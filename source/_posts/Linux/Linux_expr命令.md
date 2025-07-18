---
title: Linux_expr命令
categories:
  - - Linux
tags: 
date: 2025/7/16
updated: 
comments: 
published:
---
# 先看`var=$var2+$var3`
```sh
mrcan@ubuntu:~$ var=1+2
mrcan@ubuntu:~$ echo $var
1+2
mrcan@ubuntu:~$ var2=2
mrcan@ubuntu:~$ var3=3
mrcan@ubuntu:~$ var=$var2+$var3
mrcan@ubuntu:~$ echo $var
2+3
```
我们发现，`var=$var2+$var3`未能让2+3的结果5赋给var，而是以字符串的形式给了var。
# expr
用于计算数值类。

which发现`expr`是存在`/usr/bin/expr`下的，说明是个常用的命令。
```sh
mrcan@ubuntu:~$ expr 1+2
1+2
```
我们发现，以字符串的形式输出了expr的结果。明显不是我们想要的。
因为要计算数值的话，在运算符前后要加空格分隔开！
```sh
mrcan@ubuntu:~$ expr 1 + 2
3
```

```sh
mrcan@ubuntu:~$ expr $var2 + $var3
5
```
# 把expr的结果赋给环境变量
```sh
mrcan@ubuntu:~$ var=expr $var2 + $var3
2: command not found
```

```sh
mrcan@ubuntu:~$ var=(expr $var2 + $var3)
mrcan@ubuntu:~$ echo $var
expr
```
以上两种都是错误的。

要用“命令代换”。

看：《Linux_重定向#命令代换》![命令代换](Linux_重定向.md#命令代换)
正确的方式：****
```sh
mrcan@ubuntu:~$ var=$(expr $var2 + $var3)
mrcan@ubuntu:~$ echo $var
5
```


# 使用expr编写从1加到100的脚本
```sh
#! /bin/sh

I=1
SUM=0
while (test $I -le 100) # 可以简写为 while [ $I -le 100 ]
do
    SUM=$(expr $SUM + $I)
    I=$(expr $I + 1)
done
echo "sum of 1 ... 50 is "$SUM
```

```sh
mrcan@ubuntu:~$ ./testif.sh 
sum of 1 ... 50 is 505
```
# 乘号需要转义
```sh
mrcan@ubuntu:~$ expr 56 * 9
expr: syntax error: unexpected argument ‘a.c’
mrcan@ubuntu:~$ echo $?
2
```
在expr命令中，乘号不能直接写，而是要`\*`转义。
```sh
mrcan@ubuntu:~$ expr 56 \* 9
504
```

除号不用转义。
```sh
mrcan@ubuntu:~$ expr 56 / 9
6
```