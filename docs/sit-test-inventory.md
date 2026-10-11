# 我做过的 SIT 测试清单

这篇整理我入职以来实际跑过的 SIT（System Integration Test，系统集成测试）用例，按部件分类。

写这个主要是两个用途：一是我自己查"某个测试的日志到底在哪、上次怎么跑的"；二是后面带新人时，能直接指着某一项说"这个我做过，日志在哪儿"。

> **说明**：清单按工作区里实际留存的测试目录和日志整理，标 ✅ 的是日志完整、能直接翻的；标 📁 的是有目录但日志不全或已归档的。

---

## 1. 存储 / NVMe SSD

我做得最多的一块。NVMe 协议最多支持 64K 个队列，每个 CPU 核心可以独占一个队列，所以并发度比传统 SATA 高很多——测的时候主要看队列深度变化时的 IOPS 和延迟曲线。

### 1.1 性能类

| 测试项 | 单/多盘 | 状态 | 说明 |
| --- | --- | --- | --- |
| NVMe SSD Random IO Performance Test | 单盘 | ✅ | 4K/16K/128K/1024K 包大小，1/4 线程 × QD1/QD128 |
| NVMe SSD Random IO Performance Test | 多盘 | ✅ | nvme0n1 + nvme1n1 同时压，看互不干扰 |
| NVMe SSD Sequential IO Performance Test | 多盘 | ✅ | 1M 包长顺序读/写 |
| Same LBA Read After Write Latency | 单盘 | 📁 | 同一 LBA 写后读，验证缓存一致性 |
| Driver Assisted Striping | 单盘 | 📁 | 驱动辅助条带化下的性能表现 |

### 1.2 稳定性类

| 测试项 | 状态 | 说明 |
| --- | --- | --- |
| Single NVME SSD Random IO Stability Test | 📁 | 单盘随机 IO 长稳 |
| Single NVME SSD Sequential IO Stability Test | 📁 | 单盘顺序 IO 长稳 |
| Multiple NVME SSD Random IO Stability Test | ✅ | 双盘随机 IO 长稳，日志在 `log整理\Multiple NVME SSD Random IO Stability Test\` |
| Multiple NVME SSD Sequential IO Stability Test | ✅ | 双盘顺序 IO 长稳 |

### 1.3 可靠性类

| 测试项 | 状态 | 说明 |
| --- | --- | --- |
| Hot Plug and Remove With IO Test | 📁 | 带 IO 状态热插拔，循环 100 次拔插 |
| unaligned IO Data integrity check with mixed cmd | 📁 | 非对齐 IO + 混合命令的数据完整性校验 |
| 固件升级 / 降级验证 | ✅ | PM893，升级前后各采一次 SMART 对比 |

### 1.4 3.84T 大盘专项

`测试\3.84T\` 下是两盘（nvme0n1 / nvme1n1）的 fio 原始日志，每个盘 4 个文件：

```
nvme0n1_random_read_data.log              # 随机读
nvme0n1_random_write_data.log             # 随机写
nvme0n1_one_thread_seq_wr_precondition.log   # 单线程顺序写预置稳
nvme0n1_one_thread_random_wr_precondition.log # 单线程随机写预置稳
```

**预置稳这一步不能省**。SSD 刚格式化完测出来的数是不准的，要先用大块顺序写或 4K 随机写把它磨到稳态，数据才有可比性。我一般用 1M 顺序写循环 6 次，或者 4K 随机写跑 8 小时。

### 1.5 fio 参数（我自己常用的那套）

```bash
# 顺序读，1/4 线程 × QD1/128
fio --filename=/dev/nvme0n1 --ioengine=libaio --direct=1 --thread=1 \
    --numjobs=${jobs} --iodepth=${qd} --rw=read --bs=1024k \
    --runtime=10m --time_based=1 --size=100% --group_reporting \
    --log_avg_msec=1000 --write_iops_log=seqR_nvme0n1_${jobs}job_${qd}depth_1M.log

# 随机读，4K
fio --filename=/dev/nvme0n1 --ioengine=libaio --direct=1 --thread=1 \
    --numjobs=${jobs} --iodepth=${qd} --rw=randread --bs=4k \
    --runtime=10m --time_based=1 --size=100% --group_reporting \
    --log_avg_msec=1000 --write_iops_log=randR_nvme0n1_${jobs}job_${qd}depth_4k.log
