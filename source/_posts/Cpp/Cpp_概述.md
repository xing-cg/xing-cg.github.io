---
title: Cpp_概述
categories:
  - - Cpp
tags: 
date: 2024/3/20
updated: 
comments: 
published:
---
# 内容
1. Cpp是个什么语言，组成部分。
# Cpp组成部分
根据`legacy.cplusplus.com`手册中Reference一栏。标准C++库包含以下部分（此网站2017年停止更新）：
1. C库
2. 容器
3. 输入输出流
4. 原子和线程库（C++11之后）
5. 杂项（Miscellaneous headers）
    1. 算法库（algorithm）
    2. 时间库（chrono）
    3. 异常库（exception）
    4. 函数对象（functional）
    5. 迭代器（iterator）
    6. 内存管理（memory）：尤其是智能指针
    7. 随机（Random）
        1. 梅森旋转算法，使用的是分布律。随机性能很好。
    8. 工具（utility）：用的最多的是pair
# 名字空间
这个不是C++的专利，所有的面向对象语言，都有名字空间的概念。

以前在C语言中，随意定义全局变量可能会产生污染。此时只有“全局空间”一说。

现在在C++中，可以自定义一个名字空间，在其中定义自己的东西。防止与他人冲突。
```cpp
namespace xcg
{
class Person
{
private:
    int m_age;
public:
    int age(void) const
    {
        return m_age;
    }
    void age(int age)
    {
        m_age = age;
    }
};
}
namespace other
{
class Person
{
private:
    int m_age;
public:
    int age(void) const
    {
        return m_age;
    }
    void age(int age)
    {
        m_age = age;
    }
};
}
int main()
{
    xcg::Person person;
}
```
## 名字空间可以嵌套
第一种写法：
```cpp
namespace xcg::alpha
{
    // ...
}
```
第二种写法：
```cpp
namespace xcg
{
    namespace beta
    {
        // ...
    }
}
```