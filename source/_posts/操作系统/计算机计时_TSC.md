---
title: 计算机计时_TSC
categories:
  - - 操作系统
tags:
date: 2025/10/24
updated: 2026/1/8
comments:
published:
---
在 x86 架构的 Linux 系统中，获取纳秒级时间的核心依赖于硬件层面的 **TSC (Time Stamp Counter)** 寄存器，以及软件层面 Linux 内核提供的 **vDSO (virtual Dynamic Shared Object)** 机制。

简单来说：**硬件提供高频脉冲计数，软件通过免系统调用的方式将其快速换算为时间。**

# 3 种板载时钟机制
当前的桌面计算机有三种板载时钟机制来确定时间：
- 1）电池供电的实时时钟即使在断电的情况下也能保持计时，其精确度堪比任何石英手表。获取此类时钟的时间成本相对较高，通常操作系统只会在启动过程中查询它。
- 2）操作系统会设置一个定时器，以固定的时间间隔中断 CPU。每次中断时，内核都会增加一个计数器。Windows 和大多数 Linux 内核将此间隔设置为 10 毫秒。该时钟的频率漂移和抖动相对较低，但分辨率仅为 10 毫秒（取决于内核）。
- 3）所有现代 CPU 都包含一个在每个时钟周期递增的寄存器（例如，如果您的处理器主频为 1.0 GHz，则每秒递增 10 亿次）。不同的架构赋予该寄存器不同的名称；在本文档中，我们将其称为 TSC 寄存器。
    - 该时钟具有非常高的分辨率，但由于晶体不稳定、温度和功率波动（可能由系统负载变化引起）以及明确的电源管理（速度限制），频率漂移相对较高。

名词：**墙上时间**（Wall Clock），对应系统当前时间（1970年至今），受 NTP 修改影响。
# 1 GHz 和 1个时钟周期
G 代表 10亿（$10^9$），Hz代表“每秒钟的次数”。
GHz，是 CPU（处理器）的主频单位。

所以，**1 GHz 的意思就是：CPU 内部的时钟每秒钟震荡 10 亿次。** 这里的 10 亿次是指 CPU 内部晶体管开关切换的节奏。

那么，1 GHz 对应的 1 个时钟周期是多久？

$T=\frac{1}{f}=\frac{1}{10^9\text{Hz}}=0.000000001\text{s}=1\text{纳秒}$
1纳秒是什么概念？光在 1 纳秒能传播约 30 厘米。

# 硬件：什么是TSC（Time Stamp Counter）

这是获取高精度时间的基石。

TSC 是 x86 处理器内部的一个 64 位寄存器。它从 CPU 上电或复位开始，每一个 CPU 时钟周期（Cycle）就自动加 1。

* 精度很高：现代 CPU 主频通常在 2GHz - 4GHz，意味着 TSC 每秒增加几十亿次，1 个时钟周期小于 1 纳秒，即，理论分辨率小于 1 纳秒。
* 访问极快：CPU 提供了一条汇编指令 `rdtsc` (Read Time-Stamp Counter)，可以直接将这个寄存器的值读入通用寄存器`EDX:EAX`，开销极小。

## 历史问题与现代解决方案（Invariant TSC）
>每日单词：Invariant: never changing

早期的 TSC 有两个严重问题：
1. **变频问题：** 当 CPU 降频节能（SpeedStep）或超频时，TSC 计数速度会变，导致时间计算不准。
2. **多核同步问题：** 不同核心的 TSC 初始值或增加速度可能不一致，导致程序在不同核心间漂移时时间“倒流”。

**现代解决方案：** 现代 x86 CPU（Intel Nehalem 架构以后）引入了 **Invariant TSC (恒定速率 TSC)**。
- 无论 CPU 当前核心频率是多少，TSC 都以一个固定的“基准频率”计数。    
- 所有核心的 TSC 保持同步。
- Linux 内核启动时会检测 `CPUID` 中的 `Invariant TSC` 标志，如果支持，就会将其作为系统的首选时钟源（Clock source）。
# 软件：加速机制 vDSO（避免系统调用开销）
如果每次获取时间都要发起一次标准的系统调用（System Call），那么用户态切换到内核态的上下文切换（Context Switch）开销高达几百纳秒甚至微秒级，这会严重损耗性能。

Linux 使用 **vDSO** 技术解决了这个问题。
- **原理：** Linux 内核将一小块包含时间数据和算法的内存页（Page），直接**映射**到每一个用户进程的虚拟地址空间中。这块区域对用户是只读的。
- **过程：**
    1. 当用户调用 `clock_gettime` 时，glibc 库会检测是否支持 vDSO。
    2. 如果支持，程序并**不进入内核态**，而是直接在用户态运行 vDSO 页面中的代码。
    3. 这段代码直接读取内存页中的“基准时间”数据，并执行 `rdtsc` 指令获取当前经过的周期数，现场计算出当前时间。
# 基于 TSC 的时间计算公式
TSC 给的是“滴答数”（Cycles），用户要的是“纳秒”（Nanoseconds）。

内核（以及 vDSO 代码）通过以下公式进行转换：

$$CurrentTime = BaseTime + \frac{(CurrentTSC - LastTSC) \times Mult}{2^{Shift}}$$

- **$BaseTime$：** 内核定期（例如每 1/1000 秒）更新的一个基准时间点（存放在 vDSO 共享数据页中）。    
- **$LastTSC$：** 更新 $BaseTime$ 那一刻的 TSC 值。
- **$CurrentTSC$：** 刚才通过 `rdtsc` 读到的当前值。
- **$Mult$ 和 $Shift$：** 这是一个数学技巧。为了避免在内核中使用缓慢的浮点运算，Linux 预先计算好了一组乘数（Mult）和位移（Shift）参数，用来模拟“除以频率”的操作。

通过这种整型运算，CPU 可以极快地算出从上一次 Tick 到现在经过了多少纳秒，并加到基准时间上。
## 代码示例
### 读basetime和lasttsc、mult、shift
以下代码的工作：
计算CPU真实的平均频率。

1. 建立测量窗口 (The Measurement Window)
    * 代码并不相信 CPU 标称的频率（比如 2.5GHz），而是现场实测。
    - **动作：** 记录开始时间 `a` 和开始 Cycle `t0`。
    - **动作：** 进入一个 `do...while` 循环进行“忙等待”（Busy Wait）。
    - **目的：** 它硬生生跑满 `target_window_ns`（默认 100ms）。
    - **为什么这么做？**
        - 为了减少误差。如果只测 1ms，系统调用的几十纳秒开销占比太大，计算出的频率不准。
        - 拉长到 100ms，可以将系统调用的开销（Noise）被分母稀释掉，得到非常精准的平均频率。
