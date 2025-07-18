---
title: Linux_wc命令(word count)
categories:
  - - Linux
tags: 
date: 2025/7/16
updated: 
comments: 
published:
---
```sh
wc file.txt
```

```sh
mrcan@ubuntu:~$ wc testif.sh
  9  27 131 testif.sh
 行  词 字符
```

默认显示行、词、字符数。

可以单独指定：
```sh
mrcan@ubuntu:~$ wc -w testif.sh 
27 testif.sh
mrcan@ubuntu:~$ wc -l testif.sh 
9 testif.sh
mrcan@ubuntu:~$ wc -c testif.sh 
131 testif.sh
```
# word `-w`
只要中间没有空格、换行符，就是1个word。
# line `-l`
# character `-c`
1个英文字母算1个字符
# byte