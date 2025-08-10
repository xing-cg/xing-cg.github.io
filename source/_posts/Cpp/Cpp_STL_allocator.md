---
title: Cpp_STL_allocator
categories:
  - - Cpp
    - STL
  - - 项目
    - 内存池
tags: 
date: 2022/3/7
updated: 
comments: 
published:
---

# 内容

1. 一级配置器和二级配置器的关系
2. 类型萃取
3. allocator
4. `__malloc_alloc_template`静态成员的类外初始化，其中函数指针尤为麻烦

# 空间配置器

SGI STL第一级配置器

```cpp
template<int inst>
class __malloc_alloc_template{ /*...*/ };
```

1. allocate()直接使用malloc()，deallocate()直接使用free()。
2. 模拟C++的set\_new\_handler() 以处理内存不足的情况。

SGI STL第二级配置器

```cpp
template<bool threads, int inst>
class __default_alloc_template{ /*...*/ };
```

1. 维护16个自由串列，负责16种小型区块的此配置能力。内存池以malloc配置而得。如果内存不足，转第一级配置器进行oom处理。
2. 如果需求区块大于128bayes，转第一级配置器（直接malloc）。

# 类型萃取

iterator_traits负责萃取迭代器的特性，而SGI的__type_traits把这种技法进一步扩大到迭代器以外的世界。

我们对类型关注的属性有：

1. has_trivial_default_constructor
2. has_trivial_copy_constructor
3. has_trivial_assignment_operator
4. has_trivial_destructor
5. is_POD_type

如果某种类型的构造、拷贝、赋值、析构函数是不必要的，那么我们在对这个类型进行以上动作时，就可以采用最原始化、最有效率的措施，如malloc、memcpy等。这就是对类型属性萃取的意义。

```c++
struct __true_type {};
struct __false_type {};
```

## 小试牛刀

```c++
#pragma once
namespace xcg
{

struct __true_type {};
struct __false_type {};

template<class T>
struct __type_traits
{
	typedef __true_type		this_dummy_member_must_be_first;
	typedef __false_type	has_trivial_default_constructor;
	typedef __false_type	has_trivial_copy_constructor;
	typedef __false_type	has_trivial_assignment_operator;
	typedef __false_type	has_trivial_destructor;
	typedef __false_type	is_POD_type;
};

template<>
struct __type_traits<char>
{
	typedef __true_type		has_trivial_default_constructor;
	typedef __true_type		has_trivial_copy_constructor;
	typedef __true_type		has_trivial_assignment_operator;
	typedef __true_type		has_trivial_destructor;
	typedef __true_type		is_POD_type;
};

template<>
struct __type_traits<int>
{
	typedef __true_type		has_trivial_default_constructor;
	typedef __true_type		has_trivial_copy_constructor;
	typedef __true_type		has_trivial_assignment_operator;
	typedef __true_type		has_trivial_destructor;
	typedef __true_type		is_POD_type;
};

template<class T>
struct __type_traits<T*>
{
	typedef __true_type		has_trivial_default_constructor;
	typedef __true_type		has_trivial_copy_constructor;
	typedef __true_type		has_trivial_assignment_operator;
	typedef __true_type		has_trivial_destructor;
	typedef __true_type		is_POD_type;
};

}
```

# allocator

## STL处理内存的三大部分

STL的allocator把分配空间、释放空间分工了。分配是alloc::allocate()负责，释放是alloc::deallocate()负责；对象构建是::construct()负责，对象析构是::delete()负责，new和delete往往是全局函数。

STL标准规格的配置器定义于\<memory\>中，SGI\<memory\>内含以下两个文件：

```c++
#include<stl_alloc.h>
#include<stl_construct.h>
```

实际上，STL除了以上两个重要部分，还有stl_uninitialized.h，是处理大块内存的。

这个切入点，就可以直接把话题导向内存池了，我们可以根据具体类型的构造、析构、赋值函数是否无关紧要而采取最有效率的措施。

