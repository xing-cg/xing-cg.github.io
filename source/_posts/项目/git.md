---
title: git
categories:
  - - 项目
  - - Linux
  - - git
tags: 
date: 2022/4/10
updated: 
comments: 
published:
---

# 内容

1. 新建仓库
1. git init
1. git remote add

# 如何让本地指向远程仓库

1. 创建一个目录，使用git init命令将其变为一个可以通过git管理的仓库

   ```bash
   1.mkdir一个目录
   2.cd 创建的目录
   3.git init .
   ```

2. git remote是管理远程仓库的命令，git remote add \[name\] \[url\]可以给当前目录下的git添加一个远程仓库。这里的url我填的是gitee仓库的ssh地址。

   ```bash
   git remote add mygitee git@gitee.com:mrcan/distributed-system-framework.git
   # git remote -v 可以查看远程仓库的信息
   ```

# 本地、远程进行同步

1. 生成密钥

   ```bash
   ssh-keygen -t rsa -C "邮箱"
   ```

2. 切换到家目录下的".ssh"文件夹，发现有id\_rsa和id\_rsa.pub，后者是公钥，公钥给别人，连接时根据其他用户输入的公钥经过加密算法与服务器上的私钥比对。

3. cat id\_rsa.pub，把公钥内容拷贝下来，在gitee上的项目管理中找到部署公钥管理。

4. 配置git全局用户名、邮箱

   ```bash
   git config --global user.name 'xcg'
   git config --global user.email '1933966629@qq.com'
   ```

   

5. 使用git branch -r查看远程的分支有哪些，或者-a查看所有分支（包括本地分支）
