---
title: Cpp_初始化列表
categories:
  - - Cpp
tags: 
date: 2024/3/20
updated: 
comments: 
published:
---
# 内容
1. 三种形式的initialization
2. 构造函数的初始化列表写法
# Cpp的initialization
有三种形式：
1. 赋值（assignment形式）
2. 卷括号（大括号，curly braces）
3. 函数式（圆括号）

```cpp
int main()
{
    // assignment initialization
    double a = 17.5;
    // braces initialization
    double b{ 17.5 };
    // functional initialization
    double c(17.5);
}
```

委员会推荐新出的大括号形式的初始化，最通用，相比于圆括号来说更少有歧义。
# 构造函数中的初始化列表

冒号后面的初始化列表，除了assignment形式，小括号、大括号都可以。现代C++更推荐大括号。

```cpp
class Test
{
public:
    Test(void) : _val{ 7 }
    {
        //_val = 7;
    }
private:
    int _val;
}
```

初始化列表的初始化是先于构造函数的，即给对象创建空间后立马填成员的值。
而构造函数中赋值，是对象创建完毕后才去执行。
## 初始化列表的语法糖

如果要初始化的东西是常量，则`C++11`有语法糖：直接在成员变量后加初始化括号。
```cpp
class Test
{
public:
    Test(void)
    {
        //_val = 7;
    }
private:
    int _val{ 7 };
};
```
