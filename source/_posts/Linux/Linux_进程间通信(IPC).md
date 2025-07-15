---
title: Linux_进程间通信(IPC)
categories:
  - - Linux
  - - 操作系统
    - 多线程
tags: 
date: 2021/11/1
updated: 
comments: 
published:
---

# 内容

1. 管道
2. 信号量
3. 共享内存
4. 消息队列
5. 套接字

2-4比较正式。如果填空只有三个空，填2-4。

# 管道

管道文件是在内存中分配空间。

读端r，写端w。

分两种，有名管道、无名管道。

## 有名管道

```bash
mkfifo fifo#mkfifo [名字]
#prw-rw-r-- 1 stu stu 0 fifo
mkfifo()
```

```c
//写端程序
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<assert.h>
#include<fcntl.h>
int main()
{   
    int fdw = open("fifo",O_WRONLY);
    assert(fdw!=-1);
    printf("fdw=%d\n",fdw);
    char buff[128] = {0};
    fgets(buff,128,stdin);
    write(fdw,buff,strlen(buff));
    close(fdw);
    exit(0);
}
```



```c
int main()
{
    int fdr = open("fifo",O_RDONLY);
    assert(fdr!=-1);
    
    printf("fdr=%d\n",fdr);
    
    char buff[128] = {0};
    
    read(fdr,buff,127);
    printf("%s",buff);
    close(fdr);
    exit(0);
}
```



## 无名管道

```bash
pipe()
```



## 总结

1. 管道为空时，读会阻塞。

2. 管道写满时，写会阻塞。

3. 两个端口打开后，也写了一些数据，也读了一些数据，如果读完了，则管道内容变空，则读端进程阻塞。而如果写端关闭后，则读端不会再阻塞了，再接着read会返回一个零，可作为写端完毕的标志。

   ```c
   //写端
   int main()
   {   
       int fdw = open("fifo",O_WRONLY);
       assert(fdw!=-1);
       printf("fdw=%d\n",fdw);
       while(1)
       {
           char buff[128] = {0};
           printf("input util end:"\n);
           fgets(buff,128,stdin);
           if(strncmp(buff,"end",3)==0)
           {
               break;
           }
       }
       write(fdw,buff,strlen(buff));
       close(fdw);
       exit(0);
   }
   //读端
   int main()
   {
       int fdr = open("fifo",O_RDONLY);
       assert(fdr!=-1);
       
       printf("fdr=%d\n",fdr);
       while(1)
       {
           char buff[128] = {0};
       	int n = read(fdr,buff,127);
       	if(n == 0)//n=0肯定是因为写端close了，因为没有在没写入时阻塞
           {
               break;
           }
       	printf("%s",buff);
       }
       close(fdr);
       exit(0);
   }
   ```

4. 管道读端关闭后，写端再写入数据时，则会触发异常，则内核会给写端进程发信号。

5. 面试：

   1. 有名管道可以在任意两个进程间通信；无名管道只能在父子进程间使用。
   2. 管道通信方式：半双工的。
      1. 单工：电台从A发到B，收音机不可能给电台发数据。过程不可逆。
      2. 半双工：跟对讲机类似。需要按下按键抢占信道。其余是听。同一时刻只能读或写。
      3. 全双工：腾讯会议。
   3. 写入管道的数据，在内存中。有名管道虽然能在磁盘上看到，但只是一个标识，大小是0。

# 信号量

目的是控制对某个临界资源的访问。

## 临界资源

临界资源：同一时刻只允许一个进程访问的资源

临界区：访问临界资源的代码段

## 取值

比较特殊，只取0和正数。

第一种是0，1二值信号量。

第二种是计数信号量。

## p,v操作

p，减一，代表获取资源。

v，加一，代表释放资源。

## 代码封装

```c
/*
1.sem_init()
2.sem_p()
3.sem_v()
4.sem_destroy()
*/
```

# 共享内存

1. 创建共享内存
2. 将共享内存映射到进程中
3. 断开映射
4. 销毁共享内存

# 消息队列

1. 发送消息，发送两个信息，一是type，二是数据。
2. 获取消息，获取type。