```

几个点：
- `--log_avg_msec=1000` 是按秒记录 IOPS，日后画曲线用
- 多盘测**必须绑核**，我用 `taskset` 给每块盘分配独立物理核。单盘单线程给 1 个核，单盘 4 线程给 4 个
- 超线程的兄弟核别分给不同的盘，不然两份数据互相干扰，结果没法比
- 别忘了开 NPTL、`--group_reporting`

---

## 2. 内存

HY 和 SA 两个平台各跑了一套完整用例，六个测试项：

| 测试项 | HY | SA | 说明 |
| --- | --- | --- | --- |
| Memory Basic Information Check | ✅ | ✅ | dmidecode 读型号、容量、位宽 |
| Memory Frequency Test | ✅ | ✅ | 实际频率 vs 标称频率 |
| Memory POR Test | ✅ | ✅ | Power Of Reset，测内存插法（走线/打线） |
| Memory RAS Mode Test | ✅ | ✅ |  RAS 模式配置 |
| Memory Type Test | ✅ | ✅ | UDIMM / RDIMM / LRDIMM 识别是否正确 |
| Test Platform Information Record | 📁 | ✅ | 平台信息记录，SA 这套完整 |

### 2.1 几个容易踩的点

**POR（内存插法）测试**是最容易漏的。插错线位的话机器可能照样能点亮，但内存带宽会莫名其妙偏低。我做的时候会把每个槽位的插法都跑一遍，同时记 `dmidecode -t memory` 的输出：

```bash
dmidecode -t memory >> Dmidecode_内存型号_CPU数_DIMM数_配置.txt
```

**内存频率对不上**的时候，别急着判失败。BIOS 里有个 NUMA 设置会影响，先确认 NUMA 是开是关：

```bash
lscpu | grep -i numa     # 节点数 0/1 = 没开，≥2 = 开了
```

性能测试要求按招标参数的 BIOS 出厂设置来，NUMA 这一项必须核对。

**`lose cover RAM`** 这个参数要看一下。内核启动时会打印内存划分结果，这个值非 0 说明有内存没被利用上：

```bash
dmesg | grep -iE 'gran_size|chunk_size|num_reg|lose cover'
```

---

## 3. 显卡 / GPU

测过四个型号：**5880ada**、**A400**、**L2**、**L20**。

### 3.1 共同测试项

| 测试项 | 5880ada | A400 | L2 | L20 |
| --- | --- | --- | --- | --- |
| CUDA Bidirectional Bandwidth Test | ✅ | — | ✅ | ✅ |
| CUDA Unidirectional Bandwidth Test | — | — | ✅ | ✅ |
| CUDA P2P Bandwidth & Latency Test | ✅ | ✅ | ✅ | ✅ |
| GPU Information Check Under BMC | ✅ | — | — | ✅ |
| NVIDIA GPU Information Check Under Linux | ✅ | — | — | ✅ |
| GPU Full Loading Test under Linux | ✅ | ✅ | — | — |
| GPU DC On/Off Test | ✅ | — | — | ✅ |
| GPU Reboot Test | ✅ | — | — | — |
| Test Platform Information Record | — | — | ✅ | ✅ |
| Driver Install / Re-Install | — | ✅ | — | ✅ |

### 3.2 L20 上的专项（最全的一套）

L20 那一套最全，除了上面的基本项，还有：

- **NVIDIA GPU Performance Tuning**（调频，`nvidia-smi -pm 1` 开持久模式、`-ac` 锁频）
- **NCCL Tests**（多卡通信，`mpirun -np 16 -H ... -x NCCL_ALGO=Ring -x NCCL_NCHANNELS_PER_NET_PEER=1 ...`）
- **PCI Express Bandwidth Test / Data Eye Metric / Qualification Test**（PCIe 信号质量）
- **Thermal Qualification Test**（散热）
- **Input EDPp Test**（带外对带内，模拟带外读取 GPU 状态）
- **NVQual environment preparation**（NVQual 环境）
- **Mechanical Test**（物理装配）

### 3.3 A400 上的硬件专项

- **Hot Plug Detect (HPD) Test** — 需要 2 台显示器
- **Vertical Span-NVIDIA** — 需要 2 台显示器，且分辨率要一致
- **显卡风扇异常噪声检查** — 这个靠耳朵，只能人工过一遍

### 3.4 GPU 压测的坑

GPU 满载测试**必须先装好驱动和对应版本的 CUDA**，否则 gpu_burn 根本跑不起来。另外测 GPU 性能和散热是绑定的——跑满载时重点盯温度，而不是只盯性能数字。

L2 和 5880ada 的多卡日志我分成了 `dtoh`（device to host）和 `htod`（host to device）两个方向，测双向带宽时要分开跑。

---

## 4. 电源

### 4.1 供应商 / 功率

| 供应商 | 功率段 |
| --- | --- |
| 欧路通 | — |
| 长城 | 1300W / 2000W / 2700W |

### 4.2 测试项

| 测试项 | 说明 | 状态 |
| --- | --- | --- |
| Power Supply Module Hot Plug under Active-Active Stress Mode | 冗余双活模式下电源模块热插拔 | 📁 长城 1300W ✅ |
| Power Supply Module Hot Plug under Active-Standby Stress Mode | 主备模式下热插拔 | 📁 长城 1300W ✅ |
| Power Supply Module Hot Plug under Active-Standby Idle Mode | 主备 + 空载下热插拔 | 📁 长城 1300W / 2000W ✅ |
| AC/DC Power Cord Hot Plug under Active-Standby Idle Mode | 电源线热插拔 | 📁 长城 1300W |
| PSU 传感器阈值信息 | 记录阈值，用于告警验证 | 📁 长城 1300W |
| Always on Function Check | 常开功能 | 📁 长城 2700W |
| Last State Function Check | 上次下电状态记忆 | 📁 长城 2700W |
| Mode Setting Test | 电源模式设置 | 📁 长城 2700W |
| Full Loading Test | 满载 | 📁 长城 2700W |
| BMC Power Consumption Measurement Accuracy Test | BMC 功耗测量精度 | ✅ |
| AC Log / Bug 验证 | AC 上下电日志与问题单验证 | ✅ |

### 4.3 电源测试的坑

**电源切换必须先切主备，再断电**。顺序反了会出现"关机状态下无法复现"——这不是被测件的 bug，是操作顺序问题。我现在固定先执行：

```bash
ipmitool chassis policy always-on
```

强制关机前也要先切主备。这个坑我踩过一次，记一下。

**PDU 插座位置一定要二次确认**。脚本参数 `-m`（PDU 名）和 `-p`（插座位置）填错会误断别人的机器：

```bash
./powerCycle -i ac -s <AC_SERVER> -m <PDU名> -p '<插座位置>' -a <server编号> -l <循环数> -f NON-STOP
```

---

## 5. 整机稳定性（G40 / G50）

G40 和 G50 两个平台各跑了四类稳定性测试：

| 测试类型 | G40 | G50 | 说明 |
| --- | --- | --- | --- |
| AC powercycle 稳定性测试 | ✅ | ✅ | 交流上下电循环 |
| DC cycle 稳定性测试 | ✅ | ✅ | 直流上下电循环 |
| Warm Reboot 稳定性测试 | ✅ | ✅ | 热重启循环 |
| Stress 稳定性测试 | ✅ | ✅ | 满载压力 |

日志量很大——`Warm Reboot稳定性测试\log\` 下有 90 个 `Log_N` 目录，DC cycle 下有 180+ 个 `BlackList.log`。每个 log 目录里存的是当轮的 `messages.log`。收数据的时候建议先写脚本批量抽取，我一般用：

```bash
# 噪声过滤后的有效错误
cat Log_*/*log | grep -iE "error|Fail|warn" | grep -vE \
"BAR 13: failed| Fail to get numa node|firewalld|kdump|error during parsing file|Failed to listen on SSSD|ERST|RAS|sssd-nss.socket|rdisc.service|mst_pci: module verification failed|only decoding architectural errors|systemd|augenrules|WARNING: CPU|warning on user-space split_locks|45 urandom warning"
```

这个 `grep -vE` 的排除列表是踩出来的。不加排除的话几百个 log 里全是 systemd、firewalld 的正常噪音，真正的问题反而看不见了。**换新平台这个列表大概率要重新踩一遍。**

---

## 6. MOC（神龙 MoC 卡）

MoC = Microserver on Chip，阿里神龙的自研卡。

| 测试项 | 状态 |
| --- | --- |
| DC On_Off Test-Linux | ✅ |
| Reboot Test | ✅（100+ 轮 log） |
| OutFANx_Present（风扇在位检测） | ✅ |

风扇在位检测我是这样读的：

```bash
ipmitool -I lanplus -H <BMC_IP> -U <user> -P <pass> sensor get "FAN1_PWM"
```

---

## 7. 变更测试 / 专项

| 测试 | 说明 |
| --- | --- |
| 沐川-Alibaba XW2.0 2U Retimer-Fan BP 变更测试方案 | 背板 + Retimer + 风扇联动变更验证 |
| 神龙离线装机测试指导 | MoC 卡场景的离线装机流程 |
| PCIe 版本对比 | `测试\pcie版本.png` |
| GPU 调优手段 | `测试\gpu调优手段.png` |
| MachineCheck | 硬件错误采集 |

---

## 8. 测试之外

`测试\210020102040-【已脱敏-姓名】-基于YOLO算法的PCB板缺陷检测系统设计与开发.docx` —— 这是我自己做的毕业设计，跟 SIT 工作无关，放在一起是因为当时在同一个工作区。

---

## 相关文档

| 文档 | 内容 |
| --- | --- |
| [服务器配置指南](./server-configuration.md) | 测试前的配置基线：BIOS / BMC / NUMA / 内存 / NVMe |
| [自动化部署流程](./deployment-guide.md) | 装机、工具链、压测套件、固件升降级 |
| [监控与维护](./monitoring-maintenance.md) | 采日志、判指标、查问题 |
| [安全加固措施](./security-best-practices.md) | 带外安全、凭据管理 |