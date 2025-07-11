---
title: 高级数据结构_Bplus树
categories:
  - [高级数据结构, 树]
tags:
  - null 
date: 2022/6/18
update:
comments:
published:
---

# B+

B+树可以看作是B树的一种变形，在实现文件索引结构方面比B树使用得更普遍。

## B+树与B树区别

m阶`B+`树与m阶B树区别: 

1. 所有叶子节点包含全部关键字信息, 及指向含有这些关键字记录的指针, 且叶子节点中关键字进行有序链接; 
2. 非叶子节点相当于是叶子节点的索引, 叶子节点相当于是存储(关键字)数据的数据层; 

## B+树

一棵m阶B+树可以定义如下：

* 树中每个非叶结点最多有m棵子树;
* 根结点（非叶结点）至少有2棵子树。除根结点外，其它的非叶结点至少有${\lceil}m/2{\rceil}$棵子树;有n棵子树的非叶结点有n-1个关键码。
* 所有的叶结点都处于同一层次上，包含了全部关键码及指向相应数据对象存放地址的指针，且叶结点本身按关键码从小到大顺序链接;

# B+树的优势

## 读写代价更低

B+树的磁盘读写代价更低, B+树的内部节点没有指向具体数据的指针, 因此内部节点相对B树更小, 如果把同一内部节点的关键字放在同一块磁盘中, 盘块所能容纳的关键字数量也就越多, 一次性读入内存中的需要查找的关键字也就越多, 相对IO读写次数减少, 性能更高;

## 查询效率更加稳定

非叶节点并不是最终指向文件内容的节点, 只作为叶子节点中关键字的索引; 所以任何关键字的查找必须走一条从根节点到叶子节点的路; 所有关键字查询的路径长度相同, 于是每一个数据的查询效率基本等同; 

## 统计数据手段更灵活

由于B+树把数据都放到了最底一层, 可以很方便的把这些数据一个个链接起来形成链表, 加上B+树数据本来就是按序存放的, 这样的话有利于整表搜索或区间查找, 比B树的区间搜索快很多;

## 举个例子

假设磁盘中的一个盘块容纳16bytes, 而一个关键字2bytes, 一个关键字具体信息指针2bytes; 一棵9阶BTree的内部节点需要2个盘块; 而B+Tree内部节点只需要1个盘块, 当需要把内部节点读入内存的时候, B树就比B+树多一次盘块查找时间, 在磁盘中就是盘片旋转的时间; (这就是省了指针, 只存索引项的好处)

# B+树节点结构

```cpp
#define M 5
#define MAXSIZE (M-1)
#define MINSIZE (M/2)
typedef char KeyType;
typedef struct Record {} Record;
typedef enum{BRCH = 1, LEAF = 2}NodeType;
typedef struct BNode
{
	int num;					//包含元素的个数
    BNode * parent;
	NodeType utype;				//LEAF, BRCH
    KeyType key[M + 1];
/*
    //LEAF
    Record * recptr[M + 1];
    BNode * prev, *next;
    //BRCH
    BNode * sub[M + 1];
*/
    union
    {
        struct	//LEAF
        {
        	Record * recptr[M + 1];
            BNode * prev, * next;
        };
        //BRCH
        BNode * sub[M + 1];
    };
    struct BNode* parent;	
	ElemType data[M + 1];
	struct BNode* sub[M + 1];
}BNode;
```

B+树和B树的分支结构一样，但叶子结构不一样。

除了上述结构。还有一些规定

```cpp
#define BRCHMAX (M-1)
#define BRCHMIN (M/2)
#define LEAFMAX (M)
#define LEAFMIN (M/2+1)
```

# B+树的Insert

