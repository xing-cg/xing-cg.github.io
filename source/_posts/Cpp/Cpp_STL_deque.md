---
title: Cpp_STL_deque
typora-root-url: ../..
categories:
  - [Cpp, STL]
tags:
  - null 
date: 2022/2/28
update:
comments:
published:
---

# 内容

1. deque

# 构造

```c++
#include<iostream>
#include<vector>
using namespace std;
int main()
{
	vector<int> ar1;	//空vector
	vector<int> ar2 = { 12,23,34,45,56,67,78,89 }; //类似数组的初始化方式
	vector<int> ar3(10, 23); //存10个23
	vector<int> ar4({ 12,23,34,45,56,67 }); //初始化列表作为参数构造
	vector<int> ar5(ar2); //拷贝构造，按ar2的有效size
	vector<int> ar6(std::move(ar2)); //移动构造
}
```

# emplace和push

首先来看隐式转换的概念：

```c++
class Object
{
    int val;
public:
    Object(int val) : val(val) { }
    
};
int main()
{
    vector<Object> objvec;
    objvec.push_back(20); //隐式转换20，创建临时对象Object(20)。实际上是：emplace_back(Object(20))
    objvec.emplace_back(20); //在vec.end()处直接定位new Object(20)
    
}
```

# vector与智能指针结合

```c++
int main()
{
    std::vector<Shape*> shape;
    string name;
    while(cin >> name, name != "end")
    {
        if(name == "Circle")
        {
            shape.push_back(new Circle());
        }
        else if(name == "Square")
        {
            shape.push_back(new Square());
        }
        else
        {
            cout << "input class name error: " << name << endl;
        }
    }
    for(auto & x : shape)
    {
        x->draw();
        x->erase();
    }
}
```