## 空间的配置与释放std::alloc

对象构建前的空间配置、对象析构后的空间释放，两个动作都由<stl_alloc.h>负责。SGI对此的设计哲学如下：

1. 向system heap申请空间；
2. 考虑多线程状态；
3. 考虑内存不足时的应对措施；
4. 考虑内存碎片问题

对于第四个问题——内存碎片问题，SGI设计了两级配置器。第一级是对于足够大的内存块处理方式。第二级是对于太小的内存块处理方式。这个大小的标准以128bytes为界。

一级配置器：

1. 直接malloc
2. 直接free
3. 模拟c++的set_new_handler处理内存不足情况

二级配置器：

1. 维护16个自由链表(free lists)，内存池以malloc获取，内存不足则用一级配置器处理。
2. 如果需求区块大于128bytes，转调用一级配置器。

## 一级配置器的静态成员及其初始化

先以一个简单的例子说明静态成员的类外初始化。

```c++
class Object
{
private:
    static int num;
    static void (*pfun)();
};

int Object::num = 0;
//void (*pfun)() = nullptr;			//error
//要告诉编译器，pfun来自哪个类。
void (* Object::pfun)() = nullptr;	//correct
```

其中，template\<int inst\>代表“非类型”模板，模板中的参数只是用于标识的。

```c++
template<int inst>
class Object
{
private:
    static int num;
    static void (*pfun)();
};
template<int inst>
int Object<inst>::num = 0;
template<int inst>
void (* Object<inst>::pfun)() = nullptr;
```

### alloc类的静态成员

```c++
template<int inst>
class __malloc_alloc_template
{
private:
    static void * oom_malloc(size_t n);
    static void * oom_realloc(void * p, size_t new_sz);
    static void (*__malloc_alloc_oom_handler)();
public:
    static void * allocate(size_t n);
    static void deallocate(void * p, size_t n);
    static void * reallocate(void * p, size_t old_sz, size_t new_sz);
    //static ? set_malloc_handler(void (*f)());
    //函数名是set_malloc_handler，参数是一个void(*)()类型的变量，参数名字是f。
    //其欲返回函数指针void (*)()。
    //但是不能这样写：static void(*)() set_malloc_handler(void (*f)());
    //下面是正确写法
    static void(* set_malloc_handler(void(*f)()) ) ();
};
//此时的类外初始化
//__malloc_alloc_template类的私有成员__malloc_alloc_oom_handler
template<int inst>
void(* __malloc_alloc_template<inst>::__malloc_alloc_oom_handler)() = nullptr;
```

为了解决函数指针类型的可读性问题，使用typedef或using重定义类型名字。瞬间清爽了许多。

```c++
class __malloc_alloc_template
{
public:
    using PFUN = void(*)();
private:
    static PFUN __malloc_alloc_oom_handler;
public:
    static PFUN set_malloc_handler(PFUN p);
};
```

接下来进行类外初始化：

```c++
//__malloc_alloc_template类的私有成员__malloc_alloc_oom_handler
template<int inst>
typename __malloc_alloc_template<inst>::PFUN __malloc_alloc_template<inst>::__malloc_alloc_oom_handler = nullptr;
/*
别看名字过长，实际和之前的形式一样。
仿照以下方式写，成员类型+哪个类<T>::成员名 = ...
template<int inst>
int Object<inst>::num = 0;
/*
```

# 一级配置器函数实现

函数成员总览：

```c++
template<int inst>
class __malloc_alloc_template
{
private:
    static void * oom_malloc(size_t n);
    static void * oom_realloc(void * p, size_t new_sz);
    static void (*__malloc_alloc_oom_handler)();
public:
    static void * allocate(size_t n);
    static void deallocate(void * p, size_t n);
    static void * reallocate(void * p, size_t old_sz, size_t new_sz);
public:
    using PFUN = void(*)();
    static PFUN set_malloc_handler(PFUN p);
};
```

## oom_handler

