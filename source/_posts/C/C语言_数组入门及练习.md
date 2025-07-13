---
title: C语言_数组入门及练习
categories:
  - - C
tags:
  - tulun
date: 2021/7/17
updated: 
comments: 
published:
---
本章内容：
1. 一维数组的定义和初始化
2. 一维数组在内存中的存储
3. 一维数组的使用
4. 一维数组的应用实例
5. 二维数组的定义和初始化
6. 二维数组在内存中的存储
7. 二维数组的使用
8. 二维数组的应用实例
# 数组的定义和初始化

数组是包含给定类型的一组数据，并将这些数据依次存储在连续的内存空间中。每个独立的数据被称为数组的元素（element）。元素的类型可以是任意类型。 数组本身也是一个结构，其类型由它的元素类型延伸而来。更具体地说，数组的类型由**元素的类型和数量**所决定。如果一个数组的元素是 T 类型，那么该数组就称为“T 数组”。例如，如果元素类型为 int，那么该数组的类型就是“int 数组”。然而，int 数组类型是不完整的类型，除非指定了数组元素的数量。如果一个 int 数组有 16 个元素，那么它就是一个完整的对象类型，即“16 个 int 元素数组”。

## 一维数组的定义和初始化

数组的定义决定了数组名、元素类型以及元素个数。
其语法如下：`<类型> 数组名 [ <元素数量> ];`
元素数量在方括号`[ ]`之间，它必须是大于`0`的整型常量表达式。

```c
#include<stdio.h>
int main()
{
    int ar[10]; //elem num;elem type;
    int br[10] = {};//对数组的10个元素都初始化为0。
    int cr[10] = { 12,23,34 };//后面的值为0
    int a = 10;
    sizeof(a);//4
    sizeof(ar);//4*10=40
    int dr[] = { 12,23,34,45,56,67,78 };//方括号内可以不写，但花括号中的数据必须得写。则元素数为sizeof(dr)/sezeof(dr[0])。sizeof是在编译时期计算的，则dr数组的大小是编译时期就确定了。
}
```

### 总结

数组的类型有数组元素的类型、数组元素的个数。数组的元素个数可以通过sizeof 计算得到。

### 不同C标准

C99可以拿将要用键盘输入的变量作为方括号内的常量，但VS这样的C11不可以。

```c
int n=0;
scanf("%d",&n);
int ar[n];
return 0;
```

# 数值在内存的表

```c
int main()
{
    int ar[10]={12,23,34,45,56,67,78,89,100};//0x0c~0x64
}
```

![image-20210717150102235](../../images/C%E8%AF%AD%E8%A8%80_%E6%95%B0%E7%BB%84/image-20210717150102235.png)

# 一维数组

## 一维数组的使用

数组在存储单元中是顺序连续存放的，任何一个元素都可以单独访问，其标识方法是用数组名和下标;
`数组名[整型表达式];`

整型表达式**可以是变量，也可是常量，但必须是整型类型**。

数组的访问（读取，写入）与数组的定义不一样。

```c
int main()
{
    int ar[10];
    for(int i=0;i<10;++i)
    {
        ar[i]=i+10;
    }
    for(int i=0;i<10;++i)
    {
        printf("%d ",ar[i]);
    }
    printf("\n");
    return 0;
}
```

```c
//为了便于代码的维护
int main()
{
    const int n=5;
    int ar[n];
    for(int i=0;i<n;++i)
    {
        ar[i]=i+10;
    }
    for(int i=0;i<n;++i)
    {
        printf("%d ",ar[i]);
    }
    printf("\n");
    return 0;
}
```

const关键字在编译时期的替换是对于cpp文件而言的。对于`.c`文件，const就不适用了！
```c
const int m;//无意义
const int br[10];//也不可以。但是const int br[10]={};可以
//说明const修饰的变量必须要在定义时初始化值。
```
### 其他类型的数组

#### 常性数组
```c
const int ar[5]={1,2,3,4,5};//常性数组，数组元素的值只可读，不可写。
```
#### 存放字符（串）的数组
```c
#include<stdio.h>
int main()
{
    //以下两个数组效果一样，未存放的位置都是'\0'
    char str1[10]={"tulun"};
    char str2[10]={'t','u','l','u','n'};
    //而以下两个数组就不一样了，尤其是在被表示为字符串格式化输出时：
    char stra[]={"tulun"};//定界符""表示存入的数据是字符串，隐形在尾部有'\0'，数组元素个数被开辟6个
    char strb[]={'t','u','l','u','n'};//是一个个存的，数组元素个数被开辟5个，后面没有'\0'
    sizeof(stra);//6
    sizeof(strb);//5
    
    printf("%s \n",stra);//将输出"tulun"
    printf("%s \n",strb);//将输出：tulun????????...因为程序将从strb这个首地址开始一个一个输出，直到找到内存中存的数值为0才结束
}
```
#### 存放指针的数组
```c
int main()
{
    //每个元素是某一指针类型//是存放指针的数组
    int* par[10]={NULL};
    char* pstr[10]={NULL};
}
```
## 一维数组的应用实例
### 查表法
查表法是将一些事先计算好的结果，存储在常量数组中，用到是直接按下标取数据，以节省运行时的计算时间。是以空间换时间。
```c
//斐波那契数列
#include<stdio.h>
int main()
{
    const int n=30;
    int ar[n]={1,1};
    ar[0]=ar[1]=1;
    for(int i=2;i<n;++i)
    {
        ar[i]=ar[i-1]+ar[i];
    }
    for(int i;i<n;++i)
    {
        printf("%d ",ar[i]);
    }
}
```
# 二维数组
## 二维数组的定义
## 二维数组的逻辑和物理（内存）表示
## 二维数组的使用
# 数组与函数
## 一维数组作为函数的实参
## 二维数组作为函数的实参
# 有关指针
## `int *p[n]`
## `int (*p)[n]`
## 对比
# 不同C标准
1. C99可以拿将要用键盘输入的变量作为方括号内的常量，但VS这样的C11不可以。
```c
int n=0;
scanf("%d",&n);
int ar[n];
return 0;
```

