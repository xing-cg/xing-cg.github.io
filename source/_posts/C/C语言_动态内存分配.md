---
typora: ../..
title: C语言_动态内存分配
categories:
  - [C]
tags:
  - null 
date: 2021/9/2
update:
comments:
published:
---
# malloc

1. 返回类型为void *
2. 参数为(unsigned int size)，表示总共要开辟的字节数

```c
int* p = nullptr;
int n = 0;
scanf_s("%d", &n);
//p = (int*)malloc(n);//这是错误的参数，因为malloc的参数要写要开辟的字节数，而不是以变量数为单位。
p = (int*)malloc(sizeof(int) * n);
```



# calloc

1. 返回类型为*void
2. 参数为(unsigned int count, unsigned int size)，count表示变量数，size表示单个变量(类型)的字节数。

```c
#include<stdio.h>
#include<stdlib.h>
#include<string.h>//包含memset函数
typedef unsigned int size_t;
void* My_Calloc(size_t count, size_t size)
{
    void* p = malloc(count * size);
    if(nullptr != p)
    {
        memset(p, 0, count * size);//把p指向的内存空间中的数据赋值为0x00
    }
    return p;
}
```

# free

1. 不管是malloc还是calloc开辟的空间都是用free(p)来释放的。

2. free并不是把p释放掉了，而是把p指向堆区的空间从引用状态转换成未引用状态。

3. 空间的释放只能释放一次，不能释放多次

4. free(p)释放后，要给p赋值nullptr，防止指针失效带来的麻烦。

   ```c
   int* p = (int*)My_Calloc(10, sizeof(int));
   free(p);
   p = nullptr;
   return 0;
   ```

   

# realloc

realloc扩充或收缩之前分配的内存块（重新分配内存块）

```c
void *realloc(void *ptr, size_t new_size);
```
1. 参数中的ptr指向需要重新分配的内存区域的地址，new_size为新的字节数
2. 返回值：成功时，返回指向新分配内存的指针，返回的指针必须用free或realloc归还。；失败时，返回空指针。原指针ptr保持有效，并需要通过free或realloc归还。

重新分配给定的内存区域。它必须是之前为malloc/calloc/realloc分配的，并且仍未被free或realloc的调用所释放。否则，结果未定义。

```c
int size=5;//分配空间的变量数为5
int new_size=10;//最终要分配空间的变量数为10
int* p = (int*)malloc(sizeof(int)*size);//开辟size个int空间
for(int i=0;i<size;++i)
{
    p[i]=1;
}
p = (int*)realloc(p, sizeof(int) * new_size);//增加到new_size个int空间，即由原来5个扩充到10个
p = (int*)realloc(p, sizeof(int) * 2);//减少到2个int空间，即由原来5个收缩到2个
```