2. 捕捉锚点 (Capture Anchors)
    * 当 100ms 跑完后，它抓取了三个关键数据：
    1. `t1`：结束时的 TSC 读数。
    2. `b`：结束时的单调时间（`CLOCK_MONOTONIC_RAW`）。
    3. `rt`：当前的墙上时间（`CLOCK_REALTIME`）。

此时，它拥有了一个精确的方程：

$$\Delta \text{Cycles} (t1 - t0) \approx \Delta \text{Nanoseconds} (100ms)$$
3. 计算定点运算因子 (Magic Math)
    * 这是最核心、也是最难懂的部分：
    * **背景：** 以后算时间，我们希望用公式 $Time = \frac{Cycles}{Frequency}$。但除法在 CPU 里很慢。
    - **优化：** 我们把它变成乘法 $Time = Cycles \times Factor$。
        - 但是 $Factor$ （即 $\frac{1}{Freq}$）是一个很小的小数，整数存不下。
        - 所以我们把这个小数“左移”放大 $2^{32}$ 倍（`shift=32`），存成一个大整数 `mult`。

- **原理：**
    $$\text{mult} = \frac{\Delta \text{ns} \times 2^{32}}{\Delta \text{cycles}}$$
    这里的 dt_cy / 2 是为了四舍五入，保证除法结果最接近真实值。

**结果：** 以后只需要做 `(cycles * mult) >> 32` 就能极其快速地算出纳秒，完全避开了浮点运算和除法。

4. 建立时间参考系 (Base & Offset)
    * 不能光知道“频率”，还得知道“现在几点”。
    - **`base_tsc = t1`**: 把结束时刻的 TSC 记为基准点。
    - **`base_mono_ns`**: 把结束时刻的单调时间记为基准时间。
    - **`realtime_offset`**: 计算单调时间（开机时长）和墙上时间（1970年至今）的差值。
        - 公式：`rt_ns - base_mono_ns`。
        - 这样以后算出单调时间后，加上这个 offset，就是墙上时间。




```cpp
static inline TscCalib calibrate_tsc_fixedpoint(uint64_t target_window_ns = 100000000ull /*100ms*/)
{
    // 读取起点
    timespec a{};
    clock_gettime(CLOCK_MONOTONIC_RAW, &a);
    uint64_t t0 = rdtscp_cycles();

    // 忙等到窗口时长，避免调度带来的不确定性
    uint64_t dt_ns = 0;
    timespec b{};
    uint64_t t1 = 0;
    do
    {
        clock_gettime(CLOCK_MONOTONIC_RAW, &b);
        t1 = rdtscp_cycles();
        dt_ns = (uint64_t)(b.tv_sec - a.tv_sec) * 1000000000ull + (uint64_t)(b.tv_nsec - a.tv_nsec);
    } while (dt_ns < target_window_ns);

    // 同时取一次 REALTIME 以建立 epoch 偏移
    timespec rt{};
    clock_gettime(CLOCK_REALTIME, &rt);
    uint64_t rt_ns = (uint64_t)rt.tv_sec * 1000000000ull + (uint64_t)rt.tv_nsec;

    uint64_t dt_cy = t1 - t0;
    const uint32_t shift = 32;
    
    
    // 3. 计算定点运算因子 (Magic Math)
    // mult = (dt_ns << shift) / dt_cy  （四舍五入），用 128 位避免溢出
    __uint128_t num = ((__uint128_t)dt_ns << shift) + (dt_cy / 2);
    uint64_t mult = (uint64_t)(num / dt_cy);

    uint64_t base_mono_ns = (uint64_t)b.tv_sec * 1000000000ull + (uint64_t)b.tv_nsec;
    int64_t  realtime_offset = (int64_t)rt_ns - (int64_t)base_mono_ns;

    TscCalib c{};
    c.base_tsc = t1;
    c.base_mono_ns = base_mono_ns;
    c.realtime_offset = realtime_offset;
    c.mult = mult;
    c.shift = shift;
    c.ghz = (double)dt_cy / (double)dt_ns; // cycles per ns
    return c;
}
```

#### 问题
虽然代码逻辑是对的，但在生产环境中使用需要注意以下 3 个核心问题：

A. NTP 漂移与“失效”的 Realtime Offset
```cpp
int64_t realtime_offset = (int64_t)rt_ns - (int64_t)base_mono_ns;
```

- **问题：** `CLOCK_REALTIME` 是受 NTP 调整的（会发生时间跳变或频率微调），而 TSC（以及 `CLOCK_MONOTONIC_RAW`）是恒定的硬件频率。
- **后果：** 这个 `realtime_offset` 只有在校准的那一瞬间是准的。过几分钟后，由于晶振温漂和 NTP 的介入，你用 TSC 算出来的“当前 Realtime”会和系统显示的 `date` 时间产生偏差（可能达到毫秒级）。
- **建议：** 如果只用作性能分析（算耗时），没问题。如果用于业务逻辑（如记录日志时间戳），需要**定期重新校准**（例如每秒一次），或者接受这个偏差。

B. `CLOCK_MONOTONIC_RAW` vs `CLOCK_MONOTONIC`
- 使用了 `CLOCK_MONOTONIC_RAW`，这是一个非常聪明的选择。
    - **RAW:** 直接对应硬件晶振频率，不接受 NTP 的频率修正（Slew）。这和 TSC 的行为是一致的。
    - **非 RAW:** Linux 内核会通过 `adjtimex` 微微调整频率来对齐 NTP。如果你用 TSC 去拟合非 RAW 的时间，会发现计算出的时间有时候会“跑得比墙上时钟快/慢一点点”。
- **结论：** 坚持用 `RAW` 来计算 `mult` 是对的，这样你的 `ghz` 才是 CPU 真实的物理频率。

C. 上下文切换的干扰
```cpp
    clock_gettime(CLOCK_MONOTONIC_RAW, &b);
    t1 = rdtscp_cycles(); // <--- 风险点
```
- **风险：** 虽然几率很小，但在 `clock_gettime` 返回后、`rdtscp` 执行前，线程可能被操作系统**抢占（Preempt）**或者发生中断。
- **后果：** `t1` 读晚了，但 `b` (时间) 是旧的。会导致计算出的起点 `base_tsc` 和 `base_mono_ns` 没对齐。
- **改进建议（高阶优化）：** 可以在校准时循环多次（例如 10 次），计算 `(t_after - t_before)` 的差值，取**差值最小**（也就是被干扰最少）的那一次作为基准点。

