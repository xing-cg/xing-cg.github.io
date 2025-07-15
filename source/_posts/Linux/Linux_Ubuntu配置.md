---
title: Linux_Ubuntu配置
categories:
  - - Linux
  - - 操作系统
tags: 
date: 2023/5/9
updated: 
comments: 
published:
---

# 内容

1. 关闭Ubuntu动画

2. 配置终端代理

3. 下载安装make、g++-12/gcc-12、git、net-tools、curl

   1. g++和gcc有什么关系？d41d8c的回答 - 知乎 https://www.zhihu.com/question/20940822/answer/1768772877

   2. 将g++默认为g++-12(参考https://cloud.tencent.com/developer/ask/sof/100275)

      ```
      sudo apt-get install g++-12
      sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-12 100
      sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 100
      
      sudo update-alternatives --set g++ /usr/bin/g++-12
      sudo update-alternatives --set gcc /usr/bin/gcc-12
      ```

   3. 命令格式：`update-alternatives [<选项> ...] <命令>`(参考https://blog.csdn.net/JasonDing1354/article/details/50470109)

      ```
      Commands:
        --install <link> <name> <path> <priority>
          [--slave <link> <name> <path>] ...
                                 在系统中加入一组替换项.
        --remove <name> <path>   从 <名称> 替换组中去除 <路径> 项.
        --remove-all <name>      从替换系统中删除 <名称> 替换组.
        --auto <name>            将 <名称> 的主链接切换到自动模式.
        --display <name>         显示关于 <名称> 替换组的信息.
        --query <name>           machine parseable version of --display <name>.
        --list <name>            列出 <名称> 替换组中所有的可用替换项.
        --get-selections         list master alternative names and their status.
        --set-selections         read alternative status from standard input.
        --config <name>          列出 <名称> 替换组中的可选项，并就使用其中
                                       哪一个，征询用户的意见.
        --set <name> <path>      将 <路径> 设置为 <名称> 的替换项.
        --all                    对所有可选项一一调用 --config 命令.
      <link> 是指向 /etc/alternatives/<名称> 的符号链接>.
        (e.g. /usr/bin/pager)
      <name> 是该链接替换组的主控名.
        (e.g. pager)
      <path> 是替换项目标文件的位置.
        (e.g. /usr/bin/less)
      <priority> 是一个整数，在自动模式下，这个数字越高的选项，其优先级也就越高.
      Options:
        --altdir <directory>     指定不同的可选项目录.
        --admindir <directory>   指定不同的管理目录.
        --log <file>             设置log文件.
        --force                  allow replacing files with alternative links.
        --skip-auto              skip prompt for alternatives correctly configured
                                 in automatic mode (relevant for --config only)
        --verbose                详尽的操作进行信息，更多的输出.
        --quiet                  安静模式，输出尽可能少的信息.
        --help                   显示本帮助信息.
        --version                显示版本信息.
      ```

4. 下载安装vim

   1. ```
      git clone https://github.com/vim/vim.git
      # if you don't have local changes, update to the latest version with:
      cd vim
      git pull
      # If you made some changes, e.g. to a makefile, you can keep them and merge with the latest version with:
      cd vim
      git stash
      git pull
      git stash pop
      # If you have local changes you may need to merge. If you are sure you can discard local changes (e.g. if you were just trying a patch), you can use:
      git fetch --all
      git reset --hard origin/master
      ```

   2. ```
      # Building Vim
      cd src
      make distclean  # if you build Vim before
      make
      sudo make install
      ```

   3. ```
      checking for tgetent()... configure: error: NOT FOUND!
            You need to install a terminal library; for example ncurses.
            On Linux that would be the libncurses-dev package.
            Or specify the name of the library with --with-tlib.
      # sudo apt-get install libncurses5-dev
      ```

5. 配置镜像源 - huaweicloud, http协议

6. 更新系统软件

7. 安装ssh-server `sudo apt-get install openssh-server`

# 关闭Ubuntu动画

Ubuntu Desktop应用程序菜单带有从屏幕底角到屏幕中心的动画。虽然这看起来很酷，但感觉它很卡，尤其是在集成显卡和虚拟机的环境中。有一种方法可以关闭此动画，从而可以更快地从“应用程序”菜单中启动应用程序。

1. 打开“终端”
2. 执行以下命令，关闭动画，立即生效（不用root）

```
gsettings set org.gnome.desktop.interface enable-animations false
```

想要重新开启动画，只需要执行

```
gsettings set org.gnome.desktop.interface enable-animations true
```
