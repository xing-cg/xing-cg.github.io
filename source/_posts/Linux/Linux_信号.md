---
title: Linux_信号
categories:
  - - Linux
  - - 操作系统
    - 多线程
tags: 
date: 2021/10/30
updated: 
comments: 
published:
---
# 信号

也叫软中断。

通知进程发生了某个事件。

# signal

![image-20211030101557246](../../images/%E4%BF%A1%E5%8F%B7/image-20211030101557246.png)

signal第一个参数，信号代号。第二个参数，信号处理函数入口地址。

默认：SIG_DFL；忽略：SIG_IGN。

```c
//默认：SIG_DFL；忽略：SIG_IGN。
(void (*)0)
```



```c
//以下程序把原有的SIGINT信号处理方式改为了我们自定义的函数。
void sig_fun(int sig)
{
    printf("sig=%d\n",sig);
}
int main()
{
    signal(SIGINT, sig_fun);//只是做了一个约定。没有去调用sig_fun函数。等到信号出现才去调用。
    while(1)
    {
        printf("hello\n");
        sleep(1);
    }
}
```

