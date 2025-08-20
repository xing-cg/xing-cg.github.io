---
title: Linux_IO复用_select_poll
categories:
  - - Linux
  - - 网络
tags: 
date: 2022/3/26
updated: 
comments: 
published:
---
# 内容

1. 基本概念
2. 了解接口socket
3. 了解协议tcp/udp
4. io复用
5. select
6. poll/epoll

可以参考学习的文章：

1. [深入浅出理解select、poll、epoll的实现](https://zhuanlan.zhihu.com/p/367591714)
2. [linux在系统调用进入内核时，为什么要将参数从用户空间拷贝到内核空间？不能直接访问，或是使用memcpy吗？非要使用copy_from_user才行吗？](https://www.zhihu.com/question/19728793) - 针对为什么select/poll调用一次拷贝一次数组到内核态空间。

# I/O复用

1. select
2. poll
3. epoll

# select

先观察其API

```c
#include<sys/select.h>
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
```

1. 参数
   1. `nfds`参数通常被设置为`select`监听的所有文件描述符中的最大值加1，表示在`fd_set`集合中，我们关心的描述符的总数。为什么加1呢？因为文件描述符是从0开始计数的。`nfds`与`fd_set容量大小`不一样，容量大小指的是`FD_SETSIZE`，即`fd_set容量大小`是`fd_set`可容纳描述符的最大大小。
   2. readfds参数是select关心的读事件的集合；
   3. writefds参数是select关心的写事件的集合；
   4. exceptfds参数select关心的异常事件的集合；
   5. timeout参数设置select的超时时间。
2. 返回值
   1. 集合中有事件就绪的描述符的个数
   2. 但是并没有告诉你具体是哪一个描述符就绪

## fd_set结构体

```c
#include<typesizes.h>
#define __FD_SETSIZE 1024

#include<sys/select.h>
#define FD_SETSIZE __FD_SETSIZE
typedef long int __fd_mask;
#undef __NFDBITS
#define __NFDBITS (8*(int)sizeof(__fd_mask))
typedef struct
{
#ifdef __USE_XOPEN
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
#define __FDS_BITS(set) ((set)->fds_bits)
#else
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
#define __FDS_BITS(set) ((set)->__fds_bits)
#endif
}fd_set;
-------------------
#define FD_SETSIZE 1024
#define NFDBITS (8*(int)sizeof(long int))
typedef struct
{
    long int fds_bits[FD_SETSIZE / NFDBITS];
}fd_set;
```

其中，`__FD_SETSIZE`指出select可以关注的最大文件描述符个数，默认为1024。

`__fd_mask`被定义为`long int`的类型别称，`long int`在32位机占8个字节。

`__NFDBITS`计算的是1个`__fd_mask`元素所占用的位数，一个字节占8位，sizeof算出`__fd_mask`的字节数，相乘得其占用的bit大小。

接着，定义`fds_bits`，其是`long int`型的数组，数组大小为`__FD_SETSIZE`除以`__NFDBITS`。比如`SETSIZE`为1024位，`NFDBITS`是64位，则数组大小位`1024/64=16`。这里的计算主要是为了计算出数组的大小，以确定多大的数组可以正好容纳`1024`个位数，来记录文件描述符信息。

## 用到的宏函数

`fd_set`集合对于文件描述符的管理是按位进行的，而位只有0和1两种状态。

假如`SETSIZE=1024`，则可管理1024个文件描述符，如果`文件描述符7`有效，我们需要对位操作，使其位变为`1`。

由于位操作过于繁琐，select API中提供了一系列宏函数来方便我们访问、操作`fd_set`集合状态。

```c
#include<sys/select.h>
FD_ZERO(fd_set *fdset);				/*清除fdset的所有位*/
FD_SET(int fd, fd_set *fdset);		/*设置fdset的位fd*/
FD_CLR(int fd, fd_set *fdset);		/*清除fdset的位fd*/
int FD_ISSET(int fd, fd_set *fdset);/*测试fdset的位fd是否被设置*/
```

## select编程思路

最好另外定义一个整型数组，其大小为我们预测将要出现的描述符的最多数目。用作我们存放描述符的容器。初始化时将数组值一律设为`-1`，表示容器中该位置还没有存放描述符。如果在某一时刻有一个描述符有了消息，我们就将该描述符数值覆盖到这个容器中第一个为`-1`的地方。

```c
#define MAX 10
void fds_init(int fds[])
{
    for(int i = 0;i<MAX;++i)
    {
        fds[i] = -1;
    }
}
void fds_add(int fd, int fds[])//向fds容器中添加描述符fd
{
    if(fd<0)
    {
        printf("无效的描述符\n");
        return;
    }
    for(int i = 0;i<MAX;++i)
    {
        if(fds[i]==-1)
        {
            fds[i] = fd;
            return;
        }
    }
    printf("容器已满，无法添加该描述符\n");
}
void fds_del(int fd, int fds[])
{
    for(int i = 0;i<MAX;++i)
    {
        if(fds[i]==fd)
        {
            fds[i] = -1;
            return;
        }
    }
    printf("没有找到该描述符\n");
}
int main()
{
    int fds[MAX];
    fds_init(fds);
}
```

## 示例-TCP服务使用select处理多个套接字

```c
int main()
{
    int sockfd = socket_init();//socket_init封装了bind(ip:port)的操作，还封装了对sockfd进行listen的操作，并设置了监听队列大小。
    assert(sockfd!=-1);
    
    int fds[MAX];
    fds_init(fds);
    
    fds_add(sockfd, fds);
    
    fd_set fdset;//此处的fd_set即<sys/select.h>库中API提供的fd_set结构体，实际上是一个有1024个“二进制位”的数组。
    while(1)//将fds[MAX]中所有有效的(即>=0)描述符全部“加入”fdset中，
    //即把fdset中与某有效描述符对应的[位]的状态设为1。
    {
        FD_ZERO(&fdset);	//把1024位全清零。
        int maxfd = -1;//记录当前最大的描述符数值是多少，方便过后调用select传入参数nfds。
        for(int i = 0; i<MAX; ++i)
        {
            if(fds[i] == -1)
            {
                continue;
            }
            FD_SET(fds[i],fdset);	//fds[i] != -1, 说明有效
            if(maxfd<fds[i])//寻找最大描述符数值
            {
                maxfd = fds[i];
            }
        }//end for
        
    	//此时，我们已经把开始创建的sockfd添加到了fdset中。
        //下面就可以用select来监测该套接字是否有消息了。
        //比如，sockfd监听到了客户端的connect信息，
        //则select就可以探测到fdset中对应的sockfd位处于消息就绪态，
        //则select就可以不再阻塞，立马返回。
        struct timeval tv = {5,0};
        //返回在fdset集合中有信息的描述符的个数。
        int n = select(maxfd+1, &fdset, NULL, NULL, &tv);
        if(n < 0)
        {
            printf("select err\n");
        }
        else if(n == 0)
        {
            printf("time out\n");
        }
        else
        {
            for(int i = 0; i<MAX; ++i)//依然需要根据fdset查询:
            //目前是哪个描述符有事件产生，
            //fdset的过滤又需要根据fds的记录进行遍历。
            {
                if(fds[i] == -1)continue;
                if(FD_ISSET(fds[i], &fdset))//此处判断ISSET
                //即是判断我们关注的描述符是否有事件产生。
                //为什么此时标志位为1一定有事件产生？
                //因为在这之前我们进行了select，
                //select不仅说明有事件产生，它还做了更多的工作：
                //将我们关心的描述符却在其上没有事件产生的标志位置0。
                //因此目前所有标志位为1的描述符均有事件。
                {
                    //以下才是核心业务代码,抓住了有事件产生的描述符,
                    //对这些描述符我们的处理流程,对于不同类型的描述符
                    //需要不同的处理流程。比如sockfd用accept处理，
                    //accept返回1个新描述符c,则先将其加入fds容器,
                    //下一轮再用recv处理描述符c的消息。
                    if(fds[i] == sockfd)//处理监听套接字sockfd
                    {
                        struct sockaddr_in caddr;
                        int len = sizeof(caddr);
                        int c = accept(sockfd, (struct sockaddr*)&caddr, &len);
                        if(c < 0)
                        {
                            continue;
                        }
                        printf("accept c = %d\n", c);
                        //此处的fds_add是用户自定义的函数，添加的是fds自定义数组
                        fds_add(c, fds);//只是收到fds容器，下次while扫描才将c加到fdset
                    }
                    else//处理收发套接字，在此程序，除了sockfd皆为收发套接字c
                    {
                        char buff[128] = {0};
                        int num = recv(fds[i], buff, 127, 0);
                        if(num <= 0)
                        {
                            printf("client close\n");
                            close(fds[i]);
                            fds_del(fds[i], fds);
                        }
                        else//num > 0，读到了数据
                        {
                            printf("recv(c = %d) = %s\n",fds[i],buff);
                            send(fds[i], "ok", 2, 0);
                        }
                    }
                }//end if(ISSET(fds[i], &fdset))
            }//end for(int i = 0; i<MAX; ++i)
        }//end if(n > 0)
    }//while end
}
```

场景情况：如果客户端与select服务端已建立连接，而客户端进程结束，select会一直阻塞、未感知吗？--不会。

因为客户端的进程结束，也算是一种读事件，相当于通知服务端该套接字连接结束了。那么服务端recv会返回0，达到关闭该套接字的条件，关闭后，别忘了在fds容器中删除掉该描述符。

如果忘记了close该套接字，且忘了`fds_del`该描述符，那么如果客户端结束进程，服务端就会一直打印"`client close`"，因为select一直在探测此描述符有无读事件，若该套接字连接关闭，那么此描述符一直有读事件，`recv`返回0，由于没有`fds_del`，每次都会关注，所以每次都会打印"`client close`"。

# poll

可以理解为加强版的select。

先观察其API

```c
#include<poll.h>
int poll(struct pollfd* fds, nfds_t nfds, int timeout);
```

1. fds参数是一个`pollfd`结构类型的指针，可指向一段连续空间（数组），因此很灵活，**大小可按需声明**。它可以指定我们感兴趣的文件描述符上发生的可读、可写和异常等事件。定义如下

   ```c
   struct pollfd
   {
       int fd;			//文件描述符
       short events;	//注册的事件类型，按位标志
       short revents;	//实际发生的事件，按位标志，由内核填充
   }
   ```

   1. 其中，fd成员指定文件描述符。
   2. events成员告诉poll监听fd上的哪些事件类型，他可以是一系列事件类型的**按位或**。常见的事件类型有：`POLLIN`(数据可读)、`POLLOUT`(数据可写)。
   3. revents成员由内核修改，以通知应用程序fd上实际发生了哪些事件。

## poll编程

```c
void poll_fds_init(struct pollfd* fds)
{
    for(int i = 0;i<MAX;++i)
    {
        fds[i].fd = -1;
        fds[i].events = 0;
        fds[i].revents = 0;
    }
}
void poll_fds_add(int fd, struct pollfd* fds)
{
    for(int i = 0;i<MAX;++i)
    {
        if(fds[i].fd == -1)
        {
            fds[i].fd = fd;
            fds[i].events = POLLIN;//只关注读事件
            fds[i].revents = 0;
            break;
        }
    }
}
void poll_fds_del(int fd, struct pollfd* fds)
{
    for(int i = 0;i<MAX;++i)
    {
        if(fds[i].fd == fd)
        {
            fds[i].fd = -1;
            fds[i].events = 0;
            fds[i].revents = 0;
            break;
        }
    }
}
#define MAX 10
int main()
{
    int sockfd = socket_init();
    assert(sockfd != -1);
    struct pollfd poll_fds[MAX];
    poll_fds_init(poll_fds);
    poll_fds_add(sockfd,poll_fds);
    while(1)
    {
        int n = poll(poll_fds,MAX,5000);//5000ms timeout 
        if(n < 0)printf("poll error\n");
        else if(n == 0)printf("time out \n");
        else
        {
            for(int i = 0;i<MAX;++i)
            {
                if(poll_fds[i].fd == -1)continue;
                //short has 16bits,POLLIN is 10000000 ...,
                //when revents is 10000000 ...,then the read event is going
                if(poll_fds[i].revents & POLLIN)//revents & POLLIN 不为0 则代表有读事件产生
                {
                    if(poll_fds[i].fd == sockfd)
                    {
                        struct sockaddr_in caddr;
                        int len = sizeof(caddr);
                        int c = accept(sockfd, (struct sockaddr*)&caddr, &len);
                        if(c < 0)
                        {
                            continue;
                        }
                        printf("accept:%d\n", c);
                        poll_fds_add(c, poll_fds);
                    }
                    else
                    {
                        char buff[128] = {0};
                        int num = recv(poll_fds[i].fd, buff, 127, 0);
                        if(num <= 0)
                        {
                            close(poll_fds[i].fd);
                            poll_fds_del(poll_fds[i].fd, poll_fds);
                            printf("client close\n");
                        }
                        else
                        {
                            printf("recv(%d):%s\n", poll_fds[i].fd, buff);
                            send(poll_fds[i].fd, "ok", 2, 0);
                        }
                    }

                }
                if(poll_fds[i].revents & POLLOUT)
                {
					//...
                }
            }
        }
    }    
}
```

与select的一处细节区别：

每次select除了监测`fd_set`有效描述符上有无事件，其次还将没有事件的描述符从`fd_set`移除（将该描述符对应在`fd_set`上的位进行`置0`操作）（**这样就得每次select之前都要重新注册一遍我们关注的描述符（即用户和内核共同操作FD_SET、FD_CLR等）**），然后下面过滤有事件的描述符时，只要找到`fd_set`集合哪个位是1状态即就找到了有事件产生的描述符。

而poll的用法是：用户只管注册`events`，实际上的有无事件**由内核来进行对`revents`的填充**，以此来更好地区别该描述符是否有事件产生。**这样，就不用在每次poll之前重新注册一遍我们关注的描述符的结构体里的`events`了**。我们只要把要关心的描述符的`fd`置成`非-1`，以及管理好要关心的哪些事件类型`events`即可。

## 与select相比的优点

1. 可以监听的描述符的最大数目可以超过`1024`个，大小按需自拟。
2. 可以监听的事件类型数目变多、变细了，更强大了。
3. **不用**在每次`poll`之前**重新注册**一遍我们关注的描述符的结构体里的`events`了

# 总结

select和poll的总体实现流程：
* 用户程序

select用`fd_set`结构体（默认是1024个bit位的数组）；poll用`struct pollfd fds[MAX]`结构体。

select和poll通常被循环调用。每调用一次，就拷贝一次结构体数组给内核。

> [linux在系统调用进入内核时，为什么要将参数从用户空间拷贝到内核空间？不能直接访问，或是使用memcpy吗？非要使用copy_from_user才行吗？](https://www.zhihu.com/question/19728793) - 针对为什么select/poll调用一次拷贝一次数组到内核态空间。

* 内核轮询
内核对若干个文件描述符进行轮询扫描。查看之上是否有事件发生。
时间复杂度，为O(n)
* select/poll返回后
返回值仅仅告诉用户程序发生了事件的描述符个数，并未告诉具体哪个描述符。
用户程序还需要再次轮询一遍。O(n)
![select](../../images/Linux_IO复用_select&poll/select.mp4)

