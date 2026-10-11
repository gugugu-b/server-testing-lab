# 监控与排障笔记

测的时候盯什么、采什么、出问题怎么顺。这篇主要是把我自己那套流程固定下来。

> IP 和口令已脱敏，用的时候填回去。

---

## 1. 我的三层看法

每次排查我基本都按这三层走，缺一层就容易下错结论：

| 层 | 看什么 | 特点 |
| --- | --- | --- |
| **带外 BMC** | 温度、电压、风扇、电源、SEL 硬件事件 | OS 挂了还在，硬件问题最准 |
| **带内 OS** | CPU、内存、盘、网络负载 | 要 OS 活着 |
| **压测脚本** | 进程还在不在、指标有没有超阈值 | 提前发现工具假死 |

**OS 日志说没问题，不等于硬件没问题**——SEL 里有记录但带内干干净净的情况我遇到过。

---

## 2. 采日志

### 一次采全

测之前和测之后各来一遍，这是我的标配：

```bash
cat /var/log/messages > messages.log
dmesg > dmesg.log
cat /var/log/mcelog > mcelog
ipmitool sel elist > sel.log
ipmitool fru > fru.log
ipmitool sdr list > sdr.log
```

硬件信息：

```bash
dmidecode -t memory >> Dmidecode_内存型号_CPU数_DIMM数_配置.txt
dmidecode -t 1        # BIOS
dmidecode -t system   # 型号 / SN
lspci > pcie_device.txt
```

### dmesg 几个开关

```bash
dmesg -H        # 可读时间戳
dmesg -T        # 简短时间
dmesg --follow  # 跟着刷
dmesg | tail -10
```

### 内存划分

`dmesg` 启动时这段挺重要，实际可用内存对不上标称的话先看它：

```bash
dmesg | grep -iE 'gran_size|chunk_size|num_reg|lose cover'
```

`lose cover RAM` 非 0 = 有内存没利用上，报告里要写。

---

## 3. 带外

```bash
ipmitool -I lanplus -H <BMC_IP> -U <user> sel list
ipmitool -I lanplus -H <BMC_IP> -U <user> sel elist
ipmitool -I lanplus -H <BMC_IP> -U <user> sensor list
ipmitool -I lanplus -H <BMC_IP> -U <user> sdr list
ipmitool -I lanplus -H <BMC_IP> -U <user> fru
ipmitool chassis status
ipmitool lan print
```

⚠️ 别把口令写在命令行 `-P` 里，会进 shell history 和进程列表。用交互输入或者环境变量。

盯的项：

| 类别 | 看什么 |
| --- | --- |
| 温度 | CPU、DIMM、硬盘、进风口、出风口 |
| 电压 | 各供电轨，波动超规格就是电源或主板的事 |
| 风扇 | 转速和状态；冗余设计下单个坏了会自动切 |
| 电源 | PSU 状态和功率 |
| SEL | 过热、掉电、硬件故障、固件更新 |

DIMM 温度、PCH 温度有些平台要专门驱动才读得到。

---

## 4. 带内

### 盘

```bash
iostat -x 1           # util / await / r/s / w/s
iostat -d -x 1 2      # 只看盘，排除 dm- 设备
```

压测期间我会起一个后台持续采：

```bash
iostat -x 1 > disk_monitoring/iostat_results.log 2>/dev/null &
```

**只开一个**，开多个会互相干扰还白吃 CPU。

### CPU

```bash
mpstat -P ALL 1 1
top -bn1 | grep "Cpu(s)"
uptime
ps -p <PID> -o etime
```

判断有没有真跑满我看的是 `mpstat -P ALL` 的 **idle**，不是总利用率——有个别核饱和其余空着的情况，只看总数会被骗。

### 内存

```bash
free -h
free -m
vmstat 1
```

### 网络

```bash
sar -n EDEV 1 1
ip -o link show
```

双机对联我用 network namespace 隔开，免得互相干扰：

```bash
ip netns add t1 && ip netns add t2
ip link set ens32f0np0 netns t1
ip link set ens32f1np1 netns t2
ip netns exec t1 ip addr add 1.1.1.1/24 dev ens32f0np0
ip netns exec t2 ip addr add 1.1.1.2/24 dev ens32f1np1
ip netns exec t1 iperf -s -p 45000 &
ip netns exec t2 iperf -c 1.1.1.1 -p 45000 -P 10 -i 1 -t 43200
```

单进程对联多网卡（12 小时）：

```bash
iperf3 -s -B <对端IP> -p 5201 &
iperf3 -c <对端IP> -B <本机IP> -p 5201 -t 12h -b 0
```

### nmon

出图好用，采完用 Excel 分析：

```bash
./nmon_x86_64_rhel7 -f -s 1 -c 600 &   # 每秒一次，600 个点
```

---

## 5. 硬盘健康

```bash
smartctl -a /dev/nvme0n1
smartctl -i /dev/sda
smartctl -a -d megaraid,<DID> /dev/<disk>
```

阵列卡后面挂的盘要一条条捞：

