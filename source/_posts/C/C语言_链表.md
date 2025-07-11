---
title: C语言_链表
categories:
  - - C
  - - 基础数据结构
tags: 
date: 2024/1/17
updated: 
comments: 
published:
---
# 链表

1. 不要求地址连续，因此可以建立在堆上。
2. 插入的时间复杂度低。

## 节点定义

以下为无头节点的单链表。

```c
typedef struct _Node
{
    unsigned id;
    char val;
    struct _Node * next;
}Node, *PNode;
// 指向链表中第1个有效元素的指针，不是教科书上的头节点（头节点是无效值，仅代表标识）
PNode header = NULL;
```

## push_back

```c
void push_back(PNode node)
{
    PNode current = header;
    if(current == NULL)
    {
        header = node;
        return;
    }
    while(current->next != NULL)
    {
        current = current->next;
    }
    current->next = node;
    return;
}
```

## make_node

```c
PNode make_node(unsigned id, char val)
{
    PNode node = (PNode)malloc(sizeof(Node));
    if (node != NULL)
    {
        node->id = id;
        node->val = val;
        node->next = NULL;
    }
    return node;
}
```

测试

```c
int main()
{
    PNode node = make_node(1, 'a');
    push_back(node);
    node = make_node(2, 'b');
    push_back(node);
    node = make_node(3, 'c');
    push_back(node);
}
```

## traversal

```c
void traversal()
{
    PNode current = header;
    while(current != NULL)
    {
        printf("id: %u, val: %c\n", current->id, current->val);
        current = current->next;
    }
}
```

## `insert_after_id`

```c
void insert_after_id(unsigned id, PNode node)
{
    PNode current = header;
    while(current != NULL)
    {
        if(id == current->id) break;
        current = current->next;
    }
    if(current != NULL)
    {
        node->next = current->next;
        current->next = node;
    }
    else
    {
        push_back(node);
    }
}
```

测试

```c
int main()
{
    PNode node = make_node(1, 'a');
    push_back(node);
    node = make_node(2, 'b');
    push_back(node);
    node = make_node(3, 'c');
    push_back(node);
    traversal();
    node = make_node(4, 'd');
    insert_after_id(2, node);
    traversal();
}
//第一次遍历：1 2 3
//第二次遍历：1 2 4 3
```

## `insert_before_id`

自己写的版本

```c
void insert_before_id(unsigned id, PNode node)
{
    PNode current = header;
    if(current == NULL)
    {
        push_back(node);
        return;
    }
    if(current->id == id)
    {
        header = node;
        node->next = current;
        return;
    }
    while(current->next != NULL)
    {
        if(id == current->next->id) break;
        current = current->next;
    }
    if(current->next != NULL)
    {
        node->next = current->next;
        current->next = node;
    }
    else
    {
        push_back(node);
    }
}
```

测试

```c
int main()
{
    PNode node = make_node(1, 'a');
    push_back(node);
    node = make_node(2, 'b');
    push_back(node);
    node = make_node(3, 'c');
    push_back(node);
    traversal();
    node = make_node(4, 'd');
    insert_before_id(1, node);
    traversal();
}
//第一次遍历：1 2 3
//第二次遍历：4 1 2 3
```

双指针法

```c
void insert_before(unsigned id, PNode node)
{
    PNode p = header, pp = NULL;
    while(p)
    {
        if(p->id == id)
        {
            break;
        }
        pp = p;
        p = p->next;
    }
    if(!p)//插入的位置为空
    {
        push_back(node);
        return;
    }
    // 找到id，且p不为空
    if(p != NULL && pp == NULL)//第一次while循环即找到了id，即id在首元素，需要插到首位置，要换头。
    {
        header = node;
        node->next = p;
        return;
    }
    // 剩下就是在中间插入的情况了。
    pp->next = node;
    node->next = p;
}
```

## remove_node

删除指定id

```c
void remove_node(unsigned id)
{
    PNode current = header;
    if(header != NULL && header->id == id)
    {
        free(header);
        header = NULL;
        return;
    }
    PNode prev = NULL;
    while(current != NULL)
    {
        if(id == current->id) break;
        prev = current;
        current = current->next;
    }
    if(current != NULL)
    {
        prev->next = current->next;
        free(current);
        current = NULL;
    }
}
```

测试

