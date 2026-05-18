---
title: Linux_GRUB启动参数
categories:
  - - Linux
tags:
  - 性能调优
date: 2026/4/29
updated: 2026/4/29
comments:
published:
---
# 什么是 GRUB？

**GRUB**（GNU GRand Unified Bootloader）是 Linux 系统中最常用的**启动加载程序**。

简单来说，它是计算机启动时运行的第一个程序（在 BIOS/UEFI 之后，操作系统内核之前）。它的主要任务是：

- **引导内核**：找到硬盘上的 Linux 内核文件（vmlinuz）并将其加载到内存中。
- **传递参数**：这是最重要的功能。GRUB 允许在启动时向内核传递一串“命令行参数”（Boot Parameters），通过这些参数，可以从底层改变内核的行为（如关闭节能、隔离 CPU、禁用安全补丁等）。

Ubuntu写法：
```config
GRUB_DEFAULT=0
GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=0
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian` 这是Ubuntu的写法
# GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)" 这是CentOS7的写法
GRUB_CMDLINE_LINUX="quiet splash intel_idle.max_cstate=0 intel_iommu=off intel_pstate=disable idle=poll iommu=off mitigations=off pcie_aspm=performance pcie_port_pm=off processor.max_cstate=0 nohz=on clocksource=tsc tsc=reliable spec_store_bypass_disable=off spectre_v2=off audit=0 nmi_watchdog=0 selinux=0 nosoftlockup acpi_irq_nobalance irqaffinity=0 isolcpus=1-22 nohz_full=1-22 rcu_nocbs=1-22 rcu_nocb_poll"
```

---

# 如何构建高性能调优知识库

底层调优笔记，采用“分类-功能-原理-副作用”的结构。按以下模板记录：

- **分类**：例如“CPU 电源管理”、“中断与隔离”。
- **参数名**：GRUB 配置文件中的具体字段。
- **目的**：解决什么问题（如：降低 P99 抖动）。
- **原理简述**：底层发生了什么。
- **副作用/代价**：换取性能丢掉了什么（如：发热、安全性）。

---
# 在配置隔核参数之前要做的信息查询
## 确认CPU拓扑
```bash
lscpu -e
```
要观察，是否有超线程的逻辑核。尽量关闭超线程，让线程都对应一个物理核。防止逻辑核抢占L1/L2缓存。
## 检查NUMA拓扑
```bash
sudo yum install -y numactl
numactl -H
```
如果回测数据存在 Node 0 的内存里，但程序绑在了 Node 1 的 CPU 上，数据的 P99 延迟会因为跨 CPU 访问总线（UPI）而直接翻倍。

**建议**： 如果要隔离核心，确保回测线程绑定的核心都在**同一个 NUMA Node** 下，并且用 `numactl --membind` 强制程序只在该 Node 申请内存。


当确定了隔离核心（比如 2-15）分布在哪个 Node 上后，运行回测程序的最佳姿势是：
```bash
# 假设核心 2-7 属于 Node 0
# --cpunodebind=0: 只在 Node 0 的核上跑
# --membind=0: 只在 Node 0 的内存条上申请内存
numactl --cpunodebind=0 --membind=0 ./your_backtester
```

# 按功能分类

## GRUB 界面与系统引导
- **`GRUB_DEFAULT=0`**：默认启动菜单中第一个内核。
- **`GRUB_TIMEOUT=0`**：开机菜单停留 0 秒，直接启动。（不建议）
- **`GRUB_TIMEOUT_STYLE=hidden`**：不显示引导菜单界面。
- **`quiet splash`**：静默启动，不显示那一串滚动的内核日志（生产环境常用，调试时可去掉）。

|**参数**|**作用**|**形象理解**|
|---|---|---|
|**`GRUB_DEFAULT=0`**|指定默认启动哪一个内核。`0` 表示启动菜单里的第一个（通常是最新版本）。|**默认选项**：开机后如果不动，自动选第一个。|
|**`GRUB_TIMEOUT=0`**|开机时 GRUB 菜单停留的时间，单位为秒。|**零秒等待**：开机瞬间直接加载内核，不给人按键选择的机会。|
|**`GRUB_TIMEOUT_STYLE=hidden`**|隐藏启动菜单。|**静默模式**：屏幕上不会出现那一排排的内核列表。|
## 极致电源管理 (消除唤醒延迟)
- **`intel_idle.max_cstate=0` / `processor.max_cstate=0`**：禁止 CPU 进入任何节能睡眠状态（C-state），始终保持 C0（全速）。
- **`idle=poll`**：让空闲 CPU 执行忙等待（Polling）而不是休眠。响应延迟降至物理最低，但发热剧增。
- **`intel_pstate=disable`**：禁用 Intel 自动调频驱动。通过手动控制频率，避免 CPU 在高低频之间跳变。
- **`pcie_aspm=performance`**：禁用 PCIe 总线的活动状态电源管理。保持 PCIe 链路（如网卡、硬盘通道）永不切入低功耗模式。通俗来讲，就是告诉设备“别睡”。
- **`pcie_port_pm=off`**：关闭 PCIe 端口的电源管理。配合aspm使用，告诉主板上的PCIe端口控制器“别管电源管理”，两者结合，杜绝总线层面的抖动。
### pcie_aspm是什么
`pcie_aspm` 是针对 **PCI Express 总线** 的“节能管家”。

**ASPM** 的全称是 **Active State Power Management**（活动状态电源管理）。简单来说，它的作用是：**当 PCIe 总线（比如连接网卡、NVMe 硬盘的通道）上没有数据传输时，让这条“高速公路”进入低功耗睡眠状态。**

和 CPU C-states 逻辑非常相似，ASPM 的核心矛盾在于 **“唤醒延迟”**。
- **L0 状态**：全速运转，数据随时秒发。
- **L0s/L1 状态**：节能状态。当有新的数据（比如一条新的 Tick 行情从网卡进来，或者回测程序要从硬盘读数据）时，总线必须先从节能状态“苏醒”，恢复电压和时钟频率。

**影响：** 这种从“休眠”到“全速”的切换需要 **几个微秒到几十微秒** 的时间。在回测中，如果程序每隔一段时间才处理一组数据，PCIe 链路可能就在停顿的间隙偷偷“打了个盹”。等到下一组数据过来，第一条数据的延迟就会无故增加，从而直接拉高了 **P99**。

可选值：
- **`default`**：由 BIOS 决定怎么管。
- **`powersave`**：极力节能（低延迟的大忌）。
- **`performance`**：**禁用**大部分节能状态，强制让 PCIe 链路保持在 **L0（全速）** 状态。
- **`off`**：完全关闭内核对 ASPM 的控制。
## CPU 隔离与任务调度 (解决 P99 的核心)
- **`isolcpus=2-15`**：物理隔离核心。让内核调度器完全忽略这些核，普通程序绝不会跑上去。
- **`nohz_full=2-15`**：开启无滴答模式。当核心只有一个任务时，停止时钟中断，减少内核对计算的干扰。
- **`rcu_nocbs=2-15`**：将 RCU（只读复制更新）回调任务外包。不让隔离核处理这些清理工作，扔给系统核。
- **`rcu_nocb_poll`**：让 RCU 服务线程主动轮询，而不是被动等待唤醒，进一步减少中断。
- **`irqaffinity=0`**：强制所有硬件中断（网卡、磁盘等）默认绑定到 0 号核。
- **`acpi_irq_nobalance`**：禁用 ACPI 中断平衡。防止内核自动把中断分配到隔离核上。
### intel_pstate是什么
`intel_pstate` 驱动是 Linux 内核用来自动调整 CPU 频率的。它非常“聪明”，会根据当前的系统负载，动态地在几百兆赫兹（MHz）到几千兆赫兹（GHz）之间跳变。
对于普通用户，这个驱动很棒，因为它省电且发热小。但对于**量化回测和高频交易**，它有两个致命缺点：**引入频率抖动**和**算法黑盒**。
禁用后，内核会回退（Fallback）到传统的 **`acpi-cpufreq`** 驱动。虽然没那么聪明，但它非常**听话**。允许配合 `cpupower` 工具，将所有核心强制锁定在 **Performance（高性能）** 模式。

当禁用了 `pstate` 以后，建议在重启进入系统后，手动执行以下命令来配合：
```sh
# 确保安装了工具
yum install -y cpupowerutils

