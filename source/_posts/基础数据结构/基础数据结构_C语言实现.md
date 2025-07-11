---
title: 基础数据结构_C语言实现
categories:
  - - C
  - - 基础数据结构
tags:
  - null 
date: 2021/9/2
update:
comments:
published:
---
# 内容

1. 数据结构的基础概念和时间空间复杂度
2. 线性表：顺序表、链表
   1. 顺序表（定长的、扩容的）
   2. 单链表（带头结点的、不带头结点的）
      1. 需要动手实践，大量的面试题
3. 栈（顺序栈、链栈）
   1. 用两个栈实现一个队列
4. 队列（顺序队、链队）
   1. 用两个队实现一个栈
5. 串（字符数组）
   1. API实现（strcpy strcat srtcmp strlen）
   2. 字符串的查找（BF、KMP算法）
6. 双向链表、循环链表、双向循环链表
7. 八大排序算法
   1. 冒泡排序
   2. 二路归并排序
   3. 插入排序
   4. 希尔排序
   5. 堆排序
   6. 基数排序（桶排序）
   7. 快速排序
   8. 选择排序

# 概念

## 数据结构

1. 集合
2. 线性（现实生活中的排队）
3. 树型（部门架构、族谱）
4. 图型（交通网络、六度分割理论、网络拓扑）

## 时间复杂度

执行语句与问题规模之间的函数

1. O(1): 没有循环，或者有循环但是循环的退出条件和问题规模之间没有关系；
2. O(n): 有循环，循环的退出条件与问题规模之间存在关系，并且控制循环的变量以++或者--的方式执行
3. O(n^2): 有循环，嵌套一层。
4. O(logn): 有循环，循环的退出条件与问题规模之间存在关系，并且控制循环的变量每次\*2或/2的方式执行。

## 空间复杂度

算法所需的额外存储空间（除了函数本身计算申请的内存外）与问题规模之间的关系。

```c
void Show(int arr[], int len)
{
    for(int i = 0;i<len;++i)
    {
        printf("%d ",arr[i]);
    }
}
```

此处，函数参数中的int arr[]和int len不是额外存储空间。而是for循环。

# 线性表

* 唯一的头（此处的头指的是第一个有效节点），唯一的尾。
* 除了头，剩余的节点都有前驱。除了尾，剩余的节点都有后继。

## 定长顺序表

相当于一个特殊使用的数组。

但存放数据是按顺序来存的，不能随意位置存放。

```c
//sqlist.h
#pragma once	//防止头文件重复
typedef int ElemType;
typedef struct Sqlist
{
    ElemType arr[10];//节点
    int length;//当前有效数据节点的个数
}Sqlist,*PSqlist;
//初始化
void Init_sqlist(PSqlist plist);
//增删改查
//按位置插入（头插、尾插）
bool Insert_Pos(PSqlist plist, int pos, int val);
//删除：按值删、按位置删（删除这个值第一次出现的位置）
bool Del_val(PSqlist plist, int val);
bool Del_pos(PSqlist plist, int pos);
//查找值为val的节点
Sqlist* FindNode(PSqlist plist, int val);
//判空 判满
bool IsEmpty(PSqlist plist);
bool IsFull(PSqlist plist);
//清空
void Clear(PSqlist plist);
//销毁
void Destory(PSplist plist);
```

