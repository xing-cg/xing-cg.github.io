---
title: C语言_const与volatile和mutable
categories:
  - - C
  - - Cpp
tags: 
date: 2021/9/26
updated: 2025/8/7
comments: 
published:
---
# 一句话结论
volatile是要求读值时，必须每次从内存中读取。不能优化缓存到寄存器中，因为这个变量实际的值可能被修改，从寄存器中读取缓存可能出现脏值。

**典型用途​**​：硬件寄存器和多线程共享的只读状态。

在嵌入式编程中，硬件寄存器可能被映射到固定地址。这个寄存器可能是只读的（比如状态寄存器），**但它的值会随着硬件状态改变**。所以我们既需要`const`（因为程序不能写它）又需要`volatile`（因为值会变，编译器不能优化读取）。

volatile 与程序所做的更改完全无关。这意味着**内存可能会因为编译器无法控制的原因而改变**，因此编译器每次都必须读取或写入内存地址，并且不能将内容缓存在寄存器中。

而const只是给程序员的指导性建议，实际可以强制修改const变量。而且const只限制了程序内部行为，它修饰的变量仍然可能被外部修改。
# const和volatile结合的汇编区别

```c
int main()
{
	volatile const int a = 0;
	int b = a;
	int* p = (int*)&a;
	*p = 10;
}
```

![image-20210926094934967](../../images/C%E8%AF%AD%E8%A8%80_volatile&const/image-20210926094934967.png)
区别在于：`int b = a;`：去取了内存中 a 的地址中实际的值到寄存器，寄存器给 b 赋值。

```c
int main()
{
	const int a = 0;
	int b = a;
	int* p = (int*)&a;
	*p = 10;
}
```

![image-20210926095031711](../../images/C%E8%AF%AD%E8%A8%80_volatile&const/image-20210926095031711.png)
区别在于：`int b = a;`：没有去取内存中 a 的地址中实际的值，而是经过了优化，直接给 b 填了 0 。

如果变量被volatile修饰，则是在给编译器说明：该变量可能随时被改写。
所以编译器就不会轻易地去优化代码结构。

const：本程序段中不能对此变量作修改，任何修改都是不通过的，或者至少是粗心，编译器应该报错，防止这种粗心。
但const修饰了的变量不允许被修改，**不代表不允许通过别名的方式修改**，比如：
```c
int i = 5;
const int* p = &i;
*p = 6; // 不可以；
i = 7;  // 完全可以，而且那个“const”的“*p”也跟着变成了7。
```
# const和volatile放在一起的意义
const和volatile放在一起的意义在于：
（1）本程序段中不能对a作修改，任何修改都是不通过的，或者至少是粗心，编译器应该报错，防止这种粗心；
（2）另一个程序段则完全有可能修改，因此编译器最好不要做太激进的优化。

“const”含义是“请作为常量使用”，**但不是**：“放心吧，那肯定是个常量”。
“volatile”的含义是“请不要做没谱的优化，这个值可能变掉的”，**但不是**“你可以修改这个值”。
因此，它们不是矛盾的。
## 例子
```c
const volatile int i = 10;
```
这行代码有没有问题？如果没有，那 `i` 到底是什么属性?

回答一：
1. 没有问题。
2. 例如只读的状态寄存器。它是volatile，因为它可能被意想不到地改变。
3. 它是const，因为程序不应该试图去修改它。
4. volatile和const并不矛盾，只是控制的范围不一样，一个在程序本身之外，另一个是程序本身。

回答二：
1. 没问题，const和volatile这两个类型限定符不矛盾。
2. const表示（运行时）常量语义：被const修饰的对象在所在的作用域无法进行修改操作，编译器对于试图直接修改const对象的表达式会产生编译错误。
3. volatile表示“易变的”，即在运行期对象可能在当前程序上下文的控制流以外被修改（例如多线程中被其它线程修改；对象所在的存储器可能被多个硬件设备随机修改等情况）：被volatile修饰的对象，编译器不会对这个对象的操作进行优化。
4. 一个对象可以同时被const和volatile修饰，表明这个对象体现常量语义，但同时可能被当前对象所在程序上下文意外的情况修改。
# mutable和const的关系
- **规则​**​：`mutable`​**​不能​**​直接与`const`或`volatile`组合修饰变量（如`mutable const`非法）。
- ​**​语义​**​：
    - `mutable`：专门用于类的成员变量，允许其在`const`成员函数中被修改。
    - `const`：在成员函数中禁止修改对象状态。
- ​**​用途​**​：突破`const`限制，修改与逻辑无关的内部状态（如缓存、互斥锁）。

```cpp
class Cache
{
    mutable int cached_value; // 允许const函数修改
    int expensive_calculation() const;
public:
    int get_value() const
    {
        if (!cache_valid)
        {
            cached_value = expensive_calculation(); // mutable突破const限制
        }
        return cached_value;
    }
};
```

- ​**​关键限制​**​：
    - `mutable`​**​仅用于类的非静态数据成员​**​。
    - 不可修饰类外的局部变量、函数参数或类中静态成员。
    - 不允许组合`mutable const int x;`❌（语法错误）。

# volatile 和 mutable可以组合吗？
`C++`标准人为规定了：`volatile`和`mutable`​**​不能组合​**​（如`mutable volatile`非法）。
volatile的语义：告诉编译器，这个变量可能被外部修改，不能优化访问的方式
mutable的语义：告诉编译器，即使对象是const，也可以修改这个成员变量

组合起来的语义是：我希望这个成员变量，即使const对象也可以修改其成员变量，而且每次访问都要从内存中读取，不能缓存。

虽然语义是不冲突的，逻辑是通顺的，但是`C++`标准明确禁止了这种组合。
可能是碍于编译器实现的复杂度、实际需求太少。

可以有替代方案：用`const_cast`突破。
```cpp
class Device
{
    volatile int status_register;
public:
    void reset() const 
    {
        const_cast<volatile int&>(status_register) = 0;
    }
};
```
# 总结

| 关键字组合              | 是否允许 | 用途场景                                   | 关键限制                                         |
| ------------------ | ---- | -------------------------------------- | -------------------------------------------- |
| `const volatile`   | ✅    | 只读但可能被外部修改的变量                          | 程序自身不可写                                      |
| `mutable`          | ✅    | 类的成员（在const函数中可修改）                     | 仅限类的非静态成员                                    |
| `const mutable`    | ❌    | 无！                                     | 语义冲突                                         |
| `volatile mutable` | ❌    | const对象也可以修改其成员变量，而且每次访问都要从内存中读取，不能缓存。 | 语义上不冲突，但是人为规定禁止。可以用`const_cast`代替mutable的功能。 |
核心区别
- `const volatile`→ ​**​全局性限制​**​（硬件/多线程只读数据）
- `mutable`→ ​**​局部性突破​**​（针对类的const成员函数，修改特定状态）
