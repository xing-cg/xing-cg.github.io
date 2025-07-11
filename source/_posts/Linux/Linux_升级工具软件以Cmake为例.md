---
title: Linux_升级工具软件以Cmake为例
typora-root-url: ../..
categories:
  - [Linux]
tags:
  - null 
date: 2022/10/26
update:
comments:
published:
---

参考文章：[【安全|正确】Ubuntu升级CMake](https://blog.csdn.net/zzh516451964zzh/article/details/126373715)

主要通过建立新的软链接来升级软件，同样适用于升级软件：无需卸载，通过建立软链接即可解决问题。很多教程会让你使用autoremove，但该操作会将之前编译过的许多文件连带删除。

1. 下载CMake，[下载地址](https://cmake.org/files/)

2. 下载压缩包，请注意无需编译，这是已经编译好的版本

3. 解压

4. 移动

   `sudo mv cmake-3.22.1-linux-x86_64 /usr/share/camke-3.22.1`

5. 建立软链接

   `sudo ln -sf /usr/share/cmake-3.22.1/bin/cmake /usr/bin/cmake `

6. 检查是否成功

   `cmake --version`
