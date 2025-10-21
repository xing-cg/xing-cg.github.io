---
title: Linux_gcc_gdb
categories:
  - - Linux
tags: 
date: 2022/5/12
updated: 2025/7/17
comments: 
published:
---
# 内容

1. gcc
2. gdb
# gcc的文件类型约定规则
![](../../images/Linux_gcc_gdb/image-20250717221711701.png)
# gcc常用选项
![](../../images/Linux_gcc_gdb/image-20250717221742017.png)


# gdb - 调试工具

* 调试的对象：可执行程序
* 编译时需要增加调试信息`-g`

## 常用命令

| 命令标识                | 含义           |
| ------------------- | ------------ |
| `l`                 | 显示代码         |
| `Enter（回车键）`        | 重复上一条命令      |
| `b 行号`              | 为某行添加断点      |
| info break          | 查看断点信息(bnum)   |
| delete bnum         | 删除断点对应的编号  |
| `r`、`run`           | 启动程序         |
| `n`、`next`          | 单步执行         |
| `p 变量名`、`print 变量名` | 打印变量名内容      |
| `q`                 | 退出调试         |
| `s`、`step`          | 进入函数         |
| `f`、`finish`        | 跳出函数         |
| `continue`          | 继续程序（到下一个断点） |

![](../../images/Linux_gcc_gdb/image-20250717223939391.png)

# gcc配置
目标：让新服务器上的`gcc/g++`命令默认指向GCC 15，且不破坏系统原默认版本。

## 定位GCC 15路径，常见目录为`/opt/GCC/v15`，指向具体版本

```bash
ls -l /opt/GCC/v15/bin || find /opt/GCC -maxdepth 3 -name 'gcc-15' -o -name 'g++-15'
```

输出：
```
cxing@forrest:~$ ls -l /opt/GCC/v15/bin || find /opt/GCC -maxdepth 3 -name 'gcc-15' -o -name 'g++-15'
total 480156
-rwxr-xr-x 4 root root  12542192 8月   9 17:15 c++-15
-rwxr-xr-x 1 root root  12537688 8月   9 17:15 cpp-15
-rwxr-xr-x 4 root root  12542192 8月   9 17:15 g++-15
-rwxr-xr-x 3 root root  12535504 8月   9 17:15 gcc-15
-rwxr-xr-x 2 root root    156184 8月   9 17:15 gcc-ar-15
-rwxr-xr-x 2 root root    151976 8月   9 17:15 gcc-nm-15
-rwxr-xr-x 2 root root    151984 8月   9 17:15 gcc-ranlib-15
-rwxr-xr-x 1 root root  11483480 8月   9 17:15 gcov-15
-rwxr-xr-x 1 root root   9305832 8月   9 17:15 gcov-dump-15
-rwxr-xr-x 1 root root   9477192 8月   9 17:15 gcov-tool-15
-rwxr-xr-x 1 root root 360127440 8月   9 17:15 lto-dump-15
-rwxr-xr-x 4 root root  12542192 8月   9 17:15 x86_64-linux-gnu-c++-15
-rwxr-xr-x 4 root root  12542192 8月   9 17:15 x86_64-linux-gnu-g++-15
-rwxr-xr-x 3 root root  12535504 8月   9 17:15 x86_64-linux-gnu-gcc-15
-rwxr-xr-x 2 root root    156184 8月   9 17:15 x86_64-linux-gnu-gcc-ar-15
-rwxr-xr-x 2 root root    151976 8月   9 17:15 x86_64-linux-gnu-gcc-nm-15
-rwxr-xr-x 2 root root    151984 8月   9 17:15 x86_64-linux-gnu-gcc-ranlib-15
-rwxr-xr-x 3 root root  12535504 8月   9 17:15 x86_64-linux-gnu-gcc-tmp
```

## 在用户目录创建覆盖用的符号链接
```bash
mkdir -p ~/bin
ln -sf /opt/GCC/v15/bin/gcc-15 ~/bin/gcc
ln -sf /opt/GCC/v15/bin/g++-15 ~/bin/g++
```

## 确保 `~/bin` 在 PATH 最前（加到 `~/.bashrc` 末尾，首次配置只需一次）
```bash
# Ensure $HOME/bin is at the front of PATH for user-level overrides (e.g., gcc/g++)
if [ -d "$HOME/bin" ]; then
  case ":$PATH:" in 
    *":$HOME/bin:"*) ;; 
    *) PATH="$HOME/bin:$PATH" ;; 
  esac
fi
```

之后刷新并验证

```bash
source ~/.bashrc && hash -r
which gcc && gcc --version | head -n1
which g++ && g++ --version | head -n1
```


## 系统级方案（需root，所有用户生效）
使用 update-alternatives 统一默认：
```bash
sudo update-alternatives --install /usr/bin/gcc gcc /opt/GCC/v15.2.0/bin/gcc-15 150 \
  --slave /usr/bin/g++ g++ /opt/GCC/v15.2.0/bin/g++-15
sudo update-alternatives --config gcc   # 选择刚添加的 15
gcc --version
```
注意：系统级修改会影响所有用户与任务，需谨慎。