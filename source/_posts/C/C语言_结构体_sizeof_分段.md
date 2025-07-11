---
title: C语言_结构体_sizeof_分段
categories:
  - - C
tags: 
date: 2024/1/16
updated: 
comments: 
published:
---
# 结构体

>关于`;`的问题：
>
>1. 一般来说，大括号`{}`结尾后不用再加`;`。但是结构体的定义中的大括号`{}`后面要加`;`。因为结构体的定义属于数据的定义，只是有了大括号的形式而已，而数据的定义后面都要加`;`，因此结构体的定义中的大括号`{}`后面要加`;`。
>2. 函数、if语句、for语句中的`{}`是语句的定义，则不用加`;`。

```c
struct Stu
{
    unsigned id;
    unsigned char age;
    char name[10];
};
```

C语言中，`struct`和`结构体名字`是一体的，共同才能合成一个类型，不能只写`Stu`。

# 结构体的定义

```c
int main()
{
    struct Stu stu;
    stu.age = 17;
    stu.id = 123456;
    strcpy(stu.name, "xcg");
    printf("Age: %u, ID: %u, Name: %s\n", stu.age, stu.id, stu.name);
}
```

# 结构体的大小

```c
struct Test
{
    int a;          // 4
    char b;         // 1  --> 实际: 4
};
int main()
{
    printf("%u\n", sizeof(struct Test));
}// 8
```

## 字节对齐

为了效率，即使char只占用一个字节，也会以结构体中大的数据类型为参考多占一些空间。如此在传输过程中就无需再去一个一个找分量，而是可以直接按照整体处理。一个时钟周期，64位数据都会一次性传输完成，所以说虽然浪费了一些空间，但换来了提高效率。

1. 每一个元素的偏移量（相对0），都是自身类型大小的整数倍（0也算）。
2. 结构体的整体大小，是内部最大的基础数据类型大小的整数倍。
3. 我们控制不了的：**结构体的首地址，是体内最大基本数据类型大小的整数倍。**

```c
struct Test
{
    int a;          // 4
    char b;         // 1  --> 实际: 2
    short c;        // 2
};//8
struct Test
{
    int a;          // 4
    char b;         // 1  --> 实际: 2
    short c;        // 2
    char d;         // 1  --> 实际: 4
};//12
struct Test
{
    int a;          // 4
    char b;         // 1
    char d;         // 1
    short c;        // 2
    short e[10];    // 10 --> 12
};//20
```

> 有办法关闭字节对齐特性：`#pragma comment package 1`

# typedef

```c
typedef struct _Stu
{
    unsigned id;
    unsigned char age;
    char name[11];
}Stu;
```

## 结构体指针类型

```c
typedef struct _Stu
{
    unsigned id;
    unsigned char age;
    char name[11];
}*PStu;
```

如此定义，则效果为：`PStu`相当于`Stu*`

`Stu* stu = (Stu*)malloc(sizeof(Stu));`就可以写为

`PStu stu = (PStu)malloc(sizeof(Stu));`

## 星号的结合问题

```c
typedef int * PMyInt;
```

如此定义，`PMyInt`相当于`int*`，即是一个int指针类型。但是，实际上，`*`星号是跟着名字`PMyInt`结合的。`int*`这种写法只是大多数人的习惯而已。

因此，`typedef int MyInt, *PMyInt;`才解释得通。如果`typedef int * MyInt, PMyInt`，则就变成了`MyInt`是指针类型，而`PMyInt`变成了int类型！

# 堆创建结构体

## stdlib

C Standard General Utilities Library

### malloc

分配指定字节数的内存块，返回一个无类型指针，需要强制转换类型。此空间的伸缩不受限制，因此需要显式分配、释放。

在小型设备上如果内存不够，则有可能会返回NULL。

```c
#include<stdlib.h>
int main()
{
    Stu* stu = (Stu*)malloc(sizeof(Stu));
}
```

### free

```c
#include<stdlib.h>
int main()
{
    Stu* stu = (Stu*)malloc(sizeof(Stu));
    free(stu);
    stu = NULL;
}
```

为什么要置空？不成文的规定：指针是否有效？由指针是否为空决定。如果指针不空则认为是指向的内容有效。

如果不置空，即在释放指向的内容后依旧保留此指针，就成为了无意义的乱指。称为“野指针”或“迷途指针”。

> 如果反过来：空间内容有效，但指针值丢失，则叫做内存泄漏。
>
> 怎么解决？
>
> 1. 小心编程
> 2. 关闭进程，让操作系统收回所有内存

## 结构体指针访问成员

1. `(*stu).age` - `*`优先级是2，`.`优先级是1
2. `stu->age` - `arrow`运算符。

# 分段

结构体不仅可以把小类型组合成大类型，还能把基本数据类型拆成小块。

使用“位域”运算符（bit field operator）。

```c
struct Test
{
    int a : 2;
};
int main()
{
    struct Test test;
    test.a = 1;
    printf("%i\n", test.a);
}// 1
```

给`test.a`赋值1是可以的。但给`test.a`赋值2则会打印成`-2`

```c
int main()
{
    struct Test test;
    test.a = 2;
    printf("%i\n", test.a);
}// -2
```

因为：a是有符号int型，但只有2位的空间。除去符号位，则只有1位空间。那么，把2这个int字面常量给了a，就相当于：`(10)2`给了a。那么，a实际空间存的是(10)2，因为它是有符号int，而又因为当前符号位是1，所以实际表达的值为：`(10)2`的补码，算出无符号值后，再加个负号，即$(01)_2+1=(10)_2$​得出2，再加个负号，即`-2`。

此时的sizeof大小为多少呢？

```c
struct Test
{
    int a : 2;
};// 4  -->  整个int的大小
```

```c
//a、b一起分了12个字节，还有20个没分配。因此还是1个int的大小
struct Test
{
    int a : 2;
    int b : 10;
};// 4  -->  1个int的大小
```

## 分隔符

```c
//a先分了2个字节，但是a剩下的空间不想再用，加个separator分隔符。
//在分隔符后定义b。此时是第2个int空间。
struct Test
{
    int a : 2;
    int : 0;     // separator  //无名
    int b : 10;
};// 8  -->  2个int的大小
```

下面这个分隔符的意义在于，虽然实际有效的是int类型，但是long long分隔符的作用在于在第一个和第二个int之间划分了8字节的位置，即a实际被提升为8字节大小，b也被提升为8字节大小。

```c
//a先分了2个字节，但是a剩下的空间不想再用，加个separator分隔符。
//在分隔符后定义b。此时是第2个int空间。
struct Test
{
    int a : 2;
    long long : 0;     // separator  //无名
    int b : 10;
};// 16  -->  2个long long的大小
```

## 实际用处

1. 如果小于8 bits，则用位域运算符。如果是8 bits，则直接用char。
2. separator一般不用，直接占满上面的空位即可，然后紧接着定义下一个类型。
