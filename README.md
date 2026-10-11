# server-testing-lab

我做服务器 SIT 测试时攒下来的一些笔记。

不是什么规范文档，就是把每次测试前要查的东西、踩过的坑、常用的命令整理了一下，省得下次再翻一遍散落在各个目录里的笔记。

---

## 先看这个

**[我做过的 SIT 测试清单](./docs/sit-test-inventory.md)** —— 存储、内存、显卡、电源、整机稳定性、MOC，各类测试项和日志位置都在这里。想知道自己做过什么、日志在哪，从这篇开始。

---

## 四篇笔记

| 文档 | 里面记的是什么 | 什么时候翻 |
| --- | --- | --- |
| [服务器配置指南](./docs/server-configuration.md) | 测之前 BIOS 和 BMC 要怎么设，NUMA、内存拓扑、NVMe namespace 怎么处理 | 测试开始前对配置 |
| [自动化部署流程](./docs/deployment-guide.md) | 批量装机、工具怎么装、压测套件参数、固件怎么升降级 | 准备环境 |
| [监控与维护](./docs/monitoring-maintenance.md) | 日志怎么采、指标怎么看、出问题怎么定位 | 测试过程中和排查 |
| [安全加固措施](./docs/security-best-practices.md) | 带外管理安全、口令怎么管、明文凭据怎么处理 | 交机器前 |

另外 [服务器测试学习计划](./docs/learning_plan.md) 是我刚入职时的学习路线。

---

## 常用命令

放在这里是因为每次都要翻，常忘。

```bash
# —— 采日志，测前测后各来一遍 ——
cat /var/log/messages > messages.log
dmesg > dmesg.log
cat /var/log/mcelog > mcelog
ipmitool sel elist > sel.log      # 硬件事件，跟带内无关
ipmitool fru > fru.log
ipmitool sdr list > sdr.log

# —— 带外 ——
ipmitool -I lanplus -H <BMC_IP> -U <user> sel list
ipmitool -I lanplus -H <BMC_IP> -U <user> sensor list
ipmitool chassis power status
ipmitool chassis policy always-on     # 长测必设，不设带外会跟着 OS 睡

# —— 硬件健康 ——
smartctl -a /dev/nvme0n1
nvme smart-log /dev/nvme0n1 | grep "^temperature"
lspci -s <bus> -vvv | grep -iE "LnkCap|LnkSta"    # 两边对不上就是降速了

# —— 资源 ——
iostat -x 1          # 磁盘
mpstat -P ALL 1 1    # CPU，要看逐核 idle，不能只看总利用率
free -h
```

---

## 几个反复踩的坑

写在这儿主要是提醒我自己。

1. **SSD 测性能前必须预置稳**。刚格式化完测的数不能看，1M 顺序写循环 6 次，或者 4K 随机写 8 小时。
2. **多盘测试必须绑核**。`taskset` 给每块盘独立物理核，超线程兄弟核别拆开分给不同盘。
3. **NUMA 没按要求设**，性能数据就是废的。`lscpu | grep -i numa` 先确认。
4. **电源切换先切主备再断电**，顺序反了会误判成 bug。
5. **查日志一定要先 `grep -vE` 排噪声**，不然 systemd / firewalld 的正常信息把真问题埋了。
6. **BMC 的 IPv4 改完要手动点保存**，IPv6 不用。最常返工的一条。

---

## 约定

- 所有内网 IP、口令、域名、内部链接**已做脱敏**，统一替换成 `【内网:xxx】` `【已脱敏:xxx】` 占位符。要用真实值时自己填回去。
- 占位符用全角 `【】`，因为半角 `<>` 在 bash 里会被当成重定向符，用了脚本直接语法报错。
- 文档里的命令都是能直接复制跑的，但**换机型要重新核对一遍**——绑核、核数、PDU 位置这些跟具体配置走。