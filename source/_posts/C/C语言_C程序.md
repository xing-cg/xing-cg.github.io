---
title: C语言_C程序
categories:
  - - C
tags: 
date: 2023/12/17
updated: 
comments: 
published:
---
# 语言的发展
1. Procedure
2. Function
3. Object
4. Meta (template)
5. Component

# Visual Studio使用

Solution（解决方案）指的是解决某一问题整个的方案，需要1个或多个Project（项目）来协同完成。不同的项目可以是不同的语言如`C++`、`C#`等，可以把这些项目放在一个解决方案中联合编译。

## 项目目录结构

项目编译成功后，可执行exe文件会生成在项目目录下的`x64/Debug`中，名字为项目名称

# 第一个C程序

```c
#include <stdio.h> /* standard input output. */

// forward declarations.
void bar(void);

int main(void)
{
    bar();
    return 0;
}

void bar(void)
{
    printf("Hello\n");
}
```

前置**声明**的主要作用体现在：

1. 编译器可以在代码中检测函数是否正确调用，如检查函数名、返回类型、参数类型。
2. 如果代码中函数调用书写正确，则通过编译检测。我们可以写和前置声明中一样的调用书写来欺骗编译器，但是后面链接器会进一步检查是否在下面有函数的正确**定义**。

如果我们删除bar函数定义

```c
#include <stdio.h> /* standard input output. */

// forward declarations.
void bar(void);

int main(void)
{
    bar();
    return 0;
}
```

则链接器报错：

```
1>------ 已启动生成: 项目: Project1, 配置: Debug x64 ------
1>Source.c
1>Source.obj : error LNK2019: 无法解析的外部符号 bar，函数 main 中引用了该符号
1>C:\Users\xcg\source\repos\Solution1\x64\Debug\Project1.exe : fatal error LNK1120: 1 个无法解析的外部命令
1>已完成生成项目“Project1.vcxproj”的操作 - 失败。
```

# 什么情况下需要前置声明

1. 如果把函数实现放在调用处的后面，则需要在调用处前面前置声明；
2. 如果函数在此文件外，则需要在调用处前面前置声明；
3. 如果函数在系统库中，也需要在调用处前面前置声明，只不过写到了`#include<XXX>`中去了，比如printf函数包含在`stdio.h`中；

# 尖括号和双引号的区别

1. 尖括号搜索范围小，编译速度快。
2. 双引号会优先到当前工程路径下去扫描，没有扫描到则去系统库中搜索。

# 编译链接步骤

1. 预处理，如`#include`
2. 编译，把每一个`.c`文件生成一个`.obj`文件，即目标文件，是CPU可识别的机器码，但无法直接执行。
3. 汇编
4. 链接，把所有的`.obj`文件组合为`.exe`文件（Linux下为`.out`）

## `.exe`文件由什么组成

1. 所有`.obj`文件（自己编写的内容，用户库）
2. 系统库内容，如`printf`函数
3. C启动代码
   1. 首先需要一个调用者来调用main函数，程序才能从入口启动
   2. 其次，有一些全局变量，需要启动代码在main函数执行前加载

# C语言程序顶层角度

程序由模块组成，即一个个功能单元。可以说：大的工厂分了好多车间，各个车间有各自的原料。那么，在程序里，语句把这些原料（数据），按多种方法（顺序、分支、循环）送到某一个位置。而表达式则是这些原料（数据）的载体。表达式由运算符、数据组成。

```
C Program
         functions
                   statements
                              expressions __
                                   |        \
                                   |         \
                                  operators   \
                                              elementary data type
```

