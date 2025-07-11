---
title: Cpp_继承和多态
typora-root-url: ../..
categories:
  - [Cpp]
tags:
  - null 
date: 2021/11/9
update:
comments:
published:
---

# 内容

# 引入

类与类之间的关系：

组合/嵌套：一个类中声明了另一个类，是属于的关系。一个类是另一个类的一部分。

代理：一个类的接口是另一个类接口的子集

继承：fish--goldFish

# 隐藏

子类会隐藏父类的同名成员。----同名隐藏

访问被隐藏的成员需要加上（父类）作用域。

# 动多态

产生：使用指针或者引用调用虚函数，就会产生动多态调用

动多态调用的过程：

1. 使用指针或者引用调用虚函数
2. 在对象中找到vfptr
3. 根据vfptr找到vftable
4. 在vftable中找到要调用的函数
5. 调用

vftable

编译时期如果发现类中有虚函数，就会对这个类生成vftable（虚函数表）。将当前

# 面试

1. 父子类/组合类的构造顺序
   1. 先构造父类，再构造子类。
   2. 先构造内部类，再构造大类。
2. 类的编译顺序：
   1. 编译类名
   2. 编译成员名=====编译嵌套类
   3. 编译成员方法体
3. 什么是多态
   1. 面向对象方法学中，派生类继承基类后表现出来的形式叫多态。这个形式为在基类的行为基础上又派生出各自派生类的特性
4. 动多态的产生条件
   1. 父类有虚函数
5. 动多态的过程
   1. 子类继承父类时，拷贝父类虚函数表，覆盖虚函数表中的同名方法。
   2. 在调用子类方法时，如果发现其是虚函数，则由虚函数表指针找到虚函数表，查出函数指针，调用继承或重写的方法。
6. vftable什么时候产生？在哪里存储？
   1. 在构造子类前，构造父类时产生。在内存的只读数据段中存储。
7. 构造函数能不能写成虚函数
   1. 不能，因为调用虚函数需要去构造好的对象中找虚函数表指针。但是他还没构造出来。
8. 静态函数能不能写成虚函数
   1. 静态函数属于类的函数，理论上可以被定义为虚函数并继承。
9. 析构函数能不能写成虚函数
   1. 可以，并且在运用多态时，常用。
10. 虚函数能不能被处理为内联
    1. 不能。内联是在编译期代码段展开，而动多态是在运行时。
11. 什么情况下析构函数必须写成虚函数
    1. 父类指针指向子类对象
12. 父类指针能不能指向子类对象？子类指针能不能指向父类对象？
    1. 前者可以，后者不可。
13. 什么是RTTI？在什么时候产生？存储在哪里？
14. 父类指针如何转化为子类指针？转化有什么条件？
15. 什么是菱形继承？菱形继承有什么问题？如何解决？
16. 纯虚函数有什么作用？
17. 隐藏
18. 覆盖

## 答案

1. 
   1. 静多态
      1. 编译时期的多态，又被称为早绑定
      2. 代表：函数重载、模板
   2. 动多态
      1. 运行时期的多态，又被称为晚绑定
      2. 代表：继承中的多态

2. 
   1. 编译类名
   2. 编译成员名=====编译嵌套类
   3. 编译成员方法体

3. 
4. 
   1. 指针或引用调用虚函数 + 对象必须完整
   2. 完整对象：构造函数执行完毕，虚构函数还没开始

5. 
   1. 使用指针或者引用调用虚函数
   2. 在对象中找到vfptr
   3. 找到vftable
   4. 在表中找到对应的函数

6. 
   1. 编译期产生
   2. 放在rodata区

7. 
   1. 不能，构造函数无法通过指针或者引用调用，写成虚函数没有意义。
   2. vfptr是在构造时才写入对象，而调用虚构造函数时没有对象构造出来。

8. 不能，因为静态函数不依赖于对象。

10. 
    1. 不能。已内联的函数在编译期代码段展开，在release版本没有地址。

