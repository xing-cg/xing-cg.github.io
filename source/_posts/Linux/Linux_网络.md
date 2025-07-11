---
title: Linux_网络
categories:
  - - Linux
  - - 网络
tags: 
date: 2021/12/4
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

# 分层模型

OSI：七层
TCP/IP协议族：四层
## 问题

1. 为什么要分层？
    1. 解耦，分工。
2. 网络：
3. 互联网：网络和网络连接起来，就是互联网
4. ipv4：是32位的，用“.”分成4个段，每个段是8位。每个段用1个10进制表示。
5. ipv6：是128位的，用“:”分成8个段，每个段是16位。每个段用4个16进制表示。
6. MAC地址：物理地址，又称硬件地址。
7. 有了MAC地址，为什么还要有ip地址？寻址！
8. 端口号：软件层次的概念。因为最终要达到进程的通信。
9. 协议：
    1. tcp
    2. http
    3. ip
    4. udp
# TCP编程流程

面试唯一要写代码的。

1. 服务器端：
    1. 创建套接字socket()
        1. 所需地址：ip+port
    2. bind()
    3. listen()
    4. c = accept()
    5. recv()
    6. send()
    7. close()
2. 客户端：
    1. socket()
    2. connect()
    3. send()
    4. recv()
    5. close()
# TCP

特点：
1. 面向连接的
    1. 三次握手--在客户端connect()时
        1. 必须是三次
    2. 四次挥手--在任一方close时
        1. 有时可以三次
2. 可靠的
    1. 应答确认
    2. 超时重传
    3. 乱序重排
    4. 去重
    5. 滑动窗口进行流量控制
3. 流式服务
    1. 发送和接收的次数可能不一致。
        1. 连续多次发送的数据可能会被对方一次性收到。
            1. 起始末尾加标记
            2. send之后recv隔开
4. tcp是有状态的
    1. 开始closed
    2. listen--connecting(三次握手中)
    3. established(已完成握手)
    4. FIN_WAIT_1/2
    5. TIME_WAIT
        1. 可靠地终止TCP的连接
        2. 让迟来的报文在这一段时间被识别，即收集后丢弃，以防止误传给下一个使用该端口的连接。

问题：1、TCP/IP协议详解 卷一；2、UNIX网络编程 卷一

1. 三次握手，四次挥手
2. 应答确认、超时重传机制
3. 乱序重排、去重、滑动窗口进行流量控制
4. 什么是粘包？怎么解决？
5. 中间转换状态的意义？TIME_WAIT状态的意义？
# UDP

特点：
1. 无连接的
    1. 没有严格意义上的服务端/客户端，只要知道ip:port就可以给对方sendto发消息。recvfrom谁的消息都可以收到，也可以收到不同终端的消息。
2. 不可靠的
    1. 没有应答确认机制，发出后是否成功看天意。
3. 数据报服务
    1. 如果send了很多信息，recv一次没收完，则剩下的数据丢包。
    2. UDP的send和recv也有缓冲区，但是其与收发的动作是一个整体，对应着发送/接收的那一次。
4. 无状态的
# UDP编程流程

在UDP场景中，其实没有严格意义上的服务端/客户端。因为编程流程很相似。

## 服务端

socket()
bind()
recvfrom()
sendto()
close()
## 客户端

socket()
【bind()：可以绑定也可以不绑定】
recvfrom()
sendto()
close()
## 与tcp的区别

1. 创建socket时的第二个参数TCP中是SOCK_STREAM，UDP中是SOCK_DGRAM(指datagrams)