#### 代码改进建议
在校准部分加入“最小误差过滤”，并封装一下 calibration 结构体。
这里有一个微调后的建议版本：

```cpp
#include <time.h>
#include <stdint.h>
#include <stdio.h>
#include <x86intrin.h> // 包含 rdtscp 的内建支持

// 简单的自旋等待，防止编译器优化
static void spin_wait_ns(uint64_t ns) {
    struct timespec start, now;
    clock_gettime(CLOCK_MONOTONIC_RAW, &start);
    do {
        clock_gettime(CLOCK_MONOTONIC_RAW, &now);
    } while ((uint64_t)(now.tv_sec - start.tv_sec) * 1000000000ull + 
             (uint64_t)(now.tv_nsec - start.tv_nsec) < ns);
}

static inline TscCalib calibrate_tsc_robust() {
    TscCalib c = {};
    const int iterations = 10; // 尝试多次捕捉最佳对齐点
    const uint64_t wait_ns = 20000000; // 20ms 间隔，不需要太长，太长容易受温漂影响
    
    struct timespec t_start, t_end;
    uint64_t tsc_start, tsc_end;
    
    // 1. 先热身一下，把代码加载到 cache
    clock_gettime(CLOCK_MONOTONIC_RAW, &t_start);
    
    // 2. 测量频率 (Mult 计算)
    // 采用长窗口计算频率更准
    clock_gettime(CLOCK_MONOTONIC_RAW, &t_start);
    tsc_start = __rdtscp(&c.shift); // 借用一下参数位，实际上这里不需要 aux
    
    spin_wait_ns(100000000); // 100ms
    
    clock_gettime(CLOCK_MONOTONIC_RAW, &t_end);
    tsc_end = __rdtscp(&c.shift);
    
    uint64_t dt_ns = (t_end.tv_sec - t_start.tv_sec) * 1000000000ull + (t_end.tv_nsec - t_start.tv_nsec);
    uint64_t dt_cy = tsc_end - tsc_start;
    
    c.shift = 32;
    __uint128_t num = ((__uint128_t)dt_ns << c.shift) + (dt_cy / 2);
    c.mult = (uint64_t)(num / dt_cy);
    c.ghz = (double)dt_cy / dt_ns;

    // 3. 确定 Base (对齐锚点)
    // 我们需要 tsc 和 monotonic 时间尽可能的“紧挨着”
    // 运行多次，取系统调用开销看起来最小的那一次
    uint64_t min_delta = (uint64_t)-1;
    
    for (int i = 0; i < 50; i++) {
        struct timespec t0, t1;
        uint64_t tsc;
        
        clock_gettime(CLOCK_MONOTONIC_RAW, &t0);
        tsc = __rdtscp(&c.shift); // 这里的 aux 可以扔掉
        clock_gettime(CLOCK_MONOTONIC_RAW, &t1);
        
        // 计算 gettime 这一来回花了多久
        uint64_t delta = (t1.tv_sec - t0.tv_sec) * 1000000000ull + (t1.tv_nsec - t0.tv_nsec);
        
        if (delta < min_delta) {
            min_delta = delta;
            c.base_tsc = tsc;
            // 认为 tsc 是在 t0 和 t1 中间发生的
            // 或者保守点，认为 tsc 对应 t0 (因为 rdtscp 是序列化的)
            c.base_mono_ns = (uint64_t)t0.tv_sec * 1000000000ull + t0.tv_nsec;
        }
    }
    
    // 计算 Realtime Offset (仅供参考，随时间失效)
    struct timespec rt;
    clock_gettime(CLOCK_REALTIME, &rt);
    uint64_t rt_ns = (uint64_t)rt.tv_sec * 1000000000ull + rt.tv_nsec;
    // 这里简单计算，生产环境最好也多次采样
    c.realtime_offset = (int64_t)rt_ns - (int64_t)((__uint128_t)(c.base_tsc - c.base_tsc) * c.mult >> c.shift) - c.base_mono_ns; 
    // 上面这行其实就是 rt_ns - base_mono_ns，因为 base_tsc 刚好对应 base_mono
    c.realtime_offset = (int64_t)rt_ns - (int64_t)c.base_mono_ns;

    return c;
}
```

如果发现时间戳和系统日志里的时间偶尔有几十微秒的对不齐，那是 Calibration 过程中 `clock_gettime` 和 `rdtscp` 之间的微小缝隙（Jitter）造成的。使用上面的“多次采样取最小值”策略可以优化这一点。
### 读currentTSC
- **指令选择：** 使用 `rdtscp` 是正确的。相比 `rdtsc`，它不仅读取计数器，还强制让前面的指令先执行完（Serializing），这对精准计时至关重要。
- **Memory Clobber：** `:: "memory"` 告诉编译器不要把这行汇编上下的内存操作重排，这对于 benchmark 代码非常关键。

```cpp
#if defined(__x86_64__)
static inline __attribute__((always_inline)) uint64_t rdtscp_cycles() {
    unsigned aux, lo, hi;
    asm volatile("rdtscp" : "=a"(lo), "=d"(hi), "=c"(aux) :: "memory");
    return ((uint64_t)hi << 32) | lo;
}
```
### 计算ns
- **定点运算：** 使用 `(delta * mult) >> shift` 是标准的内核级时间转换算法。
- **防溢出：** 计算中，`delta` (cycles) 乘以 `mult` 可能会非常大。
    - 假设 3GHz CPU，1秒对应 $3 \times 10^9$ cycles。
    - `shift=32` 时，`mult` 大约为 $0.33 \times 2^{32} \approx 1.4 \times 10^9$。
    - 两者相乘 $\approx 4.2 \times 10^{18}$。
    - `uint64_t` 最大值是 $\approx 1.8 \times 10^{19}$。
    - 如果不转成 `__uint128_t`，只要时间超过 **4秒** 左右，乘法就会溢出。**使用了 `__uint128_t`，这是完美的做法，支持极长时间不溢出。

```cpp
static inline __attribute__((always_inline)) uint64_t tsc_now_ns_mono_raw(const TscCalib& c) {
    uint64_t t = rdtscp_cycles();
    uint64_t d = t - c.base_tsc;
    __uint128_t prod = (__uint128_t)d * (uint64_t)c.mult;
    return c.base_mono_ns + (uint64_t)(prod >> c.shift);
}

static inline __attribute__((always_inline)) uint64_t tsc_now_ns_realtime(const TscCalib& c) {
    return (uint64_t)((int64_t)tsc_now_ns_mono_raw(c) + c.realtime_offset);
}
```
# 获取纳秒的方式一览
在 Linux 环境下使用 C++ 开发时，获取时间的方法主要分为 **C++ 标准库层**、**POSIX 系统调用层** 和 **硬件指令层**。
这三者在**可移植性**、**精度**和**性能**上各有千秋。
## `C++`标准库层（`std::chrono`）

