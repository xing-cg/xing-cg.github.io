---
title: SGI-STL和Nginx内存池剖析
categories:
  - - 项目
    - 内存池
  - - Cpp
    - STL
tags: 
date: 2022/4/8
updated: 2025/8/7
comments: 
published:
---
# 内容

内存池是为了更高效地管理小块内存的频繁开辟、释放。
1. `C++` STL：标准模板库。
2. SGI STL：第三方厂商开发的，后来被纳入`C++`标准，成为了`C++` STL中管理内存的底层实现。
3. Nginx内存池设计
# `C++ STL`空间配置器

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
# SGI STL 的两级 allocator

提供了两个allocator的实现：
1. 一级allocator，实际上就是`malloc/free`；
2. 二级allocator，是基于内存池的内存管理。
本文主要剖析SGI的二级allocator，即内存池的实现。

通过阅读源码，发现：

SGI STL底层对于容器的对象的构造、析构是通过自定义的全局模板函数Construct和Destroy完成的。
而点进去发现本质上仍是通过**定位new**、**调用对象的析构**函数完成的。这些都是内存申请、释放工作之外的动作。

因此，可以推断，SGI STL的空间配置器**主要工作的区别在于 allocate 和 deallocate**，即**对容器内存申请、释放的管理**。
# SGI STL 内存池的实现
内存池的粒度信息
```cpp
enum { _ALIGN = 8 };
enum { _MAX_BYTES = 128 };
enum { _NFREELISTS = 16 };
```
每一个内存chunk块的头信息
```cpp
union _Obj
{
    union _Obj* _M_free_list_link;
    char _M_client_data[1];
};
```
组织所有自由链表的指针数组。
这是静态变量，多线程共享，volatile避免了读取缓存的脏数据。
```cpp
static _Obj* __STL_VOLATILE _S_free_list[_NFREELISTS];
```
图示：
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250807041312313.png)
一个数组，第 1 个位置存放的是 8 字节的内存池，第 2 个位置存放的是 16 字节的内存池，...，最后一个位置存放的是 128 字节的内存池。数组的大小和每个位置对应的字节数由**内存池的粒度信息**决定。
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250807041752139.png)
## `allocate(size_t)`
定义在`stl_alloc.h`中

统一的allocate接口。外部申请 n 个字节。如果是大块内存，则普通malloc。
如果是128字节及以下的内存，则会从内存池中分配，这就是二级空间配置。

