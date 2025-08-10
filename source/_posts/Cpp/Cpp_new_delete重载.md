---
title: Cpp_new_delete重载
categories:
  - - Cpp
tags: 
date: 2021/10/28
updated: 2024/5/26
comments: 
published:
---
# 内容

# new的动作

new是关键字。其实是new operator，本身不可重载。

1. 申请内存
    1. new第一步的申请内存调用的是operator new(可看作一个运算符函数)，可以重载。
2. 调用构造
3. 返回地址
# delete的动作

delete是关键字。其实是delete operator，本身不可重载。

1. 调用析构
2. 内存释放
    1. delete内存释放调用的是operator delete(可看作一个运算符函数)，可以重载。
# operator new重载
```cpp
void* operator new(size_t size)
{
    void* p = malloc(size);
    cout << "调用了自定义operator new" << endl;
    return p;
}
```

有何作用？

堆上内存：属于是外部资源。

申请/释放外部资源，用户态没有权限访问，内核态才有权限。外部资源很多，比如我们的鼠标、键盘、屏幕。

我们申请/释放资源的过程需要内核态完成，此时，需要从用户态切换到内核态，再做**系统调用**。

平时接触的系统调用有很多，比如：cout、printf、cin、scanf

应运而生，会出现内存池的概念。`C++`自带的内存池：`_Alloc`

内存池是：用户态自己维护的**一大段**内存。

重载new和delete相当于留下了一个可修改的端口，可以**自定义申请、释放内存的动作**。这就是重载new和delete的意义所在。
比如，可以自己编写malloc把内存分配到显存中。除了在改变分配内存的不同设备，还可以在这里面进行内存分配动作的统计、标记等，常用于游戏开发、内存池。

但是要知道，`C++`只允许把分配内存的任务交给程序员自定义，没有开放在重载new中调用构造函数的权限。
# operator delete重载

```cpp
void operator delete(void* p)
{
    cout << "调用了自定义operator delete" << endl;
    free(p);
}
```
# 
```cpp
#include<iostream>
using namespace std;
class Tmp
{
public:
	Tmp()
	{
		cout << "Tmp()" << endl;
	}
	~Tmp()
	{
		cout << "~Tmp()" << endl;
	}
	void* operator new(size_t size)
	{
		void* p = malloc(size);
		cout << "调用了operator new" << endl;
		return p;
	}
	void operator delete(void* p)
	{
		cout << "调用了operator delete" << endl;
		free(p);
	}
};
int main()
{
	Tmp* p = new Tmp();

}
```
# 作业

1. mstring三种：普通的、写时拷贝的、迭代器的
2. 刷题
# 区别

1. new是关键字。malloc是函数；
```c++
int * p = new int(10);
int * s = (int*)malloc(sizeof(int));
```
2. new不需要类型转换，malloc需要强转
3. new自动计算类型大小，malloc需要sizeof
4. new先malloc，再构造，再返回指针。malloc只申请内存
5. 申请空间失败时
    1. new抛出`std::bad_malloc`
    2. malloc返回NULL
6. new可以申请一组，需要`new[]`,`delete[]`。而malloc必须手动指明字节数，free不知道释放的是不是一组。
7. 对于内置类型、POD类型，new/malloc可以混用；对于自定义类型能否混用？主要看类型中有没有析构函数，如果有，在`new[]`时会记录申请了多少对象，于是在`delete[]`时可以自动溯源。
8. new是运算符，可以进行重载；malloc则不行
9. new有三种调用方式；malloc没有
10. new可以设置`nothrow`不抛异常，返回NULL
11. `set_new_handler`