从 C++11 开始引入，是现代 C++ 获取时间的标准方式。
- **特点：** 类型安全、跨平台、代码可读性高。
- **底层：** 在 Linux 上，编译器通常会将其优化为对 `clock_gettime` 的内联调用，性能几乎没有损耗。
- **常用时钟：**
    - `std::chrono::system_clock`：**墙上时间**（Wall Clock），对应系统当前时间（1970年至今），受 NTP 修改影响。
    - `std::chrono::steady_clock`：**单调时钟**，保证时间只增不减，适合计算耗时、超时判断。
    - `std::chrono::high_resolution_clock`：通常是上面二者之一的别名（取决于实现）。

**代码示例：**
```cpp
#include <iostream>
#include <chrono>

void cpp_style() {
    // 获取当前时间点
    auto start = std::chrono::steady_clock::now();
    
    // ... 做一些操作 ...
    
    auto end = std::chrono::steady_clock::now();
    
    // 计算耗时 (纳秒)
    auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);
    std::cout << "耗时: " << duration.count() << " ns" << std::endl;
}
```

## POSIX 系统调用层 (`clock_gettime`)

这是 Linux 系统获取纳秒级时间的核心 API，也是 C 语言的首选。
- **特点：** 精度高（纳秒级）、性能极高（通过 vDSO 避免进入内核态）、控制粒度最细。
- **常用 Clock ID：**
    - `CLOCK_REALTIME`：系统绝对时间（受 NTP 影响）。
    - `CLOCK_MONOTONIC`：单调时间（受 NTP 频率微调影响，但不会回退）。
    - `CLOCK_MONOTONIC_RAW`：纯硬件计数换算，**完全不受 NTP 影响**（精度最高最稳，但可能与墙上时间有偏差）。

**代码示例：**
```cpp
#include <time.h>
#include <stdio.h>

void linux_style() {
    struct timespec ts;
    // 获取单调时间
    clock_gettime(CLOCK_MONOTONIC, &ts);
    
    long long now_ns = (long long)ts.tv_sec * 1000000000 + ts.tv_nsec;
    printf("当前纳秒: %lld\n", now_ns);
}
```

## Legacy/过时方法
- **`gettimeofday()`**：
    - **精度：** 微秒（us）。
    - **缺点：** 已经被 POSIX 标记为**废弃**（Obsolescent）。它的开销并不比 `clock_gettime` 小，但精度低了1000倍。
- **`time()`**：
    - **精度：** 秒（s）。
    - **缺点：** 精度太低，仅适用于显示日期等粗糙场景。        

## 他们输出的值都代表什么？什么样子？
为了让你直观地感受这些时间函数输出的区别，我把它们“打印”出来给你看。它们的核心区别在于**数据结构**（是整数还是结构体）以及**数值的含义**（是当前时间还是开机时间）。

以下是常见的输出形式对比：
### `std::chrono` (C++ 标准库)

`std::chrono` 的输出通常不是直接打印的，它是一个**强类型的对象**（TimePoint 或 Duration）。需要调用 `.count()` 才能拿到里面的数值。

- **输出样子：** 巨大的整数 (通常是 `long long`)

| **时钟类型**         | **原始值示例 (.count())**  | **含义**                                                |
| ---------------- | --------------------- | ----------------------------------------------------- |
| **system_clock** | `1704787200000000000` | **从 1970-01-01 00:00:00 到现在**的纳秒数。                    |
| **steady_clock** | `45000000000000`      | **从系统启动到现在**的纳秒数（这个例子约 12.5 小时）。这个值本身对人类无意义，适合来算相对耗时。 |

## `clock_gettime` (Linux 原生)
也是墙上时间（相对于1970年）

它输出的是一个结构体 `struct timespec`，把时间拆成了“秒”和“纳秒”两部分。**这是为了避免 32 位系统溢出**。

- **数据结构：**
    
    ```c
    struct timespec {
        time_t   tv_sec;  // 秒
        long     tv_nsec; // 纳秒 (0 ~ 999,999,999)
    };
    ```
    
- **输出样子：** `{1704787200, 123456789}`
- **直观理解：** 1704787200 秒 **又** 123 毫秒 456 微秒 789 纳秒。

## `gettimeofday` (旧式 C)
也是墙上时间，精度为微秒。

它输出的是 `struct timeval`，把时间拆成“秒”和“微秒”。

- **数据结构：**
    
    ```c
    struct timeval {
        time_t      tv_sec;  // 秒
        suseconds_t tv_usec; // 微秒 (0 ~ 999,999)
    };
    ```
    
- **输出样子：** `{1704787200, 123456}`
- **区别：** 精度比 `timespec` 低 1000 倍。

## `time()` (最原始)
墙上时间，精度为秒。

- **数据结构：** `time_t` (通常是 `long`)
- **输出样子：** `1704787200`
- **含义：** 仅仅是秒数。丢失了所有毫秒级精度。
    
## `rdtsc` / `rdtscp` (硬件指令)

- **数据结构：** `uint64_t` (无符号 64 位整数)
- **输出样子：** `13500000000000`
- **含义：** **CPU 既然上电以来跳动的总次数**。
    - 如果不除以 CPU 频率，这个数字对人类完全没有时间概念（你不知道它是 1 秒还是 10 年）。
    - 它**每次读取都在变大**，且增加速度极快（每纳秒增加 2-4 次）。

## 注意要点

- 读区间时，常用模式：`start=rdtsc(); … ; end=rdtsc();`  
    若对序列化严格：`lfence; rdtsc;`……`rdtscp; lfence;`    
- 线程**绑核**（`sched_setaffinity`）可避免跨核不同步导致的抖动。
- 仅做**相对时间**，不要当作“系统时间”。

**要“对点墙钟”的真实时间戳（日志、审计）** ：用 `CLOCK_REALTIME`（受 NTP/chrony 调整）； 若机器跑了 **PTP** 且网卡有 **PHC**：读 `/dev/ptpX` 对应的 clock（最准的对时）。

系统墙钟（受 NTP/chrony 校时）：
```cpp
uint64_t realtime_ns = clock_gettime_ns(CLOCK_REALTIME);
```

