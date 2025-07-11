---
title: MIT-6.S081
categories:
  - - 操作系统
tags: 
date: 2021/9/12
updated: 
comments: 
published:
---
# 内容

Schedule: [6.S081 / 2020年秋季 --- 6.S081 / Fall 2020 (mit.edu)](https://pdos.csail.mit.edu/6.828/2020/schedule.html)
1. Basic ideas of OS
2. Study of the code in xv6 (a small teaching OS)
    1. talk about how it works
    2. look at the code and show the code executing
    3. reading assignments, a book that describes how xv6 operates and why it's designed that way.
    4. xv6 is a simplified unix-like operating system, which runs on the RISC-V microprocessor that is the same microprocessor and the focus of 6.004, but in this course we will run xv6 under the QEMU machine emulator which runs on Linux.
3. Background to help do the labs, like about C works, how the RISC-V which is the microprocessor that will be used.
# Labs

Either implementing basic operating system features or adding a kernel extensions to the xv6 operating system.

2023 Guidance: [6.1810 / 2023 年秋季 --- 6.1810 / Fall 2023 (mit.edu)](https://pdos.csail.mit.edu/6.828/2023/tools.html)
## 安装中遇到的棘手问题（Apple M芯片、macOS 14）

1. 不需要安装xcode IDE，只需要安装`$ xcode-select --install`即可，是xcode的命令行工具
2. 安装Homebrew，按照官网给的一句话命令安装
3. https://github.com/riscv/homebrew-riscv 这是homebrew下安装risc-v，旧系统、旧芯片可能会装到`/usr/local/opt/riscv-gnu-toolchain/bin`，但新系统、新芯片则装到了`/opt/homebrew/Cellar/riscv-gnu-toolchain/main.reinstall/bin`。
4. The brew formula may not link into /usr/local. You will need to update your shell's rc file (e.g. [~/.bashrc](https://www.gnu.org/software/bash/manual/html_node/Bash-Startup-Files.html)) to add the appropriate directory to [$PATH](http://www.linfo.org/path_env_var.html).
    1. 此处需要手动添加PATH，例子中说的是bashrc，M2、macOS14系统下默认终端是zsh，因此需要在家目录下的`.zshrc`中添加：`PATH=$PATH:/opt/homebrew/Cellar/riscv-gnu-toolchain/main.reinstall/bin`。意味着将此目录添加到当前用户的环境变量 `PATH` 中。这个操作的作用是将该目录下的可执行文件路径包含进系统的执行路径中，使得系统可以直接在命令行中找到并执行这些命令。
    2. 修改 `.zshrc` 后，需要执行 `source ~/.zshrc` 命令使其生效，或者关闭当前终端窗口重新打开一个新窗口。
5. 用brew安装qemu，默认装到了`/opt/homebrew/Cellar/qemu/9.0.1`
6. 然后，以上这些和xv6没关系，第一节课演示的xv6系统需要git源码：`git clone git://g.csail.mit.edu/xv6-labs-2023`然后进入make qemu。
7. 如何退出xv6：先按`Ctrl+A`后单击`X`。
## Lab1

Writing applications that make the system calls
## The Last Lab

Add a network stack and a network driver, to be able to connect in over the network to the operating system that you run.
# 零碎

## Xv6 I/O 与文件描述符

https://blog.csdn.net/u012419550/article/details/113850465

# OS设计原则

OS should be defensive.
- app cannot crash the OS
- app cannot break out of its isolation

Strong isolation between app and OS
- typical : Hardware Support
    - user / kernel mode
    - virtual memory system
# E01 - open & fork

Operating Systems provide a lot of features and a lot of services, but they actually tend to interact, and sometimes in odd ways that require a lot of thought, even the simple examples given with **open** and **fork**.
if a program allocates a file descriptor with the **open** system call, and then that same program forks.
The semantics of fork turned out to be that you create a new process that's a copy of the current process, this file descriptor you opened is truly to be a copy, this file descriptor still has to be present and usable in the child. So that is the files, the opened file descriptors, interact with fork in this interesting way. 
fork的语义是：创建一个进程，是当前进程的副本。
刚才打开的文件描述符在子进程中仍然存在并可用。