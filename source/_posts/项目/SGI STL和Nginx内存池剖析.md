---
title: SGI STL和Nginx内存池剖析
typora-root-url: ../..
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