PTP 硬件时钟（更准，对时最强）
```cpp
#include <sys/timex.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <linux/ptp_clock.h>

int fd = open("/dev/ptp0", O_RDONLY);
clockid_t clk = FD_TO_CLOCKID(fd);
uint64_t phc_ns = clock_gettime_ns(clk); // 读取网卡 PHC
close(fd);
```

若系统跑了 `linuxptp`（`ptp4l`/`phc2sys`），PHC 与系统时钟会保持纳秒级一致。

## 对点墙钟是什么

| 对比项          | 传统挂钟               | 智能对点墙钟                              |
| ------------ | ------------------ | ----------------------------------- |
| ​**​核心功能​**​ | 自行走时，依赖内部机芯（石英或机械） | ​**​自动接收信号校准时间​**​，消除累积误差           |
| ​**​对时方式​**​ | 手动调节               | 自动（如GPS、NTP、电波）                     |
| ​**​时间精度​**​ | 有机芯本身存在的误差，会累积     | ​**​极高​**​，与标准时间源保持同步               |
| ​**​多钟同步​**​ | 难以实现，各钟显示时间可能存在差异  | ​**​轻松实现​**​，所有时钟显示完全一致的时间          |
| ​**​典型应用​**​ | 家居、普通办公室           | ​**​学校、医院、车站、工厂、办公楼​**​等需要统一时间的公共场所 |
# 纳秒时间戳怎么实现
我们通过记录 TSC，最后批量换算 ns 即可。

**单次真实“纳秒时间戳”调用 <1 ns 不现实**：
- `clock_gettime`（vDSO）通常十几到数十 ns；    
- `RDTSC/RDTSCP` 也要 ~5–15 ns（视 CPU/栅栏而定）。

解决：在**热路径**记录**TSC**（CPU 周期计数），把换算成 ns 的工作放到**批量/后台**。只要你的业务需要“本地单调时序”而不是“墙钟对时”，这就完美契合。


读 TSC（建议 RDTSCP + 绑核）

```cpp
static inline __attribute__((always_inline)) uint64_t rdtsc_ordered() {
#if defined(__x86_64__)
    unsigned aux, lo, hi;
    asm volatile("rdtscp" : "=a"(lo), "=d"(hi), "=c"(aux) :: "memory");
    // RDTSCP 自带乱序屏障，读后不会被重排到指令前
    return (uint64_t(hi) << 32) | lo;
#else
#  error "Use aarch64 cntvct_el0 or platform-specific counter"
#endif
}
```

运行前检查 CPU 有 **invariant/constant_tsc**；线程用 `sched_setaffinity` 绑核，避免跨核 TSC 偏差


固定点比例换算（高效、无浮点除法）

思路：`ns = base_ns + ((tsc - base_tsc) * mult) >> shift`。  
在启动/定时校准时，用 `CLOCK_MONOTONIC_RAW` 标定 `mult/shift`。

```cpp
#include <time.h>
#include <atomic>

struct TscCalib {
    uint64_t base_tsc;
    uint64_t base_ns;
    uint32_t mult;   // cycles -> ns 的乘子（放大后右移）
    uint32_t shift;  // 右移位数
};

// 简易标定：100ms 窗口；生产里可做多次取中位数
inline TscCalib calibrate_tsc_fixedpoint() {
    timespec a{}, b{};
    // 读起始
    clock_gettime(CLOCK_MONOTONIC_RAW, &a);
    uint64_t t0 = rdtsc_ordered();
    // 睡 100ms（或忙等更稳定）
    struct timespec req{0, 100000000};
    nanosleep(&req, nullptr);
    uint64_t t1 = rdtsc_ordered();
    clock_gettime(CLOCK_MONOTONIC_RAW, &b);

    uint64_t dt_ns = uint64_t(b.tv_sec - a.tv_sec) * 1000000000ull
                   + uint64_t(b.tv_nsec - a.tv_nsec);
    uint64_t dt_cy = t1 - t0;

    // 计算 (ns/cycle) 的定点表示：找个 shift 使 mult 不溢出
    // 我们想要：ns ≈ (cycles * mult) >> shift
    uint32_t shift = 24; // 经验值；可根据 dt_cy 调整
    uint64_t mult64 = ((dt_ns << shift) + (dt_cy/2)) / dt_cy;

    TscCalib c{};
    c.base_tsc = t1;
    c.base_ns  = uint64_t(b.tv_sec) * 1000000000ull + b.tv_nsec;
    c.mult = (uint32_t)mult64;
    c.shift = shift;
    return c;
}

inline uint64_t tsc_to_ns(uint64_t tsc, const TscCalib& c) {
    uint64_t d = tsc - c.base_tsc;
    // 128bit 乘法更稳；GCC/Clang 下内建 __int128
    __uint128_t prod = (__uint128_t)d * c.mult;
    return c.base_ns + (uint64_t)(prod >> c.shift);
}
```


 热路径怎么存储：
只做`tsc = rdtsc_ordered();` 把 `tsc` 写入环形队列 / 结构体。

大概 5 - 15 ns。

可以不在每次操作时都真的读取 TSC，而是每 N 次 / 每一批读一次，写同一个批次的时间共用一个时间戳（或首尾时间戳 + 序号插值）。这样平均到每次调用就会 小于 1 ns 了。

批量阶段：
取出`tsc`批量`tsc_to_ns()`，一次循环里用固定点乘法右移，吞吐很高（无 syscalls）。

若必须每次都带 ns 值：就只能接受`~10` ns 级开销（TSC + 换算）或`~20-60` ns （vDSO `clock_gettime`），做不到 `<1` ns。

## 如果需要“对时”怎么办（墙钟/与行情时间对齐）
- 本机“墙钟”用 `CLOCK_REALTIME`；但开销 > TSC。
- 有 PTP 的交易网卡（X710/E810 等）可读 PHC（/dev/ptpX）做对时，再把 PHC 与 TSC 做一次线性拟合，得到**TSC→UTC** 的映射（同样固定点转换）。这样热路径仍旧只记 TSC，离线批量转 **UTC 纳秒**。

## 工程化清单
- 绑核 + 关闭频率波动（performance governor）、确认 **invariant TSC**。
- 位图常驻：把位图和它的读写者放同核，避免跨核伪共享；读多写少时用**双缓冲或 RCU**，避免读到撕裂的 64 位。
- 禁止异常、RTTI，开 LTO；必要时 `-fno-plt -fno-asynchronous-unwind-tables`。
- 事件日志：用**无锁环形队列**，记录 `{idx, result, tsc}`；批量 flush 时统一换算。
- 监控：定期重标定 TSC（如每几秒做一次短窗口），防极端漂移；记录校准参数版本号到事件中，确保可逆。

