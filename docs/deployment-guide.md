# 部署笔记

装机、装工具、跑压测、刷固件。这篇记的是我的流程和几个不踩不知道的坑。

> IP 和口令已脱敏成 `【内网:xxx】` `【已脱敏:xxx】`，用的时候自己填。

---

## 1. 怎么装系统

测试中心的路子有好几种：

| 方式 | 什么时候用 |
| --- | --- |
| USB / SATA 光驱 | 单机、没有网络 |
| BMC 挂虚拟介质 | 机架机器没光驱 |
| USB 启动盘 | 单机刷写 |
| **PXE** | **批量装机，我基本都用这个** |

人工一台台装是效率瓶颈，也是版本不一致的根源。

### PXE 批量装

```bash
ipmitool chassis bootdev pxe                  # 下次 PXE 启动，只生效一次
ipmitool chassis bootdev disk                 # 下次从硬盘启动
ipmitool chassis bootdev pxe options=persistent   # 永久生效
```

**`bootdev` 是一次性的，必须在开机之前设好**，顺序反了就得重来。

BIOS 里 `Network Stack Configuration` 要开，不然 PXE 引导不起来。装完记得改回 `bootdev disk`，不然下次异常重启会掉进 PXE 循环。

### 装完固化

内核升级或者换 initramfs 的时候，先把原文件留着：

```bash
mv initramfs-4.19.91- /boot/initramfs-4.19.91- /boot/initramfs-4.19.91-
007.ali4000.alios7.x86_64.img.bak
```

---

## 2. 工具怎么装

统一从内部 FTP 拉，别各处散着装，版本会乱。

```bash
wget -nH http://<工具服务器>/test_tool/<包名> -O <包名>
```

### smartmontools

系统自带的版本经常不够，脚本会自己检测（要求 ≥ 7.1）然后源码编译：

```bash
smartctl -V | grep -i release

yum install -y gcc gcc-c++ gcc-g77
wget -nH http://<工具服务器>/test_tool/smartmontools-7.3.tar.gz -O smartmontools-7.3.tar.gz
tar xvf smartmontools-7.3.tar.gz
cd smartmontools-7.3/
./configure
make -j $(cat /proc/cpuinfo | grep physical | sort -n | uniq -c | head -1 | awk '{printf $1}')
make install
cp -f smartctl /sbin/smartctl
```

### storcli（阵列卡）

```bash
wget -O storcli-007.2203.0000.0000-1.noarch.rpm -nH http://<工具服务器>/test_tool/
rpm -ivh storcli-007.2203.0000.0000-1.noarch.rpm --force
ln -f /opt/MegaRAID/storcli/storcli64 /bin/storcli
```

### mcelog（硬件错误）

```bash
wget http://<工具服务器>/test_tool/mcelog-191.tar.gz
tar xvf mcelog-191.tar.gz && cd mcelog-191/ && make && make install

cp mcelog.service /usr/lib/systemd/system
sed -i '/^ExecStart/cExecStart=/usr/sbin/mcelog --daemon --foreground --syslog --logfile=/var/log/mcelog' \
    /usr/lib/systemd/system/mcelog.service
systemctl enable mcelog.service
systemctl restart mcelog.service
```

**mcelog 默认只写 `/var/log/mcelog`，不写系统日志。** 要在系统日志里也能看到就得改 `ExecStart`，不然事后排查会漏。

### 工具清单

| 工具 | 用途 |
| --- | --- |
| fio | 存储压测 |
| nmon | 长时资源采集，导 Excel 出图 |
| iostat / mpstat / sar | 实时和周期指标（sysstat） |
| smartctl | 硬盘健康 |
| storcli | 阵列卡 |
| nvme-cli | NVMe 设备 |
| hdparm | SATA/SAS 刷固件 |
| ipmitool | 带外 |
| iperf3 | 网络带宽 |
| memtester / SAT | 内存压测 |
| mcelog | 硬件错误 |

---

## 3. 压测套件

