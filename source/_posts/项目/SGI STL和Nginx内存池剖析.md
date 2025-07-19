---
title: SGI STL和Nginx内存池剖析
categories:
  - - 项目
  - - Cpp
tags: 
date: 2022/4/8
updated: 
comments: 
published:
---

# 内容

内存池是为了更高效地管理小块内存的频繁开辟、释放。
1. `C++` STL：标准模板库。
2. SGI STL：第三方厂商开发的，后来被纳入`C++`标准。
# `C++ STL`普通空间配置器

先用vector举个例子

```cpp
template<typename T, typename _Alloc = allocator<T>>
class vector
{
    
};
```

可以看到，vector第二个模板参数是一个空间配置器。

主要包含了四个方法：
1. `allocate`：负责给容器开辟内存空间=>`malloc`
2. `deallocate`：负责释放容器内存空间=>`free`
3. `construct`：负责在容器中构造对象=>`定位new`：是基于已经开辟好了的容器空间中直接构造的。
4. `destroy`：负责析构容器中的对象=>`p->~T()`

空间配置器的核心作用：
1. 拆开了new的两个操作——对象的内存开辟、对象构造；
2. 拆开了delete的两个操作——对象的析构，内存的内存释放。
把空间和对象本身分开，在容器这个场景下更为适合。
# SGI STL

提供了两个allocator的实现：
1. 一级allocator，实际上就是`malloc/free`；
2. 二级allocator，是基于内存池的内存管理。
本文主要剖析SGI的二级allocator，即内存池的实现。

通过阅读源码，发现：

SGI STL底层对于容器的对象的构造、析构是通过自定义的全局模板函数Construct和Destroy完成的。而点进去发现本质上仍是通过定位new、调用对象的析构函数完成的。
因此，可以推断，SGI STL的空间配置器主要工作的区别在于allocate和deallocate，即对容器内存空间的管理。
# 高效内存池设计和实现

## 实现思路（分而治之）
思路1：对于每个请求或者连接都会建立相应的内存池，建立好内存池之后，我们可以直接从内存池中申请所需要的内存，不用去管内存的释放，当内存池使用完成之后一次性销毁内存池。
思路2：区分大小内存块的申请和释放，大于池尺寸的定义为大内存块，使用单独的大内存块链表保存，即时分配和释放；小于等于池尺寸的定义为小内存块，直接从预先分配的内存块中提取，不够就扩充池中的内存，在生命周期内对小块内存不做释放，直到最后统一销毁。
## Nginx内存池结构图
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250719234700476.png)

## 关键数据结构
```c
typdef strcut
{
    u_char        *last;       // 当前数据块中内存分配指针的当前位置   
    u_char        *end;        // 内存块的结束位置
    ngx_pool_t    *next;       // 内存池由多块内存块组成，指向下一个数据块的位置
    ngx_uint_t    failed;      // 当前数据块内存不足引起分配失败的次数
} ngx_pool_data_t;

struct ngx_pool_s
{
    ngx_pool_data_t  d;        // 上面那个结构体：内存池当前的数据区指针的结构体
    size_t           max;      // 当前数据块最大可分配的内存大小（字节数）
    ngx_pool_t       *current; // 当前正在使用的数据块的指针
    ngx_pool_large_t *large;   // pool中指向大数据块的指针（大数据块是指size > max的数据）
};
```
`npx_pool_t`结构示意图（大小为1024的池）
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250719235439957.png)

