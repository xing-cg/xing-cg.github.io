---
title: Cpp_函数调用过程
typora-root-url: ../..
categories:
  - [Cpp]
tags:
  - null 
date: 2021/9/30
update:
comments:
published:
---

# 函数调用过程

1. 参数入栈
2. 函数栈帧开辟
3. 返回值返回
4. 函数栈退出

# 代码

```c
//4字节入栈
#include<stdio.h>
void fun(int a, int b)
{
    int c;
    c = a + b;
    return c;
}
int main()
{
    int a = 10;
    int b = 20;
    fun(a,b);
    return 0;
}
```

```c
//8字节入栈
struct Node
{
    int _1;
    int _2;
};
int fun(struct Node a, struct Node b)
{
    int c = 30;
    return c;
}
int main()
{
    struct Node a = {10, 20};
    struct Node b = {30, 40};
    fun(a, b);
    return 0;
}
```

```c
//12字节入栈
struct Node
{
    int _1;
    int _2;
    int _3;
};
int fun(struct Node a, struct Node b)
{
    int c = 30;
    return c;
}
int main()
{
    struct Node a = {10, 20, 30};
    struct Node b = {40, 50, 60};
    fun(a, b);
    return 0;
}
```

![image-20211001184239521](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211001184239521.png)

# 寄存器

可以理解为CPU中的变量

如何去标志一个栈？——栈底和栈顶指针。

esp：栈顶寄存器。

ebp：栈底寄存器。

# 函数调用过程

1. 参数入栈（C语言）

   1. 4字节参数（dword）入栈

      1. 顺序：从右向左
      2. 方式：使用寄存器push带入

   2. 8字节参数入栈

      1. 顺序：从右向左
      2. 方式：使用寄存器分两次push带入

   3. （>8字节）12字节参数入栈

      ![image-20211001184251000](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211001184251000.png)

      1. 顺序：从右向左
      2. 方式：变了，**栈顶先上移，即开辟足够的空间**，之后利用寄存器将数据分批次复制进去。

   4. C++中，只要是自定义类型，无论多大字节，都采用先esp上移开辟空间，之后分次赋值的方式（即方式3），即>8字节的情况。

2. 函数栈帧开辟
   其中，1、2步是为了保存现场。3、4步是开辟新的栈帧。

   1. 函数入参的动作完成后，汇编代码执行call(_fun)语句，esp向上移4位，将**调用方函数原来的下一行指令地址**存入栈。因为main中调用完成外部函数后，还要回到以前的位置。
      ![image-20211018205543731](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018205543731.png)
      ![image-20211018205624299](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018205624299.png)
   2. 将**调用方函数的栈底寄存器**（ebp）入栈
      ![image-20211018205322518](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018205322518.png)
   3. **让`ebp=esp`**（让ebp上移到esp的位置）
   4. **让`esp=esp-0**h`**（上移若干空间，开辟此函数的空间）
   5. 将某些寄存器（线程保护）入栈
   6. 将新开辟的栈帧空间中全部赋为`0xcccccccc`

3. 函数返回值

   1. 4字节返回值：将返回值先存到eax再赋给接收变量
      ![image-20211018210146635](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018210146635.png)
      ![image-20211018210420665](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018210420665.png)

   2. 8字节返回值：将返回值分开放到两个寄存器，再赋给接收变量
      ![image-20211018210458734](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018210458734.png)

      ![image-20211018210558730](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018210558730.png)
      ![image-20211018210607525](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018210607525.png)
      ![image-20211018210900593](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018210900593.png)

   3. （>8字节）12字节返回值：

      1. 在函数参数入栈之后，入栈一个调用方（如main）栈帧上的地址（靠近栈顶位置）
      2. 在返回值返回的时候，将返回数据分次写入到之前入栈的调用方地址上。
      3. 返回之后，将从该地址（调用方（如main）栈帧上的地址）将数据分次取出（一次4字节）到接收变量中。

      ![image-20211001213921252](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211001213921252.png)

   4. C++中，自定义类型都按照入栈一个调用方地址的方式

4. 函数栈退出

   1. 进行当前函数栈帧的校验
   2. 将线程保护的寄存器出栈
   3. 让`esp=ebp`（回缩栈帧）
   4. `pop`操作（即将pop的位置是esp指向的位置）。
      `pop ebp`，意为：`ebp=pop`，`esp`指向的是原来`main`的`ebp`地址，赋给`ebp`，同时`esp`下移4字节。则`ebp`回指到原来的位置。形成现场恢复。
      ![image-20211018233157527](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018233157527.png)
   5. 将下一行指令的地址还原（`ret`），实际上，也是一个`pop`，返回保留的地址值，`esp`下移4字节。
   6. esp再次下移若干，清除参数和之前入栈的调用方地址。下移之后，则esp和ebp回到了main函数调用外部函数之前的原始位置。函数调用结束。
      ![image-20211018233852672](../../images/%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8%E8%BF%87%E7%A8%8B/image-20211018233852672.png)

# 说明

当前演示的函数调用规则是依赖于C语言默认的调用约定`__cdecl`，其他的还有`__stdcall`/`__fastcall`。三种差异并不大，只是负责的事情不同，清除参数是调用方执行的，`stdcall`是被调用方执行的。

```c
struct Node __cdecl fun(int a, int b)
{
    struct Node c = {10, 20, 30};
    return c;
}
```

