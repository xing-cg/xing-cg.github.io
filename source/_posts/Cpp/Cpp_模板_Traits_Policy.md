---
title: Cpp_模板_Traits_Policy
categories:
  - - Cpp
    - 模板
tags: 
date: 2024/4/20
updated: 2025/8/22
comments: 
published:
---
# 模板是为了什么？

> 利用了模板的元编程，可以像面向对象那样复用、组合可用的代码（即也可以把一段代码作为组件使用），但是这不是Cpp的本意，实际上是Cpp的“副作用”，只是貌似是一种优势。
> 利用了模板的元编程的更有价值的作用在于：可以编译期执行、计算。

模板是为了编译期运算，编译期运算就可以达到零开销的效果。

传统的运行时运算的程序的性能主要依赖于客户端的计算机性能。而编译期运算的性能则主要依赖开发侧的计算机性能。`C++`的设计目标就在于`Zero Overhead`，即零开销，意思是尽量把工作都在编译期间完成。

主要体现在模板、元编程上，指导编译器生成代码

比如，现在有一个需求，是编写add函数，返回两个操作数的加和值。

刚开始可能只想到了两个整型。
```cpp
#include<iostream>
int add(int a, int b)
{
    return a + b;
}
int main()
{
    int c = add(1, 2);
}
```
但是问题在于，后期可能要加上float型的情况。那我们可以通过Cpp的函数重载功能进行解决。
```cpp
float add(float a, float b)
{
    return a + b;
}
```
由此可以看出，需要模板的地方在于：
1. 需要函数重载
2. 虽然是函数重载，但是这些函数的结构相似，具体的操作、行为都一样。
3. 区别仅在于操作的对象的数据类型不同。

在这种情况下，可以不用反复重载，而是利用模板，让编译器为我们自动生成。当然，编译器不会一下子全部把所有情况都生成，而是我们当时的代码具体用哪个，就在编译期特别地生成哪个。这就叫做模板编程。
# 模板编程

以下叫做函数模板。不能叫做函数。这个函数模板也不是编译单元，即不是可编译的代码。
```cpp
template<typename T>
T add(T a, T b)
{
    return a + b;
}
```
而是从`add<float>`开始到之后才是编译单元。总之：模板不会生成代码，模板不能编译，而是在使用模板的时候，指导编译器生成相应的函数代码，才有了可编译的代码。
```cpp
int main()
{
    float c = add<float>(1.0f, 2.0f);
}
```

# 模板特化
Specialization
## 全特化
Full Specialization

如果模板中的所有类型T都被具体类型替代了，那么`< >`中就不用写`typename T`了，空着。叫做全特化。

```cpp
template < >
int add(int a, int b)
{
    return a + b;
}
```
### 灵活利用特化

```cpp
template<typename T, typename R>
R add(T a, T b)
{
    return static_cast<R>(a + b);
}
```
以下语句是无法编译通过的，因为调用语句只说明了函数参数的类型，而函数返回类型系统是无法推断的。
```cpp
int main()
{
    float f = add(1, 2);
}
```
如果以下这样写呢？也不行，因为模板中类型的顺序问题，导致float对应的是第一个类型T，因此系统还是未知函数返回类型。
```cpp
int main()
{
    float f = add<float>(1, 2);
}
```
只能全部写出：`add<int, float>(1, 2);`

或者：使用类似于后位缺省的写法习惯，来解决这个问题。即把T、R换个顺序，那么在调用add时，就可以略去T了。
```cpp
template<typename R, typename T>
R add(T a, T b)
{
    return static_cast<R>(a + b);
}
```
以下调用，走的是`template<typename R, typename T>`的add。会生成：`float add(int, int)`。
```cpp
int main()
{
    float f = add<float>(1, 2);
}
```
如果：配合上全特化模板函数。如以下main函数调用`add<int>`，则系统就判断出我们调用的是`int add(int, int)`了。总之，只要R和T全都对应上了特化的模板函数的所有类型，就会走模板特化函数。
```cpp
template<typename R, typename T>
R add(T a, T b)
{
    return static_cast<R>(a + b);
}
template < >
int add(int a, int b)
{
    return a + b;
}
int main()
{
    int c = add<int>(1, 2); //int add(int a, int b)
}
```
### 简便的写法

模板类型的后面的参数是可以有默认类型的。
如果还是T、R的顺序，如下写：就是在说：如果不指定模板第二个类型参数，就会默认返回类型为int。
```cpp
template<typename T, typename R = int>
R add(T a, T b)
{
    return static_cast<R>(a + b);
}
template < >
int add(int a, int b)
{
    return a + b;
}
```
而我们模板第一个类型参数T又是可以通过函数参数推断的，那么就可以全部省略：
```cpp
int main()
{
    int c = add(1, 2); //int add(int a, int b)
}
```