```c
#include<stdio.h>
#include<assert.h>
#include<stdlib.h>
#include"sqlist.h"
#include<string.h>
//初始化
void Init_sqlist(PSqlist plist)
{
    assert(plist!=NULL);
    memset(plist->arr,0,sizeof(arr));
    plist->length = 0;
}
//增删改查
//按位置插入（头插、尾插）
bool Insert_Pos(PSqlist plist, int pos, int val)
{
    assert(plist!=NULL);
    if( pos<0 || pos > plist->length || IsFull(plist) )return false;
    
    //移动数据
    for(int i = plist->length-1; i>=pos ;i--)
    {
        //不会越界，因为如果length为10时，不能进入此循环
        plist->arr[i+1] = plist->arr[i];
    }
    //插空
    plist->arr[pos] = val;
    //长度加1
    plist->length++;
    return true;
}
//头插
bool Push_Front(PSqlist plist, int val)
{
    assert(plist!=NULL);
    return Insert_Pos(plist, 0, val);
}
//尾插
bool Push_Back(PSqlist plist, int val)
{
    assert(plist!=NULL);
    return Insert_Pos(plist, plist->length, val);
}
//删除：（删除这个值第一次出现的位置）
//按值删
bool Del_val(PSqlist plist, int val)
{
    assert(plist!=NULL);
    return Del_pos(plist, FindPos_withVal(plist, val) );
}
//按位置删
bool Del_pos(PSqlist plist, int pos)
{
    assert(plist!=NULL);
    if( pos<0 || pos > plist->length || IsEmpty(plist) )return false;
    //长度减1
    plist->length--;
    //移动数据（相当于清除了arr[pos]）
    for(int i = pos; i < plist->length-1 ;++i)//i < plist->length-1  容易出错
    {
        plist->arr[i] = plist->arr[i+1];
    }
    return true;
}
//头删
bool Pop_Front(PSqlist plist)
{
    assert(plist!=NULL);
    Del_pos(plist, 0);
}
//尾删
bool Pop_Back(PSqlist plist)
{
    assert(plist!=NULL);
    Del_pos(plist, plist->length);
}
//查找值为val的节点
int FindPos_withVal(PSqlist plist, int val)
{
    assert(plist!=NULL);
    int pos = -1;
    int i = 0;
    while(i< plist->length && val!=plist->arr[i])
    {
        ++i;
    }
    if(i!=length)//则while是因为val==plist->arr[i]跳出的。
    {
        pos = i;
    }
    return pos;
}
//判空 判满
bool IsEmpty(PSqlist plist)
{
    return plist->length == 0;
}
bool IsFull(PSqlist plist)
{
    return plist->length == sizeof(plist->arr)/sizeof(plist->arr[0]);
}
//清空
void Clear(PSqlist plist)
{
    assert(plist!=NULL);
    plist->length = 0;
}
//销毁
void Destory(PSplist plist)
{
    assert(plist!=NULL);
    Clear(plist);
}
```

## 不定长顺序表

```c
//sqlist.h
#pragma once	//防止头文件重复
typedef int ElemType;
#define INIT_SIZE 10
typedef struct DSqlist
{
    ElemType* ar;//基地址
    int cursize;//当前有效数据节点的个数
    int capacity;//当前最长容量
}DSqlist,*PDSqlist;
//初始化
void Init_sqlist(PDSqlist plist);
//增删改查
//按位置插入（头插、尾插）
bool Insert_Pos(PDSqlist plist, int pos, int val);
//头插
bool Push_Front(PDSqlist plist, int val);
//尾插
bool Push_Back(PDSqlist plist, int val);
//删除：按值删、按位置删（删除这个值第一次出现的位置）
bool Del_val(PDSqlist plist, int val);
bool Del_pos(PDSqlist plist, int pos);
//头删
bool Pop_Front(PDSqlist plist);
//尾删
bool Pop_Back(PDSqlist plist);
//查找值为val的节点
int FindPos_withVal(PDSqlist plist, int val);
//判空 判满
bool IsEmpty(PDSqlist plist);
bool IsFull(PDSqlist plist);
//清空
void Clear(PDSqlist plist);
//销毁
void Destory(PDSplist plist);
void Inc(PDsqlist plist);
```

```c
void Inc(PDsqlist plist)
{
    
}
```

# 单链表

![image-20210902180653639](../../images/C%E8%AF%AD%E8%A8%80_%E5%9F%BA%E7%A1%80%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20210902180653639.png)



![image-20210902181055988](../../images/C%E8%AF%AD%E8%A8%80_%E5%9F%BA%E7%A1%80%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20210902181055988.png)

1. 第一种大厂笔试/面试中常用，主要考察对边界条件检测的掌握力；
   * 不带头结点的逆置和带头结点的逆置难度不一样。
2. 第二种实际编程中常用，方便好用；
3. 第三种的循环无意义，因为功能和头指针差不多。
4. 第四种是加了一个尾指针，使尾插也变O(1)，但代价是需要对尾指针进行密切维护。



# 面试

![image-20210904191948363](../../images/C%E8%AF%AD%E8%A8%80_%E5%9F%BA%E7%A1%80%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/image-20210904191948363.png)
