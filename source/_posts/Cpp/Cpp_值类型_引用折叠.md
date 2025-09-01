---
title: Cpp_值类型_引用折叠
categories:
  - - Cpp
    - Modern
tags: 
date: 2022/3/12
updated: 
comments: 
published:
---
# `static_cast`

`int a`不能直接用`int && e`接收。
```cpp
int main()
{
    int a = 12;
    int && e = a;  // error
    e = 13;
}
```

`int a`强转为`int&&`时，可以用`int && e`接收，修改e，会连带修改a。
```cpp
int main()
{
    int a = 12;
    int && e = static_cast<int&&>(a);
    e = 13;     // e = 13; a = 13
}
```
`int a`强转为`int`时，可以用`int && e`接收，修改e，不会连带修改a。

> 为什么？
> 强转为值类型时，相当于拷贝了一份临时对象（另创空间）。

```cpp
int main()
{
    int a = 12;
    int && e = static_cast<int>(a);
    e = 13;     // e = 13; a = 12
}
```

# 移动构造（拷贝右值构造）
1. 参数不能加const，因为语义上，是要把other对象的资源移动过来，是需要改动other的。
2. 经验上，要把移动构造函数声明为noexcept，因为涉及到资源的移动的动作过程中即使出现了异常也无法处理、修复。

```cpp
class Test
{
public:
    Test(int val) : _val { new int(val) }
    {}
    // 以非右值对象拷贝构造
    Test(Test const & other)
    {
        _val = new int(*other._val);
    }
    // 以右值对象移动构造
    Test(Test && other) noexcept
    {
        _val = other._val;
        other._val = nullptr;
    }
    ~Test()
    {
        if(_val)
        {
            delete _val;
            _val = nullptr;
        }
    }
    void set_value(int v)
    {
        if(_val)
        {
            *_val = v;
        }
    }
private:
    int * _val;
};
```
测试移动构造：以下代码未能走到移动构造函数的断点——编译器优化了

> 编译器优化了：`Test test{ get_test() }`。没走移动构造，而是直接把`Test test{5}`在main函数中的test上构造了。
> 这个现象叫做：具名的返回值优化。

```cpp
Test get_test(void)
{
    Test test{5};
    return test;
}
int main()
{
    Test test{ get_test() };
}
```
如果不优化的话，会是以下的情况：一共创建了3个对象，有2个是中间无用的临时对象
```cpp
Test get_test(void)
{
    Test test{ 5 };         // create a new test object
    return test;            // generated a temporary test object
}
int main()
{
    Test test{ get_test() };// create a new object by calling copy-constructor
}
```
为了能看到移动构造函数的断点：不通过函数返回的临时值，通过自己手动创建一个test，来强转为右值引用，从而移动构造新的test。
```cpp
int main()
{
    Test test2{ static_cast<Test&&>(test) };
}
```
# 赋值重载

以非右值对象赋值
```cpp
Test & operator=(Test const& other)
{
    if(&other == this) return *this;
    if(_val != nullptr)
    {
        delete _val;
        _val = nullptr;
    }
    if(other._val != nullptr) 
    {
        _val = new int { *other._val };
    }
    return *this;
}
```

以右值对象赋值
```cpp
Test & operator=(Test && other)
{
    if(&other == this) return *this;
    if(_val != nullptr)
    {
        delete _val;
        _val = nullptr;
    }
    _val = other._val;
    return *this;
}
```

# `static_cast`三种值类型的行为

没有变量接收时的行为：
```cpp
int main()
{
    Test test{ 2 };

    static_cast<Test>(test);
    static_cast<Test&>(test);
    static_cast<Test&&>(test);
    
}
```

通过断点调试，发现，强转为普通值类型`<Test>`时，而且是在不进行其他操作的情况下，仍会调用一次普通拷贝构造函数。其他两句不会进行拷贝。

接下来研究其他两个语句——在有变量接收时的行为。
## 强转为普通值类型
1. 以普通值类型接收时，会拷贝，调用的是普通拷贝构造；
2. 不可以左值引用接收普通值类型。
3. 以右值引用接收时，**也会拷贝**，调用的是普通拷贝构造。

```cpp
int main()
{
    Test test{ 2 };

    Test    test11 = static_cast<Test>(test);
    Test &  test12 = static_cast<Test>(test);
    Test && test13 = static_cast<Test>(test);
}
```
## 强转为左值
1. 以普通值类型接收时，会拷贝，调用的是普通拷贝构造；
2. 以左值引用接收时，不会拷贝，**是高效的用法**。
3.  不可以右值引用接收左值。

```cpp
int main()
{
    Test test{ 2 };

    Test    test21 = static_cast<Test&>(test);
    Test &  test22 = static_cast<Test&>(test);
  //Test && test23 = static_cast<Test&>(test);  // error
}
```
## 强转为右值
1. 以普通值类型接收时，会拷贝，**调用右值构造**（如果右值构造较好地处理了资源的移动，则较高效。**若没有明确写右值构造，则仍会进行普通拷贝，则降低了效率**）；
2. 不可以左值引用接收右值。
3. 以右值引用接收时，不会拷贝，**是高效的用法**。

```cpp
int main()
{
    Test test{ 2 };

    Test    test31 = static_cast<Test&&>(test);  // decide on Move Constructor
  //Test &  test32 = static_cast<Test&&>(test);  // error
    Test && test33 = static_cast<Test&&>(test);
}
```