来看看以下的情况：此时走的是`R add(T a, T b)`，是`int add(double, double)`
```cpp
int main()
{
    int c = add(1.0, 2.0);
}
```
而如果这样：就会编译不通过，因为无法推断T的类型。
```cpp
int main()
{
    int c = add(1.0, 2.0f);
}
```
那么就需要明确在add后加`< >`指出，T是什么。如果加的是double，那么`2.0f`会转为double型进行计算。
```cpp
int main()
{
    int c = add<double>(1.0, 2.0f);
}
```
## 部分特化（实际上是函数模板的重载）

```cpp
template<typename T, typename R = int>
R add(T a, T b)
{
    return static_cast<R>(a + b);
}
template<typename R = float>
R add(long long a, long long b)
{
    return a + b;
}
template < >
int add(int a, int b)
{
    return a + b;
}
int main()
{
    auto c = add(1ll, 2ll);
}
```
以上，add函数优先和特化的模板函数匹配，而不是和`R add(T a, T b)`匹配，因为，参数`1ll`和`2ll`与特化模板函数中的long long对应上了，即`R add(long long a, long long b)`，所以最后生成的函数是`float add(long long a, long long b)`。

实际上，`template<typename R = float>`，这个形式，本质上是一种函数模板的重载。因为，给出了参数在某些特别类型下，函数的重定义，体现了多态。总之：同一个名字，不同的形式，都叫overload。
### 再举一个例子_1

```cpp
template<typename R = float>
R add(long long a, long long* pb)
{
    return a + *pb;
}
```

```cpp
int main()
{
    auto b = 2ll;
    auto c = add(1ll, &b);
}
```
走的是`R add(long long a, long long* pb)`。这也是一个function template overload。
### 再举一个例子_2

```cpp
template<typename R = float>
R add(long long a, long long b)
{
    return a + b;
}

template<typename R = float>
R add(long long a, long long& b)
{
    return a + b;
}
```

```cpp
int main()
{
    auto b = 2ll;
    auto c = add(1ll, &b);
}
```
此时，如果没有用到`add(long long, long long&)`，是可以正常编译的。
但是，如果一旦用到了：
```cpp
int main()
{
    auto b = 2ll;
    auto c = add(1ll, b);
}
```
就会报错：
```
'add': ambiguous call to overloaded function

more than one instance of overloaded function "add" matches the argument list:
    function template "R add(long long a, long long b)"
    function template "R add(long long a, long long &b)"
```
# 类模板

```cpp
template <typename R, typename T>
class Addition
{
public:
    R add(T a, T b) const noexcept
    {
        return a + b;
    }
}
```

```cpp
int main()
{
    Addition<int, int> addition;
    auto c = addition.add(1, 2);
}
```
1. `Addition<int, int>`是对类模板的实例化，产生了类。
2. `Addition<int, int> addition;`是对类的实例化，产生了对象。
3. `addition.add(1, 2);`调用类模板函数。生成了代码。
## 简便的写法

能不能省一个模板参数，写出构造函数？
```cpp
template<typename R, typename T = int>
class Addition
{
public:
    Addition(void)
    {
    }
    Addition(T a, T b) : _a{ a }, _b{ b }
    {
        
    }
    R add(T a, T b) const noexcept
    {
        return a + b;
    }
    R add(void) const noexcept
    {
        return _a + _b;
    }
private:
    T _a;
    T _b;
};
```

```cpp
int main()
{
    Addition addition(1, 2);     // C++14标准无法编译通过
    auto c = addition.add(1, 2); 
}
```

## 类的部分特化（偏特化）
Class Template Partial Specialization

```cpp
template<typename R, typename T>
class Addition
{
public:
    R add(T a, T b) const noexcept
    {
        return a + b;
    }
};
```
类的部分特化，定义时，要在类名后写尖括号，写入模板参数。
```cpp
template <typename T>
class Addition<T, int>
{
public:
    int add(T a, T b) const noexcept
    {
        return a + b;
    }
};
```

```cpp
int main()
{
    Addition<int, int> addition;
    auto c = addition.add(1, 2); 
}
```
类的实例化走的是`class Addition<T, int>`。
## 类的全特化
Class Template Full Specialization

```cpp
template < >
class Addition<int, int>
{
public:
    int add(int a, int b) const noexcept
    {
        return a + b;
    }
};
```

