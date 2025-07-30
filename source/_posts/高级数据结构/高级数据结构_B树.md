---
title: 高级数据结构_B树
categories:
  - - 高级数据结构
    - 树
tags: 
date: 2022/3/24
updated: 
comments: 
published:
---
# B树
B代表Balanced，所有叶子节点都在同一层。
# B树代码示例

```cpp
#include<iostream>
using namespace std;
#define M 5
#define MAXSIZE (M-1)
#define MINSIZE (M/2)
typedef char KeyType;
typedef struct Record {} Record;
typedef struct ElemType
{
	KeyType key;
	Record* recptr;
}ElemType;
typedef struct BNode
{
	int num;
	struct BNode* parent;
	ElemType data[M + 1];
	struct BNode* sub[M + 1];
}BNode;
typedef struct BTree
{
	struct BNode* root;
	int cursize;
}BTree;
typedef struct Result
{
	struct BNode* pnode;
	int index;
	bool tag;
}Result;
BNode* Buynode()
{
	BNode* s = (BNode*)malloc(sizeof(BNode));
	if (nullptr == s)exit(1);
	memset(s, 0, sizeof(BNode));
	return s;
}
void Init_BTree(BTree& tree)
{
	tree.root = nullptr;
	tree.cursize = 0;
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
BNode* MakeRoot(const ElemType& item, BNode* left, BNode* right)
{
	BNode* s = Buynode();
	s->num = 1;
	s->parent = nullptr;
	s->data[1] = item;
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
bool Insert(BTree& tree, const ElemType& item)
{
	if (tree.root == nullptr)
	{
		tree.root = MakeRoot(item, nullptr, nullptr);
		tree.cursize = 1;
		return true;
	}
	//不为空
	Result res = FindKey(tree, item.key);
	if (res.pnode != nullptr && res.tag)return false;
	//在res.pnode处插入
	BNode* ptr = res.pnode;
	int pos = res.index;
	Insert_Item(ptr, pos, item, nullptr);

	if (ptr->num > MAXSIZE)
	{
		BNode* newroot = Splice(ptr);	//分裂
		if (newroot != nullptr)
		{
			tree.root = newroot;
		}
	}
	tree.cursize += 1;
	return true;
}
BNode* FindPrev(BNode* ptr, int pos)
{
	BNode* p = ptr->sub[pos];
	while (p != nullptr && p->sub[p->num])
	{
		p = p->sub[p->num];
	}
	return p;
}
BNode* FindNext(BNode* ptr, int pos)
{
	BNode* p = ptr->sub[pos];
	while (p != nullptr && p->sub[0])
	{
		p = p->sub[0];
	}
	return p;
}
bool Remove(BTree& tree, KeyType kx)
{
	Result res = FindKey(tree, kx);
	if (res.pnode == nullptr || !res.tag)return false;
	BNode* ptr = res.pnode;
	int pos = res.index;
	BNode* pre = FindPrev(ptr, pos - 1);
	BNode* nt = FindNext(ptr, pos);
	if (pre != nullptr && ptr->num > MINSIZE)
	{
		ptr->data[pos] = pre->data[ptr->num];
		ptr = pre;
		pos = pre->num;
	}
	else if (nt != nullptr && nt->num > MINSIZE)
	{
		ptr->data[pos] = nt->data[1];
		ptr = nt;
		pos = 1;
	}
	else if (pre != nullptr)
	{
		ptr->data[pos] = pre->data[pre->num];
		ptr = pre;
		pos = pre->num;
	}
	Delete_Leaf(ptr, pos);
	if (ptr->parent == nullptr && ptr->num == 0)
	{
		free(ptr);
		tree.root = nullptr;
	}
	else if (ptr->num < MINSIZE)
	{
		BNode* newroot = Merge_Leaf(ptr);
		if (newroot != nullptr)
		{
			tree.root = newroot;
		}
	}
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

# `B*`

与B+树的结构一致。

但是

```cpp
bool Move_Left_Right(BNode * ptr)
{
    bool res = false;	//是否移动
    BNode * left = ptr->prev;
    BNode * right = ptr->next;
    if(left != nullptr && left->num < LEAFMAX)
    {
        //左移
        res = true;
    }else if(right != nullptr && right->num < LEAFMAX)
    {
        //右移
        res = true;
    }
    return res;
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
        // 移动
		BNode* newroot = Splice_Leaf(ptr);	//分裂
		if (newroot != nullptr)
		{
			tree.root = newroot;
		}
	}
	tree.cursize += 1;
	return true;
}
```





