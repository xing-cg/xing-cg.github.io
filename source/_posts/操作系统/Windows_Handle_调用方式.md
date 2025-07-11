---
title: Windows_Handle_调用方式
categories:
  - - Windows
  - - 操作系统
tags: 
date: 2024/6/12
updated: 
comments: 
published:
---
# 句柄的实现

以HWND为例。

```cpp
DECLARE_HANDLE      (HWND);
```

```cpp
#define DECLARE_HANDLE(name)
struct name##__{
    int unused;
};
typedef struct name##__ *name
```
以上是带参数的宏定义，意味着，后面所有的“name”将被替换。
`##`是粘连符号，以区分`"name__"`和`name + __`，即如果不用粘连符号，假如在程序中写`DECLARE_HANDLE(HWND)`时就无法替换name了，而是生成了`name__`。如果正确用粘连符号，那么就生成`HWND__`。
即：
```cpp
DECLARE_HANDLE(HWND);

==等效于

struct HWND__{ int unused; };
typedef struct HWND__ * HWND;
```

其中，`int unused;`仅作为占位符，即垫片。实际是不使用的，内容值无意义。主要来定义一个类型，使用其类型的指针，以防止与其他类型的HANDLE指针误互相赋值。

相似地，HDC（Handle of Device Context）也是用这个宏定义的，只是类型名不同。

# CALLBACK

```cpp
#define CALLBACK __stdcall
```

调用方式是在C语言层面上的话题，和操作系统是无关的，是给编译器的提示。

`__stdcall`和`__cdecl`是有主要区别的。在其他编译器下，可能有不同的设置方法。从`C++17`标准后，可设置attributes，逐渐变统一了。
在MAC系统上，常用的是`__pascal`调用方式。

1. 压栈方向上，
    1. CDECL是从右向左压栈
    2. PASCAL是从左向右压栈
    3. STDCALL是从右向左压栈
2. CDECL的含义是C语言的调用方式，为了支持可变参数，由于被调用者不清楚参数数目，因此需要调用者来清理栈。
3. PASCAL和STDCALL也称为快速调用。因为在被调用者返回时，顺便也会清理栈，效率稍高。
    1. STDCALL不支持可变参数。
    2. CDECL支持可变参数（如printf）。
4. Windows刚面世时，在286的计算机年代，主频可能才60～80 MHz，效率很宝贵。而CALLBACK涉及到多次调用，而且参数数目是固定的4个（见window_procedure），因此STDCALL作为Windows特意的默认调用方式。
5. 效果是：定义函数时，不主动写调用方式时默认为`CDECL`。显式写CALLBACK时就变成了STDCALL调用方式。
    1. cdecl 是 C 语言的默认调用约定，广泛用于 C 语言编写的程序中。由于调用者负责清理堆栈，这种调用约定允许可变参数函数（如 printf）。
    2. stdcall 主要用于 Windows API 函数调用，是 Windows 平台上的标准调用约定之一。由于被调用者负责清理堆栈，这种调用约定在某些情况下可以提高函数调用的效率。
    3. pascal 调用约定则是历史上在 Pascal 语言中使用的调用约定，现在使用较少。