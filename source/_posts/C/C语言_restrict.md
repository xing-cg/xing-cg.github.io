---
title: C语言_restrict
categories:
  - [C]
tags:
  - null 
date: 2022/3/23
updated:
comments:
published:
---

参考文章：[C++ 需要 restrict 关键字吗？ - 蓝色的回答 - 知乎](https://www.zhihu.com/question/32106315/answer/54660335)

# 演示

C/Cpp编译器，无论是GCC，Clang，VCpp，IBM XL Cpp等，这些主流的Cpp编译器都提供了restrict关键字的支持，只是书写的形式有所变化，如可能是\_\_restrict\_\_，\_\_restrict等 ，而restrict是限制**Pointer Alias**的，限制Pointer Alias有助于编译器做优化，这和unique_ptr完全是两码事。

## restrict

以GCC产生汇编指令的例子来补充一下，比较直观

```c
void f(int *a, int *b, int *c)
{
    *a += *c;
    *b += *c;
}
```

-O3后的汇编代码

```x86asm
f(int*, int*, int*):
	movl	(%rdx), %eax
	addl	%eax, (%rdi)
	movl	(%rdx), %eax
	addl	%eax, (%rsi)
	ret
```

加上restrict

```c
void f(int * __restrict__ a, int* __restrict__ b, int* __restrict__ c)
{
  *a += *c;
  *b += *c;
}
```

-O3后

```x86asm
f(int*, int*, int*):
	movl	(%rdx), %eax
	addl	%eax, (%rdi)
	addl	%eax, (%rsi)
	ret
```

**可以很清楚的看见是4条指令变为了3条指令，而少掉的一条就是第二次的load c**

## unique_ptr

```cpp
#include <memory>
using namespace std;
void f(std::unique_ptr<int> a, std::unique_ptr<int>b, std::unique_ptr<int> c)
{
  *a += *c;
  *b += *c;
}
```

-O3 -std=c++11

```x86asm
f(std::unique_ptr<int, std::default_delete<int> >, std::unique_ptr<int, std::default_delete<int> >, std::unique_ptr<int, std::default_delete<int> >):
	movq	(%rdx), %rdx
	movq	(%rdi), %rax
        
        ; *a += *c
	movl	(%rdx), %ecx
	addl	%ecx, (%rax)

	movq	(%rsi), %rax
        
        ; *b += *c
	movl	(%rdx), %edx
	addl	%edx, (%rax)
	ret
```

所以，可见，unique_ptr和restrict完全是两码事。

# 个人理解

至于为什么会和unique_ptr混淆，因为他两个语义上有共同之处：只有一个指针指向某个变量。但是restrict的目的是给编译器做优化，而且它的限制效果与const修饰符很类似，需要自己去体会；

---

评论区见解：

提问者 - dawnmist：按照unique_ptr的语义，它是资源的唯一持有者。如果资源是内存，那也只能通过这一个unique_ptr访问。用unique_ptr时，编译器能不能像restrict那样优化这个指针访问呢？

回答者 - 蓝色：不行，unique_ptr可以通过get()方法转为原始指针；unique_ptr只是一个智能指针，所以你不能这样限制unique_ptr的语义。