oom_handler，指out of memory时的处理。

```c++
static PFUN set_malloc_handler(PFUN p)
{
    PFUN old = __malloc_alloc_oom_handler;
    __malloc_alloc_oom_handler = p;
    return old;
}
```

```c++
static void * oom_malloc(size_t size)
{
    void* result = NULL;
    void (*my_malloc_handler)() = nullptr;
    for(;;) //infinite loop
    {
        my_malloc_handler = __malloc_alloc_oom_handler;
    	if(nullptr == my_malloc_handler)
        {
            __THROW_BAD_ALLOC;
        }
        my_malloc_hanler();
        result = malloc(size);	//after handling, retry to malloc
        if(nullptr != result)
        {
            return result;
        }
    }
}
```

```c++
static void * oom_realloc(void * p, size_t new_sz)
{
    void* result = NULL;
    void (*my_malloc_handler)() = nullptr;
    for(;;) //infinite loop
    {
        my_malloc_handler = __malloc_alloc_oom_handler;
    	if(nullptr == my_malloc_handler)
        {
            __THROW_BAD_ALLOC;
        }
        my_malloc_hanler();
        result = realloc(p, new_sz);	//after handling, retry to malloc
        if(nullptr != result)
        {
            return result;
        }
    }
}
```

## allocate

```c++
static void* allocate(size_t size)
{
    void* result = malloc(size);
    if(nullptr == result)
    {
        result = oom_malloc(size);
    }
    return result;
}
```

## 对oom_handler的简要测试

```c++
#include"my_alloc.h"

char* chunk1 = (char*)malloc(sizeof(char)*10000000);
void handler1()
{
    if(chunk1!=nullptr)
    {
        free(chunk1);
    }
    chunk1 = nullptr;
    xcg::malloc_alloc::set_malloc_handler(nullptr);
}
int main()
{
    //设置hanler函数
    xcg::malloc_alloc::set_malloc_handler(handler1);
    //尝试申请100个int空间
    //在allocate函数中，可以通过调试器强制将result赋为nullptr则可进行oom处理测试。
    int * p = (int*)xcg::malloc_alloc::allocate(sizeof(int)*100);
    return 0;
}
```

## deallocate

```c++
static void deallocate(void * p, size_t n)
{
    free(p);
}
```

## reallocate

```c++
static void * reallocate(void * p, size_t old_sz, size_t new_sz)
{
    void * result = realloc(p, new_sz);
    if(nullptr == result)
    {
        result = oom_realloc(p, new_sz);
    }
    return result;
}
```

# 二级配置器

* 一级配置器的问题
  * 直接的malloc后的数据上下各有一个越界标记。上越界标记之外还有信息，包括申请了多少字节，还有next域、prev域以便连接到链表中进行内存管理。除此之外可能还有有效/失效(是否释放)的标记。
  * 除了包装信息占空间较多之外，还有malloc时花费的时间也多。这样就造成对于比较小的对象数据在malloc时入不敷出。
  * 结论就是：对于较小数据，适用于内存池来进行内存管理。

* **对于客户端申请较小空间（128bytes）的具体处理流程**：
  1. 每次配置一大块内存，并维护对应的自由链表(free-list)
  2. 下次如果再有相同大小的内存需求，直接从free-list中取出若干小块。
  3. 如果客户端释放小块内存，就由配置器回收到free-list中。所以配置器除了分配空间，还负责回收空间。

* 为了方便管理，SGI第二级配置器会主动按任何小块内存的内存需求量计算成8的倍数。（比如客户端要求30bytes，则计算为32bytes。

* 共维护16个free-lists，各自管理大小分别为8、16、24、32、40、48、56、64、72、80、88、96、104、112、120、128的小额区块。

* free-lists的单结点结构如下：

  ```cpp
  union obj
  {
      union obj * free_list_link;
      char client_data[1];	//the client sees this
  }
  ```

## 总览

