---
title: Linux_rg检索
categories:
  - - Linux
tags: 
date: 2025/8/18
updated: 
comments: 
published:
---
# 介绍

rg，全称ripgrep，文本搜索工具，有grep、Ack、Ag。rg全称RipGrep，是一个按行搜索工具，它根据提供的pattern递归地在指定的目录里搜索。由Rust写成，相较于同类工具的特点是快、省电。

1. 自动递归搜索
2. 自动忽略.gitignore中的文件以及二进制文件
3. 可以限制文件类型
   4. `rg -t py foo`限定python文件
   5. `rg -T js foo`排除js文件
6. 支持各种文件编码（UTF-8、UTF-16、GBK）
7. 支持搜索常见的压缩文件（gzip、xz、bzip2）
8. 自动高亮匹配的结果

>扩展：强中更有强中手，现在更有rg的加强版ripgrep-all，能搜索pdf, e-book, office documents, zip, tar.gz等

# 小试牛刀

以下面的文件为测试文件, 命名为a.txt

```
test
a
b
c
d
e
f
g
test1
TEST2
```

1. 基础搜索

```bash
rg 'test' a.txt
1:test
9:test1
```

```bash
grep 'test' a.txt
test
test1
```

可见，rg和grep的区别有一点在于rg还额外加了行号

2. `-w`有word之意，表示搜索的字符串作为一个独立的单词时才能被匹配到。

```bash
rg -w 'test' a.txt
1:test
```

3. 使用`-i`选项，不区分大小写

```bash
rg -i 'test' a.txt
1:test
9:test1
10:TEST2
```

4. `-l`只打印有匹配信息的文件名字

```bash
rg -l 'test' a.txt
a.txt
```

5. `-C`输出匹配上下几行的内容

```bash
rg -C 2 'c' a.txt#意思是，输出'c'所在这一行的上下两行信息
2-a
3-b
4:c
5-d
6-e
```

# 高级搜索

1. 使用`-e REGEX`来指定正则表达式

```bash
rg -e "*sql" -C2
```

2. 默认rg会忽略`.gitignore`和隐藏文件，可以使用`-uu`来查询所有内容

```bash
rg -uu "word" .
```

3. 可以使用`-t type`来指定文件类型

```bash
rg -t markdown "mysql" . #在md文件中查找"mysql"关键字
```

# 搜索文件

列出当前文件夹会进行查询的所有文件，该选项其实可以相当于`find . -type f`，查找当前目录所有文件。

```bash
rg --files .
```