```c
int main()
{
    PNode node = make_node(1, 'a');
    push_back(node);
    node = make_node(2, 'b');
    push_back(node);
    node = make_node(3, 'c');
    push_back(node);
    traversal();
    node = make_node(4, 'd');
    insert_before_id(1, node);
    traversal();
    remove_node(1);
    traversal();
    remove_node(2);
    traversal();
    remove_node(3);
    traversal();
    remove_node(4);
    traversal();
    remove_node(5);
    traversal();
}
/* 结果：
id: 1, val: a
id: 2, val: b
id: 3, val: c

id: 4, val: d
id: 1, val: a
id: 2, val: b
id: 3, val: c

id: 4, val: d
id: 2, val: b
id: 3, val: c

id: 4, val: d
id: 3, val: c

id: 4, val: d

空
*/
```

## 环

### 制造环

标记一个id，之后让尾节点next指向该id

```c
PNode find_id(unsigned id)
{
    PNode current = header;
    while(current != NULL)
    {
        if(current->id == id) break;
        current = current->next;
    }
    return current;
}
void make_cycle(unsigned id)
{
    // 确保有两个节点
    if(header == NULL || header->next == NULL) return;
    PNode cycle_begin = find_id(id);
    // 确保有这个id  并且  确保cycle_begin不是尾节点
    if(cycle_begin == NULL || cycle_begin->next == NULL) return;
    PNode current = header;
    while(current->next != NULL)
    {
        current = current->next;
    }
    current->next = cycle_begin;
}
```

### 如何判断是否有环？快慢指针

一个指针走一步，一个指针走两步。

如果两个指针重合则有环，如果无环的话，快指针肯定能到终点NULL。

```c
int has_cycle()
{
    if (header == NULL || header->next == NULL) return 0;
    PNode fast = header;
    PNode slow = header;
    while (fast != NULL)
    {
        fast = fast->next;
        if(fast != NULL)
            printf("fast1: -> %i\n", fast->id);
        if (fast != NULL)
        {
            fast = fast->next;
            if (fast != NULL)
                printf("fast2: -> %i\n", fast->id);
            slow = slow->next;
            printf("slow: -> %i\n\n", slow->id);
            if (fast == slow && fast != NULL)
            {
                fast = header;
                while (fast != slow)
                {
                    fast = fast->next;
                    slow = slow->next;
                }
                printf("有环: %i\n", slow->id);
                return 1;
            }
        }
        else
        {
            printf("无环");
            return 0;
        }
    }
}
int main()
{
    PNode node = make_node(7, '7');
    push_back(node);
    node = make_node(5, '5');
    push_back(node);
    node = make_node(6, '6');
    push_back(node);
    node = make_node(8, '8');
    push_back(node);
    node = make_node(2, '2');
    push_back(node);
    node = make_node(0, '0');
    push_back(node);
    node = make_node(4, '4');
    push_back(node);
    traversal();
    make_cycle(6);
    has_cycle();
}
// 有环：6
```

让一个快指针和一个慢指针再次相遇当然可以说明此链表有环。但，难点在于如何找到环的入口？

设$x$为头节点到循环入口处的距离，$y$为循环入口处到相遇点的距离，$z$为相遇点到尾节点的距离（即该节点下一个是循环口）。并结合$fast$指针和$slow$指针的步数关系$fast=2slow$，得出在相遇点：$fast=x+n(y+z)+y$，$slow=x+y$（慢指针肯定不会在圈内转完一圈，可以想想，是合理的，因为快指针总比慢指针快$n$圈，肯定在慢指针走完一圈前和它相遇！），则
$$
\begin{align}
x+n(y+z)+y&=2(x+y)\\
x&=n(y+z)-y\\
x&=n(y+z)-(y+z)+z\\
x&=(n-1)(y+z)+z\\
x&=z
\end{align}
$$
$y+z$是圈长，我们可以想到，如果快指针要和慢指针相遇，$n$必须要大于等于$1$。至于$n$，即圈数具体为几，实际上无所谓，因为快指针在里面转了$n$圈和$1$圈，效果是一样的！所以我们可以得出，$x=z$！于是，已知$z$为相遇点到尾节点的距离（尾节点下一个是循环口），并且已知$z=x$（$x$为头节点到循环入口处的距离）。那么，让一个指针在相遇点，另一个指针在头节点，同时步进，直到相遇，就是循环入口处！

参考：https://www.bilibili.com/video/BV1if4y1d7ob

## 保存到文件

保存所有节点信息到文件中。成功返回1，失败返回0。

不应当把结构体作为整体完全保存到一行。

而是每个属性一行一行地保存。因为next指针在下次运行失效的。需要按照每个属性重新构建每个节点。

```c
int save(char const * const filename)
{
    return 0;
}
```

加载的时候，next指针的信息已经是失效的，需要重新`make_node`。

```c
int load(char const * const filename)
{
    return 0;
}
```