11. 父类指针指向堆上的子类对象时，必须把父类的虚构函数写为虚函数。

13. 
    1. runtime type info，是一个指向类型信息的指针。
    2. 编译期产生，RTTI指针放在vftable里面，类型信息放在rodata段。

14. ```cpp
    Derive* pd = dynamic_cast<Derive*>(p);
    dynamic_cast : 父类指针转为子类指针专用的类型强转
    要求：
        1.必须有RTTI
        2.父类指针指向的对象中的RTTI确实是子类类型。
    ```

15. 

17. 子类中的成员会隐藏父类中同名的成员。
18. 子类中的成员方法会覆盖父类中相同（同返回值、参数列表（因为函数指针的类型要相同））的虚函数。

# 菱形继承

1. 菱形继承引入的问题
   1. 造成公共父类在子对象中存在多个实例
2. 菱形继承的解决
   1. 虚继承
3. PowerShell中命令查看类的构造
   1. `cl /d1 report Single Class Layout D(要查看结构的类名) ./main.cpp > 1.txt`

# 虚继承

1. 虚继承的逻辑
   1. 被虚继承的类会变成虚基类
   2. 虚基类在子类对象中存放在vbtable中
   3. 原本应该存储该父类对象的位置上替换为vbptr
   4. vbptr指向vbtable中虚基类实例的位置，从而保证虚基类在子类中只会有一个实例的存在
   5. 注意：虚基类在子类构造时候会被当做直接父类进行构造（子类的初始化列表需要加虚基类的初始化）

# 抽象类

1. 有纯虚函数的类叫抽象类
2. 

# 隐藏和重写

“隐藏”是比“重写”更重要的法则。“重写”虚函数需要函数同名，必定要遵守“隐藏”法则，这不意味着我们不能调用被隐藏的基类函数，“虚”的意义正是要通过另一种工具——虚表指针来找出隐藏的基类函数。

```c++
#include<iostream>
using namespace std;
class Object
{
	int value;
public:
	Object(int x = 0) : value(x) {}
	virtual void add() { cout << "O:add" << endl; }
	virtual void fun() { cout << "O:fun" << endl; }
};
class Base : public Object
{
private:
	int num;
public:
	Base(int x = 0) : num(x + 10) {}
	virtual void add() { cout << "B:add" << endl; }
	virtual void fun(int x) { cout << "B:fun(int)" << endl; }
};
int main()
{
	Base base(10);
/*	'op', 'bp' actually point to a 'Base' instance. 
	There are 'B:add()', 'O:fun()', 'B:(int)' in this instance's vft*/
	Object* op = &base;
	Base* bp = &base;
	op->add();		//O:add()
	op->fun();		//O:fun()
/*	op->fun(133);*/	//complie error! 'op' is a 'Object' poniter, 'Object' class not has fun(int)

	bp->add();		//B:add()	->	overide the O:add()
/*	bp->fun();*/	//compile error! 'bp' is a 'Base' poniter, though it's vft has the O:fun(), but O:fun() is hidden by B:fun(int)
	bp->fun(233);	//B:fun(int)->	hide	the O:fun()
    
	//compile error! 'op' stated a 'Base' poniter, though it's vft has the O:fun(), but O:fun() is hidden by B:fun(int)
/*	dynamic_cast<Base*>(op)->fun();	*/
    
    //ok! 'op' is a 'Object' poniter, it's vft not has the 'B:fun(int)'
	dynamic_cast<Base*>(op)->fun(1);
    
    
    //ok! Here explicitly stated 'bp' a 'Object' pointer, now compiler feels 'O:fun()' instead of being hidden.
	dynamic_cast<Object*>(bp)->fun();
    
    //compile error! 'bp' stated a 'Object' pointer, now compiler think the 'bp' not has the 'fun(int)' function.
/*	dynamic_cast<Object*>(bp)->fun(2);*/

}
```