# 将所有核心设置为性能模式
cpupower -c all frequency-set -g performance
```
## 时间源与硬件控制 (提高打点计时精度)
- **`clocksource=tsc`**：强制系统使用 CPU 时间戳计数器作为时钟源。
- **`tsc=reliable`**：标记 TSC 为可靠时钟。跳过复杂的时钟校验，显著提升 `std::chrono` 等计时函数的调用效率。
- **`nohz=on`**：动态时钟开关。配合 `nohz_full` 使用。
- **`intel_iommu=off` / `iommu=off`**：关闭 I/O 内存管理单元。减少网卡/磁盘通过 DMA 访问内存时的地址转换开销。
    - **`intel_iommu=off`**：专门针对 Intel 平台的 VT-d 技术。
    - **`iommu=off`**：通用的 IOMMU 关闭开关。
### IOMMU是什么

在正常模式下，当一个外设想要通过 **DMA（直接内存访问）** 读取内存时，IOMMU 会介入：
- **地址转换**：将外设看到的“虚拟地址”转换成真实的“物理地址”。
- **内存隔离**：确保网卡只能访问分配给它的那块内存，不能乱读内核或其他程序的内存。
- **虚拟机支持**：这是实现虚拟机“硬件直通”（Passthrough）的核心技术。

### 把IOMMU设为 `off`的结果？

对于回测程序和实盘交易系统，追求的是**零延迟**。

- **消除翻译开销（Latency）**： 每次外设访问内存都要经过 IOMMU 的“翻译”。虽然这个过程很快，但它会引入额外的纳秒级延迟。在高频场景下，每一次 DMA 操作的延迟累加起来会直接推高 **P99**。
- **避免 IOTLB 未命中**： 像 CPU 有缓存一样，IOMMU 也有缓存（IOTLB）。如果回测时数据量极大，导致 IOTLB 未命中，系统就必须去内存里查表，这会产生明显的延迟尖刺（Spike）。
- **简化内存路径**： 关闭它后，外设将直接使用物理地址访问内存。路径变短了，不确定性（Jitter）也就减少了。
## 安全补丁与辅助功能 (减少内核损耗)

- **`mitigations=off`**：关闭所有针对 CPU 漏洞（如熔断、幽灵）的安全补丁。这能显著提升系统调用和上下文切换的速度。
- **`spectre_v2=off` / `spec_store_bypass_disable=off`**：具体关闭某些分支预测的安全限制，提升 C++ 执行效率。
- **`selinux=0`**：禁用 SELinux 安全子系统。减少内核在权限检查上的 CPU 消耗。
- **`audit=0`**：关闭内核审计系统。减少日志写入开销。
- **`nmi_watchdog=0`**：禁用不可屏蔽中断监控器。防止这个周期性中断干扰实时任务。
- **`nosoftlockup`**：告诉 Linux 内核：**“即便发现某个 CPU 核心长时间不响应，也不要报警，更不要尝试干预。”**

### 什么是Soft Lockup
在正常的 Linux 运行中，内核会为每个核心启动一个优先级极高的守护线程（通常叫 `watchdog/N`）。

- **工作机制**：这个守护线程会定期“喂狗”（更新一个时间戳）。
- **触发条件**：如果某个 CPU 核心被某个任务死死占住，超过一定时间（默认通常是 20 秒）没有给这个守护线程运行的机会，内核就会认为发生了“软锁定”。
- **后果**：内核会在日志（`dmesg`）里打印一大堆堆栈信息，报警说“BUG: soft lockup - CPU#N stuck for 22s!”，甚至在某些配置下直接导致系统重启。

#### Soft Lockup vs Hard Lockup
- **Soft Lockup**（软锁定）：CPU 在内核态循环，不给别的任务机会，但还能响应中断。由 `nosoftlockup` 控制。
- **Hard Lockup**（硬锁定）：CPU 彻底卡死，连中断都不响应了。由 `nmi_watchdog=0` 控制。
### 为什么要开启nosoftlockup
#### 消除“看门狗”带来的抖动 (Jitter)

为了检测软锁定，内核必须定期唤醒 `watchdog` 线程。虽然它占用资源极少，但它毕竟是一个**中断/上下文切换**。在追求纳秒级确定的环境里，任何不请自来的线程都是“刺客”。禁用它能进一步减少 CPU 被干扰的概率。
#### 防止“合法”的 CPU 霸占被误报
在配置中：
1. 开启了 `isolcpus`（隔离核心）。
2. 使用了 `SCHED_FIFO`（实时优先级）。
3. `C++`代码可能在一个 `while(true)` 循环里以 100% 的速度狂奔处理 Tick 数据。

这种情况下，程序**确实**会霸占 CPU 几十秒甚至几小时。内核的看门狗线程根本抢不到执行时间，于是它会疯狂报错。开启 `nosoftlockup` 相当于告诉内核：“别叫了，这个 CPU 是我故意占着的，一切都在掌握中。”
#### 副作用

如果代码真的因为内核态 Bug 导致死锁，系统将不再自动记录堆栈信息，可能很难发现到底是哪行内核代码锁死了 CPU。
# 应用GRUB配置
## 语法检查
`/etc/default/grub` 本质上是一个 Shell 脚本片段，最常见的错误是**引号没成对**或者**括号没闭合**。
```bash
source /etc/default/grub
```
如果没有任何输出，则语法正确。
## 逻辑检查：预览完整配置
可以运行`grub2-mkconfig -o /tmp/grub_test.cfg`，**检查重点**：在输出中找找刚加的那一串 `isolcpus=2-15...` 是否完整出现了，说明 GRUB 逻辑链路已经打通。
## 临门一脚：覆盖现有配置
根据当前的引导模式（BIOS 或 UEFI），执行最后的覆盖操作。

> - **BIOS 用户**：`grub2-mkconfig -o /boot/grub2/grub.cfg`
> - **UEFI 用户**：`grub2-mkconfig -o /boot/efi/EFI/centos/grub.cfg`

怎么看自己是 BIOS 还是 UEFI？
```bash
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS (Legacy)"
```
以上命令检查系统是否导出了 **EFI 固件接口**。如果 `/sys/firmware/efi` 这个目录**存在**，说明系统是通过 UEFI 引导的，否则是BIOS。

还有一种方法

```bash
fdisk -l /dev/sda  # 假设系统盘是 sda
```

- 看到 `Disklabel type: gpt`：通常对应 **UEFI**。
- 看到 `Disklabel type: dos` (或 MBR)：通常对应 **BIOS**。
## 重启后检查
```bash
reboot
```

```bash
cat /proc/cmdline
```
**如果看到了 `isolcpus=2-15` 等参数**：说明 GRUB 成功把指令传达给了内核。

第二看：CPU 负载 (验证隔离)

运行 `top`，然后按键盘上的 **`1`**（显示所有核心）：

- **预期现象**：发现 CPU 0 和 1 可能有一点点占用，但 **CPU 2 到 15 的占用率应该是死死地钉在 0.0%**。


# 程序优化开关
## 锁定物理内存 (mlockall)
为了防止 CentOS 7 的内核在内存压力大时把回测程序的数据置换（Swap）到磁盘，建议在 `C++` `main` 函数开头加上这一行
```cpp
#include <sys/mman.h>