```c++
enum { __ALIGN = 8};
enum { __MAX_BYTES = 128};
enum { __NFREELISTS = __MAX_BYTES / __ALIGN};	// 128 / 8 = 16
```

```c++
template<bool threads, int inst>	//参数1：是否考虑多线程，参数2：标识
class __default_alloc_template
{
private:
    union obj	/* 只是类型定义 */
    {
        union obj * free_list_link;	//next
        char client_data[1];
    };
private:
    static obj * volatile free_list[__NFREELISTS];//volatile修饰obj*，即free_list数组中对于的数据内容。
    static char* start_free;//每次配置的一大块'内存' 的内存来源的剩余部分的起始
    static char* end_free;	//每次配置的一大块'内存' 的内存来源的剩余部分的末尾
    static size_t heap_size;//上面提到的都是内存来源，而这个内存来源都是从堆区申请的，堆区申请的总空间数要记录在案
    static size_t ROUNT_UP(size_t bytes);
    static size_t FREELIST_INDEX(size_t bytes);	//hash
    static char* chunk_alloc(size_t size, int & nobjs);//size - 对象的大小，nobjs对象的个数
    static void* refill(size_t size);	//重新填充
public:	/* 用户调用 */
    static void* allocate(size_t size);
    static void deallocate(void* p, size_t n);
    static void* reallocate(void* p, size_t old_sz, size_t new_sz);
};
```

## 成员初始化

```c++
// 对free_list[__NFREELISTS]进行初始化。
template<bool threads, int inst> //	1: __default_alloc_template是模板类，需要声明
typename __default_alloc_template<threads, inst>::obj * volatile // 2: obj是__default_alloc_template<threads, inst>类内的类，需要加上“哪个类”，并且要加上"typename"以示"obj"是个类名，而非成员。
    __default_alloc_template<threads, inst>::free_list[__NFREELISTS] = {};	// 3: free_list[__NFREELISTS]是__default_alloc_template<threads, inst>类内的成员，需要加上“哪个类”，但无需加"typename"。
// 对start_free进行初始化
template<bool threads, int inst>
char* __default_alloc_template<threads, inst>::start_free = nullptr;
// 对end_free进行初始化
template<bool threads, int inst>
char* __default_alloc_template<threads, inst>::end_free = nullptr;
// 对heap_size进行初始化
template<bool threads, int inst>
size_t __default_alloc_template<threads, inst>::heap_size = 0;
```

## 函数实现

准备工作

```c++
using malloc_alloc = __malloc_alloc_template<0>;
```

### allocate

相当于单链表的头删法，返回删了的节点的地址给用户用。

```c++
static void * allocate(size_t size)
{
    if(size > (size_t)__MAX_BYTES)
    {
        return malloc_alloc::allocate(size);	//转给一级配置器。
    }
    obj * result = nullptr;
    obj * volatile * my_free_list = nullptr;	//是二级指针，要指向obj*
    my_free_list = free_list + FREELIST_INDEX(size);
    result = *my_free_list;
    if(nullptr == result)	//说明此下标处无内存块了，需要申请。
    {
        void * r = refill(ROUND_UP(size));
        return r;
    }
    *my_free_list = result->free_list_link;	//把free_list此下标的指针更新到下一个节点。相当于链表的头删法。
    return result;
}
```

### refill

```c++
static void * refill(size_t size)
{
    int nobjs = 20;
    char * chunk = chunk_alloc(size, nobjs);
    //chunk_alloc运行之后，nobjs值可能发生改变。
    if(1 == nobjs)
    {
        //如果nobjs是1，则说明内存块正好用完，不用再处理后续内存区域的链接、移动
        return chunk;
    }
    
    //接下来，把内存块分崩离析，依次串联。
    obj * volatile * my_free_list = NULL;
    obj * result = (obj*)chunk;
    obj * current_obj = NULL, * next_obj = NULL;
    my_free_list = free_list + FREELIST_INDEX(size);
    *my_free_list = next_obj = (obj*)(chunk+size);
    for(int i = 1; ; ++i)
    {
        current_obj = next_obj;
        next_obj = (obj*)((char*)next_obj + size);
        if(i == nobjs-1)//最后一块
        {
            current_obj->free_list_link = NULL;
            break;
        }
        current_obj->free_list_link = next_obj;
    }
    return result;
}
```

