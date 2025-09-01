---
title: Cpp_奇异递归模板模式（CRTP）
categories:
  - - Cpp
    - Modern
  - - Cpp
    - 模板
tags: 
date: 2025/9/1
updated: 
comments: 
published:
---
# 1
```cpp
template <typename Derived>
class Base
{
public:
    void show(void)
    {
        static_cast<Derived*>(this)->show();
    }
protected:
    void show(void)
    {
        std::wcout << L"Base::show()" << std::endl;
    }
};

class Client : public Base<Client>
{
public:

};
int main()
{
    Client client;
    client.show();
    return 0;
}
```

输出：
```
Base::show()
```
# 2
```cpp
template <typename Derived>
class Base
{
public:
    void show(void)
    {
        static_cast<Derived*>(this)->show();
    }
protected:
    void show(void)
    {
        std::wcout << L"Base::show()" << std::endl;
    }
};

class Client : public Base<Client>
{
public:
    void show(void) // hidden
    {
        std::wcout << L"Client::show()" << std::endl;
    }
};
int main()
{
    Client client;
    client.show();
    return 0;
}
```

输出：
```
Client::show()
```
# 静态多态
```cpp
#include <iostream>

// 基类模板，依赖派生类提供 impl()
template<class Derived>
struct Algo {
    void run() {
        // 共享的流程骨架
        pre();
        static_cast<Derived*>(this)->impl(); // 派生类定制点
        post();
    }
    void pre()  { /* 公共准备逻辑 */ }
    void post() { /* 公共收尾逻辑 */ }
};

// 派生类 A：提供自己的实现
struct AlgoA : Algo<AlgoA> {
    void impl() { std::cout << "AlgoA fast path\n"; }
};

// 派生类 B：另一种实现
struct AlgoB : Algo<AlgoB> {
    void impl() { std::cout << "AlgoB precise path\n"; }
};

int main() {
    AlgoA a; a.run();   // 无虚表，编译期静态解析，易于内联
    AlgoB b; b.run();
}

```
# 实际场景
WTL、ATL（活动模板库，90年代的，用于COM）