其中，以“右值引用类型”接收“强转为右值的对象”（或者`std::move`后的对象），这个行为其实是对“右值对象”的生命期的延展。如果通过右值引用来修改对象，那么对象源值也会修改。这才是右值引用最本原的功能。
```cpp
int main()
{
    Test test{ 2 };
    // test    : _val : 2

    Test && test33 = static_cast<Test&&>(test);
    test33.set_value(4);
    // test.33 : _val : 4
    // test    : _val : 4
}
```
## 总结

Modern Cpp引入了值类型的细分，不是让你装逼的，也不是为了给程序员增加痛苦的，而是为了让各种类型都有它最好的归宿：
1. 左值交给左值引用（不会有任何其他多余的操作，高效转手）
2. 右值交给右值引用（不会有任何其他多余的操作，高效转手）
3. 右值交给普通类型（调用右值构造，如果右值构造较好地处理了资源的移动，则较高效）

1. 对于前两种来说，只要引用类型匹配，则无需程序员费心，便可提高效率。
2. 而对于程序员来说，最要注意的就是第三种情况，即右值构造函数需要程序员来明确写出，指明资源转移的动作，决定了右值交给普通类型的效率。**如果没有写右值构造函数，则仍会进行普通拷贝，则降低了效率。**
# `forwarding_reference`

形式上和右值引用一样，但性质、行为是未定义的。
1. 传入模板的类型，是不确定的，需要接收后另外推断。
2. 传给auto的类型，亦如此，接收后需要推断。

困扰：
```cpp
template<typename T>
void test(T const & t)
{
    
}

int main()
{
    const int ca = 10;
    int a = 11;
    test(ca);
    test(a);
    test(9);
}
```
问题：字面常量无法传递给`T & t`。
于是加了`T const & t`，可以传递了。

但是，导致，如果传的是const int变量，则T后加const与否对应的行为不一样。
总结起来就是：
1. T后无const时
    1. 传入非常左值，T对应的类型是`const int`，t对应的类型是`const int &`
    2. 传入常左值，T对应的类型是`int`，t对应的类型是`int &`
    3. 常性在传入前后始终是对应的，很好。
    4. 但是：不能接收右值。
    5. 总结：虽然不怎么通用，但是起码没有很乱。
2. T后有const时
    1. 能接收右值了，很好。
    2. 但是，无论传入的是常左值、非常左值、右值，最后，T对应的类型都会是`int`，t对应的类型都会是`const int &`。
    3. 那么就导致：如果传入的是非常左值，由于t被污染成了const int，导致t无法修改。
## 全新的解决方案

使用`T &&`这个形式改进`T const & `。虽然样子看起来是右值引用，但它不是，因为T是不确定类型，而它又常用在参数转发中，所以起了新名叫”转发引用 (Forwarding Reference)“（也有人叫Universal Reference）。根据接收的值类型的不同，T也会跟着变化。

包容度和`T const &`一样，既能接收常左值、非常左值，又能接收右值。
最终达到的效果却比`T const &`更好：
1. 常左值，T对应的类型是`const int &`，t对应的类型是`const int &`
2. 左值，T对应的类型是`int &`，t对应的类型是`int &`
3. 右值，T对应的类型是`int`，t对应的类型是`int &&`

```cpp
template<typename T>
void test(T && t)
{
    
}

int main()
{
    int a = 11;
    test(a);
    test(9);
}
```
## 引用折叠
Reference Collapsed
## 模板参数中的`&&`

虽然经过传入，变成了对应的引用，在函数中也可以修改t的值（const类型除外），但是其实没啥意义，这个转发引用的意义就是在于让你转发的，不是修改值。

```cpp
template <typename T>
void test(T && t)
{
    t = 8;
}
int main()
{
    int a = 11;
    test(a);    // 可以把本身改为8。
    test(9);    // 9还是9，不变。   t = 8修改的是副本
}
```
## auto后的`&&`

```cpp
int main()
{
    int a = 11;
  //int  && ri = a;   // error  右值引用无法接收左值
  //auto &  la = 9;   // error  左值引用无法接收右值
    auto && ra = a;   // ok     ra : int &  
    auto && ra2= 9;   // ok     ra2: int && ra2
}
```
# move

把传入的值类型转换为右值。

```cpp
template<typename T>
T&& mymove(T && t)
{
    return static_cast<T&&>(t);
}
```

```
T: int & + && -> t: int &
T: int   + && -> t: int && 
```
现在的问题是：传入不同值类型的变量后，行为不一致。若是左值，则最终只能强转为`int &`；若为右值，则最终只能强转为`int &&`。现在我们想要追求：无论左右值，最终都转为左值。

那么可以考虑用Traits技术，消除左右值的区别、影响，统一。
```cpp
template<typename T>
class RemoveRefTraits
{
public:
    using TYPE = T;
};

template<typename T>
class RemoveRefTraits<T&>
{
public:
    using TYPE = T;
};

template<typename T>
class RemoveRefTraits<T&&>
{
public:
    using TYPE = T;
};

template <typename T>
using RemRef_t = RemoveRefTraits<T>::TYPE;
```

```cpp
template<typename T>
RemRef_t<T>&& mymove(T && t)
{
    return static_cast<RemRef_t<T>&&>(t);
}
```
# 完美转发
Perfect Forwarding