```bash
VD=$(storcli /call/vall show | grep RAID | awk '{print $1}' | awk -F '/' '{print $2}')
for vd in $VD; do
    disk=$(storcli /call/v"${vd}" show all | grep "OS Drive Name" | awk -F '/' '{print $3}')
    for did in $(storcli /call/v"${vd}" show all | grep -iE "HDD|SSD" | awk '{print $2}'); do
        smartctl -a -d megaraid,${did} /dev/"$disk"
    done
done
```

NVMe 原生：

```bash
nvme smart-log /dev/nvme0n1 | grep "^temperature"
```

### 数据完整性

热插拔这类测试一定要有 md5 基线，不然拔完盘说不好数据有没有坏：

```bash
fallocate -l 100G /mnt/<dev>p1/<dev>p1.txt
md5sum /mnt/<dev>p1/<dev>p1.txt > /mnt/<dev>p1/<dev>p1.md5
# 测完重算比对
```

### 链路

```bash
lspci -s <bus> -vvv | grep -iE "LnkCap|LnkSta"
```

`LnkCap` 和 `LnkSta` 不一致就是降速了。

---

## 6. MachineCheck / mcelog

mcelog 管的是 **CPU 和内存的硬件错误**。

关键区别在于：

- **未纠正错误**是关键异常，CPU 恢复不了会引发内核错误、应用重启
- **热重启** → mcelog 能抓到，重启完还能看
- **硬重启** → 数据丢了，mcelog 可能根本来不及写

所以偶发重启类的 case，mcelog 是第一现场。

相关文件：

| 路径 | 说明 |
| --- | --- |
| `/dev/mcelog` | 设备文件 |
| `/var/log/mcelog` | 日志 |
| `/etc/mcelog/mcelog.conf` | 配置 |
| `/var/run/mcelog.pid` | PID |

**默认不写系统日志**，要在系统日志里体现得改 `ExecStart`。

```bash
mcelog --daemon &
mcelog --client                  # 看守护进程有没有检测到错误
tail /var/log/mcelog
mcelog --ascii <file.log          # 解析二进制日志
mcelog --version
```

---

## 7. 压测巡检阈值

`stress.sh` 里我自己设的：

| 指标 | 阈值 | 采集 |
| --- | --- | --- |
| 网络利用率 | > 80% | `sar -n EDEV` |
| 磁盘利用率 | > 100% | `iostat -d -x` |
| CPU idle | < 0%（满载异常） | `mpstat -P ALL` |
| 内存使用率 | > 90% | `free -m` |

```bash
./stress.sh start / ./stress.sh -t 168 start
./stress.sh status
./stress.sh stop
```

---

## 8. 怎么查

### 先排噪声

这是最关键的一步。不排的话几百个 log 刷出来全是 systemd、firewalld 的正常信息，真问题埋在里面：

```bash
cat Log_*/*log | grep -iE "error|Fail|warn" | grep -vE \
"BAR 13: failed| Fail to get numa node|firewalld|kdump|error during parsing file|Failed to listen on SSSD|ERST|RAS|sssd-nss.socket|rdisc.service|mst_pci: module verification failed|only decoding architectural errors|systemd|augenrules|WARNING: CPU|warning on user-space split_locks|45 urandom warning"
```

这个排除列表是我踩出来的，**换新平台基本要重新踩一遍**。

### 顺序

```
1. SEL 有没有新记录？
   ├─ 有硬件事件 → 直接看硬件（温度/电压/电源/风扇）
   └─ 没有 → 往下

2. mcelog / MachineCheck 有没有？
   ├─ 有 → CPU/内存硬件错误
   └─ 没有 → 往下

3. dmesg 有没有内核报错？
   ├─ 有 → 驱动 / 内核层
   └─ 没有 → 往下

4. 是不是 SELinux 拦的？
   └─ ausearch -m AVC,USER_AVC -ts recent
      （别直接关 SELinux 验证，临时 setenforce 0 可以，验完必须恢复）

5. 应用层日志
```

### 对照表

| 现象 | 先看 |
| --- | --- |
| 带内正常带外异常 | BMC 固件、SEL |
| 带外正常带内异常 | 驱动、内核版本 |
| 性能低且抖 | NUMA、PCIe 降速、没预置稳 |
| 多盘数据互扰 | 绑核 |
| 偶发重启 | mcelog、SEL 掉电、温度 |
| 网卡速率不对 | `LnkCap` vs `LnkSta`、Retimer、线缆 |
| 内存比标称少 | `lose cover RAM` |

---

## 9. 几个习惯

1. **测前记 SEL 条目数**，不然分不清新旧。
2. **长稳必须采全量日志**，偶发问题只体现在日志里，指标上看不出来。
3. **`iostat` 只开一个**。
4. **监控本身会影响性能**，性能测试时把采集进程和压测进程绑核隔开。
5. **日志目录 chmod 755**，权限不够会归档失败。
6. **带内带外对时间**，两边时钟可能差不少，不同步的话日志对不上。

---

## 相关

- [我做过的 SIT 测试清单](./sit-test-inventory.md)
- [服务器配置笔记](./server-configuration.md)
- [部署笔记](./deployment-guide.md)
- [安全加固笔记](./security-best-practices.md)