```cpp
  static void* allocate(size_t __n)
  {
    void* __ret = 0;

    if (__n > (size_t) _MAX_BYTES)
    {
      __ret = malloc_alloc::allocate(__n);
    }
    else  // 核心代码
    {
        // 返回 freelist 的某下标的指针（二级指针）
      _Obj* __STL_VOLATILE* __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      // Acquire the lock here with a constructor call.
      // This ensures that it is released in exit or during stack
      // unwinding.
      
#     ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#     endif
        // 去看 freelist 这个下标，有没有已开辟且由空闲的块
      _Obj* __RESTRICT __result = *__my_free_list;
      if (__result == nullptr)  // 没有空闲块，或者没开辟。
      {
        __ret = _S_refill(_S_round_up(__n));
      }
      else                      // 有空闲块。
      {
          // 让 这个下标 指向下一块空闲块。（见_Obj联合体定义）
        *__my_free_list = __result -> _M_free_list_link;
          // 返回的是 __result 这个空闲块
        __ret = __result;
      }
    }
    return __ret;
  };
```
###  `_S_freelist_index(size_t)`
用于返回freelist的下标，0下标存放的是8字节的内存池块。1下标存放的是16字节的内存池块。以此类推。
比如，
外部申请 1 字节，则 $(1 + 8 - 1) / 8 - 1 = 1 - 1 = 0$
外部申请 7 字节，则 $(7 + 8 - 1) / 8 - 1 = 1 - 1 = 0$
外部申请 8 字节，则 $(8 + 8 - 1) / 8 - 1 = 1 - 1 = 0$
外部申请 9 字节，则 $(9 + 8 - 1) / 8 - 1 = 2 - 1 = 1$
```cpp
static  size_t _S_freelist_index(size_t __bytes)
{
    return (__bytes + (size_t)_ALIGN - 1 ) / (size_t)_ALIGN      - 1;
}
```
### `_S_round_up(size_t)`
外部申请 n 字节，返回的是 n 字节对应的在内存池块中，一小块的实际大小（8的整数倍）。
类似于向上取整（取8的整倍数的最小值），比如输入1到8，输出8。输入9到16，输出16。（输入0，输出0）
比如，
外部申请 1 字节，则 $(1 + 8 - 1) \& \sim(8 - 1) = 8 \& \sim(7) = 1000 \& \sim(0111) = 1000 \& 1000 = 8$
外部申请 7 字节，则 $(7 + 8 - 1) \& \sim(15 - 1) = 15 \& \sim(7) = 1110 \& \sim(0111) = 1110 \& 1000 = 8$
外部申请 8 字节，则 $(8 + 8 - 1) \& \sim(16 - 1) = 15 \& \sim(7) = 1111 \& \sim(0111) = 1111 \& 1000 = 8$
外部申请 9 字节，则 $(9 + 8 - 1) \& \sim(17 - 1) = 16 \& \sim(7) = 10000 \& \sim(00111) = 10000 \& 11000 = 16$
```cpp
static size_t
_S_round_up(size_t __bytes) 
{
    return (__bytes + (size_t)_ALIGN - 1) & ~((size_t)_ALIGN - 1);
}
```
## `_S_refill(size_t)`
调用`_S_chunk_alloc(__n, __nobjs)`，在内存池块中尽量找一个合适的小字节块。
`_S_chunk_alloc`内部会帮你处理底层的开辟内存池，处理内存碎片，管理内存池的指示信息等等。
由于`_S_chunk_alloc`第二个参数`__nobjs`传入的是引用，有可能`__nobjs`会被改变。
调用之前，`__nobjs`是我们想要申请的小字节块的个数。
调用结束后，`__nobjs`更新为了实际分配到的小字节块的个数。
如果是 1 ，此次分配完之后，这个内存池块正好用完了，不构建freelist下标的链表。
其他情况，构建相应的freelist下标的链表。
for循环中做的是遍历内存池大块中的每个单元小块，联合体`_Obj*`的`_M_free_list_link`指向紧挨着的下一个单元小块。
```cpp
/* Returns an object of size __n, and optionally adds to size __n free list.*/
/* We assume that __n is properly aligned.                                */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
void*
__default_alloc_template<__threads, __inst>::_S_refill(size_t __n)
{
    int __nobjs = 20;
    char* __chunk = _S_chunk_alloc(__n, __nobjs);
    
    _Obj* __STL_VOLATILE* __my_free_list;
    _Obj* __result;
    _Obj* __current_obj;
    _Obj* __next_obj;
    int __i;

    if (1 == __nobjs) return(__chunk);
    
    __my_free_list = _S_free_list + _S_freelist_index(__n);

    /* Build free list in chunk */
    __result = (_Obj*)__chunk;
    *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);
    for (__i = 1; ; __i++)
    {
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);
        if (__nobjs - 1 == __i)
        {
            __current_obj -> _M_free_list_link = 0;
            break;
        }
        else
        {
            __current_obj -> _M_free_list_link = __next_obj;
        }
    }
    return(__result);
}
```
图示：
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250807041752139.png)
## `_S_chunk_alloc(size_t, int& nobjs)`
在内存池块中尽量找一个合适的小字节块。
期间，可能会改变外部传入的`__nobjs`。（因为是引用，外部会受影响）

`__total_bytes`记录的是欲开辟的内存池大小（根据`__nobjs`，这个是小字节块数）。
`__bytes_left`指的是`_S_end_free - _S_start_free`，指目前未被开发的大小。
### `__bytes_left`还足够，返回，无需开辟新空间
如果`__bytes_left`还足够（至少是 1 个小字节块），则返回`_S_start_free`。同时移动`_S_start_free`到新的位置。
成功返回。无需额外操作。

---
### `__bytes_left`不足，开辟新的更大的内存池块
如果`__bytes_left`不足，则将要开辟新的更大的内存池块。`__bytes_to_get = 2 * __total_bytes + _S_round_up(_S_heap_size >> 4)`
#### `__bytes_left > 0`，头插到合适的一个链表，管理这个内存碎片
如果`__bytes_left > 0`。这时，为了不浪费`__bytes_left`而造成内存碎片，把这部分尚余的小内存给其对应的freelist下标的链头插。比如，剩余 32 字节，则找下标 3 ，**头插**进去。（见下文图示）
#### 开辟新的大块内存_malloc正常
以下是`__bytes_left`不足时，都需要做的操作，目的是开辟新的大块内存。返回的地址赋给`_S_start_free`。

---
如果malloc正常（`malloc返回 != 0`），更新`_S_heap_size += __bytes_to_get;`，更新`_S_end_free = _S_start_free + __bytes_to_get;`。

malloc成功，递归调用`return(_S_chunk_alloc(__size, __nobjs));`。
#### 开辟新的大块内存_malloc失败：找内存池中已开辟的大单元链表的空间
---
如果malloc失败（`malloc返回 == 0`）。则处理：
遍历freelist的下标各个内存池块。从`size_t __n`对应的freelist下标，依次往后找有没有还有空闲的。

`__i`初始化为`__n`，循环，每次`__i += (size_t) _ALIGN`（即加8）。比如，`__n`等于 40 字节，我们依次去找40、48、56等等的freelist下标的内存池块，看看有没有能分配出来空间的。