```c
int sockfd = socket(AF_INET,SOCK_DGRAM,0);
```
2. recvfrom中多了两个参数，前四个参数与TCP一致，后面一个传入**要接收哪个主机的sockaddr类型的指针**，最后是传入这个**sockaddr对象的长度的指针**。为什么要传指针？因为UDP中某一端接收的数据可以来自不同主机，所以传入指针是要实时的记录谁在给接收端发数据。
3. 同样的，sendto也多了两个参数，后面一个传入**要发送到哪个的主机的sockaddr类型的指针**（参数类型为`const struct sockaddr* dest_addr`），最后传入**这个`sockaddr`对象的长度大小，通常用`sizeof`计算**。相对于`recvfrom`，**为什么不传`非const指针`了**？因为：要么是服务端，`sendto`的目的主机已经在`recvfrom`收到客户端消息后明确；要么是客户端要`sendto`的目的主机已经很明确地指定。
## 一个端口在开启TCP服务的同时，也可以开启UDP服务

我们常说：一个端口标识一个进程，但是对于不同的协议，两个进程居然可以绑定到同一端口！但是，知道为什么能这样做，再去想，这句话还是没毛病的。

只有一个端口，如何能够区分不同协议？这个区分工作通常是在应用层完成的。比如，一个进程是TCP服务端，另一个进程的UDP服务端。UDP报文过来了，该给谁；下一刻TCP数据过来了再给谁。端口虽然不能区分不同协议的数据报，但是我们最终都要把该数据包传到某个进程中去！**两个进程同时监听到了端口的数据传来，此时这个数据报头已经加了其协议头，进程在去解析协议头时就清楚了该不该继续接收这个数据报**，这样就自然的完成了对不同协议的区分。即：在应用层，每个连接就需要按照五元组来区分：{协议，源ip，源port，目的ip，目的port}。即使不区分，按照SOCK_STREAM或SOCK_DGRAM，报文类型不一样，根本就不适配，所以两者使用的端口在应用层来说是独立的。
# http

1. 浏览器解析到IP后，端口约定为80，connect三次握手建立TCP连接。
2. 浏览器给服务器发送http请求报文。浏览器send
3. 服务器给浏览器回复应答报文。浏览器recv
4. 如果2、3步只发生一次，为短连接。发生了两次以上，则为长连接，之后不会很快断开。如此可以避免握手、挥手浪费的时间。
# I/O复用

1. select
2. poll
3. epoll
## select

先观察其API

```c
#include<sys/select.h>
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
```

1. 参数
    1. nfds参数通常被设置为select监听的所有文件描述符中的最大值加1，表示在fd_set集合中，我们关心的描述符的总数。为什么加1呢？因为文件描述符是从0开始计数的。nfds与fd_set容量大小不一样，容量大小指的是FD_SETSIZE，即fd_set容量大小是fd_set可容纳描述符的最大大小。
    2. readfds参数是select关心的读事件的集合；
    3. writefds参数是select关心的写事件的集合；
    4. exceptfds参数select关心的异常事件的集合；
    5. timeout参数设置select的超时时间。
2. 返回值
    1. 集合中有事件就绪的描述符的个数
    2. 但是并没有告诉你具体是哪一个描述符就绪
