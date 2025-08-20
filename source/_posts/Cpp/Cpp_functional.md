---
title: Cpp_functional
categories:
  - - Cpp
tags: 
date: 2022/3/29
updated: 
comments: 
published:
---

# 内容

1. std::function
1. std::bind

# 可调用对象

Cpp中，存在“可调用对象(Callable Objects)”这一概念。准确来说，可调用对象有如下几种定义：

1. 是一个函数指针
2. 是一个具有operator()成员函数的类对象（仿函数）。
3. 是一个可被转换为函数指针的类对象。
4. 是一个类成员（函数）指针。

* 可调用对象的使用示例

  * 函数指针

    ```cpp
    void func(void)
    {
        // ...
    }
    ```

  * 仿函数

    ```cpp
    struct Foo
    {
        void operator()(void)
        {
            // ...
        }
    };
    ```

  * 可转换为函数指针的类对象

    ```cpp
    struct Bar
    {
        using fr_t = void(*)(void);
        static void func(void)
        {
            // ...
        }
        operator fr_t(void)
        {
            return func;
        }
    }
    ```

  * 类成员函数指针

    ```cpp
    struct A
    {
        int a_;
        void mem_func(void)
        {
            // ...
        }
    };
    ```

  * 测试

    ```cpp
    int main(void)
    {
        void (* func_ptr)(void) = &func;
        func_ptr();
        
        Foo foo;
        foo();
        
        Bar bar;
        bar();
        
        void (A::*mem_func_ptr)(void) = &A::mem_func;
        int A::*mem_obj_ptr = &A::a_;
        A aa;
        (aa.*mem_func_ptr)();
        aa.*mem_obj_ptr = 123;
        
        return 0;
    }
    ```

从上述可看到，除了类成员指针之外，上面定义涉及的对象均可以像一个函数那样做调用操作。Cpp11中，这些对象(func\_ptr、foo、bar、mem\_func\_ptr、mem\_obj\_ptr)都被称作**可调用对象**。相对应地，这些对象的类型被统称为“**可调用类型**”。

> 上面对可调用类型的定义里并没有包括函数类型，这是因为函数类型并不能直接用来定义对象；也没有包括函数引用，因为引用从某种意义来说，可以看作一个const的函数指针。

Cpp中的可调用对象具有统一的操作形式，即后面加括号进行调用（除了类成员函数指针），但是定义方法却五花八门。我们试图使用统一的方式进行保存，或传递一个可调用对象时。于是Cpp11通过提供std::funciton和std::bind统一了可调用对象的各种操作。

# 可调用对象包装器std::function

std::function是可调用对象的包装器，是一个类模板，可以容纳除了类成员函数指针之外的所有可调用对象。通过指定它的模板参数，它可以用统一的方式处理函数、函数对象、函数指针，并允许保存和延迟执行它们。

* std::function的基本用法示例

  * 绑定一个普通函数

    ```cpp
    #include<iostream>
    #include<functional>
    void func(void)
    {
        std::cout << __FUNCTION__ << std::endl;
    }
    int main()
    {
        std::function<void(void)> fr1 = func;
        fr1();
        return 0;
    }
    /* func */
    ```

  * 绑定一个类的静态成员函数

    ```cpp
    #include<iostream>
    #include<functional>
    class Foo
    {
    public:
        static int foo_func(int a)
        {
            std::cout << __FUNCTION__ << "(" << a << ") ->: ";
            return a;
        }
    };
    int main()
    {
        std::function<int(int)> fr2 = Foo::foo_func;
        std::cout << fr2(123) << std::endl;
        return 0;
    }
    /* foo_func(123) ->: 123 */
    ```

  * 绑定一个仿函数

    ```cpp
    class Bar
    {
    public:
        int operator()(int a)
        {
            std::cout << __FUNCTION__ << "(" << a << ") ->: ";
            return a;
        }
    };
    int main()
    {
        Bar bar;
        fr2 = bar;
        std::cout << fr2(123) << std::endl;
        return 0;
    }
    /* operator()(123) ->: 123 */
    ```

从上面我们可以看到std::function的使用方法，当我们给std::function填入合适的函数签名（即一个函数类型，只需要包括返回值和参数表）之后，它就变成了一个可以容纳所有这一类调用方式的“函数包装器”。

* std::function作为回调函数的示例

  ```cpp
  #include<iostream>
  #include<functional>
  class A
  {
      std::function<void()> callback_;
  public:
      A(const std::function<void()> & f) : callback_(f)
      {}
      void notify(void)
      {
          callback_();	//回调到上层
      }
  };
  class Foo
  {
  public:
      void operator()(void)
      {
          std::cout << __FUNCTION__ << std::endl;
      }
  };
  int main()
  {
      Foo foo;
      A aa(foo);
      aa.notify();
      return 0;
  }
  ```

从上面例子中可以看到，std::function可以取代函数指针的作用。因为它可以保存函数延迟执行，所以比较适合作为回调函数。

* std::function还可以作为函数入参

  ```cpp
  #include<iostream>
  #include<functional>
  void call_when_even(int x, const std::funciton<void(int)> & f)
  {
      if(!(x&1))	//x % 2 == 0
      {
          f(x);
      }
  }
  void output(int x)
  {
      std::cout << x << " ";
  }
  int main()
  {
      for(int i = 0; i < 10; ++i)
      {
          call_when_even(i, output);
      }
      std::cout << std::endl;
      return 0;
  }
  /* 0 2 4 6 8 */
  ```

