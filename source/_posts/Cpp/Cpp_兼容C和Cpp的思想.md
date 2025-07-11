---
title: Cpp_兼容C和Cpp的思想
categories:
  - - Cpp
tags:
  - tulun
date: 2022/2/26
update: 2025/3/8
comments: 
published:
---
# 内容

重点是模板的思想，使得Cpp在包罗万象时如虎添翼：模板中的类型，把要用到的类型信息萃取出来给模板函数，而原本的类型信息不丢失。使得在有型和无型之间游离。这达到了很多的效果，包括把c子集和class子集中基本数据类型和自定义类型的鸿沟给填补了。
# 模板是C范式和Class范式的桥梁
1. C子集（面向过程）
2. Class子集（面向对象）
3. STL
4. 模板--可以跨越C和Class子集。
# 模板跨越c和class

先小试牛刀，初次感受。

```c++
template<class T>
void fun(T a)
{
    
}
fun<int>(12);
fun<Object>(obj);
// 奇妙之处就是--使用了模板后，你并不清楚T是内置类型还是封装类型，所以要完成统一内置类型和封装类型的操作必定很麻烦。模板的强大背后都是心酸。
```

在谈下一个知识“完美转发”之前，我们先来一段C++11之后，左值右值的重载情况：

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

结合下面的情况，我们为了在fac中调用其他函数使用到tmp时保持其右值属性，使用到完美转发

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
void fac(T && tmp)
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
    fac(a);-----------------
            fun(int& a)
            fun(int& a)
    fac(b);-----------------
            fun(const int& a)
            fun(const int& a)
    fac(20);-----------------
            fun(int& a)			-->***完美转发之前
            fun(int&& a)		-->***完美转发之后
*/
```

有一个很疑惑的问题：既然`fac(a);` `fac(b);`编译通过了，为什么完美转发后运行的结果还是`fun(int& a)`、`fun(const int& a)`？

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
void test(int&& a)			//--与上一个例子最大的区别：不是模板函数!
{
    fun(a);
    fun(std::forward<int>(a));
}
int main()
{
    int a = 10;
    const int b = 20;
    //test(a);				//参数是左值，是无法调用test(int&& a)的，因为编译器规定:无法将‘右值参数’--‘绑定到左值’。
    //test(std::move(b));	//-->编译器提示：将"int &&”类型的引用绑定到"std:remove_reference_t<const int &>”类型的初始值设定项时，限定符被丢弃   --->意思是，警告我们不要这样做。这样会丢失b的const属性。
    
    test(std::move(a));		//但是我们知道，在平时的书写中，int a = 10; int b = a;这种写法是很普遍的，也就是说，虽然右值不能放在左值的位置上，但是左值是可以充当右值使用的，只不过是编译器规定了我们不能在调用函数时与把左值和右值混淆。
    						//因为不能直接test(a);说明问题所在，所以我们std::move(a)				
}
/* 运行结果
    fun(int& a)
    fun(int&& a)
我们可以看到，在完美转发前，因为到了fun函数中，a又成为了有名变量，所以再次调用fun(a)退化为左值；而完美转发后，可以保持a的右值性。
*/
```

这个结果，与上一个代码中的结果（完美转发后运行的结果依旧是`fun(int& a)`、`fun(const int& a)`）一经对比，发现了问题：就是调用`fac(a);` `fac(b);`之前，由于`fac`是一个模板函数，而且参数类型很特殊！这是最关键的点，参数类型是`T && tmp`，这就要求，不管T是什么类型，不管要传入的实参是否是右值属性，我们都可以让其成功进入到此中，甚至他是一个左值（左值是可以充当右值使用的）。而模板机制中的T是会保留传入实参类型的所有信息的，包括const属性和左右值属性，比如`int a = 10;`的a是非常性左值，`const int b = 20;`的b是常性左值。所以，看上去tmp是一个右值，但是tmp的根源属性是外部a、b原来的属性。所以完美转发后的结果依旧是a的非常性左值`int &`，b的常性左值`const int &`。

通过这个例子，可以得出，模板函数的`\<class T\> void fun(T && tmp)`并未丢弃T元素原本的的const信息、左右值属性。

那么我们来小试牛刀一下：

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
void fac_template(T && tmp)
{
    fun(tmp);
    fun(std::forward<T>(tmp));
}
void fac_non_template(T && tmp)
{
    fun(tmp);
    fun(std::forward<T>(tmp));
}
int main()
{
    int a = 10;
    fac_template(std::move(a));			//***注意此处区别，上上个代码中是fac(a),此处是fac(std::move(a))
    
    fac_non_template(std::move(a));
    return 0;
}
/* 运行结果
	----------------
	fac_template(std::move(a));
	----------------
        fun(int& a)
        fun(int&& a)	//上上个代码fac(a)的此处的结果是fun(int& a)，而此处是fun(int&& a)
	----------------
    fac_non_template(std::move(a));
    ----------------
        fun(int& a)
        fun(int&& a)
*/
```
# 在cpp文件中调用c文件

矛盾点：

1. C++调用C
    1. C++产生函数符号-（函数名+参数类型列表），C语言产生函数符号-（函数名）
2. C语言调用C++
    1. 如果将C++的函数符号改为C语言的函数符号--需要改动C++源文件--不现实。
    2. 正确解决办法是添加自己实现的C++文件，写**C++函数作为中间层去调用需要的C++函数，然后让自实现的C++函数产生C语言符号（extern C）**。

```c
#func.c

#include<stdio.h>
void fun_C()
{
    printf("void fun_C()");
}
#main.cpp
void fun_C();
//cpp生成的符号的方式是取决于函数名和参数列表。
//而c生成的符号只是函数名
//所以导致符号找不到。
int main()
{
    fun_C();
    return 0;
}
//如何解决——使用C语言的方式编译和生成符号。
extern "C"
{
    void fun_C();//声明
}
```

反过来呢？如何解决？

```c++
#fun.cpp
void fun_CPP()//生成的符号：fun_CPP_void
{
    printf("void fun_CPP()");
}

#func.c
void fun_CPP();//生成的符号：fun_CPP
int main()
{
    fun_CPP();
    return 0;
}

```



```c++
//C调用C++的代码方法
#fun.cpp
void fun_CPP()//生成的符号：fun_CPP_void
{
    printf("void fun_CPP()");
}

#tmp.cpp
void fun_CPP();
extern "C"
{
    void fun_CPP_tmp()
    {
        fun_CPP();
    }
}

#func.c
void fun_CPP_tmp();//生成的符号：fun_CPP
int main()
{
    fun_CPP_tmp();
    return 0;
}

```

## extern关键字

两种用法：

1. C++中，干预编译器，`extern "C"`是以C方式编译，`extern "C++"`是以C++方式编译；
2. C语言中，告诉编译器，函数是外部函数，既可以用在本文件，也可用在其他文件。