### chunk_alloc

```c++
static char* chunk_alloc(size_t size, int & nobjs)
{
    char * result = NULL;
    size_t total_bytes = size * nobjs;
    size_t bytes_left = end_free - start_free;
    if(bytes_left >= total_bytes)
    {
        result = start_free;
        start_free = start_free + total_bytes;
        return result;
    }
    else if(bytes_left >= size)
    {
        nobjs = bytes_left / size;
        total = size * nobjs;
        
        result = start_free;
        start_free = start_free + total_bytes;
        return result;
    }
    else //内存不足以分配1个对象，把剩余的插入到其他下标(剩8字节插到0, 16字节插到1, ...)，然后再另外申请一个大的
    {
        size_t bytes_to_get = 2 * total_bytes + ROUND_UP(heap_size >> 4);
        if(bytes_left > 0)
        {
            obj * volatile * my_free_list = free_list + FREELIST_INDEX(bytes_left);
            //头插
            ((obj*)start_free)->free_list_link = *my_free_list;
            *my_free_list = (obj*)start_free;
        }
        //剩余的残存空间 处理完毕，接下来申请大块内存
        start_free = (char*)malloc(bytes_to_get);
        if(NULL == start_free)	//堆区居然没有资源了！无奈之计，只能向比他之后的链表找有无剩余块。
        {
            obj * volatile * my_free_list = NULL;
            for(int i = size; i <= __MAX_BYTES; i += __ALIGN)	//注意，i从size开始，而不一定从最小的块大小8开始，因为找比你小的没有用处。得找大的。
            {
                my_free_list = free_list + FREELIST_INDEX(i);
                obj * p = NULL;
                if(NULL != p)	//找到了比他大的的剩余的了！
                {
                    *my_free_list = (*my_free_list)->free_list_link;
                    start_free = (char*)p;
                    end_free = start_free + i;
                    return chunk_alloc(size, nobjs);
                }
            }
            //如果上面的for循环没能使得其找到合适的内存块，则说明真的是“山穷水尽疑无路”了，只能转1级配置器，抛出异常或者通过handler处理。
            start_free = malloc_alloc::allocate(bytes_to_get);
        }
        end_free = start_free + bytes_to_get;
        heap_size += bytes_to_get;
        return chunk_alloc(size, nobjs);	//递归此函数，相当于重新来一遍流程
    }
}
```

### ROUND_UP & INDEX

```c++
static size_t ROUNT_UP(size_t bytes)
{
    // 0 -> 0, 1~8 -> 8, 9~16 - > 16, 17~24 -> 24, ...
    return (bytes+ __ALIGN-1) & ~(__ALIGN - 1);
}
static size_t FREELIST_INDEX(size_t bytes)
{
    // 1~8 -> 0, 9~16 -> 1, 17~24 -> 2, ...
    return ((bytes+ __ALIGN-1) / __ALIGN) - 1;
}
```

### 测试allocate

```c++
#ifdef __USE_MALLOC
typedef __malloc_alloc_template<0> malloc_alloc;
typedef malloc_alloc alloc;
#else
typedef __default_alloc_template<0, 0> alloc;
#endif
```

```c++
template<class T, class Alloc>
class simple_alloc
{
public:
    static T* allocate(size_t n)	//申请n个T对象
    {
        return Alloc::allocate(sizeof(T)*n);
    }
    static T* allocate()
    {
        return Alloc::allocate(sizeof(T));
    }
    void deallocate(T *p, size_t n)
    {
        if(NULL == p)return;
        Alloc::deallocate(p, sizeof(T)*n);
    }
    void deallocate(T* p)
    {
        if(NULL == p)return;
        Alloc::deallocate(p, sizeof(T));
    }
};
```

