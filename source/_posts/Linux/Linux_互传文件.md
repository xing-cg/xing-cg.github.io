---
title: Linux_互传文件
categories:
  - - Linux
tags: 
date: 2025/10/21
updated: 
comments: 
published:
---

​`scp`（安全复制）​：这是最基础的命令之一，通过 SSH 加密传输，适合快速传输。

- ​**​从本地复制到远程​**​：`scp /path/to/local/file.txt username@remote_ip:/path/to/remote/directory/`
- ​**​从远程复制到本地​**​：`scp username@remote_ip:/path/to/remote/file.txt /local/path/`
- ​**​递归复制整个目录​**​：使用 `-r`选项，例如 `scp -r /local/dir username@remote_ip:/remote/path/`
    - 传输大文件时，可以加上 `-C`选项开启压缩以提高速度

制定端口：
使用 `-P`(大写) 指定端口

```bash
scp -P 10022 cxing@turing001:~/gcc15.bashrc .
```