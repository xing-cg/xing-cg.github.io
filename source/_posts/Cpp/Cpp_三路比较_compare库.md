---
title: Cpp_三路比较_compare库
categories:
  - - Cpp
tags: 
date: 2024/4/10
updated: 2024/4/10
comments: 
published:
---
# `<=>`三路比较运算符

`C++20`标准中，有`<compare>`库。
1. `strong_ordering`
    1. less
    2. greater
    3. equal：相等
    4. equivalent：等价
2. `weak_ordering`
    1. less
    2. greater
    3. equivalent：等价，或者说模糊的相等
3. `partial_ordering`
    1. less
    2. greater
    3. equivalent
    4. unordered
# strong_ordering
```cpp
#include<compare>
int main()
{
    int a = 3;
    int b = 4;
    auto c = a <=> b; //c的类型：std::strong_ordering
    if(c < 0)
        std::cout << "less" << std::endl;
    else if(c > 0)
        std::cout << "greater" << std::endl;
    else if(c == 0)
        std::cout << "equal" << std::endl;
    return 0;
}
```
# partial_ordering
`partial_ordering`测试如下，主要测试`unordered`的情况。用一个NaN浮点数比较时，就会出现。
>NaN定义于limits库

```cpp
#include<limits>
int main()
{
    double a = 4.0;
    double b = std::numeric_limits<double>::quiet_NaN(); //得出double类型的NaN
    auto c = a <=> b; //c的类型：std::partial_ordering
    if(c < 0)
        std::cout << "less" << std::endl;
    else if(c > 0)
        std::cout << "greater" << std::endl;
    else if(c == 0)
        std::cout << "equivalent" << std::endl;
    else
        std::cout << "unordered" << std::endl;
    return 0;
}
```
# 类中怎么使用`<=>`

对于字符串来说，我们返回一个strong或weak都可以。
strong要求更严格一些，比如区分字母大小写等等，
而weak要求则松一些，比如不管大小写，都算相等。

1. `<=>`的默认行为是**按照类中的成员顺序依次比较**。
2. 如果不是按照默认顺序依次比较，则需要自定义函数逻辑

以下是默认的例子
```cpp
class Point
{
public:
    Point(int x, int y) : _x{ x }, _y{ y }
    {
    }
    std::strong_ordering operator <=> (Point const & pt) const = default;
    
private:
    int _x;
    int _y;
};
int main()
{
    Point pt{1, 2}, pt2{1, 2};
    if(pt == pt2)
    {
        std::cout << "equal" << std::endl;
    }
}
```
## 实际的例子：见《Cpp_string仿写》