## 迷你示例：批量查位 + 批量打时间
```cpp
#include <cstdint>
#include <cstddef>

struct alignas(64) Bitset20000 {
    // 20000 bits -> 313 x u64 (199+1? 实际需要 (20000+63)/64 = 313)
    static constexpr size_t kWords = (20000 + 63) / 64;
    uint64_t w[kWords];
};

// __restrict__ + always_inline + noalias，帮助编译器矢量化/消掉别名顾虑
static inline __attribute__((always_inline))
bool bit_test(const uint64_t* __restrict bs64, size_t i) {
    // 无分支：一次加载 + 位移 + 与
    return (bs64[i >> 6] >> (i & 63)) & 1u;
}

//--------------------------------------------------------------------
struct Event { uint32_t idx; uint8_t val; uint64_t tsc; };

void process_batch(const Bitset20000& bs, const uint32_t* idxs, size_t n,
                   Event* out, const TscCalib& calib) {
    // 只在批首读一次 TSC（也可每K次读一次）
    uint64_t tsc0 = rdtsc_ordered();
    for (size_t i = 0; i < n; ++i) {
        bool v = bit_test(bs.w, idxs[i]);
        out[i] = { idxs[i], (uint8_t)v, tsc0 };
    }
    // 批处理末尾再读一次，必要时你也能给范围时间
}

// 批量转 ns（离线/低频线程）
void convert_to_ns(Event* evts, size_t n, const TscCalib& c) {
    for (size_t i = 0; i < n; ++i) {
        uint64_t ns = tsc_to_ns(evts[i].tsc, c);
        // 写回或落盘…
        (void)ns;
    }
}
```

**小结**

- **查位**：上面那段无分支 bit-test + L1 热数据，**单次 ~1 ns 级**。
- **时间戳**：想要纳秒值但**平均 <1 ns**，只能**热路径记 TSC，批量换算 ns**；若必须每次拿 ns，就接受 10–60 ns 的现实（TSC+换算 或 vDSO）。
- 如需与交易所/撮合时间对齐，用 **PHC/PTP** 做对时，再把 **TSC→UTC** 的映射用于批量转换。
# Quill
- 高效“持续不断”打印纳秒时间戳的核心在于：前端线程用 TSC 直接采集 rdtsc 值并入队，后端线程运行 RdtscClock 周期校准，采用无锁转换算法将 rdtsc 快速转换为自纪元以来的纳秒；再由格式化器按 `%Qns` 输出纳秒。
- 队列采用单生产者单消费者（SPSC）环形缓冲，生产与消费均为 wait-free（无锁）。根据配置有“有界/无界 + 阻塞/丢弃”策略，但底层 SPSC 操作是无锁的；例如“UnboundedBlocking”只在触达上限时阻塞策略层面，而非用锁。

前端使用 TSC，把原始 rdtsc 推到队列；后端用 RdtscClock 周期与墙钟同步并用无锁算法转换为纳秒：

任何线程可获取与后端 TSC 时钟同步的“纳秒 since epoch”；内部通过 `BackendManager::convert_rdtsc_to_epoch_time`：
`BackendTscClock.h`

```cpp
QUILL_NODISCARD QUILL_ATTRIBUTE_HOT static time_point now() noexcept
{
  uint64_t const ts = detail::BackendManager::instance().convert_rdtsc_to_epoch_time(detail::rdtsc());

  return ts ? time_point{std::chrono::nanoseconds{ts}}
            : time_point{std::chrono::nanoseconds{
                std::chrono::time_point_cast<std::chrono::nanoseconds>(std::chrono::system_clock::now())
                  .time_since_epoch()
                  .count()}};
}
```

后端在首次遇到 TSC 源时懒初始化 RdtscClock，然后把前端塞入的 rdtsc 转成纳秒：
`BackendWorker.h`

```cpp
    if (transit_event->logger_base->_clock_source == ClockSourceType::Tsc)
    {
      // If using the rdtsc clock, convert the value to nanoseconds since epoch.
      // This conversion ensures that every transit inserted in the buffer below has a timestamp in
      // nanoseconds since epoch, allowing compatibility with Logger objects using different clocks.
      if (QUILL_UNLIKELY(!_rdtsc_clock.load(std::memory_order_relaxed)))
      {
        // Lazy initialization of rdtsc clock on the backend thread only if the user decides to use
        // it. The clock requires a few seconds to init as it is taking samples first.
        _rdtsc_clock.store(new RdtscClock{_options.rdtsc_resync_interval}, std::memory_order_release);
        _last_rdtsc_resync_time = std::chrono::steady_clock::now();
      }

      // Convert the rdtsc value to nanoseconds since epoch.
      transit_event->timestamp =
        _rdtsc_clock.load(std::memory_order_relaxed)->time_since_epoch(transit_event->timestamp);
    }
```

RdtscClock 的核心转换：以校准基准 base_time 与 base_tsc，按 ns_per_tick 做线性换算；当 rdtsc 差值超过阈值时触发重同步：

`RdtscClock.h`

```cpp
uint64_t time_since_epoch(uint64_t rdtsc_value) const noexcept
{
  auto const index = _version.load(std::memory_order_relaxed) & (_base.size() - 1);

  auto diff = static_cast<int64_t>(rdtsc_value - _base[index].base_tsc);

  if (diff > _resync_interval_ticks)
  {
    resync(resync_lag_cycles);
    diff = static_cast<int64_t>(rdtsc_value - _base[index].base_tsc);
  }

  return static_cast<uint64_t>(_base[index].base_time +
                               static_cast<int64_t>(static_cast<double>(diff) * _ns_per_tick));
}
```

后端格式化时按纳秒传给时间戳格式化器；`%Qns` 专门支持纳秒输出：

`PatternFormatter.h`

```cpp
if (_is_set_in_pattern[Attribute::Time])
{
  _set_arg_val<Attribute::Time>(_timestamp_formatter.format_timestamp(std::chrono::nanoseconds{timestamp}));
}
```

`TimestampFormatter.h`
```cpp
 * same format specifiers as strftime() but with the following additional specifiers :
 * 1) %Qms - Milliseconds
 * 2) %Qus - Microseconds
 * 3) %Qns - Nanoseconds
```