int main() {
    // 锁定当前及未来申请的所有内存，防止被 swap
    if (mlockall(MCL_CURRENT | MCL_FUTURE) == -1) {
        perror("mlockall failed");
    }
    
    // ... 回测代码 ...
}
```
# 关键关卡参数
## `sched_rt_runtime_us`

想让程序调用`set_realtime_priority(50, 7)`，第一个参数是指定优先级，越大越好。第二个参数是绑到哪个CPU核。绑核没问题。但是设置优先级一直不成功。返回的errno是1，代表EPERM（Operation not permitted），在Linux中是权限与限制不匹配的问题。

解决路径：
1. ulimit -r unlimited，表面上是ulimit -r从0到了unlimited，但是不行。
    1. 即使执行了 `ulimit -r unlimited`，如果系统的底层配置文件、内核安全限制或者当前进程的权限位（Capabilities）没有正确开放，`sched_setschedparam` 依然会返回失败。
    2. 不要只看`ulimit -r`，运行`cat /proc/self/limits | grep "Max realtime priority"`更准确。
2. 修改`/etc/security/limits.conf`和`/etc/systemd/system.conf`。修改完了之后重新登陆终端ulimit -r还是0，需要重启。重启完了变99了，但是`set_realtime_priority`还是不行。

`/etc/security/limits.conf`

```conf
# 允许用户设置实时优先级
your_user  hard  rtprio  99
your_user  soft  rtprio  99
# 建议同时锁定内存，防止物理内存被 swap 到磁盘导致 P99 抖动
your_user  hard  memlock  unlimited
your_user  soft  memlock  unlimited
```

`/etc/systemd/system.conf`

```conf
DefaultLimitRTPRIO=99
DefaultLimitMEMLOCK=infinity
```

3. 内核“防死锁”保护：`sched_rt_runtime_us`。Linux 内核为了防止实时线程（RT Thread）死循环导致整个系统卡死，默认有一个保护机制：**每 1 秒钟强制留出 0.05 秒给非实时任务。**
    - **默认值**：`sched_rt_period_us` = 1,000,000 (1s)
    - **默认值**：`sched_rt_runtime_us` = 950,000 (0.95s)

如果把回测线程绑在某个核上，并设为 `SCHED_FIFO`，一旦计算超过 0.95 秒没有释放 CPU，内核就会强制中断。这不仅会导致计时不准，还可能引发权限冲突。

关闭该限制（在隔核环境下设置这个是安全的，因为还有其他核心在处理系统任务）：
```bash
# 设置为 -1，表示实时线程可以占用 100% 的 CPU 时间
sudo sysctl -w kernel.sched_rt_runtime_us=-1
```

生效后，`set_realtime_priority(50)`成功了。

### 永久固化内核参数
为了让这个设置在下次重启后依然生效，需要将其写入系统配置文件：
编辑 `/etc/sysctl.conf`：
在文件末尾添加一行：
```
kernel.sched_rt_runtime_us = -1
```
执行命令使其立即生效：
```
sysctl -p
```
