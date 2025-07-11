---
title: mprpc_环境配置
categories:
  - - 项目
    - rpc
tags: 
date: 2022/8/17
update: 
comments: 
published:
---

# 项目目录组织

略

# protobuf简介

protobuf是Google创始的一种数据交换的格式，独立与平台语言。

提供了多种语言的实现。每一种实现都包含了相应语言的编译器以及库文件。

由于它是一种二进制的格式，比xml、json快。

很适合用于分布式应用之间的数据通信或者异构环境下的数据交换。作为一种效率和兼容性都很优秀的二进制数据传输格式，可以用于网络传输、配置文件、数据存储等诸多领域。

# 安装配置

在github源代码下载地址：https://github.com/google/protobuf

源码包中的`src/README.md`，有详细的安装说明，安装过程如下：

1. 解压压缩包：`unzip protobuf-master.zip`
2. 进入解压后的文件夹：`cd protobuf-master`
3. 安装所需工具：`sudo apt-get install autoconf automake libtool curl make g++ unzip`
4. 自动生成configure配置文件：`./autogen.sh`
5. 配置环境：`./configure`
6. 编译源代码(时间比较长)：make
7. 安装：sudo make install
8. 刷新动态库：sudo ldconfig