```cpp
QUILL_NODISCARD QUILL_ATTRIBUTE_HOT std::string_view format_timestamp(std::chrono::nanoseconds time_since_epoch)
{
  int64_t const timestamp_ns = time_since_epoch.count();
  ...
  if (_additional_format_specifier == AdditionalSpecifier::Qns)
  {
    static constexpr std::string_view zeros{"000000000"};
    _formatted_date.append(zeros);
    _write_fractional_seconds(extracted_ns);
  }
  ...
  return std::string_view{_formatted_date.data(), _formatted_date.size()};
}
```

队列模型：线程局部的 SPSC 队列把前端日志传给后端；默认“无界 + 阻塞”策略，但本质是 SPSC：
`overview.rst`
```
Reliable Logging Mechanism

--------------------------
  
Quill utilizes a thread-local single-producer-single-consumer queue to relay logs to the backend thread, ensuring that log messages are never dropped.

Initially, an unbounded queue with a small size is used to optimise performance.

However, if the queue reaches its capacity, a new queue will be allocated, which may cause a slight performance penalty for the frontend.

The default unbounded queue can expand up to a size of `FrontendOptions::unbounded_queue_max_capacity`. If this limit is reached, the caller thread will block.

It's possible to change the queue type within the :cpp:class:`FrontendOptions`.
```

无界 SPSC 的注释直接说明“生产/消费均为 wait-free（无锁）”，满时切换到新节点继续产出：
`UnboundedSPSCQueue.h`
```cpp
/**
 * A single-producer single-consumer FIFO circular buffer
 *
 * The buffer allows producing and consuming objects
 *
 * Production is wait free.
 *
 * When the internal circular buffer becomes full a new one will be created and the production
 * will continue in the new buffer.
 *
 * Consumption is wait free. If not data is available a special value is returned. If a new
 * buffer is created from the producer the consumer first consumes everything in the old
 * buffer and then moves to the new buffer.
 */
```

有界 SPSC 明确是环形缓冲实现，作为无界队列的基础块：
`BoundedSPSCQueue.h`

```cpp
/**
 * A bounded single producer single consumer ring buffer.
 */
```

前端写队列的调用点（prepare_write/commit），无锁热路径：

`Logger.h`

```cpp
QUILL_NODISCARD QUILL_ATTRIBUTE_HOT std::byte* _reserve_queue_space(size_t total_size,
                                                                    MacroMetadata const* macro_metadata)
{
  std::byte* write_buffer =
    _thread_context->get_spsc_queue<frontend_options_t::queue_type>().prepare_write(total_size);
  ...
  else if constexpr ((frontend_options_t::queue_type == QueueType::BoundedBlocking) ||
                     (frontend_options_t::queue_type == QueueType::UnboundedBlocking))
  {
    if (QUILL_UNLIKELY(write_buffer == nullptr))
    {
      ...
      do
      {
        ...
        write_buffer =
          _thread_context->get_spsc_queue<frontend_options_t::queue_type>().prepare_write(total_size);
      } while (write_buffer == nullptr);
    }
  }

  return write_buffer;
}
```

队列策略（有界/无界 + 阻塞/丢弃）的官方说明：

`frontend_options.rst`

```
- **UnboundedBlocking**: Starts with a small initial capacity. The queue reallocates up to `FrontendOptions::unbounded_queue_max_capacity` and then blocks the calling thread until space becomes available.
- **UnboundedDropping**: Starts with a small initial capacity. The queue reallocates up to `FrontendOptions::unbounded_queue_max_capacity` and then discards log messages.
- **BoundedBlocking**: Has a fixed capacity and never reallocates. It blocks the calling thread when the limit is reached until space becomes available.
- **BoundedDropping**: Has a fixed capacity and never reallocates. It discards log messages when the limit is reached.
```

### 简短总结

- 前端用 rdtsc 极快取样，后端 RdtscClock 周期校准，以无锁算法转换为纳秒；格式化器 %Qns 输出纳秒。

- 队列为线程局部 SPSC 环形缓冲，生产/消费均 wait-free（无锁）；策略层面可选“阻塞/丢弃/有界/无界”。因此：是无锁队列。
# fasttime 库
https://fasttime.sourceforge.net/doc/internal.html
## 来自 gettimeofday 的时间戳
gettimeofday 系统调用能够提供精度高达 1 纳秒的时间戳。

该时间由中断定时器的计数（在启动时初始化为实时时钟）获取。

任何低于 10 毫秒的精度均通过对 TSC 寄存器进行插值获得；由于内核没有关于 TSC 频率的准确信息，因此通常在启动时进行简单的校准。

```c
// startup calibration
t1 = read_tsc();
t2 = read_interrupt_count();
sleep(1);
t3 = read_tsc();
t4 = read_interrupt_count();
tsc_rate = (t4 - t2) / (t3 - t1);

// gettimeofday evaluating sub 10ms time
time = read_tsc() * tsc_rate;
```


其结果是，尽管分辨率为 1 纳秒，但系统时间只能准确报告高达 10 毫秒分辨率的时间，并且任何较低的分辨率都基于 TSC 寄存器的启动时间校准。

在许多操作系统中，gettimeofday 被实现为系统调用，需要先切换到内核，然后再切换回来才能读取时间。除了性能明显下降（在奔腾等上下文切换成本高昂的平台上尤其如此）之外，如果内核利用了这种切换，应用程序还可能丢失其时间片。例如，对于需要在发送数据包之前立即获取时间片的网络应用程序来说，这可能会带来问题。


## fasttime 的 原则
fasttime 实现基于 Network Time Protocol v4 的算法，以使用 TSC 寄存器提供准确的时间估计。
- 用于将 TSC 计数转换为时间的校准会不断更新和完善，以解决频率漂移。
- 这种重复校准不得受到系统负载的影响，并且应尽量减少系统负担。
- 校准可以由守护进程完成，并且其最新的校准表可供实现 fasttime 库的所有用户进程使用。
- 校准必须快速适应系统时间的变化；例如由 NTP 守护进程或手动用户干预带来的变化。
- 该库应该在用户空间中运行，因此上下文切换时性能不会下降。

## Derived Time 派生时间
fasttime 将 TSC 值与当前时间之间的关系建模为线性函数。校准过程会保留一个截距（intercept）和梯度（gradient），并将其应用于 TSC 值以得出当前时间。最初，这些值是使用一个非常简单的校准循环计算的，该循环类似于操作系统使用的循环；然后使用锁相环（phase-locked loop）对其进行迭代调整。

锁相环（PLL）算法将当前偏移量作为输入，并返回一个调整值，用于 fasttime 的梯度和截距。该实现大致基于 NTP4 的 PLL：

