---
title: Linux_环境配置
categories:
  - - Linux
tags: 
date: 2022/7/29
updated: 
comments: 
published:
---
# `.profile`
`.profile` 文件在 Linux/Unix 系统中扮演着重要的角色，它是位于用户家目录 (`~/`) 下的一个​**​隐藏文件​**​（以点开头）。它主要用于在​**​登录 shell​**​ 的初始化过程中为特定用户设置​**​环境变量​**​和​**​启动程序​**​。

以下是关于 `.profile` 文件的详细说明：

1. ​**​主要目的：用户环境定制​**​
    
    - ​**​设置环境变量：​**​ 这是 `.profile` 最常见的用途。你可以在这里设置 `PATH`（决定系统去哪里查找可执行命令）、`EDITOR`（默认文本编辑器）、`LANG`（系统语言和区域设置）、`JAVA_HOME`、`PYTHONPATH` 等等。这些变量定义了你的工作环境。
    - ​**​运行启动命令：​**​ 你可以在登录时自动运行某些命令或小脚本。例如：
        - 显示欢迎信息或系统状态 (`uptime`, `fortune`, `echo "Welcome back, $USER"`)
        - 设置命令别名 (`alias ll='ls -l'`)
        - 设置 shell 选项 (`set -o noclobber` - 防止覆盖文件)
        - 启动代理或后台服务（以前更常见，现在可能有更好的方法）
2. ​**​执行时机：登录时​**​
    
    - 当你通过​**​登录 shell​**​ 进入系统时，`.profile` 文件会被自动读取和执行。
    - 什么是登录 shell？
        - 通过控制台（文本登录界面）登录。
        - 使用 `ssh username@hostname` 远程登录。
        - 使用 `su - username` 或 `sudo -i`（切换用户并模拟完整登录）。
        - 图形界面登录后启动的​**​第一个​**​终端（取决于桌面环境和终端模拟器的配置，有时是登录 shell，有时不是）。
    - ​**​重要：​**​ 并非所有终端会话都是登录 shell。如果你只是打开一个新的终端窗口（标签页），它通常是​**​非登录 shell​**​，不会读取 `.profile`，而是读取 `.bashrc`（对于 bash）或类似的文件。
3. ​**​与系统级配置的关系​**​
    
    - ​**​`/etc/profile`​**​: 这是一个系统级别的全局配置文件，在用户登录时，​**​最先​**​被执行。
    - ​**​`/etc/profile.d/*.sh`​**​: `/etc/profile` 通常会调用执行这个目录下所有的 `.sh` 脚本文件，用于存放系统级别的额外配置。
    - ​**​`~/.profile`​**​: 在 `/etc/profile` 之后执行。它是用户​**​定制自己登录环境​**​的主要地方，可以覆盖或补充系统级别的设置。
    - ​**​`~/.bash_profile` 或 `~/.bash_login` (bash 特有)​**​: 如果 `.profile` 所在的 shell 是 bash，并且用户家目录下存在 `~/.bash_profile` 或 `~/.bash_login`，那么 bash 会​**​优先执行它们​**​，而​**​跳过​**​ `~/.profile`（除非这些文件显式地 `source ~/.profile`）。
    - ​**​`~/.bashrc` (bash 特有)​**​: 主要用于​**​非登录的交互式 shell​**​ (即大部分新打开的终端窗口)。在登录 shell 中，如果 `~/.bash_profile` 存在，它通常会去调用 `~/.bashrc`，这样登录 shell 也能获得其中的设置（别名、函数等）。
4. ​**​文件命名与位置​**​
    
    - ​**​路径：​**​ 位于用户的家目录下，即 `~/.profile` 或 `/home/username/.profile`。
    - ​**​兼容性：​**​ `.profile` 是一个比较通用的名字，被 Bourne Shell (`sh`)、Bash、Korn Shell (`ksh`) 等广泛支持。它为不同 shell 提供了一个公共的配置点。
    - ​**​Shell 专属文件：​**​ 许多用户（尤其是 bash 用户）更喜欢或习惯使用 `~/.bash_profile`（用于登录 bash）和 `~/.bashrc`（用于交互式非登录 bash），并将 `.profile` 用作备用或兼容配置。
5. ​**​查看与编辑​**​
    
    - 查看：`cat ~/.profile` 或 `less ~/.profile`
    - 编辑：使用你喜欢的文本编辑器（如 `nano ~/.profile`, `vim ~/.profile`）。确保你编辑的是你自己的家目录下的文件。
6. ​**​重要注意事项​**​
    
    - ​**​生效需要：​**​ 修改 `.profile` 后，这些更改​**​不会​**​立即在当前已经打开的会话中生效（因为你修改时，登录过程已经完成）。要让修改生效，你需要：
        - ​**​方法一 (推荐)：​**​ 重新登录（logout 后再 login）。
        - ​**​方法二：​**​ 在当前的登录 shell 中执行 `source ~/.profile` 或 `. ~/.profile` 命令来强制重新加载该文件。
    - ​**​谨慎修改：​**​ 错误的语法（比如引号不匹配、错误的变量赋值）或执行有问题的命令可能导致你登录时出错，甚至无法登录。在修改重要文件前最好备份。
    - ​**​`~/.bash_profile` vs `~/.profile` (bash 用户):​**​ 如果 `~/.bash_profile` 存在，bash 会忽略 `~/.profile`。因此，很多系统默认创建的是 `.profile`，但如果用户创建了 `.bash_profile`，它就会接管。推荐做法是让 `~/.bash_profile` 里简单地包含 `source ~/.profile`，然后再包含 `source ~/.bashrc`，这样可以同时兼容登录配置和交互配置。

## ​总结​
`.profile` 是个人 Linux/Unix 环境的核心配置文件之一。它在你通过登录 shell 进入系统时自动运行，其主要任务是为你定义个性化的环境变量（如 `PATH`, `EDITOR`）和登录时需要运行的命令或脚本。理解它与 `/etc/profile`、`~/.bash_profile`、`~/.bashrc` 等文件的关系和执行顺序，对于有效管理和定制你的 shell 环境至关重要。
# Linux配置相关

[bashrc source后长期有效](https://www.csdn.net/tags/MtTaEg0sMDg4NTIyLWJsb2cO0O0O.html)