2. const关键字在编译时期的替换是对于cpp文件而言的。对于.c文件，const就不适用了！
# 练习
## 把ar数组中的数据赋值给br数组
```c
int main()
{
    const int n = 5;
    int ar[5] = {1,2,3,4,5};
    int br[5];
    br = ar;//能不能把ar数组中的数据赋值给br数组。你如何实现？
    return 0;
}
```

```c
//头文件是#include <string.h>,如果要从数组a复制k个元素到数组b，可以这样做
//memcpy(b,a,sizeof(int)*k);
#include <stdio.h>  
#include <string.h> 
int main()  
{
    int a[5]={0,1,2,3,4};
    int b[5];
    memcpy(b,a,sizeof(int)*3);
    for(int i = 0; i < 3; i++)
    {
        printf("%d ",b[i]);
    }
    return 0;
}
//如此，b数组变为：0,1,2,??,??
```
## 为什么数组的下标从0 开始而不是从1 开始
https://blog.csdn.net/every__day/article/details/83114080

从数组中存储的数据模型来看，下标最精确的意思是”偏移量“，`a[0]`的偏移量是0，即为首地址。`a[i]`的偏移量是`i`，寻址公式就是`a[i]_address = base_address + i*data_type_size`

如果下标从`1`开始，那对应的寻址公式`a[i]_address = base_address + (i-1) * data_type_size`
对CPU来说，每次随机访问，就多了一次运算，多发一条指令。
## 如果希望数组的下标从1 到10 而不是从0 到9，该怎么做
![image-20210718004201328](../../images/C%E8%AF%AD%E8%A8%80_%E6%95%B0%E7%BB%84/image-20210718004201328.png)
## 用查表法实现日历的一些功能
## 随机函数给数组元素赋值并且排序
定义大小为100 的整型数组，使用随机函数给数组元素赋值。数值范围`1..100`，并且排序，使用冒泡排序实现。

随机函数：链接地址： http://www.cplusplus.com/reference/cstdlib/rand/
```c
void Bubble_Sort(int* br,int n)
{
    for(int i=1;i<n;++i)
    {
        bool tag=true;
        for(int j=0;j<n-i;++j)
        {
            if(br[j]>b[j+1])
            {
                Swap_Int(&br[j],&br[j+1]);
                tag=false;
            }
        }
        if(tag) break;
    }
}
```
## 随机函数给数组元素赋值，不能重复
定义大小为100 的整型数组，使用随机函数给数组元素赋值。数值的范围是`1 .. 100`，并且不能重复。
```c
int* My_NonRepeating_RandArr(int* ar, int n)
{
    assert(ar != nullptr);
    int br[101];
    for (int i = 0;i < n + 1;++i)
    {
        br[i]=0;
    }
    int temp;
    srand(time(NULL));
    int i = 0;
    do
    {
        temp = rand() % 100 + 1;
        if (br[temp] != 1)
        {
            br[temp] = 1;
            ar[i] = temp;
            ++i;
        }
    } while (i<n);
    return ar;
}
```
## 统计字符串中每个英文字符出现的次数，不区分大小写
统计字符串中每个英文字符出现的次数，不区分大小写，只统计英文字符。
## 统计字符串中每个英文字符出现的次数，区分大小写
统计字符串中每个英文字符出现的次数，区分大小写，只统计英文字符。
## FindValue
`FindValue`，Value如果没在数组中则返回`-1`，如果在则返回下标。
```c
int FindValue(int* br, int n, int val)
{
    int i=0;
    while(i<n)
    {
        if(br[i]==val)
        {
            return i;
        }
        ++i;
        }
        return -1;
    }
int main()
{
    int br[8]={23,12,67,34,90,100,45,78};
    FindValue(br,8,34);
}

//改进后的代码
int FindValue(int* br, int n, int val)
{
    assert(br != nullptr);
    int pos=-1;
    int i=0;
    while(i<n)
    {
        if(br[i]==val)
        {
            pos = i;
        }
        ++i;
    }
    return pos;
}
//终极改进
int FindValue(int* br, int n, int val)
{
    assert(br != nullptr);
    int pos=n-1;
    while(pos>=0 && br[pos]!=val)
    {
        --pos;
    }
    return pos;
}
```
## 给超大数组随机赋值，不能重复，用哈希算法
如果第六题中数组大小改为1000000，那么数组开辟空间就很大，如果仍然按照旧的方式去做肯定是对程序不利的。考虑用哈希算法解决！
## 有序数组中FindValue，用二分查找
接第9题FindValue，如果我们的数组中的数据是按照大小顺序排列的，那么我们就不需要按顺序遍历查找，可以采取二分查找，何为二分查找，效率如何，如何二分查找？
## 二分查找的变种——斐波那契查找
[二分查找及其变种](https://www.cnblogs.com/yumingxing/p/9583154.html)：斐波那契查找的时间复杂度还是$O(\log_2 n)$，但是与折半查找相比，斐波那契查找的优点是它只涉及加法和减法运算，而不用除法，而除法比加减法要占用更多的时间。
