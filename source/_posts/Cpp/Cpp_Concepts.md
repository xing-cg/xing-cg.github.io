---
title: Cpp_Concepts
categories:
  - - Cpp
    - Modern
  - - Cpp
    - 模板
tags: 
date: 2024/4/24
updated: 2024/4/24
comments: 
published:
---
# Concepts
意为概念，用于限定类型是哪些子类型。
# Concepts中的requires约束

```cpp
#include<iostream>
#include<concepts>
template <typename T>
T add<T a, T b>
{
    return a + b;
}

int main()
{
    auto c = add(1, 2);
    return 0;
}
```
以上，会自动推断出1、2为int，返回值3亦为int。
目前有需求，仅仅想要add进行int的计算，比如禁止浮点型的计算。则可用requires约束
```cpp
#include<iostream>
#include<concepts>
template <typename T> requires(std::integral<T>)
T add(T a, T b)
{
    return a + b;
}

int main()
{
    auto c = add(1, 2);
    return 0;
}
```
如果main函数调用add时传入浮点型：则编译不通过
> 报错：
> no instance of function template "add" matches the argument list
> argument types are: (double, double)

```cpp
int main()
{
    auto c = add(1.0, 2.0);   // error
    return 0;
}
```
如果想要接受浮点型数据，可以在requires中加入`||`
```cpp
template <typename T> requires(std::integral<T> || std::floating_point<T>)
T add(T a, T b)
{
    return a + b;
}

int main()
{
    auto c = add(1.0, 2.0);     // ok
    return 0;
}
```
# `type_traits`

Type Traits就是在编译期把各种情况列举出来，根据不同的种类去判断T是不是属于某种情况。然后就可以用在concept中。

`std::is_integral`

Member types:

| member type  | definition                         |
| ------------ | ---------------------------------- |
| `value_type` | bool                               |
| type         | either `true_type` or `false_type` |

Member constants:

| member constant | definition               |
| --------------- | ------------------------ |
| value           | either `true` or `false` |

`std::is_integral`后面往往要加`<T>::value`以取出真假值，或者可以用`std::is_integral_v<T>`代替。
```cpp
// concepts 文件 中 std::integral<T>的原型
_EXPORT_STD template <class _Ty>
concept integral = is_integral_v<_Ty>;
```
有好多种写法：
1. `std::integral<T>`
2. `std::is_integral<T>::value`
3. `std::is_integral_v<T>`
# 自定义concept

针对某种类型进行约束：
```cpp
template<typename T>
concept my_concept = std::integral<T> || std::floating_point<T>;
```
如此，requires后面使用约束条件就更加方便了：
```cpp
template<typename T> requires(my_concept<T>)
T add(T a, T b)
{
    return a + b;
}
```

# 更复杂的concept

requires不仅可以直接使用concept，也可以用于定义新的concept。
如：`template<typename T> concept my_concept = requires(T t) { // ... }`
或`template<typename T> concept my_concept = requires(bool参数)`
第一种是使用一些类型参数，大括号内会去匹配你规定的形式，这些形式需要遵循官方规定的写法。
## 限制 类必须有 某个限定函数

如下定义，表达的是：t是否有名为`get_v`、无参数的方法，并且返回的是不是`bool`类型。
```cpp
template<typename T>
concept my_concept = requires(T t)
{
    { t.get_v() } -> std::same_as<bool>;
}; // 记得加;
```
使用如下：
```cpp
class Test
{
public:
    bool get_v(void)
    {
        return true;
    }
};
template<typename T> requires(my_concept<T>)
void show(T t)
{
    auto r = t.get_v();
    std::cout << r << std::endl;
}
int main()
{
    Test test;
    show(test);
}
```
如果把Test类中的`get_v`方法改名为`get_v2`，让show函数调用`t.get_v2`。最后编译不通过。因为模板参数T对应的Test类中没有名为`get_v`、无参数且返回值为bool的方法。
> 报错：
> no instance of function template "show" matches the argument list
> argument types are: (Test)

