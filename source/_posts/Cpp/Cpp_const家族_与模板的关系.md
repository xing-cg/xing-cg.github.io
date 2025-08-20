---
title: Cpp_const家族_与模板的关系
categories:
  - - Cpp
tags: 
date: 2024/4/24
updated: 
comments: 
published:
---
# 内容

1. const：运行期；只能修饰变量（即使能修饰成员函数，但是本质上修饰的是this变量）；常性
2. constexpr：可能运行期、可能编译期；可修饰变量、可修饰函数；常性
    * if constexpr
3. consteval：编译期；只能修饰函数；函数是一段代码，没有常性一说
4. 模板参数可以传入常量
    1. 传入某一类型的常量
    2. 传入auto常量
5. constinit：编译期；修饰全局变量（包括静态变量）；无常性
# 常量

不能以变量给数组的大小做定义。因为数组要确定容量，在编译时就确定了它的内存的映像、结构（即必须在运行前就需要清楚大小），执行时不能改变。
> 类似数组的大小要确定的例子，还有switch case中的值，必须用字面常量或常量来描述。

```cpp
int main()
{
    int a = 5;
    int arr[a] = { 0 };  // error
}
```
普通int肯定是不行的。但是const int却可以：
```cpp
int main()
{
    const int N = 5;
    int arr[N] = { 0 };  // ok
}
```
为什么呢？因为此处的const就代表：编译期时，N可以得到确定为5。所以，满足了数组的定义的条件即编译期时确定大小，于是可以通过。
> 但const修饰变量不是一定能在编译期确定的。比如：

```cpp
int get_size(int);
int main()
{
    const int N = get_size(5);  // ok
    int arr[N] = { 0 };        // error
}
int get_size(int a)
{
    return a;
}
```
通过函数返回值去初始化const变量时，虽然可以定义、初始化const变量，但是用在`arr[N]`中又会报错。这说明，const int N在定义时，是可以感知到后面的值是一个字面常量还是一个函数返回值（本质上是一个变量）的。如果是通过函数返回值（变量）初始化的，虽然可以得到初始化，但是却不能用于定义数组。这是因为，函数本质上是在执行代码时，通过栈帧动态进行的，这又陷入了执行期才能确定具体结果，所以不能给数组定义。

所以：const修饰的变量，不一定能在编译期决定，即也是有可能在执行期确定的。但是编译期是可以清楚地分明你这个N的来历的，即使N被声明const也无所谓。因此const这个关键字是模棱两可的。
于是，如果我们要限制一个变量必须在编译期就确定常性，就得用constexpr来修饰。而不是用const；const以后则可以用于修饰运行期的行为，以后最好不要再乱用const来修饰编译期常量。
# constexpr

Modern Cpp提供的关键字。
1. 如果用来修饰变量：就可以用于限制等式右边的值是一个常量。这个常量一定是在编译期就确定了的。
2. 如果用来修饰函数：函数被constexpr修饰后，看调用点接收返回值的变量是否也为constexpr。
    1. 如果是，则编译期就会确定死返回值，直接在调用点替换，而不生成代码。
    2. 如果不是，则改函数正常生成可编译代码，变为普通的函数在运行期流转。

如下，如果告知了N是一个constexpr变量，但发现`get_size`不是一个constexpr，则编译不通过。
```cpp
int main()
{
    constexpr int N = get_size(5);  // error
    int arr[N] = { 0 };             // error
}
int get_size(int a)
{
    return a;
}
```
如果改为5，则通过。
```cpp
int main()
{
    constexpr int N = 5;  // ok
    int arr[N] = { 0 };   // ok
}
```
那么，如何通过函数返回值初始化constexpr变量呢？就需要给函数也用constexpr修饰。但是注意，不再支持前置声明，后置定义函数的形式，而只能直接写在前置：
```cpp
constexpr int get_size();
// 错误，get_size() 必须在调用点之前完整定义
int main(void)
{
    constexpr int N = get_size();
    int arr[N] = { 0 };          
}
constexpr int get_size()
{
    return 5;
}
```
正确写法：
```cpp
constexpr int get_size(void)
{
    return 5;
}
int main(void)
{
    constexpr int N = get_size();//ok
    int arr[N] = { 0 };          //ok
}
```
因为constexpr的函数实际不会生成函数体，因此不支持前置声明、后置定义的形式。所以，constexpr的函数建议写为inline类型或者写在头文件中包含到前置。
# consteval

不能修饰变量，只能用于修饰函数，目的是比constexpr更严格地限制修饰的内容要在编译期确定。

如果发现函数中返回值表达式中有非常量（下例则是val是变量），则不能编译通过。
```cpp
consteval int get_size(int val)
{
    return 5 + val;
}
int get_value(void)
{
    return 3;
}
int main()
{
    const int b = get_value();    // ok
    const int a = get_zize(b);    // error
}
```
若改为：
```cpp
consteval int get_size(int val)
{
    return 5 + val;
}
int get_value(void)
{
    return 3;
}
int main()
{
    const int b = 3;
    const int a = get_zize(b);    // ok
}
```
> 其实a用const、constexpr修饰都已无所谓了，因为consteval已经可以严格限制`get_size(b)`的返回值是编译期确定的了，无需靠constexpr来约束，用const也行。
# 总结constexpr和consteval

1. constexpr修饰函数时，并不严格限制函数中的内容必须是编译期就确定的，分两种情况：
    1. 满足编译期就确定时，函数变为常量表达式，在调用点处替换
    2. 不满足时，如传入一个变量参数，则必须在运行期才能确定的，就退化为一个普通函数。