如果有，则`_S_start_free`指向第一个空闲块。更新`_S_end_free = _S_start_free + __i;`
好了，成功在更大单元的内存池块找到，递归调用`return(_S_chunk_alloc(__size, __nobjs));`。
#### 开辟新的大块内存_malloc失败：异常处理
---
如果以上的for循环找了后面更大单元的内存池块，仍没有可用空间，则是系统内存不足的迹象：
需要调用`malloc_alloc::allocate(__bytes_to_get);`。
内部最后一次进行普通malloc的挣扎。
如果malloc仍然返回 0 ，则进行异常处理（绑定的回调）。

---

```cpp
/* We allocate memory in large chunks in order to avoid fragmenting     */
/* the malloc heap too much.                                            */
/* We assume that size is properly aligned.                             */
/* We hold the allocation lock.                                         */
template <bool __threads, int __inst>
char*
__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, 
                                                            int& __nobjs)
{
    char* __result;
    size_t __total_bytes = __size * __nobjs;
    size_t __bytes_left = _S_end_free - _S_start_free;

    if (__bytes_left >= __total_bytes)
    {
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    }
    else if (__bytes_left >= __size)
    {
        __nobjs = (int)(__bytes_left / __size);
        __total_bytes = __size * __nobjs;
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    }
    else
    {
        size_t __bytes_to_get = 2 * __total_bytes + _S_round_up(_S_heap_size >> 4);
        // Try to make use of the left-over piece.
        if (__bytes_left > 0)
        {
            _Obj* __STL_VOLATILE* __my_free_list =
                        _S_free_list + _S_freelist_index(__bytes_left);

            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;
            *__my_free_list = (_Obj*)_S_start_free;
        }
        _S_start_free = (char*)malloc(__bytes_to_get);
        if (0 == _S_start_free)
        {
            size_t __i;
            _Obj* __STL_VOLATILE* __my_free_list;
            _Obj* __p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            for (__i = __size; __i <= (size_t) _MAX_BYTES; __i += (size_t) _ALIGN)
            {
                __my_free_list = _S_free_list + _S_freelist_index(__i);
                __p = *__my_free_list;
                if (0 != __p)
                {
                    *__my_free_list = __p -> _M_free_list_link;
                    _S_start_free = (char*)__p;
                    _S_end_free = _S_start_free + __i;
                    return(_S_chunk_alloc(__size, __nobjs));
                    // Any leftover piece will eventually make it to the
                    // right free list.
                }
            }
            _S_end_free = 0;	// In case of exception.
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
        }
        _S_heap_size += __bytes_to_get;
        _S_end_free = _S_start_free + __bytes_to_get;
        return(_S_chunk_alloc(__size, __nobjs));
    }
}
```
### 头插小内存碎片，图示
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250807063843813.png)
### oom异常处理
```cpp
template <int __inst>
class __malloc_alloc_template
{
private:
#ifndef __STL_STATIC_TEMPLATE_MEMBER_BUG
  static void (* __malloc_alloc_oom_handler)();
#endif
public:
  static void* allocate(size_t __n)
  {
    void* __result = malloc(__n);
    if (0 == __result)
    {
        __result = _S_oom_malloc(__n);
    }
    return __result;
  }
  // ...
};
```

```cpp
template <int __inst>
void*
__malloc_alloc_template<__inst>::_S_oom_malloc(size_t __n)
{
    void (* __my_malloc_handler)();
    void* __result;

    for (;;)
    {
        __my_malloc_handler = __malloc_alloc_oom_handler;
        if (0 == __my_malloc_handler)
        {
            __THROW_BAD_ALLOC;
        }
        
        (*__my_malloc_handler)();
        
        __result = malloc(__n);
        
        if (__result)
        {
            return(__result);
        }
    }
}
```
### `_S_start_free`、`_S_end_free`、`_S_heap_size`
这三个变量，只会在`_S_chunk_alloc(size_t, int&)`函数执行中改变。
```cpp
template <bool __threads, int __inst>
char* __default_alloc_template<__threads, __inst>::_S_start_free = 0;

template <bool __threads, int __inst>
char* __default_alloc_template<__threads, __inst>::_S_end_free = 0;

template <bool __threads, int __inst>
size_t __default_alloc_template<__threads, __inst>::_S_heap_size = 0;
```
## `deallocate(void* p, size_t)`
定义于`stl_alloc.h`

头插归还，看图示。
```cpp
template <bool threads, int inst>
class __default_alloc_template {
// ...
public:
// ...

  /* __p may not be 0 */
    static void deallocate(void* __p, size_t __n)
    {
        if (__n > (size_t) _MAX_BYTES)
        {
            malloc_alloc::deallocate(__p, __n);
        }
        else
        {
            _Obj* __STL_VOLATILE*  __my_free_list = 
                _S_free_list + _S_freelist_index(__n);
            _Obj* __q = (_Obj*)__p;
            
            // acquire lock
#           ifndef _NOTHREADS
            /*REFERENCED*/
            _Lock __lock_instance;
#           endif /* _NOTHREADS */
    
            __q -> _M_free_list_link = *__my_free_list;
            *__my_free_list = __q;
            // lock is released here
        }
    }

//...
};
```
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250807073716103.png)
## `reallocate(void* p, old_sz, new_sz)`
对已开辟的内存池块的扩容、缩容。

