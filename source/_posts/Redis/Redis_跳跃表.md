---
title: Redis_跳跃表
typora-root-url: ../..
categories:
  - [高级数据结构, Redis]
tags:
  - null 
date: 2021/10/9
update:
comments:
published:
---

# 内容

# 平衡树

平衡树(Balance Tree，BT) 指的是，**任意节点的子树的高度差都小于等于1**。常见的符合平衡树的有，B树（多路平衡搜索树）、AVL树（二叉平衡搜索树）等。平衡树可以完成集合的一系列操作, 时间复杂度和空间复杂度相对于“2-3树”要低，在完成集合的一系列操作中始终保持平衡，为大型数据库的组织、索引提供了一条新的途径。

