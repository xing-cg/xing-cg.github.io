---
title: Cpp_原子类型
categories:
  - - Cpp
    - Modern
  - - 操作系统
    - 多线程
tags: 
date: 2022/4/10
updated: 2024/7/24
comments: 
published:
---
# 内容

1. 线程安全问题

# 线程安全问题

```cpp
int count = 0;
count++;
count--;	/* 是线程不安全的 */
```
后置`++`有多个动作：
1. 取出count的值为tmp
2. 计算`count + 1`
3. `count + 1`的结果赋给count
4. 返回tmp

对应于多条指令。

传统地解决，可以用加锁的方式
```cpp
{
    lock_guard<std::mutex> guard(mtx);
	count++;
}
```
但是，互斥锁是比较耗费资源的，如果临界区的代码比较轻量级，那么传统mutex锁相对而言就比较小题大做了。

现在有新机制解决：指令级并发。

如果想让这么多动作在一条指令内做完的话，需要让处理器支持这样的操作，而不用锁机制，因此这种叫做非锁结构。
# 原子类型
定义于`<atomic>`
原子类型，封装了一个值，保证其的访问不会导致数据竞争，并且可用于同步不同线程之间的内存访问。
## atomic_flag
初始化可以赋值为`ATOMIC_FLAG_INIT`，意为无状态，对应false。
`test_and_set()`：置位为有状态，对应true，并返回调用之前的状态。这是一个原子操作，不会被其他线程干扰。

以下就是一个用atomic_flag做忙等待的例子：
1. 初始flag为无状态
2. 调用一次`test_and_set()`置位flag为有状态
3. while中一直调用`test_and_set()`，测试它的状态，直到为无状态时，退出循环，结束程序。

```cpp
#include <atomic>

int main()
{
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
    flag.test_and_set();
    while (flag.test_and_set())
    {
        continue;   
    }
    return 0;
}
```

给定一个`atomic_flag`，初始为无状态，同时启动10个线程，操作之前，调用`test_and_set`，如果测试为无状态，说明没有其他线程在操作，就可以进行操作。操作完毕后，clear重置flag为无状态。下一个线程就可以探测到无状态，开始它的操作。
此时，atomic_flag就相当于自旋锁的作用。
```cpp

std::atomic_flag lock_output = ATOMIC_FLAG_INIT;
std::counting_semaphore<10> sema{ 0 };
void worker(int v)
{
    while (lock_output.test_and_set())
    {}
    std::cout << "thread #" << v << std::endl;
    lock_output.clear();
}
int main()
{
    std::jthread t[10];
    for (int i = 0; i < 10; ++i)
    {
        t[i] = std::jthread(&worker, i + 1);
    }
    sema.release(10);
    using namespace std::chrono_literals;
    std::this_thread::sleep_for(5s);
    return 0;
}
```
输出：
```
thread #1
thread #5
thread #4
thread #7
thread #2
thread #10
thread #6
thread #3
thread #9
thread #8
```
每个线程有序输出，没乱。
## atomic
```cpp
template <class T> struct atomic;
```
原子对象的主要特征是，从不同线程访问值不会导致数据竞争（即，这样做是明确定义的行为，访问顺序正确）。通常，对于所有其他对象，如果同时访问同一对象而导致数据争用，则该操作将被视为未定义行为。

原子对象能够通过指定不同的内存顺序来同步对其线程中其他非原子对象的访问。
# 内存顺序
原子操作只是提供了不同线程读写的同步。
但是没有提供操作顺序的同步。
比如：线程1修改原子值a，线程2读取原子值a。
两个线程各自的读写操作，确实是保证了不会有脏值。但是线程1、线程2的顺序没有做控制，如果线程2想要读出旧值，但是线程1在线程2读值前进行写操作，还是会导致线程2读出脏值。因此需要提供内存顺序机制。

如下程序，可能存在几个内存顺序问题：
1. x、y的store顺序可能会被交换，但是不影响正确结果。
2. consume在load y时，可能produce中的y还没store完毕，读取出false，则浪费了一次load操作，性能下降。

