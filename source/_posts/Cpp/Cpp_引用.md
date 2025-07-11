---
typora-root-url: ../..
title: Cpp_引用
categories:
  - - Cpp
tags: 
date: 2024/3/27
updated: 
comments: 
published:
---
# 引用

引用的作用是给变量起一个别名。
# 引用的地址常性

```cpp
int main()
{
    int a2 = 9;
    int a = 10;
    int& b = a;
    b = a2;     // a
}
```

在定义引用时，必须指定一个被引用者（地址）。一旦定义成功，在其生命期中，就不可更换地址了。这是引用的地址常性。

相当于`int & b = a`在本质上是`int * const b = &a`。

但是这与引用的该地址上的内存数据的常性不冲突。如果需要赋予数据常性，则仍需修饰const：`int const & b = a`。此时，就无法通过引用b去修改内存上的数据值了。