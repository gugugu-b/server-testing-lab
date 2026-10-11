# 服务器配置笔记

测性能之前，配置得先对一遍。这篇记的是我一般怎么设、怎么查、以及踩过哪些坑。

> 里面的 IP 和口令已经脱敏成 `【内网:xxx】` `【已脱敏:xxx】`，用的时候自己填回去。

---

## 1. 新机器到手先做什么

上手顺序是这样：

1. 贴资产标签，核对机型 / SN / PN
2. **BMC 网线单独接管理口**，别和业务口混
3. 查 BMC 在哪——新机一般走 DHCP

```bash
ipmitool lan print      # 已知道段落的直接查
```

或者在 BIOS → `Advanced → BMC LAN Configuration` 里看。

4. 配静态 IP。**这里最容易忘：BMC 的 IPv4 改完要手动点保存，IPv6 不用。** 我返工过好几次。

---

## 2. BMC

BMC 是独立的带外管理控制器，跟 OS 没关系——OS 挂了、系统崩了，它照样能上下电、刷固件、做控制台重定向。所以我的习惯是**所有检查都走一遍带外**，不然 OS 一挂就两眼一黑。

常用几条：

```bash
ipmitool -I lanplus -H <BMC_IP> -U <user> sel list     # 硬件事件日志
ipmitool -I lanplus -H <BMC_IP> -U <user> sel elist    # 解析过的，好读
ipmitool -I lanplus -H <BMC_IP> -U <user> sensor list  # 温度电压风扇
ipmitool -I lanplus -H <BMC_IP> -U <user> fru          # FRU 部件信息
ipmitool chassis status
ipmitool lan print
```

用 `-I lanplus`（v2.0）而不是 `lan`（v1.5 明文）。

**测之前我会先记一下 SEL 的条目数**，不然测完没法分辨哪些是新增的问题。

BIOS 里 BMC 相关菜单：

| 菜单 | 记一笔 |
| --- | --- |
| BMC Support (P95) | 保持开 |
| BMC LAN Configuration (P97) | 静态 IP；网络模式选专用还是共享 |
| BMC User (P98) | 账号增删改，按资产编号命名好追溯 |
| SOL 支持 (P99) | 默认不开，要远程调才开 |
| 监控项 (P114–P127) | 温度、电压、风扇、电源的阈值和告警 |

---

## 3. BIOS

菜单结构大致是：

```
Main → Advanced → Platform Configuration → Socket Configuration
     → Server Management → Security → Boot → Save & Exit
```

Advanced 里面我会逐项过一遍：

| 配置项 | 我怎么用 |
| --- | --- |
| Trusted Computing | 安全基线要求开，TPM 相关 |
| Power Management Configuration | 电源测试要固定策略，不然测出来不可比 |
| AST2500 Super IO Configuration | 串口波特率要跟终端对上 |
| S5 RTC Wake Settings | RTC 唤醒测试用 |
| Serial Port Console Redirection | 跟 SOL 配合，注意别一直开着 |
| Option ROM Dispatch Policy | 一般 UEFI ROMs First |
| PCI Subsystem Settings | 性能或兼容性调优时才动 |
| Network Stack Configuration | PXE 装机必须开 |
| NVMe Configuration | 存储测试前核对 |
| TLS Auth Configuration | 安全基线 |
| VLAN Configuration | 跟管理网规划对齐 |

**DIMM 温度、PCH 温度这类传感器**，有些平台要装专用驱动才读得到（我这边是 Intel PTU，Fedora 系还得先编驱动装上）。读不到别以为是硬件问题。

---

## 4. 电源策略

电源测试分三类：Reboot、DC On/Off、AC On/Off。

跑带外压力或者用 AC Server 控机之前，一定要：

```bash
ipmitool chassis policy always-on
```

不设的话 BMC 会跟着 OS 一起休眠，带外就断了。我有过一次关机后带外不可达，回来看是没设这个，当成 bug 查了半天——实际是策略问题。

还有个顺序问题：**电源切换要先切主备，再断电**。反过来操作会得到"关机状态无法复现"这种结果，同样不是被测件的锅。

---

## 5. NUMA 和内存

### 5.1 NUMA

NUMA 不是硬件，是内存访问技术，所以得从 CPU 那边看：

```bash
lscpu | grep -i numa    # 0 或 1 个节点 = 没开，≥2 = 开了
```

性能测试是按招标参数的 BIOS 出厂设置来的，**NUMA 是最容易漏检的一项**。要关掉的话：

```bash
grubby --update-kernel=ALL --args="numa=off"
reboot
dmesg | grep -i numa && lscpu | grep -i numa    # 复验
```