```cpp
std::atomic<bool> x, y;
std::atomic<int> z;
void produce()
{
    x.store(true);              // step 1/2
    y.store(true);              // step 1/2
}
void consume()
{
    while (!y.load())           // step 3
    {}
    if (x.load())               // step 4
    {
        ++z;
    }
}
int main()
{
    x = false;
    y = false;
    z = 0;
    std::thread t(&consume);
    std::thread t2(&produce);
    t.join();
    t2.join();
    std::cout << z << std::endl;  // z : 1
}
```

| 类型      | 含义                                                                                                                                                                                  |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| relaxed | CPU和编译器可以重新排序变量顺序。<br>这是一种松散的内存顺序，不保证不同线程中的内存访问对原子操作进行排序。                                                                                                                           |
| consume | 针对某个原子变量的访问指令（store）重排到此指令（load）前。即自己load排到针对某个变量的release操作后。                                                                                                                       |
| acquire | 所有访问指令（store）排到此指令（load）前。即自己load排到所有release操作后。                                                                                                                                    |
| release | 所有访问指令（load）排到此指令（store）之后。即自己store排到所有consume、acquire操作之前。<br>扮演了一个同步点（synchronization point）的角色。                                                                                  |
| acq_rel | The operation loads acquiring and stores releasing。<br>该操作可能扮演两种角色。<br>比如`std::atomic::exchange`操作，两个变量交换：<br>需要load值，可能需要等待release；<br>需要store值，即产生release，需要给其他acquire、consume通知。 |
| seq_cst | sequentially consistent，意思是该操作以顺序一致的方式排序。一旦所有可能对其他线程产生可见副作用的内存访问已经发生，则所有操作使用此内存顺序。<br>这是最严格的内存顺序，保证了在非原子内存访问中线程交互之间的意外副作用最小。<br>对于consume和acquire的load，顺序一致的store操作被认为是release操作。   |

限制内存顺序之后：
```cpp
std::atomic<bool> x, y;
std::atomic<int> z;
void produce()
{
    // 两个store都是release，没有严格限制顺序
    x.store(true, std::memory_order_release); // step 1/2
    y.store(true, std::memory_order_release); // step 1/2
}
void consume()
{
    // memory_order_consume表示必须排序在y的store之后
    while (!y.load(std::memory_order_consume))// step 3
    {}
    // memory_order_acquire表示必须排序在x store、y store之后
    if (x.load(std::memory_order_acquire))    // step 4
    {
        ++z;
    }
}
```
# lock_free测试
测试以确定原子模板包含的类型是否支持无锁。
```cpp
int main()
{
    std::wcout << std::boolalpha << std::atomic<int>{}.is_lock_free() << std::endl; // true
    std::wcout << std::boolalpha << std::atomic<int>{}.is_always_lock_free << std::endl; // true
}
```
经过测试后发现，不超过8字节（64位）的数据结构，并且没有虚函数表的数据结构，是支持无锁的。
```cpp
struct A {int x, y; A() {}}
struct B {int a[2];}
struct C {int a[3];}

int main()
{
    std::wcout << std::boolalpha << std::atomic<A>{}.is_lock_free() << std::endl; // true
    std::wcout << std::boolalpha << std::atomic<B>{}.is_lock_free() << std::endl; // true
    std::wcout << std::boolalpha << std::atomic<C>{}.is_lock_free() << std::endl; // false
}
```
# CAS

可以用CAS来保证上面操作的原子特性就足够了，也称无锁操作。其实并不是不加锁，而是加锁解锁的操作不在软件层面。

CPU和内存通信要通过系统总线进行，CAS即是通过exchange/swap指令完成的，相当于给总线加锁。当一个线程在做CPU和内存之间数据的交换时，比如CPU从内存读一个数据，CPU处理之后再写回内存。如果一个线程对总线的工作没有完成，是不允许其他线程进行操作的，这是CAS的基本思想。