```cpp
template <bool threads, int inst>
void*
__default_alloc_template<threads, inst>::reallocate(void* __p,
                                                    size_t __old_sz,
                                                    size_t __new_sz)
{
    void* __result;
    size_t __copy_sz;

    if (__old_sz > (size_t)_MAX_BYTES && __new_sz > (size_t)_MAX_BYTES)
    {
        return(realloc(__p, __new_sz));
    }
    if (_S_round_up(__old_sz) == _S_round_up(__new_sz))
    {
        return(__p);
    }
    __result = allocate(__new_sz);
    __copy_sz = __new_sz > __old_sz ? __old_sz : __new_sz;
    memcpy(__result, __p, __copy_sz);
    deallocate(__p, __old_sz);
    return(__result);
}
```
## SGI STL内存池总结

SGI STL 二级空间配置器内存池的实现优点：
1. 对于每一个字节数的chunk块分配，都是给出一部分进行使用，另一部分作为备用，这个备用可以给当前字节数使用，也可以给其它字节数使用
2. 对于备用内存池划分完chunk块以后，如果还有剩余的很小的内存块，再次分配的时候，会把这些小的内存块再次分配出去，备用内存池使用的干干净净！防止小块内存频繁的分配，释放，造成内存很多的碎片出来，内存没有更多的连续的大内存块。所以应用对于小块内存的操作，一般都会使用内存池来进行管理。
3. malloc内存分配失败，还会调用`oom_malloc`这么一个预先设置好的以后的回调函数，如果没设置，则`throw bad_alloc`。设置了则`for(;;)(*oom_malloc_handler)();`。
# Nginx内存池设计和实现
区分大小内存块的申请和释放，大于池尺寸的定义为大内存块，使用单独的大内存块链表保存，即时分配和释放；小于等于池尺寸的定义为小内存块，直接从预先分配的内存块中提取，不够就扩充池中的内存，在生命周期内对小块内存不做释放，直到最后统一销毁。
## Nginx内存池结构图
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250719234700476.png)
## Nginx源码
本次分析的是Nginx-release-1.13.1的源码。
src目录下，有好多模块，其中内存池的模块位于`/src/core`目录下。使用的是C语言。
对于不同的操作系统，有不同的实现，位于`/src/os/unix`和`/src/os/win32`下。
## 指标
```c
// 位于 ngx_palloc.h
/*
 * NGX_MAX_ALLOC_FROM_POOL should be (ngx_pagesize - 1), i.e. 4095 on x86.
 * On Windows NT it decreases a number of locked pages in a kernel.
 */
#define NGX_MAX_ALLOC_FROM_POOL  (ngx_pagesize - 1)

#define NGX_DEFAULT_POOL_SIZE    (16 * 1024)

#define NGX_POOL_ALIGNMENT       16
#define NGX_MIN_POOL_SIZE                                                     \
    ngx_align((sizeof(ngx_pool_t) + 2 * sizeof(ngx_pool_large_t)),            \
              NGX_POOL_ALIGNMENT)
```
1. `NGX_MAX_ALLOC_FROM_POOL`定义了可以从内存池中申请的最大内存。默认是`Nginx页面大小减1`。x86系统下是`4096字节减1`。
2. `NGX_DEFAULT_POOL_SIZE`定义了Nginx内存池默认大小。是`16 * 1024B` 即 `16KB`。
3. `NGX_POOL_ALIGNMENT`，内存池，分配内存时的对齐大小。默认是16。
4. `NGX_MIN_POOL_SIZE`定义了内存池的最小大小。
    1. 其需要通过`ngx_align`计算，定义如下：发现和STL的`round_up`一样，向上取 d 的整 a 倍数。比如，a等于16、d等于7的话，那d取整后就是16，d等于17的话，取整后就是32。
    2. 其中d是`(sizeof(ngx_pool_t) + 2 * sizeof(ngx_pool_large_t))`。a是`NGX_POOL_ALIGNMENT`，默认是16。

