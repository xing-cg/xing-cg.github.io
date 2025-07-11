---
title: 高级数据结构_BST
typora-root-url: ../..
categories:
  - [高级数据结构, 树]
tags:
  - null 
date: 2022/5/20
update:
comments:
published:
---

# 二叉树相关概念

1. 根节点
2. 左孩子、右孩子
3. 双亲节点（父节点）
4. 祖先节点
5. 兄弟节点
6. 叔叔节点（祖先节点的另一个子节点）
7. 叶子节点
8. 左子树、右子树
9. 二叉树的高度/层数

# BST - 二叉搜索树

对于一棵二叉树，上的每一个节点如果满足`左孩子的值 < 父节点的值 < 右孩子的值`，则称作`BST(Binary Search/Sort Tree)`树，即二叉搜索树（二叉排序树）。

# 二叉搜索树节点的结构

```cpp
typedef int KeyType;
typedef struct BstNode
{
    KeyType key;
    BstNode * lchild;
    BstNode * parent;	//不必要, 但可以大大简化代码
    BstNode * rchild;
}BstNode, * BSTree;
```

# 二叉搜索树的插入

1. BST如果为空，建根。
2. BST不为空，从根节点进行比较，找到合适位置，生成新节点，新节点的地址赋给父节点的`leftchild/rightchild`。

```cpp
#include<functional>	//less<T>
using namespace std;

template<typename T, typename Compare=less<T>>
class BSTree
{
private:
    //节点定义
    struct Node
    {
        Node(T data = T())
            : data_(data), left_(nullptr), right_(nullptr)
        {}
        T data_;		//数据域
        Node *left_;	//左孩子域
        Node *right_;	//右孩子域
    };
    Node *root_;		//指向BST树的根节点
public:
    BSTree()
        : root_(nullptr)
    {
    }
    ~BSTree()
    {
    }
    //非递归插入操作
    void n_insert(const T &val)
    {
        if(root_ == nullptr)
        {
            root_ = new Node(val);
            return;
        }
        
        Node * parent = nullptr;
        Node * cur = root_;
        while(cur != nullptr)
        {
            if(val < cur->data_)
            {
                parent = cur;
                cur = cur->left_;
            }
            else if(val > cur->data_)
            {
                parent = cur;
                cur = cur->right_;
            }
            else //val == cur->data_
            {
                //不插入值相同的元素
                return;
            }
        }
        //把新节点插入到parent节点下
        if(val < parent->data_)
        {
            parent->left_ = new Node(val);
        }
        else
        {
            parent->right_ = new Node(val);
        }
    }
};
```

测试

```cpp
int main()
{
    int ar[] = {58, 24, 67, 0, 34, 62, 69, 5, 41, 64, 78};
    BSTree<int> bst;
    for(int val : ar)
    {
        bst.n_insert(val);
    }
    bst.n_insert(12);
    return 0;
}
```

## 更简洁的写法

```cpp
bool Insert(AVLNode *& tree, KeyType kx)
{
    if(tree == nullptr)
    {
        tree = MakeRoot(val);
        return true;
    }
    AVLNode * pa = nullptr;
    AVLNode * p = tree;
    while(p != nullptr && p->key != kx)
    {
        pa = p;
        p = kx < p->key ? p->leftchild : p->rightchild;
    }
    if(p != nullptr && p->key == kx) //已存在重复值
        return false;

    //把新节点插入到pa下
    if(val < pa->key)
    {
        pa->leftchild = new Node(val);
    }
    else
    {
        pa->rightchild = new Node(val);
    }
    
}
```

