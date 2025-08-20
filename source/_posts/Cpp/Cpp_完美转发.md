---
title: Cpp_完美转发
categories:
  - - Cpp
    - 右值引用
  - - Cpp
    - Modern
tags: 
date: 2022/2/25
updated: 
comments: 
published:
---

# 内容

1. 为何需要完美转发
2. 完美转发

# 为何需要完美转发

```c++
template<class T>
void fun(T a)
{
    // ...
}
// 由于T类型有可能是内置类型(C子集)，也有可能是封装类型(C++子集)，所以对于模板来说，实现c子集和class子集上的统一的背后工作必定复杂。
fun<int>(12);
fun<Object>(obj);
```

```c++
#include<iostream>
using namespace std;
void fun(int& a)
{
	cout << "fun(int& a)" << endl;
}
void fun(const int& a)//const int&是个万能引用，左右值都可以接收，但是如果fun有int &&的重载则优先被int &&接收（如右值10）；同样的道理，如果fun有int &的重载则优先被int &接收（如左值a）。
{
	cout << "fun(const int& a)" << endl;
}
void fun(int&& a)
{
	cout << "fun(int&& a)" << endl;
}
//void fun(int a){} //与fun(int &)冲突
int main()
{
	int a = 10;
    const int b = 20;
	fun(a);		//fun(int& a)
	fun(b);		//fun(const int& a)
	fun(10);	//fun(int&& a) --> 虽然10也可以被const int &接收，但是C++11开始后，重载对值的类型、左右值进行了清晰的划分。以前只按类型重载，之后按左右值也可以重载。
}
```

# 完美转发

```c++
void fun(int& a)
{
	cout << "fun(int& a)" << endl;
}
void fun(const int& a)
{
	cout << "fun(const int& a)" << endl;
}
void fun(int&& a)
{
	cout << "fun(int&& a)" << endl;
}
template<class T>
void fac(T&& tmp)
{
    //cout << std::is_rvalue_reference<T> << endl;
    //cout << std::is_lvalue_reference<T> << endl;
    fun(tmp);
    // 虽然在调用fac时，tmp传入此函数中前是一个右值，但是传参动作完成后tmp成为了一个具名变量，则又退化为了左值，导致只能调用fun(int& a)。
    //所以我们需要一个机制，来保持变量的左右值属性。即完美转发
    fun(std::forward<T>(tmp)); //依然保持tmp的右值属性，使其可以调用fun(int&& a)。
}
int main()
{
    int a = 10;
    const int b = 20;
    fac(a);
    fac(b);
    fac(20);
    return 0;
}
/* 运行结果
    fun(int& a)
    fun(int& a)
    fun(const int& a)
    fun(const int& a)
    fun(int& a)			-->***完美转发之前
    fun(int&& a)		-->***完美转发之后
*/
```

