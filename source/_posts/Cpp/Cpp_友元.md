---
title: Cpp_友元
categories:
  - - Cpp
tags: 
date: 2022/1/20
updated: 
comments: 
published:
---
# 特点
1. 不具有自反性
2. 不具有传递性
3. 不具有继承性
# 三种形式
1. 外部函数友元
2. 成员函数友元
3. 类友元
## 外部函数友元

## 成员函数友元

```c++
class Object
{
private:
    int value;
public:
    Object(int x):value(x)
    {
        
    }
};
class Base
{
private:
    int sum;
public:
    Base(int x = 0):sum(x)
    {
        
    }
    void fun(Object& obj)
    {
        obj.value = obj.value + sum;//error
    }
};
```

```c++
class Object;//在Base前要声明
class Base
{
private:
    int sum;
public:
    Base(int x = 0):sum(x)
    {
        
    }
    void fun(Object& obj);
};
class Object
{
private:
    int value;
public:
    Object(int x):value(x)
    {
        
    }
    friend
};
void Base::fun(Object& obj)
{
    obj.value = obj.value + sum;//error
}
int main()
{
    Base base(10);
    Object obja(20);
    base.fun(obja);
    return 0;
}
```