### fd_set结构体

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
```

其中，__FD_SETSIZE指出select可以关注的最大文件描述符个数，默认为1024。

__fd_mask被定义为long int的类型别称，long int在32位机占8个字节。

\_\_NFDBITS计算的是1个\_\_fd\_mask元素所占用的位数，一个字节占8位，sizeof算出\_\_fd_mask的字节数，相乘得其占用的bit大小。

接着，定义fds_bits，其是long int型的数组，数组大小为\_\_FD\_SETSIZE除以\_\_NFDBITS。比如SETSIZE为1024位，NFDBITS是64位，则数组大小位1024/64=16。这里的计算主要是为了计算出数组的大小，以确定多大的数组可以正好容纳1024个位数，来记录文件描述符信息。
### 用到的宏函数

fd_set集合对于文件描述符的管理是按位进行的，而位只有0和1两种状态。

假如SETSIZE=1024，则可管理1024个文件描述符，如果文件描述符7有效，我们需要对位操作，使其位变为1。

由于位操作过于繁琐，select API中提供了一系列宏函数来方便我们访问、操作fd_set集合状态。

```c
#include<sys/select.h>
FD_ZERO(fd_set *fdset);				/*清除fdset的所有位*/
FD_SET(int fd, fd_set *fdset);		/*设置fdset的位fd*/
FD_CLR(int fd, fd_set *fdset);		/*清除fdset的位fd*/
int FD_ISSET(int fd, fd_set *fdset);/*测试fdset的位fd是否被设置*/
```
### select编程思路

最好另外定义一个整型数组，其大小为我们预测将要出现的描述符的最多数目。用作我们存放描述符的容器。初始化时将数组值一律设为-1，表示容器中该位置还没有存放描述符。如果在某一时刻有一个描述符有了消息，我们就将该描述符数值覆盖到这个容器中第一个为-1的地方。

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
#### 示例--TCP服务使用select处理多个套接字

```c
int main()
{
    int sockfd = socket_init();//socket_init封装了bind(ip:port)的操作，还封装了对sockfd进行listen的操作，并设置了监听队列大小。
    assert(sockfd!=-1);
    
    int fds[MAX];
    fds_init(fds);
    
    fds_add(sockfd, fds);
    
    fd_set fdset;//此处的fd_set即<sys/select.h>库中API提供的fd_set结构体
    while(1)//将fds[MAX]中所有有效的(即>=0)描述符全部“加入”fdset中，即把fdset中与某有效描述符对应的[位]的状态设为1。
    {
        FD_ZERO(&fdset);
        int maxfd = -1;//记录当前最大的描述符数值是多少，方便过后调用select传入第一个参数nfds。
        for(int i = 0;i<MAX;++i)
        {
            if(fds[i]==-1)continue;
            FD_SET(fds[i],fdset);
            if(maxfd<fds[i])//寻找最大描述符数值
            {
                maxfd = fds[i];
            }
        }
    	//此时，我们已经把开始创建的sockfd添加到了fdset中。下面就可以用select来监测该套接字是否有消息了。比如，sockfd监听到了客户端的connect信息，则select就可以探测到fdset中对应的sockfd位处于消息就绪态，则select就可以不再阻塞，立马返回。
        struct timeval tv = {5,0};
        int n = select(maxfd+1,&fdset,NULL,NULL,&tv);//返回在fdset集合中有信息的描述符的个数。
        if(n<0)printf("select err\n");
        else if(n == 0)printf("time out\n");
        else
        {
            for(int i = 0;i<MAX;++i)//依然需要根据fdset进行查询目前是哪个描述符有事件产生，fdset的过滤又需要根据fds的记录进行遍历。
            {
                if(fds[i]==-1)continue;
                if(FD_ISSET(fds[i],&fdset))//此处判断ISSET即是判断我们关注的描述符是否有事件产生。为什么此时标志位为1一定有事件产生？--因为在这之前我们进行了select，select不仅说明有事件产生，它还做了更多的工作：将我们关心的描述符却在其上没有事件产生的标志位置0。因此目前所有标志位为1的描述符均有事件。
                {
                    //以下才是核心的业务代码，抓住了有事件产生的描述符，对这些描述符我们的处理流程，对于不同类型的描述符，需要不同的处理流程。比如sockfd用accept处理，accept返回一个新的描述符c，则先将其加入fds容器，下一轮再用recv处理描述符c的消息。
                    if(fds[i] == sockfd)//处理监听套接字sockfd
                    {
                        struct sockaddr_in caddr;
                        int len = sizeof(caddr);
                        int c = accept(sockfd,(struct sockaddr*)&caddr,&len);
                        if(c<0)continue;
                        printf("accept c = %d\n",c);
                        fds_add(c,fds);//只是先收集到fds容器中，下一次的while扫描才将c添加到fdset集合中。
                    }
                    else//处理收发套接字，在此程序，除了sockfd皆为收发套接字c
                    {
                        char buff[128] = {0};
                        int num = recv(fds[i],buff,127,0);
                        if(num <= 0)
                        {
                            printf("client close\n");
                            close(fds[i]);
                            fds_del(fds[i],fds);
                        }
                        else//num > 0，读到了数据
                        {
                            printf("recv(c = %d) = %s\n",fds[i],buff);
                            send(fds[i],"ok",2,0);
                        }
                    }
                }
            }
        }
    }//while end
}
```

场景情况：如果客户端与select服务端已建立连接，而客户端进程结束，select会一直阻塞、未感知吗？--不会。

因为客户端的进程结束，也算是一种读事件，相当于通知服务端该套接字连接结束了。那么服务端recv会返回0，达到关闭该套接字的条件，关闭后，别忘了在fds容器中删除掉该描述符。

如果忘记了close该套接字，且忘了fds_del该描述符，那么如果客户端结束进程，服务端就会一直打印"client close"，因为select一直在探测此描述符有无读事件，若该套接字连接关闭，那么此描述符一直有读事件，recv返回0，由于没有fds_del，每次都会关注，所以每次都会打印"client close"。
## poll

可以理解为加强版的select。

先观察其API

```c
#include<poll.h>
int poll(struct pollfd* fds, nfds_t nfds, int timeout);
```

1. fds参数是一个pollfd结构类型的指针，可指向一段连续空间（数组），因此很灵活，大小可按需声明。它可以指定我们感兴趣的文件描述符上发生的可读、可写和异常等事件。定义如下

```c
struct pollfd
{
    int fd;			//文件描述符
    short events;	//注册的事件类型，按位标志
    short revents;	//实际发生的事件，按位标志，由内核填充
};
```

   1. 其中，fd成员指定文件描述符。
   2. events成员告诉poll监听fd上的哪些事件类型，他可以是一系列事件类型的**按位或**。常见的事件类型有：POLLIN(数据可读)、POLLOUT(数据可写)。
   3. revents成员由内核修改，以通知应用程序fd上实际发生了哪些事件。
### poll编程

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
                        int c = accept(sockfd,(struct sockaddr*)&caddr,&len);
                        if(c<0)continue;
                        printf("accept:%d\n",c);
                        poll_fds_add(c,poll_fds);
                    }
                    else
                    {
                        char buff[128] = {0};
                        int num = recv(poll_fds[i].fd,buff,127,0);
                        if(num <=0)
                        {
                            close(poll_fds[i].fd);
                            poll_fds_del(poll_fds[i].fd,poll_fds);
                            printf("client close\n");
                        }
                        else
                        {
                            printf("recv(%d):%s\n",poll_fds[i].fd,buff);
                            send(poll_fds[i].fd,"ok",2,0);
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

每次select除了监测fd_set有效描述符上有无事件，其次还将没有事件的描述符从fd_set移除（其实是将该描述符对应在fd_set上的位进行置0操作）（**这样就得每次select之前都要重新注册一遍我们关注的描述符（即用户和内核共同操作FD_SET、FD_CLR等）。**），然后下面过滤有事件的描述符时，只要找到fd_set集合哪个位是1状态即就找到了有事件产生的描述符。

而poll的用法是：用户只管注册events，实际上的有无事件**由内核来进行对revents的填充**，以此来更好地区别该描述符是否有事件产生。**这样，就不用在每次poll之前重新注册一遍我们关注的描述符的结构体里的events了**。我们只要把要关心的描述符的fd置成非-1，以及管理好要关心的哪些事件类型events即可。
### 与select相比的优点

1. 可以监听的描述符的最大数目可以超过1024个，大小按需自拟。
2. 可以监听的事件类型数目变多、变细了，更强大了。
3. **不用**在每次poll之前**重新注册**一遍我们关注的描述符的结构体里的events了
## epoll

epoll是Linux特有的I/O复用函数。

epoll的使用实际上不是单独的API，而是有一组函数来完成。三个函数：

```c
#include<sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```

### 内核事件表

epoll的最大优势在于处理描述符特别多的情况，相比轮询方式：

1. 如果用传统的select、poll单纯的顺序轮询方法监测就绪的描述符，那么性能会很低下的。
2. 而且虽然select、poll能顺利查出有几个描述符上有事件产生，但是是哪个描述符？并没有告诉我们，所以又得浪费一轮时间来查找产生时间是哪个描述符。