`HW_Suite_NHkit` / `HW_Stress_FullLoading` 都靠 `env.py` 控制。

```python
run_time   = 43200   # 43200s = 12h
test_mode  = "alone" # alone 或 mix

test_mprime = 1      # CPU 压力
test_memory = 90     # 内存压力占比，0 = 不跑
test_fio    = 1      # 1=通用版, zns=ZNS 盘
test_bmc    = 1      # BMC 压力
test_iperf  = ""     # "iperf" / "iperfalone" / 其他 = 不跑
test_netperf_num = 0 # 几个网口跑 netperf，0 = 不跑
```

两种模式：

- `alone`：常规满载，每个测试项都保证跑满 `run_time`
- `mix`：和别的工具混着跑（比如 PowerCycle + FullLoading），严格不超过 `run_time`

`mix` 模式要给预热和出报告留时间：`sat_wait_time`、`fio_wait_time`、`netperf_waittime`。

几个注意的：

- `check_time` 默认 60s，到点用 `ps -ef` 查后台程序是不是真起来了。**别低于 40s**，跟机型配置有关
- SAT 前面要预读内存信息，大概 `总内存(kb) / 4000000` 秒
- GPU 压测**得先装驱动和对应版本的 CUDA**，不然起不来
- `test_dma` 要换内核版本匹配的驱动（`accel.ko`、`bytedma.ko`、`ice.ko`、`igb_uio.ko`）

---

## 4. 固件升降级

这活儿验的是三件事：能不能升、能不能退、压力下会不会挂。

```bash
./fw_down_up.sh <设备名> <升级固件> <降级固件>
```

脚本按设备类型自己分流：

| 设备 | 工具 | 命令 |
| --- | --- | --- |
| SATA/SAS SSD | hdparm | `hdparm --fwdownload <fw> --yes-i-know-what-i-am-doing --please-destroy-my-drive /dev/<dev>` |
| NVMe | nvme-cli | `nvme fw-download` + `nvme commit -s 1 -a 3` |
| MLX 网卡 | flint | `flint -y -d <bus> -i <fw> b` + `mlxfwreset -y -d <bus> reset` |
| Intel 82599 网卡 | eeupdate64e | 得先解压 `Bootutil+Eeupdate_Linux_x64.tar.gz` |

流程是**升前记一次、升完记一次、退完再记一次**：

```bash
smartctl -a /dev/$INTERFACE > pre_upgrade_info.txt
smartctl -a /dev/$INTERFACE > post_upgrade_info.txt
smartctl -a /dev/$INTERFACE > post_downgrade_info.txt
```

### 压力下反复刷

验证固件在持续 IO 下的稳定性：

```bash
# p1 写满并生成 md5 基线，p2 拿来压测和刷固件
fallocate -l 100G /mnt/<dev>p1/<dev>p1.txt
md5sum /mnt/<dev>p1/<dev>p1.txt > /mnt/<dev>p1/<dev>p1.md5

# 后台持续 QD128 4K 混合读写
fio --name=fw_upgrade_downgrade --filename=/dev/<dev>p2 \
    --ioengine=libaio --direct=1 --thread=1 --numjobs=1 --iodepth=128 \
    --rw=randrw --bs=4k --rwmixread=50 --runtime=38000 --time_based=1 \
    --size=100% --norandommap=1 --randrepeat=0 --group_reporting \
    --log_avg_msec=1000 --bwavgtime=1000 --write_iops_log=fw_upgrade_downgrade &

# 循环 10 轮：降 → 记版本 → 升 → 记版本
for i in {1..10}; do
    sleep 60
    hdparm --fwdownload down* --yes-i-know-what-i-am-doing --please-destroy-my-drive /dev/$dev
    smartctl -a /dev/$dev | grep -i "Firmware Version" >> ${i}_downgrade_fw.log
    sleep 1500
    hdparm --fwdownload up* --yes-i-know-what-i-am-doing --please-destroy-my-drive /dev/$dev
    smartctl -a /dev/$dev | grep -i "Firmware Version" >> ${i}_upgrade_fw.log
    sleep 1500
done
```

