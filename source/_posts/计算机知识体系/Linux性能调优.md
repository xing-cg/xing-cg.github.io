---
title: Linux性能调优
categories:
  - - 计算机知识体系
tags:
date: 2026/1/22
update: 2026/1/22
comments:
published:
---
怎么看目前机器是否开启了隔核（CPU Isolation），哪些核可以绑核（CPU Pinning/Affinity）？

Linux 内核启动参数是 `cmdline`。

可以通过 `cat /proc/cmdline` 查看。

**分析结果：** 需要在这个长字符串里寻找以下关键字：
1. **`isolcpus=...`** (最核心的标志)
    - **例子**：`isolcpus=2-15`（CPU ID 是 从 0 开始的）
    - **含义**：表示核心 2 到 15 被隔离了。Linux 调度器默认不会把进程分配到这些核上。
    - **如果没有**：说明机器**目前没有开启隔核**，所有核心都由 OS 自由调度。
2. **`nohz_full=...`** (进阶低延迟标志)
    - **含义**：告诉内核在这些核心上尽量减少“时钟中断” (Tickless)。这是做极致低延迟（如高频交易）时的标配。通常和 `isolcpus` 设置的范围一致。
3. **`rcu_nocbs=...`**
    - **含义**：把 RCU 回调任务移出这些核心，进一步减少内核干扰。


```bash
cxing@forrest:~/benchspace/qy-benchmark/cache_latency$ cat /proc/cmdline

BOOT_IMAGE=/boot/vmlinuz-5.15.0-119-generic
root=UUID=aa0e4d7c-9080-49d5-945d-159abdb51b56
ro quiet splash nosmt
irqaffinity=0
isolcpus=8-15
nohz_full=8-15
rcu_nocbs=8-15
rcu_nocb_poll
vt.handoff=7
```

# IRQ Affinity
IRQ 是 Interrupt Request，中断请求。

Linux 内核、SSH 服务、系统日志等后台任务通常倾向于在 **CPU 0** 上运行。我们的参数也说明，IRQ 倾向于跑在 0 号 CPU 上。
- **原则**：永远不要把高性能任务绑定在 CPU 0 上。
- **建议**：留出 **CPU 0**（甚至 CPU 1）给操作系统处理杂务和中断。

有些网卡中断可能默认绑定在特定的几个核上。如果你的程序和网卡中断挤在一个核，性能会抖动。

观察中断分布：
```bash
# 查看各个 CPU 处理中断的累计次数
watch -n 1 "cat /proc/interrupts | grep 'LOC\|CAL\|TLB\|Rescheduling'"
```


