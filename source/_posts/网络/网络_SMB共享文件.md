---
title: 网络_SMB共享文件
categories:
  - - 网络
tags: 
date: 2024/7/14
updated: 
comments: 
published:
---
本文是基于OpenWRT下的SMB4共享。
# Linux系统添加SMB用户、密码
Linux上SMB的用户和Linux用户是两码事。SMB需要另外单独添加用户。
```sh
smbpasswd -a root
```
# OpenWRT设置
## 磁盘新建分区、格式化、挂载

1. 在`系统 -> 磁盘管理`下查看磁盘，点击“弹出”右边的`修改`按钮，可以看到分区信息。新建分区，并格式化为ext4。
2. 之后，去`系统 -> 挂载点`下配置添加挂载，基本设置中，启用此挂载点，UUID选择相应选项，挂载点自定义，手写：如`/mnt/shared`。高级设置，文件系统选择ext4。
3. 记得点`保存&应用`。
## 配置SMB挂载

在openwrt的`网络存储 -> 网络共享`页面进行设置：

| name   | 目录          | 容许用户 | 只读   | 可浏览 | 创建权限掩码 | 目录权限掩码 |
| ------ | ----------- | ---- | ---- | --- | ------ | ------ |
| shared | /mnt/shared | root | 取消勾选 | 勾选  | 默认0666 | 默认0777 |
上边的“启用macOS兼容共享”经测试，不用勾选。

也可以直接修改配置文件：
```sh
vim /etc/config/samba4
```
>但是在`/etc/samba/smb.conf`文件中也有非常类似的设置，还不清楚这两个配置文件之间的从属关系。
## 相关命令
```sh
# 启动服务
service samba4 start
# 停止服务
service samba4 stop
# 重启服务
service samba4 restart
# 服务状态
service samba4 status

# 配置文件检查
testparm -v
```
# smb输入root账户和密码后拒绝访问

系统默认不允许root访问samba，需要配置`smb.conf.template`在`invalid users = root`前添加一个#号，将本行注释掉即可，这样root就不会被限制访问samba了。
查找`smb.conf.template`文件：
```sh
find / -name smb.conf.template
```
# 卸载磁盘时device is busy的处理

`lsof` + 挂载目录位置，如：`lsof /mnt/shared`
出来列表，kill掉相应的PID即可。
如果是macOS连接过此SMB，kill掉会自动重连，需要在macOS上弹出才行。
# 挂载目录文件系统问题

共享目录最好使用`Ext4`文件系统， 因为NTFS或ExFAT在macOS系统下会提示错误码100093。主要原因可能为samba4对NTFS或ExFAT文件系统使用Windows权限控制机制（openwrt是基于Linux的，使用的文件系统一般为ext4）导致权限混乱无法正常写入文件。
