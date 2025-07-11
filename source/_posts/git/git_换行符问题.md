---
title: git_换行符问题
typora-root-url: ../..
categories:
  - [Linux]
  - [git]
tags:
  - null 
date: 2022/8/29
update:
comments:
published:
---

# 问题

git diff时发现明明只改动了几行代码，但是对比时却显示全部文档都修改过了，而且到处都是`^M`字符，导致看不出修改点，达不到git diff的效果。

# 解决方案

- 分析原因
  `^M`根据ASCII码表, 查出是`\r`，即windows换行符`\r\n`的前一半。猜测可能是文件编写/查看操作系统不同导致，用`od -tc 文件`查看文件，确实是`\r`换行符导致的。

- 解决方案

搜了一下，千遍一律的解决方案是加一个配置，忽略换行符的差异：

```shell
git config --global core.whitespace cr-at-eol  
```

但是设置后并没有效果，可能和git版本有关。

既然是`\r`的问题，那把`\r`去掉试试，用UE编辑器根据正则`$\n`替换成`\n`后，换行符变成了`\n`。

把内容贴到idea后，发现换行符还是`\r`，这个时候发现可能是idea设置换行符的问题。遂百度了下idea换行符设置。

# IDE换行符设置

## idea全局换行符

`file` -> `setting` -> `code style` -> `Line separator`

默认是System-Dependent，就是根据当前操作系统进行设置。把它改成`\n`。

## 单个文件修改换行符

修改为`\n`，即LF。

最后发现，内容里的`^M`没有了。`git diff`结果也正常了。
