---
title: 安装Nginx
categories:
  - - 项目
tags: 
date: 2022/5/19
updated: 
comments: 
published:
---
# 内容

安装nginx及其依赖。

可能需要的命令：

1. Linux下`*.tar.gz`文件解压缩命令：`tar -zxvf 压缩文件名.tar.gz`
2. `chown -R` : 处理指定目录以及其子目录下的所有文件

参考文章

1. [Hexo部署至服务器（Ubuntu 20.04）](https://cloud.tencent.com/developer/article/1945550)
2. [ubuntu nginx源代码安装](https://www.jianshu.com/p/8af24b0adaf6)
3. [Ubuntu20.04安装openssl-1.1.1k](https://zhuanlan.zhihu.com/p/387379400)
4. [linux中ln -s 命令详解](https://blog.csdn.net/liangtianmeng/article/details/86736761)
5. [解决：Linux8整合Nginx过程中报错：src/os/unix/ngx_user.c: 在函数‘ngx_libc_crypt’中: src/os/unix/ngx_user.c:36:7](https://blog.csdn.net/weixin_48033662/article/details/122004967)
6. [OpenSSL library is not used](https://blog.csdn.net/beagreatprogrammer/article/details/78369638)
7. [设置Nginx在linux服务器（Ubuntu）开机启动](http://t.zoukankan.com/hoaprox-p-12416624.html)
8. [Unable to locate package sysv-rc-conf](https://blog.csdn.net/hancc_824/article/details/109181317)
9. https://blog.csdn.net/weixin_45837693/article/details/107675339

# 安装其依赖

nginx依赖pcre、openssl、zlib

安装pcre

```bash
sudo apt install libpcre3 libpcre3-dev
```

安装zlib

下载地址:http://www.zlib.net

```bash
tar -xvf zlib-<version>.tar.gz 
cd zlib-<version>/
./configure
make
make install
```