```cpp
int main()
{
    Addition<int, int> addition;
    auto c = addition.add(1, 2); 
}
```
类的实例化走的是`class Addition<int, int>`。
### 再来个例子

```cpp
template < >
class Addition<int, int*>
{
public:
    int add(int a, int* b) const noexcept
    {
        return a + *b;
    }
};
```

```cpp
int main()
{
    auto b = 2;
    Addition<int, int*> addition;
    auto c = addition.add(1, &b);
}
```
类的实例化走的是`class Addition<int, int*>`。
# Traits

Traits是特质、特性的意思，主要用来区分不同类型。在Modern Cpp中，有`<traits>`库，可以用来分析各种类型。
如果要对某种类型单独做特别的处理时，就会用到Traits，这在meta progrmming中是一种设计模式。
## AddTraits

比如拿Addition加和函数举例：整型有整型的策略，浮点型有浮点型的策略。更具体地，整型中也有不同的整型：int有int的策略，long有long的策略……
1. int数和另一个int数相加，可能会溢出，这时就需要转为long数加和并返回。
2. long数和另一个long数相加，可能会溢出，这时就需要转为long long数加和并返回。
## 在没有用到Traits时的解决方案
如果只是通过两个模板参数来解决这个问题，可以采用如下方案：
```cpp
template<typename R, typename T>
R add(T a, T b)
{
    return a + b;
}
```
但是，这个方案不够自动化。每当处理不同的类型，都需要时刻调整模板参数：
```cpp
int main()
{
    int a = 10, b = 20;
    long c = add<long, int>(a, b);
    long d = 2000;
    long long e = add<long long, long>(c, d);
}
```
## 用到Traits，让调用更爽
Traits就是为了解决这个问题。通过提前约束不同类型的行为，从而让调用更简便。这让模板函数在使用上更自动化了。

要写这样的Addition模板群，就要先声明一个主模板：
T代表操作数类型。而操作数的返回值类型通过T对应的具体的class得出（本例中为R）。
这个主模板可以不实现，因为这个主模板是一个抽象的定义。后期才会定义具体的、特化的类模板。
```cpp
template <typename T>
class AddTraits;
```
## 通过类模板的特化实现AddTraits

Traits是利用特化来实现的。通过类模板的特化，来区分不同类型的加和。

拿unsigned short的"加和"类模板举例：
```cpp
template < >
class AddTraits<unsigned short>
{
public:
    typedef unsigned int R;
};
```
以上，在unsigned short类型的加和下，规定了目标类型R（即加和的返回类型）为unsigned int。
这个R，要在模板类外部得到其实例可以如下操作：
```cpp
int main()
{
    // AddTraits<unsigned short>::R 前面需要加一个typename，不然R可能会被认作是静态变量名字。
    typename AddTraits<unsigned short>::R r = 9;
}
```
更多地：
**unsigned int**的加和类模板，规定了目标类型R（即加和的返回类型）为unsigned long long。
```cpp
template < >
class AddTraits<unsigned int>
{
public:
    typedef unsigned long long R;
};
```
更多地：
**unsigned long**的加和类模板，规定了目标类型R（即加和的返回类型）为unsigned long long。
```cpp
template < >
class AddTraits<unsigned long>
{
public:
    typedef unsigned long long R;
};
```
## 用AddTraits类模板编写Add函数模板

我们的需求、目标就是，已知两个T类型的数，加和，返回T加和后特定、自定、规定的更大包容的类型。那么这个更大包容的类型就是每一个特化类模板中的R，现在可以统一写为：`typename AddTraits<T>::R`。

>为了便于书写，可以重命名`AddTraits<T>::R`为R。

```cpp
template<typename T>
typename AddTraits<T>::R add(T a, T b)
{
    typedef AddTraits<T>::R R;
    return static_cast<R>(a) + static_cast<R>(b);
}
 ```

> 以上函数形式也可以如下写。即在模板参数中就通过缺省值的形式指明R是什么的别名，就可以用在返回值类型的简化了。

```cpp
template <typename T, typename R = AddTraits<T>::R>
R add(T a, T b)
{
    return static_cast<R>(a) + static_cast<R>(b);
}
```
测试：
```cpp
int main()
{
    unsigned short a = 1u;
    unsigned short b = 2u;
    // 调用的是unsigned int add<unsigned short>(unsigned short a, unsigned short b)
    auto c1 = add(a, b);
    // 调用的是unsigned long long add<unsigned int>(unsigned int a, unsigned int b)
    auto c2 = add(1u, 2u);
    // 调用的是unsigned long long add<unsigned long>(unsigned long a, unsigned long b)
    auto c3 = add(1ul, 2ul);
}
```
# Policy

