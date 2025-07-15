---
title: Linux_设置系统运行级别
categories:
  - - Linux
tags: 
date: 2023/3/28
updated: 
comments: 
published:
---
# 切换系统运行级别
* runlevel命令可查看系统运行级别
* 可以用init命令动态切换0-6七个级别
    1. 关机
    2. 单用户模式
    3. 多用户无网络服务
    4. 完全的多用户文本界面
    5. 未定义或自定义
    6. 图形化界面
    7. 重启

```bash
runlevel
# 结果:
	# 图形化模式时 -> N 5
	# 命令行模式时 -> 5 3
init 3 #切换到命令行模式
```

# 启动级别Runlevel
参考：https://www.bbsmax.com/A/x9J2PYpEd6/

首先来了解下启动级别(Runlevel)：

指Unix或类Unix操作系统下不同的运行模式，运行级别通常分为7级：

0. 运行级别 0：系统停机状态，系统默认运行级别不能设为0，否则不能正常启动
1. 运行级别 1：单用户工作状态，root权限，用于系统维护，禁止远程登陆，无网络连接，不运行守护进程，不允许非超级用户登录。（如果忘记了root密码，可通过开机热键进入单用户模式进行重置，前提是要接触到主机）
2. 运行级别 2：多用户状态，无网络连接，不运行守护进程
3. 运行级别 3：完全的多用户状态(有NFS)，登陆后进入控制台命令行模式，正常登录状态
4. 运行级别 4：系统未使用，保留，用户自定义
5. 运行级别 5：X11控制台，登陆后进入图形GUI模式，就是常见的带有图形界面的模式
6. 运行级别 6：系统正常关闭并重启，默认运行级别不能设为6，否则不能正常启动

# 设置开机进入命令行模式

在全新的Linux systemD中已经使用target代替Runlevel，如multi-user.target相当于init 3，graphical.target相当于init 5，但是SystemD仍然兼容运行级别(Runlevel)。

当前绝大多数发行版已采用systemd代替UNIX System V。

要设置开机进入无图形化界面的命令行模式，就是将默认的开机进入运行级别 5 更改为开机进入运行级别 3。

步骤：

1. `sudo vim /etc/default/grub`
2. sudo update-grub
3. `systemctl set-default multi-user.target`
4. reboot

# 设置开机进入图形界面

步骤：

1. `sudo vim /etc/default/grub`
2. sudo update-grub
3. `systemctl set-default graphical.target`
4. reboot