```cpp
#include<iostream>
using namespace std;
#define M 5
#define BRCHMAX (M-1)
#define BRCHMIN (M/2)
#define LEAFMAX (M)
#define LEAFMIN (M/2+1)
typedef char KeyType;
typedef struct Record {} Record;
typedef enum{BRCH = 1, LEAF = 2}NodeType;
typedef struct BNode
{
	int num;					//包含元素的个数
    BNode * parent;
	NodeType utype;				//LEAF, BRCH
    KeyType key[M + 1];
/*
    //LEAF
    Record * recptr[M + 1];
    BNode * prev, *next;
    //BRCH
    BNode * sub[M + 1];
*/
    union
    {
        struct	//LEAF
        {
        	Record * recptr[M + 1];
            BNode * prev, * next;
        };
        //BRCH
        BNode * sub[M + 1];
    };
    struct BNode* parent;	
	ElemType data[M + 1];
	struct BNode* sub[M + 1];
}BNode;

BNode* Buynode()
{
	BNode* s = (BNode*)malloc(sizeof(BNode));
	if (nullptr == s)exit(1);
	memset(s, 0, sizeof(BNode));
	return s;
}
BNode* BuyLeaf()
{
    BNode * s = Buynode();
    s->parent = nullptr;
    s->utype = LEAF;
    return s;
}
void Init_BTree(BTree& tree)
{
	tree.root = nullptr;
    tree.first = nullptr;
	tree.cursize = 0;
}
Result FindRoot(BNode * ptr, KeyType kx)
{
    Result res = {nullptr, -1, false};
    BNode * p = ptr;
    while(p!=nullptr&&p->utype==BRCH)
    {
        p->key[0] = kx;
        int i = p->num;
        while(kx < p->key[i]) --i;
        p = p->sub[i];
    }
    res = FindLeaf(p, kx);
    return res;
}
Result FindLeaf(BNode * ptr, KeyType kx)
{
    Result res = {nullptr, -1, false};
    BNode * p = ptr;
    while(p!=nullptr&&p->next!=nullptr && kx > p->key[p->num -1])
    {
        p = p->next;
    }
    if(p == nullptr)return res;
    int pos = p->num -1;
    while(pos >= 0 && kx < p->key[pos])
    {
        --pos;
    }
    res.pnode = p;
    res.index = pos;
    if(pos < 0 && p->prev != nullptr)
    {
        res.pnode = p->prev;
        res.index = p->prev->num -1;
    }else if(pos >= 0 && kx == p->key[pos])
    {
        res.tag = true;
    }
    return res;
}
Result FindKey(BTree& tree, KeyType kx)
{
	Result res = { nullptr, -1,false };
	struct BNode* ptr = tree.root;
	while (ptr != nullptr)
	{
		ptr->data[0].key = kx;
		int i = ptr->num;
		while (i>=0 && ptr->data[i].key)
		{
			--i;
		}
		res.pnode = ptr;
		res.index = i;
		if (i > 0 && kx == ptr->data[i].key) 
		{ 
			res.tag = true;
			break;
		}
		ptr = ptr->sub[i];
	}
	return res;
}
BNode* MakeRoot(const KeyType kx, BNode* left, BNode* right)
{
	BNode* s = Buynode();
	s->num = 1;
	s->parent = nullptr;
	s->key[1] = kx;		//
	s->sub[0] = left;
	if (left != nullptr)left->parent = s;
	s->sub[1] = right;
	if (right != nullptr)right->parent = s;
	return s;
}
void Insert_Item(BNode* ptr, int pos, const ElemType& item, BNode* right)
{
	for (int i = ptr->num; i > pos; --i)
	{
		ptr->data[i + 1] = ptr->data[i];
		ptr->sub[i + 1] = ptr->sub[i];
	}
	ptr->data[pos + 1] = item;	//?
	ptr->sub[pos + 1] = right;	//?
	ptr->num += 1;
}
ElemType Move_Item(BNode* s, BNode* ptr, int pos)
{
	for (int i = 0, j = pos + 1; j <= ptr->num; ++i, ++j)
	{
		s->data[i] = ptr->data[j];
		s->sub[i] = ptr->sub[j];
		if (s->sub[i] != nullptr)
		{
			s->sub[i]->parent = s;
		}
	}
	s->num = MINSIZE;
	ptr->num = MINSIZE;
	s->parent = ptr->parent;
	return s->data[0];
}
BNode* Splice(BNode* ptr)
{
	BNode* s = Buynode();
	ElemType item = Move_Item(s, ptr, MINSIZE);
	if (ptr->parent == nullptr)
	{
		return MakeRoot(item, ptr, s);
	}
	BNode* pa = ptr->parent;
	int pos = pa->num;
	pa->data[0] = item;
	while (pos > 0 && item.key < pa->data[pos].key) { --pos; }
	Insert_Item(pa, pos, item, s);
	if (pa->num > MAXSIZE)
	{
		return Splice(pa);
	}
	else
	{
		return nullptr;
	}
}
void Insert_Leaf_Item(BNode * ptr, int pos, KeyType kx, Record *rec)
{
    for(int i = ptr->num-1; i>pos; --i)
    {
        ptr->key[i+1] = ptr->key[i];
        ptr->recptr[i+1] = ptr->recptr[i];
    }
    ptr->key[pos+1] = kx;
    ptr->recptr[pos+1] = rec;
    ptr->num+=1;
}
KeyType Move_Leaf_Item(BNode *s, BNode* ptr)
{
    for(int i = 0;j = LEAFMIN;j<ptr->num;++i,++j)
    {
        s->key[i] = key[j];
        s->recptr[i] = ptr->recptr[j];
    }
    s->num = LEAFMIN;
    ptr->num = LEAFMIN;
    s->parent = ptr->parent;
    s->next = ptr->next;
    s->prev = ptr;
    ptr->next = s;
    if(s->next != nullptr)
    {
        s->next->prev = s;
    }
    return s->key[0];
}
BNode* Splice_Leaf(BNode * ptr)
{
    BNode * s = BuyLeaf();
    KeyType kx = Move_Leaf_Item(s, ptr);
    if(ptr->parent == nullptr)
    {
        return MakeRoot(kx, ptr, s);
    }
    //
}
bool Insert(BTree& tree, KeyType kx, Record * rec)
{
	if (tree.root == nullptr)
	{
		BNode *s = BuyLeaf();
		s->key[0] = kx;
        s->recptr[0] = rec;
        s->num = 1;
        tree.root = tree.first = s;
        return true;
	}
	//不为空
	Result resr = FindRoot(tree.root, kx);
	Result resf = FindLeaf(tree.first, kx);
	if(resf.pnode == nullptr){cout << "BTree struct err"<<endl;return false;}
    if (resf.tag){cout << "exist"<<endl;return false;}
	//在res.pnode处插入
	BNode* ptr = resf.pnode;
	int pos = resf.index;
	Insert_Leaf_Item(ptr, pos, kx, rec);
	
	if (ptr->num > LEAFMAX)
	{
		BNode* newroot = Splice_Leaf(ptr);	//分裂
		if (newroot != nullptr)
		{
			tree.root = newroot;
		}
	}
	tree.cursize += 1;
	return true;
}

int main()
{
	BTree myt;
	Init_BTree(myt);
	char ch[] = { "qwertyuiopasdfghjklzxcvbnm" };
	int i = 0;
	while (ch[i] != '\0')
	{
		ElemType item = { ch[i],nullptr };
		cout << Insert(myt, item);
		i++;
	}
	cout << endl;
}
```

# 总结

B树的每一个节点, 存了关键字和对应的数据地址, 而B+树的非叶子节点只存关键字, 不存数据地址。因此B+树的每一个非叶子节点存储的关键字是远远多于B树的; B+树的叶子节点存放关键字和数据, 因此从树的高度上来说B+树的高度要小于B树, 使用的磁盘IO次数少, 因此查询会更快一些。

由于B树每个节点都存储关键字和数据, 因此离根节点近的数据, 查询的就快, 离根节点远的数据, 查询的就慢; B+树所有的数据都存在叶子节点上, 因此在B+树上搜索关键字, 找到对应数据的时间是比较平均的, 没有快慢之分。

如果在B树上做区间查找, 遍历的节点是非常多的; B+树所有叶子节点被连接成了有序链表结构, 因此做整表遍历和区间查找是非常容易的。