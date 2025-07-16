---
title: Linux_编辑器
categories:
  - - Linux
tags: 
date: 2025/7/15
updated: 
comments: 
published:
---

# 有哪些
vi、vim、Emacs、ee（Unix自带的）

Emacs是最难用的，但是如果掌握了，能获益不少？
# ee编辑器
```shell
ee 1st.sh
```
进入编辑器后的样子：

上半部分：按Ctrl加相应的符号可以调用一些功能。

![](../../images/Linux_编辑器/image-20250715235658871.png)

## 按Ctrl加`[`或按ESC进入菜单
![](../../images/Linux_编辑器/image-20250716000216760.png)

在按`a`离开编辑器时，如果没保存文件会提示你保存：
![](../../images/Linux_编辑器/image-20250716000318642.png)

# Emacs编辑器
打开文件。

```sh
emacs 3th.sh
```

按一下ESC，再按一下x（映射Emacs的M-x命令）。在底行输入linum，表示：Line Num，设置显示行号。
![](../../images/Linux_编辑器/image-20250716182031711.png)


## 退出、保存
Ctrl加x、Ctrl加c是退出。
之后，底行会问你，是否保存，按y即可保存。
![](../../images/Linux_编辑器/image-20250716185429960.png)

