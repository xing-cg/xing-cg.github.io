---
title: makefile
typora-root-url: ../..
categories:
  - - 项目
  - - Linux
tags: 
date: 2022/5/12
updated: 
comments: 
published:
---

# 内容

1. makefile的编写
1. 命令

# 示例

示例所用到的代码内容：

```c
//main.c
#include<stdio.h>
int main()
{
    int a = 2;
    int b = 3;
    printf("a+b = %d\n",add(a,b));
    printf("Max = %d\n",max(a,b));
    return 0;
}
```

```c
//add.c
int add(int x, int y)
{
    return x+y;
}
```

```c
//max.c
int max(int x, int y)
{
    return x > y ? x : y;
}
```

# makefile编写

```bash
# Makefile
all : main

main : main.o add.o max.o
	gcc -o main main.o add.o max.o
	
main.o : main.c
	gcc -c main.c

add.o : add.c
	gcc -c add.c
	
max.o : max.c
	gcc -c max.c
	
clean:
	rm -f *.o main
```

# 命令

```bash
make makefile	#make命令--读取makefile文件
```