### 5.2 内存模组

| 类型 | 特点 |
| --- | --- |
| UDIMM | 无缓冲，同频率延迟最小，容量频率都受限 |
| RDIMM | 带寄存器，缩短并行距离，容量频率更容易拉高 |
| LRDIMM | 低负载缓冲，功耗约 RDIMM 的一半 |

### 5.3 看懂型号

`1RX4`：

- `1R` = 单排
- `X4` = 每颗芯片 4 条数据线
- `1B = 8b`

### 5.4 容量对不上就查这个

```bash
free -h
dmesg | grep -iE 'gran_size|chunk_size|lose cover'
```

`lose cover RAM` 非 0 就说明有内存没被利用上，这个得在报告里写清楚。

部件引入测试要记插法（POR 那个维度）：

```bash
dmidecode -t memory >> Dmidecode_内存型号_CPU数_DIMM数_配置.txt
```

---

## 6. 存储和 NVMe

NVMe 最多 64K 个队列，每核可以占一个，所以并发度很高。测的时候重点看不同 QD 下的 IOPS / 延迟曲线。

### 6.1 命名空间

要测性能的话，容量和格式得先统一：

```bash
nvme id-ctrl /dev/nvme0n1 | grep -iE 'cntlid|tnvmcap'
nvme format /dev/nvme0n1 -l 1
```

改容量的话顺序不能乱：

```bash
nvme detach-ns /dev/nvme0n1 -n 1 -c $cntlid
nvme ns-rescan /dev/nvme0n1
nvme delete-ns /dev/nvme0n1 -n 1
nvme create-ns /dev/nvme0n1 -s $SIZE -c $SIZE -f 0 -d 0 -m 0
nvme attach-ns /dev/nvme0n1 -n 1 -c $cntlid
nvme ns-rescan /dev/nvme0n1
```

### 6.2 盘名会漂

Linux 设备路径重启后**不保证一致**——SCSI 子系统是异步扫设备的，重启完盘名可能就换了。靠脚本绑盘名的话会绑错，得用 udev 按序列号/WWN 做持久化。

### 6.3 链路

```bash
lspci | grep -i NON-V | cut -b 1-7                  # 列出所有 NVMe 控制器
lspci -s <bus> -vvv | grep -iE "LnkCap|LnkSta"
```

`LnkCap`（能力）和 `LnkSta`（当前）**对不上就是降速了**。多盘场景掉盘、掉速优先查这里，背板、Retimer、线缆、插槽都可能是原因。

### 6.4 中断聚合

也叫中断合并，**拿延迟换 CPU 开销**——减少控制器的中断频率，从而降低主机服务开销，多盘随机读会受益。各家算法不一样，厂商自己定。

### 6.5 背板和阵列卡

```
硬盘背板 ──SAS/SCSI──> 阵列卡 ──PCIe──> 主板/CPU/内存
```

- SATA 是 SAS 的子标准，物理层兼容，SAS 多一组 pin
- 高速信号衰减一般用 **Retimer** 解决，比换高速 PCB 主板便宜得多
- 阵列卡驱动要和 FW 版本配套

---

## 7. 测之前的自查

我一般按这个顺序过：

- [ ] BMC IP 是静态的，带外能通
- [ ] BMC IPv4 **保存**了
- [ ] BIOS / BMC / CPLD 三个版本都记下来了
- [ ] NUMA 设置符合要求
- [ ] 内存插法对过，`lose cover RAM` 是 0
- [ ] NVMe 容量格式统一了
- [ ] PCIe 链路速率宽度对得上
- [ ] `ipmitool chassis policy always-on` 设了
- [ ] SEL 基线条目数记了
- [ ] **盘预置稳做完了**（这条漏了数据就是废的）

---

## 8. 对不上的时候先看哪

| 现象 | 我先查 |
| --- | --- |
| BMC IP 改了没生效 | IPv4 是不是忘了保存 |
| 重启完盘名变了 | 设备名漂移 |
| 性能偏低还抖 | NUMA 没配 / PCIe 降速 / 没预置稳 |
| 关机后带外不通 | `chassis policy always-on` |
| 内存比标称少 | `lose cover RAM` 和插法 |
| 盘偶发掉 | 背板线缆、Retimer、链路、阵列卡驱动 |

---

## 相关

- [我做过的 SIT 测试清单](./sit-test-inventory.md)
- [自动化部署流程](./deployment-guide.md)
- [监控与维护](./monitoring-maintenance.md)
- [安全加固措施](./security-best-practices.md)