上面谈到的Traits是关于类型的封装。

而Policy——策略，是关于行为的封装。比如把加法、减法、乘法、除法都封装成一样的行为，就是Policy。再如，日志系统，有的要写到文件中，有的则要写到服务器中，或者直接控制台输出。
## 封装AddPolicy

比如要把加法封装为Policy，就是要封装上面的`R add(T a, T b)`：
> AddPolicy即是一个具体的OperatePolicy，那么，OperatePolicy都将有一个calculate方法。

以加法Policy来说，它的特征就是：
1. 有两个操作数a、b，类型为T。
2. 有一个计算的指令，指令名可以都叫做calculate，作为函数名。函数名中则是具体的计算行为，加法Policy中则是`a + b`。
3. 会返回一个值，类型为R。

> 我们只是利用类的外壳，实际有用的是静态方法。
> 再利用Traits，通过具体T指明R将返回什么。`R = AddTraits<T>::R`

```cpp
// T是操作数类型，R是返回类型
template <typename T, typename R = AddTraits<T>::R>
class AddPolicy
{
public:
    static R calculate(T a, T b)
    {
        return static_cast<R>(a) + static_cast<R>(b);
    }
};
```
## 封装MultiplyPolicy

现在我们要编写第二个具体的Policy，即乘法Policy。因为加法和乘法最后都表现出同样的特质，所以乘法Traits可以复用AddTraits。
```cpp
template <typename T, typename R = AddTraits<T>::R>
class MultiplyPolicy
{
public:
    static R calculate(T a, T b)
    {
        return static_cast<R>(a) * static_cast<R>(b);
    }
};
```
# 封装 Traits 和 Policy 为 Operate 函数模板

设计一个函数模板，把数据特性 Traits 和行为抽象 Policy 封装。

其中，T 是原始数据类型，U 是一个 Policy，如 AddPolicy。
## 封装AddOperate

首先可以尝试封装一个具体的 Policy 如 AddPolicy 。
这个函数返回 AddPolicy 的 calculate 的计算结果，即返回数据类型是`AddTraits<T>::R`。
```cpp
// T是操作数类型，U是Policy
template<typename T, typename U = AddPolicy<T> >
typename AddTraits<T>::R AddOperate(T a, T b)
{
    return U::calculate(a, b);
}
```

```cpp
int main()
{
    std::cout << AddOperate(1, 2) << std::endl; // 3
}
```
### 优化

AddOperate 函数的返回类型书写太冗长，考虑可以用个简化的别名。可以利用`AddTraits<T>::R`在 AddPolicy 中存在、使用这个特点，则可以在 AddPolicy 中另起模板参数R的别名为RTNTYPE（除了R，其他名字都行）。
好处在于：类中另起的 RTNTYPE 和模板参数的 R 相比，前者可以在类外部直接使用，而后者不可以。
```cpp
// T是操作数类型，R是返回类型
template <typename T, typename R = AddTraits<T>::R>
class AddPolicy
{
public:
    using RTNTYPE = R;
    static R calculate(T a, T b)
    {
        return static_cast<R>(a) + static_cast<R>(b);
    }
};
```

> T为操作数类型；U为Policy；R为返回值类型。

```cpp
template <typename T, U = AddPolicy<T> >
U::RTNTYPE AddOperate(T a, T b)
{
    return U::calculate(a, b);
}
```

```cpp
int main()
{
    unsigned short a = 7u;
    unsigned short b = 3u;
    auto c = AddOperate(a, b);
    std::cout << c << std::endl;
}
```
## 封装抽象Operate

抽象的Operate是真正的可以传入任意的Policy参数的。

```cpp
template <typename U, typename T>
U::RTNTYPE Operate(T a, T b)
{
    return U::calculate(a, b);
}
```
但是，如果这样写的话，得给AddPolicy后面加具体的操作数类型才能编译通过。
```cpp
int main()
{
    unsigned short a = 3u;
    unsigned short b = 7u;
    auto c = Operate<AddPolicy<decltype(a)> >(a, b);
    std::cout << c << std::endl;
}
```
有没有什么办法能不传入`decltype(a)`就能进行的呢？那样的话，就可以非常地简洁：`Operate<AddPolicy>(a, b)`。
方法就是把Operate中的U参数指明为类模板。
```cpp
template <typename T, template<typename, typename> class Policy>
Policy<T, Policy::RTNTYPE>::RTNTYPE Operate(T a, T b)
{
    return Policy<T, Policy::RTNTYPE>::calculate(a, b);
}
```
但是以上代码肯定编译不过，因为出现了无限递归解析：Policy不是一个具体类，因此无法通过Policy指明具体RTNTYPE，于是就得加第三个模板参数Traits。