2. consteval是加强版的constexpr，不能修饰变量，因为修饰变量没意义。主要是用于修饰函数。主要是看函数参数是否全支持编译期确定。
3. 被constexpr、consteval修饰的函数体内可以是递归。比如递归求和：（此时把鼠标挪在a上面，发现直接计算出了结果，55）
```cpp
consteval int sum(int n)
{
    if(n == 0)
        return n;
    return n + sum(n - 1);
}
int main()
{
    constexpr int a = sum(10);
}
```
但是，递归的层数默认最多为512层。这个值，不同编译器可以通过不同命令修改。
# 模板参数传入常量
## 模板参数传入某一类型的常量

```cpp
template <int N>
constexpr int get_number(void)
{
    return N;
}
int main()
{
    constexpr int a = get_number<15>();
}
```
实际编译完后，相当于产生以下代码：
```cpp
int get_number(void)
{
    return 15;
}
int main()
{
    constexpr int a = get_number();
}
```
> 此例中的模板参数N必须是常量，最好用constexpr修饰。总之：变量不行；const可能不行（若等号右边的值不是字面常量、常量）
> 同理，因为模板函数也是编译期确定的，所以N这种特定值参数必须也像上面谈的常量的用法一样，编译期必须确定。

```cpp
template <int N>
constexpr int get_number(void)
{
    return N;
}
int main()
{
    //int b = 15;
    //constexpr int a = get_number<b>(); // error
    
    constexpr int b = 15;
    constexpr int a = get_number<b>();   // ok
}
```

### `get_element`

编译期判断。用constexpr修饰if语句

```cpp
struct Obj
{
    int a;
    std::string b;
}

template<int N>
auto get_element(Obj & obj)
{
    if constexpr (N == 0)
        return obj.a;
    else if constexpr (N == 1)
        return obj.b;
}
```
如果编译时确定`N == 1`，则会生成相应的函数。auto将切换成对应类型。
```cpp
std::string get_element(Obj & obj)
{
    return obj.b;
}
```

```cpp
int main()
{
    Obj obj { 5, "Hello" };
    int a = get_element<0>(obj);           // 5
    std::string b = get_element<1>(obj);   // Hello
}
```

## 模板参数传入auto常量

`template<int N>`是int类型的特定值，而`template<auto V>`是任意类型的特定值。

static的特点是，生命周期和全局变量一样，但是可见范围仅限于一部分，下面的value就仅限于类中可见。
```cpp
template <auto V>
class Constant
{
public:
    static constexpr auto value = V;
}
```

```cpp
int main()
{
    auto a = Constant<19>::value;
}
```
### constexpr配合auto，配合模板auto常量使用

如果把a修饰为constexpr，则就会变为编译期计算。
```cpp
int main()
{
    constexpr auto a = Constant<19>::value;
}
```
这么做的意义在于，可以让一个同名的变量具有不同的类型、不同的值，即复用了名字。模板原先只能用于类模板、函数模板，这样，套了一个类模板的外壳，让变量也具有了模板的能力。
# 优化全局变量的初始化：constinit
1. 与作用域无关
2. 与常性无关
3. 修饰全局变量（包含静态变量）

```cpp
void bar(void)
{
    static int val = 8;
    std::cout << val << std::endl;
}
int main()
{
    bar();
}
```
以上程序，val的初始化不在bar函数中，而是在main函数执行之前。所以调试的断点会掉下来。
但是这不代表val是编译期执行的。
```cpp
int get_int(void)
{
    return 7;
}
void bar(void)
{
    static int val = get_int();
    std::cout << val << std::endl;
}
int main()
{
    bar();
    bar();
}
```
以上代码，通过函数确定val值时，第一次调用bar，static语句是在bar函数执行时才执行的。而第二次调用bar，static语句不执行任何动作。

全局变量，可能有两种行为：
1. 在static语句前就执行完了
2. 在static语句执行

能不能优化，让static语句在static语句前就执行完？或者说，让它在编译期就完成行为？
用constinit修饰就能起到这个作用。
```cpp
int get_int(void)
{
    return 7;
}
void bar(void)
{
    // constinit static int val = get_int();  // error, get_int() is not const value
    constinit static int val = 8;             // ok
    std::cout << val << std::endl;
}
int main()
{
    bar();
    bar();
}
```
与consteval有异曲同工之妙，被constinit修饰的变量，必须保证等号右边是一个常量（即编译期确定下来），否则编译不通过。这样，就能保证static语句在编译期初始化完毕。
所以，val不能直接接受普通的`get_int()`函数返回值，如果要编译通过，需要修饰函数为`consteval`。

虽然关键字带有const，但是不代表被修饰的变量是常性的，后期该值会不会被修改，是未定义的。如果要保证这个全局变量是常性，需要另外修饰const。

> 其实用constexpr直接修饰全局变量，也可以起到让全局变量在编译期执行的效果，但是constinit的特点在于，它修饰后，变量不带有常性，可以后期更改，而constexpr修饰变量后，后面就没法更改了。

# 总结

模板都是和类型打交道。各种的策略最后本质上都是以不同类型决定的。本质上做的工作就是帮助编译器在编译期分辨各种类型，组装类、函数、实例，供开发人员使用。

判断类型的特性，比如看是否是指针？需要用到`if constexpr`

```cpp
template<typename T>
auto get_val(T t)
{
    if constexpr (std::is_pointer_v<T>)
        return *t;
    else
        return t;
}

int main()
{
    int a = 9;
    auto v = get_val(&a); // &a 识别为 int *  返回值得到的是int型，值为9
    auto v2 = get_val(a); // a 识别为  int    返回值得到的是int型，值为9
}
```