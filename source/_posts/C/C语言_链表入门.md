---
title: C语言_链表
categories:
  - - C
  - - 基础数据结构
tags:
  - tulun
date: 2021/8/20
updated: 
comments: 
published:
---
# 单链表

## 不带头

```c
typedef int ElemType;
typedef struct ListNode
{
    ElemType data;
    struct ListNode* next;
}ListNode;
typedef struct
{
	ListNode* first;
	int cursize;
}LinkList;
//1.初始化
void InitList(LinkList* plist);
//2.清空链表
void ClearList(LinkList* plist);
//3.销毁链表
void DestroyList(LinkList* plist);
//4.获取链表的数据元素的个数
int GetSize(const LinkList* plist);
//5.链表判空
bool IsEmpty(const LinkList* plist);
//6.在ptr指针之后插入数据元素
bool Insert_Next(LinkList* plist, ListNode* ptr, ElemType val);
//7.头插数据元素
void Push_Front(LinkList* plist, ElemType val);
//8.尾插数据元素
void Push_Back(LinkList* plist, ElemType val);
//9.删除ptr所指向的后续结点（数据元素）
bool Erase_Next(LinkList* plist, ListNode* ptr);
//10.删除第一个数据结点
void Pop_Front(LinkList* plist);
//11.删除尾部数据节点
void Pop_Back(LinkList* plist);
//12.删除与val值相等的数据节点
void Remove(LinkList* plist, ElemType val);
//13.删除与val值相等的所有数据节点
void Remove_All(LinkList* plist, ElemType val);
//14.打印链表中的数据元素
void PrintList(const LinkList* plist);
//15.返回与val值相等节点的前驱节点指针
ListNode* FindValue_pre(const LinkList* plist, ElemType val);
//16.返回与val值相等节点指针
ListNode* FindValue(const LinkList* plist, ElemType val);
//17.返回pos位置的前驱节点指针
ListNode* FindPos_pre(const LinkList* plist, int pos);
//18.返回pos位置的节点指针
ListNode* FindPos(const LinkList* plist, int pos);
```

```c
//1.初始化
void InitList(LinkList* plist);
//2.清空链表
void ClearList(LinkList* plist);
//3.销毁链表
void DestroyList(LinkList* plist);
//4.获取链表的数据元素的个数
int GetSize(const LinkList* plist);
//5.链表判空
bool IsEmpty(const LinkList* plist);
//6.在ptr指针之后插入数据元素
bool Insert_Next(LinkList* plist, ListNode* ptr, ElemType val);
//7.头插数据元素
void Push_Front(LinkList* plist, ElemType val);
//8.尾插数据元素
void Push_Back(LinkList* plist, ElemType val);
//9.删除ptr所指向的后续结点（数据元素）
bool Erase_Next(LinkList* plist, ListNode* ptr);
//10.删除第一个数据结点
void Pop_Front(LinkList* plist);
//11.删除尾部数据节点
void Pop_Back(LinkList* plist);
//12.删除与val值相等的数据节点
void Remove(LinkList* plist, ElemType val);
//13.删除与val值相等的所有数据节点
void Remove_All(LinkList* plist, ElemType val);
//14.打印链表中的数据元素
void PrintList(const LinkList* plist);
//15.返回与val值相等节点的前驱节点指针
ListNode* FindValue_pre(const LinkList* plist, ElemType val);
//16.返回与val值相等节点指针
ListNode* FindValue(const LinkList* plist, ElemType val);
//17.返回pos位置的前驱节点指针
ListNode* FindPos_pre(const LinkList* plist, int pos);
//18.返回pos位置的节点指针
ListNode* FindPos(const LinkList* plist, int pos);
```