⚠️ 刷固件**会毁数据**。刷之前一定确认这块盘不在压测 DISK 数组里、不是系统盘。

### 三星 PM893

按控制器分流，失败返回非 0：

```bash
ps3_cnt=$(ps3cli show ctrlcount | grep "Controller Count" | awk -F= '{print $NF}' | tr -d ' ')
ps3_type=$(ps3cli /c0 show | grep "Product Name" | awk '{print $NF}')

if [[ $ps3_cnt -gt 0 && ! $ps3_type =~ 2120 ]]; then
    sh ps3.sh        # 有控制器且不是 2120
else
    sh dctoolkit.sh
fi
```

---

## 5. 多盘多卡怎么绑核

这个不绑，结果就没法比。

```bash
# 单盘单线程：每盘一个物理核
taskset -c 4   nohup sh ./NVME_..._1_jobs.sh nvme0n1 1200 &
taskset -c 5   nohup sh ./NVME_..._1_jobs.sh nvme1n1 1200

# 单盘多线程：每盘一组物理核
taskset -c 4-7   nohup sh ./NVME_..._4_jobs.sh nvme0n1 1200 &
taskset -c 8-11  nohup sh ./NVME_..._4_jobs.sh nvme1n1 1200
```

**绑核必须按实际 core 数调，而且超线程的兄弟核别分给不同的盘。** 我第一次测双盘就是没注意这个，两份数据互相干扰，返工了一遍。

GPU 多卡同理：

```bash
declare -A cpu_affinity_map=(
    [0]="0,1"   [1]="2,3"   [2]="4,5"
    [3]="32,33" [4]="34,35" [5]="36,37"
)
```

---

## 6. 长稳

```bash
./stress.sh start        # 24h
./stress.sh -t 168 start # 168h
./stress.sh status
./stress.sh stop
```

SSH 断了不影响，用 `nohup` 或者 `screen`。依赖：`fio iperf3 sar iostat bc ip ping mpstat memtester ptat`，要 root。

日志目录：`./logs/stress_test_YYYYMMDD_HHMMSS/`，状态文件是先写 `.tmp` 再 `mv`，不会读到半截。

---

## 7. 产物目录

```
logs/<测试项>_<YYYYmmdd_HHMMSS>/
├── status/
├── cpu_monitoring/
├── memory_monitoring/
├── disk_monitoring/
└── network_monitoring/
```

收尾前对一遍：

- [ ] OS 版本、kernel 对得上，驱动都加载了（阵列卡驱动必须配 FW 版本）
- [ ] 工具装全了、版本达标（smartctl ≥ 7.1）
- [ ] 套件参数跟测试方案一致
- [ ] 盘分区格式化做完，md5 基线生成了
- [ ] 固件升降级前后的信息文件齐
- [ ] 日志目录权限 755
- [ ] **预置稳做了**

---

## 8. 出过岔子的地方

| 现象 | 多半是 |
| --- | --- |
| PXE 起不来 | BIOS Network Stack 没开，或者 BMC 跟 PXE 服务器不同网段 |
| 批量装完版本不一致 | 镜像源没统一，改 PXE 统一发 |
| 阵列卡认不到 | storcli 版本不匹配，或内核驱动没加载 |
| 固件刷不进去 | 设备类型和工具对不上，或者误刷了系统盘 |
| 压测起不来 | `check_time` 太短，或者进程被 OOM Killer 杀了 |
| 多盘数据互相干扰 | 绑核没按物理核分 |
| 重启完盘名变了 | 设备名漂移，要上持久化 |

---

## 相关

- [我做过的 SIT 测试清单](./sit-test-inventory.md)
- [服务器配置笔记](./server-configuration.md)
- [监控与维护笔记](./monitoring-maintenance.md)
- [安全加固笔记](./security-best-practices.md)