$$
\begin{align}
prediction_k &= offset_{k-1} + mu_k * (gradient_k - gradient_{k-1});\\
correction_k &= offset_k - prediction_k / 2;\\
interceptk_k &= correction_k;\\
gradient_k &= gain * correction_k / mu_k\\
\end{align}
$$

其中 $k$ 是迭代次数，$gain$ 是某个常数，$mu$ 是 TSC 值 与上一次迭代之间的差值。

两个因素决定了 PLL 的行为：迭代间隔时间（称为环路延迟（loop delay））和增益值（gain）。

较短的环路延迟允许派生时钟（derived clock）更快地收敛（converge）到系统时间，但更容易出现振荡（oscillation）和偏移测量误差。
较长的环路延迟通常可以提供更稳定的性能，但派生时钟中的任何误差都需要更长时间才能纠正。

PLL 增益也起着类似的作用：较高的增益会导致振荡，而较低的增益则需要更长的时间来稳定。

在 fasttime 中，当时钟稳定时，环路延迟会周期性地延长；当时钟不稳定时，环路延迟会缩短。

控制延迟加大或减小多少以及 PLL 增益的可以通过 fasttimed 的命令行参数指定。
## Clock Sampling 时钟采样
从 fasttime 调用 gettimeofday 到 计算出时间（when the time is evaluated）之间存在延迟，并且在调用返回之前再次存在延迟。估算 派生时钟 与 系统时间 之间偏移量的一种简单方法是 对系统调用中的派生时间进行平均：

```c 
t1 = get_derived_time();  
t2 = gettimeofday();  
t3 = get_derived_time();  
offset = t2 - (t1 + t3) / 2;
```

如果 `(t2 - t1) == (t3 - t2)`，即延迟对称，则 offset 为 0 ，gettimeofday 结果准确。

但有很多原因可能导致结果不准确，例如在上下文切换期间丢失了时间片（这种情况已在 Pentium 4 处理器上运行 Linux 时进行过测量，并确认至少会发生）。

`[Shalunov 2000]`描述了一种更好的方法，该方法只需假设延迟是随机对称分布的，即可进行多次采样并组合。该方法既在 fasttime 中实现，用于采样系统时间；也在演示应用程序中实现，用于测试 fasttime 的准确性。


## Rate change amortisation 速率变动摊销

获取当前时间的主要应用之一是测量某个进程的运行时间。对于这类应用来说，时间的绝对准确性并不那么重要，只要速率稳定且正确即可。fasttime 通过尽可能避免引入剧烈的速率变化来适应这些程序。

当速率变化相对较小时（由于正常的 PLL 程序），变化会在几秒钟内逐渐完成。较大的误差仍会立即得到纠正（例如，系统时间的变化）。

## Clock filtering 时钟滤波
由于操作系统对系统时钟的校准不佳，其返回值偶尔会出现 30-100 微秒左右的抖动（glitches）。这种抖动（jitter）通常不会被大多数应用程序察觉，但对于 fasttime 来说却很显著，因为这导致引入时钟振荡（oscillation），而振荡需要一些时间来校正。

为了解决这个问题以及预期的硬件抖动，在将偏移传递给 PLL 之前对偏移施加滤波器。该偏移量会与最近 10 个样本的中位数进行比较；如果偏移量超过该中位数一定量（例如 5 倍），则不会调整时钟。实际上，该样本会被丢弃，尽管它确实会对后续的滤波有所贡献，因此，当系统时间发生真正的变化时，它会在短时间内通过滤波器。

## Shared memory protection 共享内存保护

FastTime 在客户端/服务器模型中运行，其中校准表格（calibration table）在单独的进程或线程中不断更新到客户端应用程序。在 fasttime 处于单独进程（fasttimed 守护进程）的情况下，POSIX 共享内存用于允许客户端访问校准。

保护共享内存的标准方法是使用互斥体、信号量或消息传递。在这种情况下，这些都不能应用，因为它需要客户端应用程序进行系统调用，这是不使用 gettimeofday 的主要动机之一。

相反，fasttime 维护一个校准表格的循环数组，其中只有一个在任何时候处于活动状态（这意味着它用于计算客户端的派生时间）。守护进程或校准线程更新未使用的校准表，然后以原子方式更新指向活动表格（active table）的索引。这种原子更新是通过 CPU 指令而不是系统调用完成的。

## Terminology  术语

* System time  系统时间
    * The current time, as returned by gettimeofday  当前时间，由 gettimeofday 返回
* Derived time  派生时间
    * Current time, calculated by fasttime  当前时间，由 fasttime 计算
* Offset  偏移
    * The difference between system and derived time  系统时间与派生时间的差值
* Rate  率
    * The frequency, or speed of a clock. For system time this is ideally 1 sec/sec, but may vary due to wander or NTP adjustment.  时钟的频率或速度。对于系统时间，理想情况下为 1 秒/秒，但可能会因漂移或 NTP 调整而变化
* Loop delay  循环延迟
    * Time between iterations of the PLL  PLL 迭代之间的时间
## References  引用

[Clock Discipline Algorithms for the Network Time Protocol Version 4](http://www.eecis.udel.edu/~mills/database/reports/allan/secureb.pdf), D. Mills 1997  
[网络时间协议第 4 版的时钟规则算法](http://www.eecis.udel.edu/~mills/database/reports/allan/secureb.pdf) ，D. Mills 1997

[Adaptive Hybrid Clock Discipline Algorithm for the Network Time Protocol](http://www.eecis.udel.edu/~mills/database/papers/allan.pdf), D. Mills 1998  
[网络时间协议的自适应混合时钟规则算法](http://www.eecis.udel.edu/~mills/database/papers/allan.pdf) ，D. Mills 1998

[PC Based Precision Timing Without GPS](http://parapet.ee.princeton.edu/~sigm2002/papers/p1-pasztor.pdf), A. Pasztor & D. Veitch 2002  
[基于 PC 的无 GPS 精确计时](http://parapet.ee.princeton.edu/~sigm2002/papers/p1-pasztor.pdf) ，A. Pasztor 和 D. Veitch 2002

[NTP Implementation and Assumptions about the Network](http://www.internet2.edu/%7Eshalunov/ntp-min.pdf), S. Shalunov 2000  
[NTP 实施和关于网络的假设](http://www.internet2.edu/%7Eshalunov/ntp-min.pdf) ，S. Shalunov 2000

[Source code](http://ntp.isc.org/bin/view/Main/SoftwareDownloads) for NTP4 is also an excellent reference as it differs from the descriptions above.  
NTP4 的[源代码](http://ntp.isc.org/bin/view/Main/SoftwareDownloads)也是一个很好的参考，因为它与上面的描述不同。