```cpp
template<typename T>
concept my_concept = requires(T t)
{
    { t.get_v() } -> std::same_as<bool>;
};
class Test
{
public:
    bool get_v2(void)
    {
        return true;
    }
};
template<typename T> requires(my_concept<T>)
void show(T t)
{
    auto r = t.get_v2();  // 不报错，但main函数中的show(test)报错
    std::cout << r << std::endl;
}
int main()
{
    Test test;
    show(test);    // error
}
```
### 对成员方法的const修饰符忽略
`bool get_v(void) const`虽然加了const，但可以通过
```cpp
template<typename T>
concept my_concept = requires(T t)
{
    { t.get_v() } -> std::same_as<bool>;
};
class Test
{
public:
    bool get_v(void) const // 虽然加了const 但可以通过
    {
        return true;
    }
};
template<typename T> requires(my_concept<T>)
void show(T t)
{
    auto r = t.get_v();
    std::cout << r << std::endl;
}
int main()
{
    Test test;
    show(test);   // ok
}
```
## 复合
1. T类型有名为`get_v`、无参数的方法，并且返回`bool`类型。
    1. 必须有`{ } -> type`的形式
2. T类型有名为read的方法，且参数必须是u的类型`std::string const &`。没有规定返回值类型
3. T类型必须有名为val的成员变量
    1. 发现如果定义val为private（甚至static private）是不可以通过的，必须是可以从外部访问的。

```cpp
template <typename T>
concept my_concept = requires(T t, std::string const& u)
{
    { t.get_v() } -> std::same_as<bool>;
    t.read(u);
    T::val;  // 或者写成t.val;
};

class Test
{
public:
    bool get_v(void) const // 虽然加了const 但可以通过
    {
        return true;
    }
    void read(std::string const& str) const
    {
    }
public:
    int val{ 5 }; // ok
/*
private:
    static int val;  //error
*/
};
```
## 嵌套requires
可以用来限定某一个成员变量的类型

requires的内容是：T类型中要有val，而且val的类型要和float一样。
```cpp
template <typename T>
concept my_concept = requires(T t)
{
    requires std::same_as<decltype(t.val), float>;
};
```
# 应用
## 通过concepts约束迭代器类型

在没有concept之前，编程时乱用不匹配的迭代器编译时是不知道对错的，运行的时候才报错。
而Modern `C++`之后随着模板和concept的发展，可以约束迭代器的行为。比如规定此迭代器类必须支持`++`、`--`操作，从而此迭代器是Bidirectional。
有了concepts，在编译期就能知道程序的对错了。
## 通过concepts约束谓词类
1. 首先编写针对Unary Predicate的concept
2. 再加到`find_if`这个模板函数后限定：
    1. “模板参数1 UnaryPredicate”符合`UnaryPredicateConcept`中T的要求。（T有`t(*u)`且返回bool的方法）
3. 注意，`t(*u)`中u前必须有`*`，不然`t()`的参数将被限定为InputIterator。那么`bool operator () (int const& v)`由于参数是int将无法通过。

```cpp
template <typename T, typename U>
concept UnaryPredicateConcept = requires(T t, U u)
{
    { t(*u) } -> std::same_as<bool>;
};

template <class InputIterator, class UnaryPredicate>
    requires(UnaryPredicateConcept
        <UnaryPredicate, InputIterator>)
InputIterator find_if(InputIterator first, InputIterator last, UnaryPredicate pred)
{
    while (first != last)
    {
        if (pred(*first))
            return first;
        else
            ++first;
    }
    return last;
}

class IsOdd
{
public:
    bool operator () (int const& v)
    {
        return v % 2 != 0;
    }
};

int main()
{
    std::vector<int> vec{ 0, 1, 2, 3, 4, 5, 6, 7, 8 };
    IsOdd is_odd_functor;
    // 此find_if目前受UnaryPredicateConcept的制约。
    auto it = ::find_if(vec.begin(), vec.end(), is_odd_functor);
    while (it != vec.end())
    {
        std::cout << *it << std::endl;
        it = ::find_if(it + 1, vec.end(), is_odd_functor);
    }
}
```
此`find_if`目前受UnaryPredicateConcept的制约。
现在，`find_if`的参数3必须是：“包含`t(*u)`且返回bool的函数”的类型。
## 升级模板库

把一个算法升级为通用算法，放到库中支持所有类型。
首先升级为模板函数。再用concept对模板类型做限制，从而在编译期就可以做相关检查。