```c
// 位于 ngx_config.h
#define ngx_align(d, a)     ( ( (d) + (a - 1) ) & ~(a - 1))
```
## 关键数据结构
定义于`ngx_palloc.h`
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
    ngx_pool_data_t    d;        // 上面那个结构体：内存池当前的数据区指针的结构体
    
    size_t             max;      // max 指的是 Nginx 一次分配小块内存大小 的最大值。
    ngx_pool_t         *current; // 当前正在使用的数据块的指针
    ngx_chain_t        *chain;   // 把内存池链接起来
    ngx_pool_large_t   *large;   // 指向大数据块的指针（大数据块是指size > max的数据）
    ngx_pool_cleanup_t *cleanup; // 类似于析构函数，在 内存free 之前，对内存上的数据进行处理，比如释放指针对应的外部资源
    ngx_log_t          *log;     
}; // in ngx_core.h: typedef struct ngx_pool_s ngx_pool_t;
```
在`ngx_core.h`中，
```c
typedef struct ngx_pool_s ngx_pool_t;
```
即`npx_pool_t`是`struct ngx_pool_s`的别名。

在Nginx内存池中，`npx_pool_t`这个结构只出现在第一个内存池块的头部上，后续链接的内存池块头部只有`ngx_pool_data_t`。

`struct ngx_pool_s`（别名`ngx_pool_t`）结构示意图（大小为1024的池）
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250719235439957.png)

## `npx_create_pool`创建内存池
声明于`ngx_palloc.h`，定义于`ngx_palloc.c`。
```c
ngx_pool_t * ngx_create_pool(size_t size, ngx_log_t *log);
```
返回`npx_pool_t *`。

开辟指定大小的内存池。根据不同系统、不同的对齐方法，调用不同的API。
一般是普通的malloc。
```c
ngx_pool_t * ngx_create_pool(size_t size, ngx_log_t *log)
{
    ngx_pool_t  *p;
    // 如果不用内存对齐，则实际就是普通的 malloc
    // 如果需要内存对齐，则调用posix_memalign(void ** p, size_t alignment, size_t size)
    // 或 memalign(size_t alignment, size_t size)
    p = ngx_memalign(NGX_POOL_ALIGNMENT, size, log);
    if (p == NULL)
    {
        return NULL;
    }
    // 最后一个内存池大块，指向这个新开辟的内存池大块的头信息之后。
    p->d.last = (u_char *) p + sizeof(ngx_pool_t);
    // 内存池的结束位置，整个大块的尾部。
    p->d.end = (u_char *) p + size;
    p->d.next = NULL;
    p->d.failed = 0;

    // size 更新为 除去头信息大小的 实际存储数据的大小
    size = size - sizeof(ngx_pool_t);
    // 若 size 小于 MAX_ALLOC，则 max为size，size 大于 MAX_ALLOC 则 max 为 MAX_ALLOC
    // max 指的是 Nginx 一次分配小块内存大小 的最大值。如果用户申请大于max，则按大块数据块处理。
    p->max = (size < NGX_MAX_ALLOC_FROM_POOL) ? size : NGX_MAX_ALLOC_FROM_POOL;
    // 指向自己，首地址（包含头信息）
    p->current = p;
    p->chain = NULL;
    p->large = NULL;
    p->cleanup = NULL;
    p->log = log;

    return p;
}
```
### `ngx_memalign(POOL_ALIGNMENT, size, log)`或`ngx_alloc(size, log)`
这是个宏定义，定义于`/src/os/unix`和`/src/os/win32`下的`ngx_alloc.h`。

在Unix下，有对齐的区别：如果要做内存对齐，则`size_t alignment`参数生效，否则忽略对齐参数
```c
#if (NGX_HAVE_POSIX_MEMALIGN || NGX_HAVE_MEMALIGN)
void *ngx_memalign(size_t alignment, size_t size, ngx_log_t *log);
#else
#define ngx_memalign(alignment, size, log)  ngx_alloc(size, log)
#endif
```
在Win32下，没有对齐限制。`#define ngx_memalign(alignment, size, log)  ngx_alloc(size, log)`

名字上，暂时一样。但是相应的`ngx_alloc(size, log)`函数，在两个操作系统上就是不同的实现了。
我们看Unix的：实际就是套了个**普通malloc**。然后输出了一些日志信息。
定义于`/src/os/unix/ngx_alloc.c`
```c
void * ngx_alloc(size_t size, ngx_log_t *log)
{
    void  *p;

    p = malloc(size);
    if (p == NULL)
    {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "malloc(%uz) failed", size);
    }

    ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, log, 0, "malloc: %p:%uz", p, size);

    return p;
}
```
## 从创建好的内存池中申请内存
定义于`ngx_palloc.c`
3个接口：`ngx_palloc`、`ngx_pnalloc`、`ngx_pcalloc`
1. `ngx_palloc`和`ngx_pnalloc`的区别在于在申请**小块内存**时，**前者考虑对齐**，后者不考虑对齐
2. `ngx_pcalloc`是调用`ngx_palloc`，之后的额外操作是**清零申请的区域**。
### `ngx_palloc(pool, size)`
如果用户申请小于等于pool头信息中max大小的内存，则按小块管理。
```c
void * ngx_palloc(ngx_pool_t *pool, size_t size)
{
#if !(NGX_DEBUG_PALLOC)
    if (size <= pool->max)
    {
        // 第3个参数是标志位，1表示考虑对齐，0表示不考虑对齐。
        return ngx_palloc_small(pool, size, 1);
    }
#endif
    return ngx_palloc_large(pool, size);
}
```

