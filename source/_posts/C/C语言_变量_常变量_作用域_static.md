---
title: C语言_变量_常变量_作用域_static
categories:
  - - C
tags: 
date: 2024/1/8
updated: 
comments: 
published:
---
# 作用域

## 局部变量

```c
#include<stdio.h>
void bar();
int main()
{
    bar();
    bar();
    return 0;
}
void bar()
{
    int val = 10;
    printf("%i\n", val++);
}
```

以上程序两次都打印10。因为val的作用域只生存在函数当时所分配的栈帧中，此变量为自动/局部变量（auto/local variable），函数调用结束后变量自动销毁。

## 全局变量

全局变量（global variable）。

```c
#include<stdio.h>
void var();
int val = 10;
int main()
{
    bar();
    bar();
    return 0;
}
void bar()
{
    printf("%i\n", val++);
}
```

第一次打印10，第二次打印11。

### 多文件下使用全局变量 - extern

```c
//文件A
#include<stdio.h>
void var();
//int val = 10;
extern int val;
int main()
{
    bar();
    bar();
    return 0;
}
void bar()
{
    printf("%i\n", val++);
}
//文件B
int val = 10;
```

1. 只需要声明，不用赋值。
2. extern声明之后不会再次分配额外的空间
3. extern在编译时不检查。
4. extern在链接时会检查其他文件中实际有没有这个变量。

## 更优雅的全局变量 - 静态变量

### static修饰变量和函数的作用

- **static修饰变量：** 在函数内部使用static修饰变量时，该变量称为静态局部变量。它的生命周期延长到整个程序运行期间，但作用域仅限于定义它的函数内部。静态局部变量在每次函数调用时不会被重新初始化，而是保留上一次调用结束时的值。

```c
#include <stdio.h>

void exampleFunction()
{
    // 静态局部变量
    static int counter = 0;
    counter++;
    printf("Counter: %d\n", counter);
}

int main()
{
    exampleFunction();  // 输出：Counter: 1
    exampleFunction();  // 输出：Counter: 2
    exampleFunction();  // 输出：Counter: 3
    
    return 0;
}
```

在这个例子中，静态局部变量`counter`在每次函数调用之间保留其值，而不会被重新初始化。

- **static修饰函数：** 在函数声明或定义时使用static修饰，将函数的链接属性变为内部链接，限制其作用域在当前源文件内。这样，其他源文件无法调用这个静态函数。

```c
#include <stdio.h>

// 静态函数声明，限制其作用域在当前源文件内
static void staticFunction()
{
    printf("This is a static function.\n");
}
int main()
{
    staticFunction();  // 可以调用静态函数
    return 0;
}
```

在这个例子中，使用static修饰的函数`staticFunction`只能在当前源文件内被调用。

### static声明语句什么时候执行

```c
#include<stdio.h>
void var();

int main()
{
    bar();
    bar();
    return 0;
}
void bar()
{
    static int val = 10;
    printf("%i\n", val++);
}
```

之前我一直以为static在函数第一次调用时会被执行，其他时候函数调用则跳过。经过调试，发现，static语句在第一次调用函数时也会直接跳过这条语句。这说明static变量在main函数执行前就被初始化了。

### static的本质

1. 本质上还是一个全局变量
2. 在应用程序建立的时候，静态变量就创建好了。
3. 在程序中写的意义只是说明变量名字的可见范围。**如果写在大括号（不仅是函数体大括号，所有大括号都算）里，则出了大括号后就不认识该标识符**；如果写在main函数外，则可见范围仅限于此文件，所有函数都可见。
   1. 如果函数里、函数外定义了同名static变量则：就近原则，优先选择本函数里的！

```c
#include<stdio.h>
void fun()
{
	static int val = 1;
}
int main()
{
	printf("val: %i\n", val);  // error, can't find val
}
```

```c
#include<stdio.h>
static int val = 34;
void fun()
{
	static int val = 1;
}
int main()
{
	fun();
	printf("val: %i\n", val);  // 函数外的val，不认识fun函数里的val
}
```

```c
#include<stdio.h>
static int val = 34;
void fun()
{
	static int val = 1;
	val = 2;  // 优先选择本函数里的val
}
int main()
{
	fun();
	printf("val: %i\n", val); // 打印34。读取的是函数外的val，不认识fun函数里的val
}

```

```c
#include<stdio.h>
static int val = 34;
void fun()
{
	{
		static int val = 1;
	}
	val = 2;  // 只认识函数外的val，函数里的上面这个大括号里的val不认识
}
int main()
{
	fun();
	printf("val: %i\n", val); // 函数外的val，不认识fun函数里的val
}

```

```c
#include<stdio.h>
static int val = 34;
void fun()
{
	static int val = 1;
	val = 2;  // 函数外的val
}
int main()
{
	static int val = 33;
	fun();
	printf("val: %i\n", val); // 打印33，优先选择本函数里的val
}

```

# 宏

```c
#include<stdio.h>
#define PI 3.14
int main()
{
    int i = PI;
    return 0;
}
```

# 常变量

如果PI赋予了普通的int型变量`i`，那么`i`可能会被篡改。这时可以用常变量修饰`i`。

```c
#include<stdio.h>
#define PI 3.14
int main()
{
    //int i = PI;
    //i = 9.78;
    const int i = PI; // or: int const i = PI;
    return 0;
}
```

但是常变量不建议这么使用。而是如下使用：

```c
#include<stdio.h>
const float PI = 3.14f;
int main()
{
    float i = PI;
    return 0;
}
```

好处：

1. 宏能做的，常变量都可以做到。
2. 宏不能做到的：变量类型检查，常变量可以做。
   1. 虽然`3.14f`可以正确地表达float字面常量
   2. 但是`90`却不能正确地表达short字面常量，char、short诸如此类都是整型兼容的，形式与整型无区别，所以无法在宏定义时把类型信息附加上，因此丢失了类型信息。

# 其他总结

1. **#define宏定义和const的区别：**
    - **#define宏定义：** 使用#define创建的宏是一种简单的文本替换机制。在预处理阶段，所有的宏名称都会被对应的文本替换，没有类型信息。例如：`#define PI 3.14`。
    - **const关键字：** const用于创建常量，具有类型信息，并且在编译时进行类型检查。例如：`const double PI = 3.14;`。const创建的常量在内存中有分配空间，而宏定义只是简单的文本替换。
2. **定义和声明的关系：**
    - **定义：** 定义是创建一个变量、函数、或其他实体，并分配相应的存储空间。例如：`int x = 5;`。
    - **声明：** 声明是告诉编译器某个实体的存在而不进行实际的创建。例如：`extern int x;`。
    - **关系：** 定义包含声明，但声明不一定包含定义。如果在某个文件中声明了一个变量，而在另一个文件中定义了该变量，那么编译时需要链接这两个文件。
