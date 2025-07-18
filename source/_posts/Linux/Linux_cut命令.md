---
title: Linux_cut命令
categories:
  - - Linux
tags: 
date: 2025/7/17
updated: 
comments: 
published:
---
cut是`/usr/bin/`下的程序。

```sh
cut  -d.  -f1  -s  IPs.txt
```
以上命令说的是：
1. `-d`：以`.`为分割。
2. `-f1`：切割下来第1列。
3. `-s`：以哪个文件的内容切割。


```sh
mrcan@ubuntu:~$ echo 192.168.0.1 > IP.txt
mrcan@ubuntu:~$ cut -d. -f1 -s IP.txt 
192
mrcan@ubuntu:~$ cut -d. -f2 -s IP.txt 
168
mrcan@ubuntu:~$ cut -d. -f3 -s IP.txt 
0
mrcan@ubuntu:~$ cut -d. -f4 -s IP.txt
1
mrcan@ubuntu:~$ cut -d. -f5 -s IP.txt
（空）
```