```c
void * ngx_pnalloc(ngx_pool_t *pool, size_t size)
{
#if !(NGX_DEBUG_PALLOC)
    if (size <= pool->max)
    {
        // 第3个参数是标志位，1表示考虑对齐，0表示不考虑对齐。
        return ngx_palloc_small(pool, size, 0);
    }
#endif
    return ngx_palloc_large(pool, size);
}
```

```c
void * ngx_pcalloc(ngx_pool_t *pool, size_t size)
{
    void *p;
    p = ngx_palloc(pool, size);
    if (p)
    {
        ngx_memzero(p, size);    
    }
    return p;
}
```
### `ngx_palloc_small(pool, size, align)`
第3个参数是标志位，1表示考虑对齐，0表示不考虑对齐。

找current，即从该内存池大块之中分配内存。

`ngx_align_ptr`，将`p->d.last`指向的可用数据地址按`NGX_ALIGNMENT`对齐到指定数（默认是32）的倍数，比如如果地址是8，则舍弃一部分内存，对齐到32。

```c
static ngx_inline void *
ngx_palloc_small(ngx_pool_t *pool, size_t size, ngx_uint_t align)
{
    u_char      *m;
    ngx_pool_t  *p;
    // 从 pool 头信息找到当前可用的内存池大块
    p = pool->current;

    do {
        // 当前数据块中可分配的内存的开始位置
        m = p->d.last;

        if (align)
        {
            m = ngx_align_ptr(m, NGX_ALIGNMENT);
        }
        // end 是当前数据块中可分配的内存的结束位置，与开始位置相减，如果大于等于用户申请的大小
        // 则 直接返回 m ，即开始位置，把 last 位置 下移 size，表示分配了 size 大小
        if ((size_t) (p->d.end - m) >= size)
        {
            p->d.last = m + size;

            return m;
        }
        // 如果 目前的数据块 大小 不够 size，则遍历找下一个数据块。
        p = p->d.next;

    } while (p);
    // 直到所有数据块都不够用，则
    return ngx_palloc_block(pool, size);
}
```
#### `ngx_align_ptr(p, a)`内存对齐
定义于`ngx_config.h`

如果没有定义`NGX_ALIGNMENT`，则默认是`sizeof(unsigned long)`，按照32位对齐。这是内存单元对齐。
>注意要和`ngx_palloc.h`中定义的`#define NGX_POOL_ALIGNMENT 16`区分。那是内存池对齐。

即向定义的对齐大小（32）向上取整32倍。

操作是：传入的指针地址，加上指针大小32（4字节，32位）减1，和32减1取反后，相与。
比如：传入的指针地址是8：`0000 1000`（前面补0），则加32减1：`0010 1000`
`0010 0000 - 1 = 0001 1111`取反`1110 0000`（前面补1）。`0010 1000`和`1110 0000`相与后：`0010 0000`，由原先的8，对齐到了32。
```c
#ifndef NGX_ALIGNMENT
#define NGX_ALIGNMENT   sizeof(unsigned long)    /* platform word */
#endif

#define ngx_align_ptr(p, a)                                                   \
    (u_char *) (((uintptr_t) (p) + ((uintptr_t) a - 1)) & ~((uintptr_t) a - 1))
```

## `ngx_palloc_block(pool, size)`再开辟一大块内存池
从pool的头块找到`end - pool`的大小，这是内存池每一个大块的大小（包含头信息的大小）

这是为了**再开辟一大块内存池**做准备。

`ngx_memalign`和第一次创建内存池一样，如果没有对齐要求，则普通malloc。