引入list进行测试

```c++
#include"my_list.h"
using namespace std;
int main()
{
    xcg::my_list<char> mylist;
    
    for(int i = 0; i < 23; ++i)
    {
        mylist.push_back(i + 'a');
    }
    return 0;
}
```

list构建流程：

```c++
template<class _Ty, class _A = xcg::alloc>
class my_list
{
public:
    typedef _Ty							 value_type;
    typedef _Ty&						 reference;
    typedef _Ty*						 pointer;
    typedef const _Ty&					 const_reference;
    typedef const _Ty*					 const_pointer;
    
    typedef _A							 allocator_type;
    typedef xcg::simple_alloc<_Node, _A> data_allocate;		//simple_alloc中使用的是第二个参数的allocate。在这里即使用_A::allocate，默认参数是xcg::alloc，而根据开关语句，若是使用malloc_alloc则为一级配置器，若为default_alloc则用二级配置器。我们默认使用的是__default_alloc_template<0, 0>。
    
public:
    my_list() : _Head(_Buynode()), _Size(0) {}
protected:
    _Nodeptr _Buynode(_Nodeptr _parg = NULL, _Nodeptr _narg = NULL)
    {
        _Nodeptr _S = data_allocate::allocate();
        _Acc::_Prev(_S) = _parg == NULL ? _S : _parg;
        _Acc::_Next(_S) = _narg == NULL ? _S : _narg;
        return _S;
    }
private:
    _Nodeptr _Head;
    size_t _Size;
};
```

### deallocate

```c++
static void deallocate(void * p, size_t size)	// n 表示空间大小
{
    if(size > (size_t)__MAX_BYTES)
    {
        malloc_alloc::deallocate(p, size);
        return;
    }
    //归还给配置器
    obj * q = (obj*)p;
    obj * volatile * my_free_list = free_list + FREELIST_INDEX(size);
    //头插
    q->free_list_link = *my_free_list;
    *my_free_list = q;
    return;
}
```

### 测试deallocate

```c++
iterator erase(iterator _P)
{
    _Nodeptr _S = _P++._Mynode();
    _Acc::_Next(_Acc::_Prev(_S)) = _Acc::_Next(_S);
    _Acc::_Prev(_Acc::_Next(_S)) = _Acc::_Prev(_S);
    destroy(&_Acc::_Value(_S));	//不删除节点，而是释放value域。
    _Freenode(_);
    
    return _P;
}
void _Freenode(_Nodeptr _P)
{
    data_allocate::deallocate(_P);
}
void pop_back()
{
    erase(--end());
}
void pop_front()
{
    erase(begin());
}
```

```c++
class Object
{
private:
    int value;
public:
    Object(int x = 0) : value(x)
    {
        cout << "create object" << this << endl;
    }
    ~Object()
    {
        cout << "destroy object" << this << endl;
    }
};
int main()
{
    xcg::my_list<Object> objlist;
    for(int i = 0; i < 10; ++i)
    {
        objlist.push_back(Object(i));
    }
    objlist.pop_back();
    return 0;
}
```

### reallocate

```c++
static void* reallocate(void * p, size_t old_sz, size_t new_sz)
{
    if(old_sz > (size_t)__MAX_BYTES && new_sz > (size_t)__MAX_BYTES)
    {
        return malloc_alloc::reallocate(p, old_sz, new_sz);
    }
    // old_sz == 20, new_sz == 22	-->24, 24
    if(ROUND_UP(old_sz) == ROUND_UP(new_sz))
    {
        return p;
    }
    // old_sz > 128, new_sz < 128
    // old_sz < 128, new_sz < 128
    // old_sz < 128, new_sz > 128
    size_t sz = old_sz < new_sz ? old_sz : new_sz;
    void * s = allocate(new_sz);
    memmove(s, p ,sz);
    deallocate(p, old, sz);
    return s;
}
```