> 注意，`Policy<T, typename Traits::R>`中的`Traits::R`前面应该加typename。以明确区分传入的是类型而不是常量值（因为模板参数可以传入常量值）

```cpp
template <template<typename, typename> class Policy, typename Traits, typename T>
Traits::R Operate(T a, T b)
{
    return Policy<T, typename Traits::R>::calculate(a, b);
}
```

```cpp
int main()
{
    unsigned short a = 3u;
    unsigned short b = 7u;
    auto c = Operate<AddPolicy, AddTraits<decltype(a)>>(a, b);
    std::cout << c << std::endl;
}
```

这样的话，还是得在AddTraits后加一个`decltype(a)`才行，能不能彻底消灭呢？类比指明Policy是个类模板的经验，把Traits也指明为一个类模板，即可：
```cpp
template <template<typename, typename> class Policy,
            template<typename> class Traits,
            typename T>
Traits<T>::R Operate(T a, T b)
{
    return Policy<T, typename Traits<T>::R>::calculate(a, b);
}
```

```cpp
int main()
{
    unsigned short a = 3u;
    unsigned short b = 7u;
    auto c = Operate<AddPolicy, AddTraits>(a, b);
    std::cout << c << std::endl;                  // 10
    c = Operate<MultiplyPolicy, AddTraits>(a, b);
    std::cout << c << std::endl;                  // 21
}
```
如此，终于把Operate的调用变得简洁、美观了！这个调用形式也体现了Policy和Traits合二为一、相辅相成的美感。
# 最终代码

```cpp
#include<iostream>

template <typename T>
class AddTraits;

template < >
class AddTraits<unsigned short>
{
public:
    typedef unsigned int R;
};

template < >
class AddTraits<unsigned int>
{
public:
    typedef unsigned long long R;
};

template < >
class AddTraits<unsigned long>
{
public:
    typedef unsigned long long R;
};

template <typename T, typename R = typename AddTraits<T>::R>
class AddPolicy
{
public:
    using RTNTYPE = R;
    static R calculate(T a, T b)
    {
        return static_cast<R>(a) + static_cast<R>(b);
    }
};

template <typename T, typename R = typename AddTraits<T>::R>
class MultiplyPolicy
{
public:
    using RTNTYPE = R;
    static R calculate(T a, T b)
    {
        return static_cast<R>(a) * static_cast<R>(b);
    }
};

template <template<typename, typename> class Policy, template<typename> class Traits, typename T>
typename Traits<T>::R Operate(T a, T b)
{
    return Policy<T, typename Traits<T>::R>::calculate(a, b);
}

int main()
{
    unsigned short a = 3u;
    unsigned short b = 7u;
    auto c = Operate<AddPolicy, AddTraits>(a, b);
    std::cout << c << std::endl;
    c = Operate<MultiplyPolicy, AddTraits>(a, b);
    std::cout << c << std::endl;
}
```
# 更加简化
升级AddTraits为`OperationTraits`
```cpp
#include<iostream>

template <typename T>
class OperationTraits;

template < >
class OperationTraits<unsigned short>
{
public:
    using R = unsigned int;
};

template < >
class OperationTraits<unsigned int>
{
public:
    using R = unsigned long;
};

template < >
class OperationTraits<unsigned long>
{
public:
    using R = unsigned long long;
};

template <typename T>
class AddPolicy
{
public:
    using RTNTYPE = typename OperationTraits<T>::R;
    static RTNTYPE calculate(T a, T b)
    {
        return static_cast<RTNTYPE>(a) + static_cast<RTNTYPE>(b);
    }
};

template <typename T>
class MultiplyPolicy
{
public:
    using RTNTYPE = typename OperationTraits<T>::R;
    static RTNTYPE calculate(T a, T b)
    {
        return static_cast<RTNTYPE>(a) * static_cast<RTNTYPE>(b);
    }
};

template <template<typename> class Policy, typename T>
typename Policy<T>::RTNTYPE Operate(T a, T b)
{
    return Policy<T>::calculate(a, b);
}

int main()
{
    unsigned short a = 3u;
    unsigned short b = 7u;
    auto c = Operate<AddPolicy>(a, b);
    std::cout << c << std::endl;
    c = Operate<MultiplyPolicy>(a, b);
    std::cout << c << std::endl;
}
```