```c
static void *
ngx_palloc_block(ngx_pool_t *pool, size_t size)
{
    u_char      *m;
    size_t       psize;
    ngx_pool_t  *p, *new;
    // 一整个内存池大块（包含头信息）的大小。
    psize = (size_t) (pool->d.end - (u_char *) pool);

    m = ngx_memalign(NGX_POOL_ALIGNMENT, psize, pool->log);
    if (m == NULL)
    {
        return NULL;
    }

    new = (ngx_pool_t *) m;
    // psize是整个内存池块的大小。m是开始，则end指向这个新开辟的内存池块的末尾
    new->d.end = m + psize;
    // next暂时指向NULL。
    new->d.next = NULL;
    new->d.failed = 0;

    // 存放普通内存池块的头信息
    m += sizeof(ngx_pool_data_t);
    // 内存对齐 m ，存储实际数据的开始地址
    m = ngx_align_ptr(m, NGX_ALIGNMENT);
    // 分配出去 size 大小的内存，last（下一个空闲内存的开始地址）更新到新位置
    new->d.last = m + size;
    
    // 从整个内存池的current到后续一长串，给每一个内存池块的failed计数加1。
    // 如果发现当前这个内存池块的failed是4，则current指向这个failed次数过多内存池块的下一个内存池块。 
    for (p = pool->current; p->d.next; p = p->d.next)
    {
        if (p->d.failed++ > 4)
        {
            pool->current = p->d.next;
        }
    }
    // 退出for 循环后，p 更新到了 最后一个内存池块。同时，当前pool的current指向的是第一个failed不为4的内存池块
    // 尾插。链接起来新开辟的这个内存池块。
    p->d.next = new;

    return m;
}
```
## `ngx_palloc_large(pool, size)`大块内存分配管理
用户申请比`pool->max`大的内存时，按大块内存分配管理。
用`ngx_alloc`分配size大小的内存（和分配内存池块的方法一样）见[`ngx_memalign(POOL_ALIGNMENT, size, log)`或`ngx_alloc(size, log)`](#`ngx_memalign(POOL_ALIGNMENT,%20size,%20log)`或`ngx_alloc(size,%20log)`)。
遍历找large链的前5个，看是否有large的alloc为空的，直接让alloc指向一开始malloc得到的p。
如果连续找了5个，发现large的alloc都不为空，则跳出循环，不再找了。
这个for循环是为了快速在前5个large中，找到一个之前开辟的，但已经空闲了的大内存块头信息，其alloc管理的大内存块已经释放了，所以为NULL，直接让alloc指向一开始malloc得到的p即可。

如果没有进入循环，说明large一开始就是空的，内存池没有申请过大块内存；或者是找了5个发现large的alloc都不空：执行下面的操作（头插大内存块的头信息到pool的large链）：

记录大内存块（large管理）的 头信息`ngx_pool_large_t`，是按照`ngx_palloc_small`方法，存放到了内存池块的小块内存中。
`ngx_palloc_small`返回空说明系统内存不够用了，失败，释放一开始malloc得到的大块内存，返回NULL。

如果正常，则在这个大块内存头信息填写：
`alloc`，即这个大块内存的地址，为一开始malloc得到的地址。
`next`，指向pool的large链头，pool的large指向这个新大块内存的头信息。相当于头插！只不过插的是大内存块的头信息。
```c
static void *
ngx_palloc_large(ngx_pool_t *pool, size_t size)
{
    void              *p;
    ngx_uint_t         n;
    ngx_pool_large_t  *large;

    p = ngx_alloc(size, pool->log);
    if (p == NULL)
    {
        return NULL;
    }

    n = 0;

    for (large = pool->large; large; large = large->next)
    {
        if (large->alloc == NULL)
        {
            large->alloc = p;
            return p;
        }

        if (n++ > 3)
        {
            break;
        }
    }
    // 记录 大块 内存（large管理）的 头信息，存放到内存池块管理的小块内存中。
    // 返回空说明 系统 内存不够用了，失败。
    large = ngx_palloc_small(pool, sizeof(ngx_pool_large_t), 1);
    if (large == NULL)
    {
        ngx_free(p);
        return NULL;
    }

    large->alloc = p;
    large->next = pool->large;
    pool->large = large;

    return p;
}
```
图示：
![](../../images/SGI%20STL和Nginx内存池剖析/image-20250807115333694.png)

如图，大块内存的头信息，是按小块内存管理，分配到了内存池块中。

下面要提到的外部资源所绑定的清理的头信息，也像是大块内存头信息一样，按小块内存管理，分配到了内存池块中。
## `ngx_pool_cleanup_add`分配一个需要管理外部资源的数据（比如指针、fd）
按小块内存分配`ngx_pool_cleanup_t`，这是清理的头信息块。
有：handler、data、next。

之后才是分配size。

之后，像头插大内存块的头信息链表一样，头插清理类的头信息链表。
```c
ngx_pool_cleanup_t * ngx_pool_cleanup_add(ngx_pool_t *p, size_t size)
{
    ngx_pool_cleanup_t  *c;

    c = ngx_palloc(p, sizeof(ngx_pool_cleanup_t));
    if (c == NULL)
    {
        return NULL;
    }

    if (size)
    {
        c->data = ngx_palloc(p, size);
        if (c->data == NULL)
        {
            return NULL;
        }
    }
    else
    {
        c->data = NULL;
    }

    c->handler = NULL;
    c->next = p->cleanup;

    p->cleanup = c;

    ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, p->log, 0, "add cleanup: %p", c);

    return c;
}
```
### 通过返回的`ngx_pool_cleanup_t *`绑定清理回调函数
```c
struct Data
{
    char * p;
    // ...
};
struct Data *pData = ngx_alloc(512);
pData->p = (char*)malloc(128);
strcpy(pData->p, "hello world");

void my_release(void *p)
{
    free(p);
}

ngx_pool_cleanup_t *pclean = ngx_pool_cleanup_add(pool, sizeof(char*));
pclean->handler = &my_release;
pclean->data = pData->p;
```
## `ngx_pfree(pool, void *p)`大块内存释放
用于释放大块内存。先free，后把large头信息中的alloc置空。
```c
ngx_int_t
ngx_pfree(ngx_pool_t *pool, void *p)
{
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next)
    {
        if (p == l->alloc)
        {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "free: %p", l->alloc);

            ngx_free(l->alloc);
            l->alloc = NULL;

            return NGX_OK;
        }
    }
    return NGX_DECLINED;
}
```
## 小块内存释放：无
Nginx的小块内存一旦分配了之后，就无法精确地回收。

不像SGI STL一样，假设A、B、C三个小内存单元相邻，A、C空闲，A是freelist的第一个空闲块，A的next是C。现在释放B，则B头插到freelist，现在是B连着A连着C。这样可以精确地释放，并下次还能重新分配（可以发现，由于是头插到了freelist的第一个空闲区域，所以最后释放的最先分配）。

Nginx呢，每个内存池块只是由`last`和`end`两个指针管理，只能指示当前内存池块的未分配的部分。

当已分配的部分中，有 1 个小块内存要释放，无法精确管理。所以Nginx只能连续地释放一整段空间，与last相连。
## `ngx_reset_pool(pool)`
遍历large链表，释放每个大块内存。最后large置空。
遍历每个内存池块，把last拉到头信息的末尾即可，相当于释放了内存池的数据。最后current置为pool。
注意，只是释放了大块内存，所有内存池块都没有free，只是更新了last。
```c
void ngx_reset_pool(ngx_pool_t *pool)
{
    ngx_pool_t        *p;
    ngx_pool_large_t  *l;

    for (l = pool->large; l; l = l->next)
    {
        if (l->alloc)
        {
            ngx_free(l->alloc);
        }
    }
    
    for (p = pool; p; p = p->d.next)
    {
        if (p == pool) // 第一个内存池，有pool头信息
        {
            p->d.last = (u_char*)p + sizeof(ngx_pool_t);
        }
        else           // 其余的内存池，只有小头信息，没有pool大头
        {
            p->d.last = (u_char *) p + sizeof(ngx_pool_data_t);
        }
        p->d.failed = 0;
    }

    pool->current = pool;
    pool->chain = NULL;
    pool->large = NULL;
}
```
## `ngx_destroy_pool(pool)`
1. 遍历“清理”链表，每个都按照绑定的handler释放其data外部资源。
2. 遍历large链表，释放每个大块内存。
3. 遍历每个内存池块，释放每个内存池块。

```c
void
ngx_destroy_pool(ngx_pool_t *pool)
{
    ngx_pool_t          *p, *n;
    ngx_pool_large_t    *l;
    ngx_pool_cleanup_t  *c;

    for (c = pool->cleanup; c; c = c->next)
    {
        if (c->handler)
        {
            ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                           "run cleanup: %p", c);
            c->handler(c->data);
        }
    }
#if (NGX_DEBUG)
    /*
     * we could allocate the pool->log from this pool
     * so we cannot use this log while free()ing the pool
     */
    for (l = pool->large; l; l = l->next)
    {
        ngx_log_debug1(NGX_LOG_DEBUG_ALLOC, pool->log, 0, "free: %p", l->alloc);
    }
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next)
    {
        ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, pool->log, 0,
                       "free: %p, unused: %uz", p, p->d.end - p->d.last);
        if (n == NULL)
        {
            break;
        }
    }
#endif
    for (l = pool->large; l; l = l->next)
    {
        if (l->alloc)
        {
            ngx_free(l->alloc);
        }
    }
    for (p = pool, n = pool->d.next; /* void */; p = n, n = n->d.next)
    {
        ngx_free(p);
        if (n == NULL)
        {
            break;
        }
    }
}
```
# Nginx和STL释放内存的策略适用的场景
Nginx大块内存分配=》内存释放ngx_free函数
Nginx小块内存分配=》没有提供任何的内存释放函数。

实际上，从小块内存的分配方式来看（直接通过last指针偏移来分配内存），根本没法进行中间部分的小块内存的回收。

Nginx本质：HTTP服务器是一个短链接的服务器，客户端（浏览器）发起一个request请求，到达Nginx服务器以后，处理完成，Nginx给客户端返回一个response响应，HTTP服务器就主动断开tcp连接。
假设HTTP 1.1 keep-alive：60s，HTTP服务器（nginx）返回响应以后，需要等待60s，60s之内客户端又发来请求，重置这个时间；
否则60s之内没有客户端发来的响应，Nginx也是最终会主动断开连接，此时Nginx可以调用ngx_reset_pool重置内存池了，等待下一次客户端的请求。

因此，Nginx内存池的设计适用于间歇性、短连接的服务。虽然有内存泄漏，但效率高，空间换时间。

如果是长连接，且小块内存分配、释放较多，最好用STL二级空间配置器，避免内存泄漏过多。