从上例可以看到，std::function比普通函数指针更灵活、便利。

# std::bind绑定器

当std::function和std::bind配合起来使用时，所有的可调用对象（包括类成员函数指针和类成员指针）都将具有统一的调用方式。

std::bind用来将可调用对象与其参数一起进行绑定，绑定后的结果可以使用std::function进行保存，并延迟调用到任何我们需要的时候。

通俗来讲，bind主要有两大作用：

1. 将可调用对象与其参数一起绑定成一个仿函数。
2. 将多元（参数个数为n, n>1）可调用对象转成一元或者（n-1）元可调用对象，即只绑定部分参数。

* 实际使用示例

  ```cpp
  #include<iostream>
  #include<functional>
  void call_when_even(int x, const std::function<void(int)> & f)
  {
      if(!(x&1))	//x % 2 == 0
      {
          f(x);
      }
  }
  void output(int x)
  {
      std::cout << x << " ";
  }
  void output_add_2(int x)
  {
      std::cout << x + 2 << " ";
  }
  int main()
  {
      {
          auto fr = std::bind(output, std::placeholders::_1);
          for(int i = 0; i < 10; ++i)
          {
              call_when_even(i, fr);
          }
          std::cout << std::endl;
      }
      {
          auto fr = std::bind(output_add_2, std::placeholders::_1);
          for(int i = 0; i < 10; ++i)
          {
              call_when_even(i, fr);
          }
          std::cout << std::endl;
      }
      return 0;
  }
  /* 
  0 2 4 6 8 
  2 4 6 8 10
  */
  ```

同样还是上面std::function中最后的一个例子，只是在这里我们使用了std::bind，在函数外部通过绑定不同的函数，控制了最后的执行结果。

我们使用auto fr保存std::bind的返回结果，是因为我们并不关心std::bind真正的返回类型（**实际上std::bind的返回类型是一个stl内部定义的仿函数类型**），只需要知道它是一个仿函数，可以直接赋值给一个std::function。当然，这里直接使用std::function类型来保存std::bind的返回值也是可以的。

std::placeholders::\_1是一个占位符，代表这个位置将在函数调用时，被传入的第一个参数所替代。

因为有了占位符的概念，std::bind的使用非常灵活。

```cpp
#include<iostream>
#include<functional>
void output(int x, int y)
{
    std::cout << x << " " << y << std::endl;
}
int main()
{
    std::bind(output, 1, 2)();							//输出1 2
    std::bind(output, std::placeholders::_1, 2)(1);		//输出1 2
    std::bind(output, 2, std::placeholders::_1)(1);		//输出2 1
/*  std::bind(output, 2, std::placeholders::_2)(1);*/	//error, 缺少第二个参数
    std::bind(output, 2, std::placeholders::_2)(1, 2);	//输出2 2, 相当于第1个参数仅仅是标志
    std::bind(output, std::placeholders::_1,
              		  std::placeholders::_2)(1, 2);		//输出1 2
    std::bind(output, std::placeholders::_2,
              		  std::placeholders::_1)(1, 2);		//输出2 1
    return 0;
}
```

上面例子对std::bind的返回结果直接施以调用，可以看到，**std::bind可以直接绑定函数的所有参数，也可以仅绑定部分参数**。

在绑定部分参数的时候，通过使用std::placeholders，来决定空位参数将会属于调用发生时的第几个参数。

* 下面来看bind和function配合使用

  ```cpp
  #include<iostream>
  #include<functional>
  class A
  {
  public:
      int i_ = 0;
      void output(int x, int y)
      {
          std::cout << x << " " << y << std::endl;
      }
  };
  int main()
  {
      A a;
      std::functional<void(int, int)> fr = 
          std::bind(&A::output, &a, std::placeholders::_1,
                    std::placeholders::_2);//目标函数A::output, 第一个参数是this指针, 还有其他两个参数。
      fr(1, 2);	//输出1 2
      std::function<int&(void)> fr_i = std::bind(&A::i_, &a);
      fr_i() = 123;
      std::cout << a.i_ << std::endl;	//输出123
      return 0;
  }
  ```

fr的类型是std::function<void(int, int)>。我们通过使用std::bind，将A的成员函数output的指针和a绑定，并转换为一个仿函数放入fr中存储。

之后，std::bind将A的成员i\_的指针和a绑定，返回的结果被放入std::function<int&(void)>中存储，并可以在需要时修改访问这个成员。

# 实质

```cpp
class Alloc
{
    size_t _sz;
    int *& _ptr;
public:
    Alloc(size_t sz, int *& p) : _sz(sz), _ptr(p) {}
    void operator()()
    {
        _ptr = (int*)malloc(_sz);
    }
}
void alloc(size_t sz, int *& p)
{
    p = (int*)malloc(sz);
}
int main()
{
    int * p = nullptr;
    int sz = sizeof(int) * 10;
    auto fr = std::bind(alloc, sz, std::ref(p));
    fr();
    if(p != nullptr)
    {
        cout << p << endl;
    }
}
```

