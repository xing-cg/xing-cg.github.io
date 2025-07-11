---
title: 协程_Cpp协程框架简单实现
categories:
  - - Cpp
    - Modern
  - - 操作系统
    - 多线程
tags: 
date: 2023/5/18
updated: 
comments: 
published:
---

# 进程、线程、协程的对比

|                | 进程                                    | 线程                               | 协程                              |
| -------------- | --------------------------------------- | ---------------------------------- | --------------------------------- |
| 切换者         | 操作系统                                | 操作系统                           | 用户                              |
| 切换时机       | 根据操作系统的调度策略，对用户透明      | 根据操作系统的调度策略，对用户透明 | 用户自己决定                      |
| 切换内容       | 1、页全局目录；2、内核栈；3、硬件上下文 | 1、内核栈；2、硬件上下文           | 硬件上下文                        |
| 切换内容的保存 | 保存于内核栈中                          | 保存于内核栈中                     | 保存于用户自己的变量（用户栈/堆） |
| 切换过程       | 用户态->内核态->用户态                  | 用户态->内核态->用户态             | 用户态                            |

硬件上下文可以在网上查阅资料。

## 协程的优缺点

1. IO阻塞时，协程的切换效率比较高
2. 不需要锁机制。实际上，创建了大量的协程之后，它们是按顺序执行的，不存在竞争关系。

但是不能利用多核CPU。

# 如何实现协程

利用操作系统提供的接口来实现协程。

1. Linux的ucontext
2. Windows的Fiber

## `ucontext_t`结构

根据UNIX环境高级编程第三版280页。`ucontext_t`被用作标识进程的上下文信息。该结构至少包含下列字段：

```cpp
struct ucontext_t
{
    ucontext_t * uc_link; //pointer to context resumed when this context returns
    stack_t uc_stack;     //stack used by this context
    mcontext_t uc_mcontext;//machine-specific representation of saved context
}
```

`uc_stack`字段描述了当前上下文使用的栈，至少包括下列成员：

```cpp
struct uc_stack
{
    void * ss_sp;   //stack base or pointer
    size_t ss_size; //stack size
    int ss_flags;   //flags
}
```

## ucontext提供的四个接口

```cpp
int getcontext(ucontext_t *ucp);
int setcontext(const ucontext_t *ucp);
void makecontext(ucontext_t *ucp, (void *func)(), int argc, ...);
int swapcontext(ucontext_t *oucp, const ucontext_t *ucp);//保存oucp, 换到ucp
```

1. getcontext: 保存上下文，将当前运行到的寄存器的信息保存到`ucontext_t`结构体中。
2. setcontext: 恢复上下文，将`ucontext_t`结构体变量中的上下文信息重新恢复到CPU中并执行。
3. makecontext: 修改上下文，给`ucontext_t`上下文指定一个程序入口函数，让程序从该入口函数中开始执行。
4. swapcontext: 切换上下文，保存当前上下文，并将下一个要执行的上下文恢复到CPU中。

## ucontext简单的使用示例

```cpp
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<ucontext.h>

int main()
{
    int i = 0;
    ucontext_t ctx;
    
    getcontext(&ctx);        //获取当前上下文
    printf("i = %d\n", i++);
    sleep(1);
    
    setcontext(&ctx);        //恢复上下文
    return 0;
}
```

# 协程框架设计

## 协程类

三个状态：就绪态、运行态、挂起态。

成员：

1. 上下文`m_ctx`
2. 状态`m_status`
3. 栈空间指针`m_stack`，及栈空间大小`m_stack_size`

成员方法：

1. run：纯虚函数，需要子类来实现协程的具体业务逻辑
2. resume：继续执行已经挂起的线程。
3. yield：当前协程“放弃”执行，让调度器调度另一协程继续执行。

```cpp
class Routine
{
public:
    enum Status { RT_READY = 0, RT_RUNNING, RT_SUSPEND };
    Routine();
    virtual ~Routine();
    
    virtual void run() = 0;
    void resume();
    void yield();
    int status();
protected:
    static void func(void * ptr);
private:
    ucontext_t m_ctx;
    int m_status;
    char * m_stack;
    int m_stack_size;
};
```

默认无参构造函数不分配栈空间，只有yield时才分配。

```cpp
Routine::Routine() : m_status(RT_READY), m_stack(nullptr), m_stack_size(0)
{}
```

### yield实现

```
注意：yield一般是在run方法中最后一句话中主动调用的。而run方法是在用户指定的m_ctx中执行的，所以，yield中定义的变量地址也是处于m_ctx指定的栈空间中分配的。
```

主要的工作是：

1. 把协程栈中的内容复制保存到`m_stack`中存储。**利用了一个dummy变量，探测协程栈实际使用了的边界地址（最低地址，栈的增长方向是从高到低）。**

其他的工作：

2. 把自身放入到调度器中的任务队列中；
3. 调用swapcontext。保存`m_ctx`上下文，切换到`s->m_main`上下文。

```cpp
void Routine::yield()
{
    auto s = Singleton<Schedule>::instance();
    char dummy = 0;
    assert(s->m_stack + s->m_stack_size - &dummy <= s->m_stack_size);
    if(m_stack_size < s->m_stack + s->m_stack_size - &dummy)
    {
        delete [] m_stack;
        m_stack_size = s->m_stack + s->m_stack_size - &dummy;
        m_stack = new char[m_stack_size];
    }
    std::memcpy(m_stack, &dummy, m_stack_size);
    m_status = RT_SUSPEND;
    s->append(this);
    swapcontext(&m_ctx, &s->m_main);
}
```



### resume实现

通常是Schedule单例对象来从其任务队列中出队一个Routine，并调用该routine的resume方法。

挂起状态：

1. 把`m_stack`中保存的内容复制到`s->m_stack + s->m_stack_size - m_stack_size`的位置，即从`s->m_stack`位置开始正好存储`m_stack_size`。
2. 调用`swapcontext`，保存`s->m_main`上下文，切换到`m_ctx`上下文。

```cpp
case RT_SUSPEND:
    std::memcpy(s->m_stack + s->m_stack_size - m_stack_size, m_stack, m_stack_size);// dest src size
    m_status = RT_RUNNING;
    swapcontext(&s->m_main, &m_ctx);
```

准备状态：

1. getcontext将当前运行到的寄存器的信息保存到`ucontext_t`结构体中。
2. 设置`m_ctx`的栈空间，用`m_ctx.uc_stack.ss_sp`指定，此时我们使用调度器s的共享栈空间。
3. `m_ctx.uc_link`，表示pointer to context resumed when this context returns。指向`s->m_main`。
4. 调用makecontext，指定其运行func函数，参数为1个，即this指针，并且指定其在`m_ctx`上下文中执行。
5. 调用swapcontext，保存`s->main`上下文，切换到`m_ctx`上下文。

```cpp
case RT_READY:
    getcontext(&m_ctx);
    m_ctx.uc_stack.ss_sp = s->m_stack;
    m_ctx.uc_stack.ss_size = s->m_stack_size;
    m_ctx.uc_link = &s->m_main;
    m_status = RT_RUNNING;
    makecontext(&m_ctx, (void (*)(void))Routine::func, 1, this);
    swapcontext(&s->m_main, &m_ctx);
```

以上代码的目的：指定协程使用s的公共栈。并且指定协程执行完毕回到调度器。

### 实例

```cpp
#include<iostream>
#include<routine/routine.h> 
using namespace xcg::routine;

class ARouine : public Routine
{
public:
    ARoutine(int * num) : Routine(), m_num(num) {}
    ~ARoutine()
    {
        if(m_num != nullptr)
        {
            delete m_num;
            m_num = nullptr;
        }
    }
    void run() overide
    {
        for(int i = 0; i < *m_num; ++i)
        {
            std::cout << "a run: num=" << i << std::endl;
            yield();
        }
    }
private:
    int * m_num;
}
```

main

```cpp
#include<iostream>

#include<routine/schedule.h>
using namespace xcg::routine;

#include<test/a_routine.h>
#include<test/b_routine.h>

int main()
{
    Logger::instance()->open("./main.log");
    Logger::instance()->console(false);
    
    auto s = Singleton<Schedule>::instance();
    s->create();
    
    int * num1 = new int;
    *num1 = 5;
    Routine * a = new ARoutine(num1);
    s->append(a);
    
    int * num2 = new int;
    *num2 = 10;
    Routine * b = new BRoutine(num2);
    s->append(b);
    
    while(!s->empty())
    {
        std::cout << "main run" << std::endl;
        s->resume();
    }
    return 0;
}
```

## 总结

总而言之，协程在不断地进行yield、resume。在准备/挂起、运行态中不断切换。更进一步说，本质上就是`swapcontext(&s->m_main, &m_ctx);`和`swapcontext(&m_ctx, &s->m_main);`的不断调用。

## 调度器Schedule

成员：

1. 记录上下文的变量`m_main`
2. 栈指针`m_stack`，及栈大小`m_stack_size`
3. 任务队列`m_queue`，用list存放协程指针

成员方法：

1. create：初始化协程调度器
2. append：添加一个新协程到队列
3. resume：唤醒队列里某个协程执行

```cpp
class Schedule
{
    friend class Routine;
public:
    Schedule();
    ~Schedule();
    
    void create(int stack_size = 1024 * 1024);
    void append(Routine * c);
    bool empty() const;
    int size() const;
    
    void resume();
private:
    ucontext_t m_main;
    char * m_stack;
    int m_stack_size;
    std::list<Routine *> m_queue;
}
```

### 创建实现

使用`new char[]`创建一个栈，并初始化为0。

```cpp
void Schedule::create(int stack_size)//默认1024*1024
{
    m_stack_size = stack_size;
    m_stack = new char[stack_size];
    std::memset(m_stack, 0, stack_size);
}
```

### 析构实现

```cpp
Schedule::~Schedule()
{
    for (auto it : m_queue)
    {
        delete it;
    }
    m_queue.clear();
    if (m_stack != nullptr)
    {
        delete [] m_stack;
        m_stack_size = 0;
    }
}
```

### 入队实现

```cpp
void Schedule::append(Routine * c)
{
    m_queue.push_back(c);
}
```

### resume实现

```cpp
void Schedule::resume()
{
    if (m_queue.empty())
    {
        return;
    }
    auto c = queue.front();
    m_queue.pop_front();
    c->resume();
}
```

# 基于协程的C++异步网络框架

## Server

```cpp
#pragma once
#include<string>
using std::string;

#include <utility/system.h>
#include <utility/logger.h>
#include <utility/ini_file.h>
#include <utility/singleton.h>
using namespace xcg::utility;

namespace xcg{
namespace frame{

class Server
{
public:
    Server();
    ~Server();
    
    void listen(const string & ip, int port);
    void start();
    void set_connects(int connects);
private:
    string m_ip;
    int m_port;
    int m_connects;
};
}
}
```

### 构造函数

1. 按照配置文件初始化Server类成员的ip和端口信息；
2. 初始化日志模块

```cpp
Server::Server() : m_connects(1024)
{
    System * sys = Singleton<System>::instance();
    sys->init();
    
    // init inifile
    Inifile * ini = Singleton<IniFile>::instance();
    ini->load(sys->get_root_path() + "/config/server.ini");
    
    m_ip = (string)(*ini)["server"]["ip"];
    m_port = (*ini)["Server"]["port"];
    m_connects = (*ini)["server"]["max_conn"];
    
    int level = (*ini)["server"]["log_level"];
    bool console = (*ini)["server"]["log_console"];
    
    // init logger
    Logger::instance()->open(sys->get_root_path() + "/log/server.log");
    Logger::instance()->level(level);
    Logger::instance()->console(console);
}
```

### 启动函数

1. 启动、初始化socket连接池
2. 启动调度器
3. 用`m_ip, m_port`构造ServerSocket
4. 用封装好的ServerSocket构造ServerRoutine，然后把ServerRoutine加入到调度器的任务队列中。
5. resume任务队列中的协程。

```cpp
void Server::start()
{
    Singleton<ObjectPool<Socket>>::instance()->init(m_connects);
    auto s = Singleton<Schedule>::instance();//调度器
    s->create();                             //创建第一个协程，服务协程
    s->append(new ServerRoutine(new ServerSocket(m_ip, m_port)));
    while(!s->empty())
    {
        s->resume();
    }
}
```



## 服务协程

1. 监听和建立客户端的连接
2. 当客户端没有新连接时立刻让出CPU，切换到下一个协程。
3. 当客户端有新连接时，给对应的连接新建一个client routine。

```cpp
#pargma once
#include<cerrno>
#inlcude<socket/socket.h>
using namespace xcg::socket;

#include<routine/routine.h>
#inlcude<routine/schedule.h>
using namespace xcg::routine;

namespace xcg
{
namespace async
{

class ServerRoutine : public Routine
{
public:
    ServerRoutine(Socket * socket);
    ~ServerRoutine();
    
    void run() override;
private:
    Socket * m_socket;
};

}
}
```

```cpp
#include<async/server_routine.h>
#include<async/client_routine.h>
using namespace xcg::async;

#include<utility/singleton.h>
#include<utility/object_pool.h>
using namespace xcg::utility;

ServerRoutine::ServerRoutine(Socket * socket) : Routine(), m_socket(socket)
{}

ServerRoutine::~ServerRoutine()
{
    if(m_socket != nullptr)
    {
        delete m_socket;
        m_socket = nullptr;
    }
}
void ServerRoutine::run()
{
    log_debug("Server routine run");
    while(true)
    {
        int sockfd = m_socket->accept();
        if(sockfd < 0)
        {
            log_debug("server accept would block: sockfd=%d errno=%d error=%s", sockfd, errno, strerror(errno));
            yield();                     //没有新连接，让出CPU
        }
        else//有有效连接
        {
            Socket * socket = nullptr;
            while(true)//如果连接池无资源则一直卡住申请
            {
                //ObjectPool为连接池，从连接池中取一个资源给socket。
                socket = Singleton<ObjectPool<Socket>>::instance()->allocate();
                if(socket != nullptr)
                {
                    break;
                }
                log_error("socket pool is empty");
                yield();                //连接池已满，让出CPU
            }
            // 有效连接并且申请得到资源 后的操作
            socket->m_sockfd = sockfd;
            socket->set_non_blocking(); //套接字设置为非阻塞的
            //新建一个Client Routine，添加到调度器队列。
            Singleton<Schedule>::instance()->append(new ClientRoutine(socket));
        }
    }
}
```

## 客户端协程

1. 当客户端没有数据到达，让出CPU，切换到下一个协程。
2. 当客户端有数据到达，立刻处理；
3. 当客户端关闭连接，协程自动销毁。

```cpp
#pragma once
#include<socket/socket.h>
using namespace xcg::socket;

#include<routine/routine.h>
using namespace xcg::routine;

namespace xcg{
namespace async{
class ClientRoutine : public Routine
{
public:
    ClientRoutine(Socket * socket);
    ~ClientRoutine();
    void run() override;
private:
    Socket * m_socket;
};
}
}
```

```cpp
#include<async/client_routine.h>
using namespace xcg::async;

#include<utility/singleton.h>
#include<utility/object_pool.h>
using namespace xcg::utility;

ClientRoutine::ClientRoutine(Socket * socket) : Routine(), m_socket(socket)
{}

ClientRoutine::~ClientRoutine()
{
    if(m_socket != nullptr)
    {
        Singleton<ObjectPool<Socket>>::instance()->release(m_socket);
        m_socket = nullptr;
    }
}

void ClientRoutine::run()
{
    log_debug("client routine run");
    char buf[1024];
    while(true)
    {
        std::memset(buf, 0, 1024);
        //socket已经在创建时设置为非阻塞模式，返回值有三种情况:0 ->closed <0 ->此时没数据到达 >0 ->有数据到达
        int len = m_socket->recv(buf, 1024);
        if(len == 0)
        {
            log_info("socket closed by peer");
            m_socket->close();
            break;
        }
        else if(len < 0)
        {
            log_debug("socket recv would block: len=%d, errno=%d, error=%s", len, errno, strerror(errno));
            yield();
        }
        else
        {
            log_info("socket recv: len=%d data=%s", len, buf);
            m_socket->send(buf, strlen(buf));
            yield();
        }
    }
}
```

## 入口函数

```cpp
#include<iostream>
#include<frame/server.h>
using namespace xcg::frame;

int main()
{
    Server * server = Singleton<Server>::instance();
    server->listen("127.0.0.1", 8080);
    server->start();
    
    return 0;
}
```

# 基于协程网络框架的聊天服务器

## 封装一个TcpServer类

```cpp
#pragma once
class TcpServer : noncopyable
{
public:
    using ThreadInitCallback = std::function<void(EventLoop*)>;
public:
    TcpServer(EventLoop *loop, const InetAddress &listenAddr,
              const std::string &nameArg, Option option = kNoReusePort);
    ~TcpServer();
    void setThreadInitCallback(const ThreadInitCallback &cb)
    {
        m_threadInitCallback = cb;
    }
    void setConnectionCallback(const ConnectionCallback &cb)
    {
        m_connectionCallback = cb;
    }
    void setMessageCallback(const MessageCallback &cb)
    {
        m_messageCallback = cb;
    }
    void setWriteCompleteCallback(const WriteCompleteCallback &cb)
    {
        m_writeCompleteCallback = cb;
    }

    /* 设置底层subLoop个数 */
    void setThreadNum(int numThreads);
    /* 开启服务器监听 */
    void start();
private:
    void newConnection(int sockfd, const InetAddress & peerAddr);
    void removeConnection(const TcpConnectionPtr & conn);
    void removeConnectionInLoop(const TcpConnectionPtr & conn);
private:
    ConnectionCallback      m_connectionCallback;   //新连接时的回调
    MessageCallback         m_;      //有读写消息时的回调
    WriteCompleteCallback   m_writeCompleteCallback;//消息发送完成后的回调
    ThreadInitCallback      m_threadInitCallback;   //loop线程初始化的回调
private:
    using ConnectionMap = std::unordered_map<std::string, TcpConnectionPtr>;
    ConnectionMap m_connections;
private:
    std::atomic_int m_started;
    int m_nextConnId;
};
```

```cpp
/**
 * 有新客户端连接时, acceptor会执行这个回调操作; 
 */
void TcpServer::newConnection(int sockfd, const InetAddress &peerAddr)
{
    EventLoop *ioLoop = m_threadPool->getNextLoop();
    char buf[64] = {0};
    snprintf(buf, sizeof buf, "-%s#%d", m_IPport.c_str(), m_nextConnId);
    ++m_nextConnId;
    std::string connName = m_name + buf;
    LOG_INFO("%s [%s] - new connection [%s] from %s\n",
             __FUNCTION__, m_name.c_str(), connName.c_str(), peerAddr.toIPport().c_str());
    
    //通过sockfd获取其绑定的ip地址和端口信息
    sockaddr_in local;
    bzero(&local, sizeof local);
    socklen_t addrlen = sizeof local;
    if(::getsockname(sockfd, (sockaddr*)&local, &addrlen) < 0)
    {
        LOG_ERROR("socket::getsockname\n");
    }
    InetAddress localAddr(local);

    //根据连接成功的sockfd, 创建TcpConnection连接对象
    TcpConnectionPtr conn(std::make_shared<TcpConnection>
        (ioLoop, connName, sockfd, localAddr, peerAddr));
    m_connections[connName] = conn;
    //下面的回调是用户给TcpServer设置的
    conn->setConnectionCallback(m_connectionCallback);
    conn->setMessageCallback(m_messageCallback);
    conn->setWriteCompleteCallback(m_writeCompleteCallback);

    //设置如何关闭连接 - 用户调用conn->shutdown() 时回调
    conn->setCloseCallback(std::bind(&TcpServer::removeConnection, this, _1));
    //直接调用TcpConnection::connectEstablished, 把state置为connected, channel tie操作, enableReading;
    ioLoop->runInLoop(std::bind(&TcpConnection::connectEstablished, conn));
}
```

## TcpConnection

```cpp
#pragma once
#include"noncopyable.h"
#include"inetaddress.h"
#include"buffer.h"
#include"callbacks.h"
#include<memory>
#include<atomic>
class EventLoop;
class Socket;
class Channel;
class TcpConnection : noncopyable,
               public std::enable_shared_from_this<TcpConnection>
{
public:
    enum StateE{
        kDisconnected, kConnecting, kConnected, kDisconnecting
    };
    TcpConnection(EventLoop *loop, const std::string& name, int sockfd,
                  const InetAddress& localAddr, const InetAddress& peerAddr);
    ~TcpConnection();
public:
    void connectEstablished();
    void connectDestoryed();
public:
    /* 关闭连接 */
    void shutdown();
public:
    void setConnectionCallback(const ConnectionCallback & cb)
    {
        m_connectionCallback = cb;
    }
    void setMessageCallback(const MessageCallback & cb)
    {
        m_messageCallback = cb;
    }
    void setWriteCompleteCallback(const WriteCompleteCallback & cb)
    {
        m_writeCompleteCallback = cb;
    }
    void setCloseCallback(const CloseCallback &cb)
    {
        m_closeCallback = cb;
    }
    void setHighWaterMarkCallback(const HighWaterMarkCallback & cb, size_t highWaterMark)
    {
        m_highWaterMarkCallback = cb;
        m_highWaterMark = highWaterMark;
    }
public:
    bool connected() const {return m_state == kConnected;}
    void setState(StateE state) {m_state = state;}
public:
    EventLoop * getLoop() const {return m_loop;}
    const std::string& name() const {return m_name;}
    const InetAddress& localAddress() const {return m_localAddr;}
    const InetAddress& peerAddress() const {return m_peerAddr;}
private:
    void handleRead(Timestamp receiveTime);
    void handleWrite();
    void handleClose();
    void handleError();
private:
    /**
     * 发送数据;
     * 用户会给TcpServer注册onMessageCallback, 
     * 已建立连接的用户有读写事件时, 尤其是读事件, onMessage会响应; 
     * 处理完客户端发来的事件后(onMessageCallback), 服务端会send给客户端回发消息; 
     * 
     * 收发数据的方式: 
     * 本项目的数据收发统一使用json或protobuf格式化的字符串进行, 
     * 所以此send函数的参数为了方便起见, 直接规定为string类型; 
     */
    void send(const std::string & buf);
private:
    /**
     * 发送数据;
     * 应用写的快, 而内核发送数据慢, 
     * 需要把待发送数据写入缓冲区, 而且设置了水位回调.
     */
    void sendInLoop(const void * data, size_t len);
private:
    void shutdownInLoop();
/*************************属性*************************/
private:
    ConnectionCallback	    m_connectionCallback;       //关于连接的回调
    MessageCallback	        m_messageCallback;          //已连接用户有读写事件发生时的回调
    WriteCompleteCallback   m_writeCompleteCallback;    //消息发送完成后的回调
    HighWaterMarkCallback   m_highWaterMarkCallback;    //高水位回调，为了控制收发流量稳定
    CloseCallback           m_closeCallback;            //关闭连接的回调
private:
    EventLoop *m_loop;                  //subLoop
private:
    std::unique_ptr<Socket>  m_socket;  //unique_ptr管理
    std::unique_ptr<Channel> m_channel; //unique_ptr管理
private:
    const InetAddress m_localAddr;      //本地地址信息
    const InetAddress m_peerAddr;       //对端地址信息
private:
    Buffer m_inputBuffer;               //写缓冲区
    Buffer m_outputBuffer;              //读缓冲区
private:
    const std::string m_name;
    
    std::atomic_int m_state;            //atomic，用枚举类变量赋值
    
    bool m_reading;
    
    size_t m_highWaterMark;             //高水位阈值
};
```

```cpp
void TcpConnection::handleRead(Timestamp receiveTime)
{
    int savedErrno = 0;
    ssize_t n = m_inputBuffer.readFd(m_channel->fd(), &savedErrno);
    if(n > 0)
    {
        //已建立连接的用户，有可读事件发生了，调用用户传入的回调操作onMessage
        m_messageCallback(shared_from_this(), &m_inputBuffer, receiveTime);
    }
    else if(n == 0) //客户端断开
    {
        handleClose();
    }
    else
    {
        errno = savedErrno;
        LOG_ERROR("%s\n", __FUNCTION__);
        handleError();
    }
}
void TcpConnection::handleWrite()
{
    if(m_channel->isWriting())
    {
        int savedErrno = 0;
        ssize_t n = m_outputBuffer.writeFd(m_channel->fd(), &savedErrno);
        if(n > 0)
        {
            m_outputBuffer.retrieve(n);
            if(m_outputBuffer.readableBytes() == 0)
            {
                m_channel->disableWriting();
                if(m_writeCompleteCallback)
                {
                    /* 唤醒loop对应的thread线程, 执行回调 */
                    m_loop->queueInLoop(std::bind(m_writeCompleteCallback,
                                                  shared_from_this()));
                }
                if(m_state == kDisconnecting)
                {
                    shutdownInLoop();
                }
            }
        }
        else
        {
            LOG_ERROR("%s\n", __FUNCTION__);
        }
    }
    else
    {
        LOG_ERROR("%s: connection fd = %d is down, no more writing.\n",
                   __FUNCTION__, m_channel->fd());
    }
}
void TcpConnection::handleClose()
{
    LOG_INFO("%s: fd = %d, state: %d\n", __FUNCTION__, m_channel->fd(), m_state.load());
    setState(kDisconnected);
    m_channel->disableAll();

    TcpConnectionPtr connPtr(shared_from_this());
    m_connectionCallback(connPtr);
    m_closeCallback(connPtr);
}
void TcpConnection::handleError()
{
    int optval;
    socklen_t optlen = sizeof optval;
    int err = 0;
    if(::getsockopt(m_channel->fd(), SOL_SOCKET, SO_ERROR, &optval, &optlen) < 0)
    {
        err = errno;
    }
    else
    {
        err = optval;
    }
    LOG_ERROR("%s name: %s - SO_ERROR: %d\n", __FUNCTION__, m_name.c_str(), err);
}
/**
 * 1. 判断当前连接的状态是否为connected; 
 * 2. 判断此loop是否在本thread中, 如果是则调用sendInLoop; 否则runInLoop, 绑定的函数也是sendInLoop;
 */
void TcpConnection::send(const std::string &buf)
{
    if(m_state == kConnected)
    {
        if(m_loop->isInLoopThread())
        {
            sendInLoop(buf.c_str(), buf.size());
        }
        else
        {
            m_loop->runInLoop(std::bind(&TcpConnection::sendInLoop,
                              this, buf.c_str(), buf.size()));
        }
    }
}
/**
 * 发送数据;
 * 应用写的快, 而内核发送数据慢, 
 * 需要把待发送数据写入缓冲区, 而且设置了水位回调.
 */
void TcpConnection::sendInLoop(const void * data, size_t len)
{
    ssize_t nwrote = 0;
    size_t remaining = len;
    bool faultError = false;	//记录是否产生错误
    if(m_state == kDisconnected)
    {
        LOG_ERROR("Disconnected, give up writing!");
        return;
    }
    /** 
     * m_channel->isWriting()为false表示channel第一次开始写数据, 
     * readableBytes()为0说明缓冲区没有待发送数据; 
     */
    if(!m_channel->isWriting() && m_outputBuffer.readableBytes() == 0)
    {
        nwrote = ::write(m_channel->fd(), data, len);
        if(nwrote >= 0)
        {
            remaining = len - nwrote;
            if(remaining == 0 && m_writeCompleteCallback)
            {
                //如果此时数据全部发送完毕, 不用再给channel设置epollout事件
                m_loop->queueInLoop(std::bind(m_writeCompleteCallback, shared_from_this()));
            }
        }
        else //nwrote < 0
        {
            nwrote = 0;
            if(errno != EWOULDBLOCK)
            {
                LOG_ERROR("%s\n", __FUNCTION__);
                if(errno == EPIPE || errno == ECONNRESET)// SIGPIPE or RESET
                {
                    faultError = true;
                }
            }
        }
    }
    if(!faultError && remaining > 0)//没有出错, 没有发送完毕, 剩余数据需要存到缓冲区, 然后给channel注册epollout事件, LT模式, poller发现tcp的发送缓冲区有空间, 会通知相应的sock->channel, 调用writeCallback方法, 即调用handleWrite, 直到把发送缓冲区中数据全部发送
    {
        size_t oldlen = m_outputBuffer.readableBytes();
        if(oldlen + remaining >= m_highWaterMark && oldlen < m_highWaterMark && m_highWaterMarkCallback)
        {
            m_loop->queueInLoop(std::bind(m_highWaterMarkCallback, shared_from_this(), oldlen+remaining));
        }
        m_outputBuffer.append((char*)data+nwrote, remaining);//data+nworte即剩余的位置
        if(!m_channel->isWriting())//m_channel->isWriting()为false表示channel第一次开始写数据, 之前没有注册epollout, 现在需要注册
        {
            m_channel->enableWriting();
        }
    }
}
```



## 重点在于ClientRoutine的数据处理代码编写

```cpp
   void ClientRoutine::run()
   {
       log_debug("client routine run");
       char buf[1024];
       while(true)
       {
           std::memset(buf, 0, 1024);
           //socket已经在创建时设置为非阻塞模式，返回值有三种情况:0 ->closed <0 ->此时没数据到达 >0 ->有数据到达
           int len = m_socket->recv(buf, 1024);//接受客户端发的消息
           if(len == 0)
           {
               log_info("socket closed by peer");
               m_socket->close();
               break;
           }
           else if(len < 0)
           {
               log_debug("socket recv would block: len=%d, errno=%d, error=%s", len, errno, strerror(errno));
               yield();
           }
           else
           {
               log_info("socket recv: len=%d data=%s", len, buf);
               //handleRead
               //已建立连接的用户，有可读事件发生了，调用用户传入的回调操作onMessage
               //messageCallback(m_socket, buf, time);
               
               //handle
               m_socket->send(buf, strlen(buf));//给客户端发送消息
               yield();
           }
       }
   }
```

### onMessage

```cpp
/* 上报读写事件相关信息的回调函数*/
void ChatServer::onMessage(const TcpConnectionPtr& conn,
                           Buffer* buffer,
                           Timestamp time)
{
    /**
     * muduo库或从网络上读取的数据首先在数据缓冲区;
     * 收到一个消息后，retrieveAllAsString可以把缓冲区的数据;
     * 取出到一个字符串中。
     */
    string buf = buffer->retrieveAllAsString();
    /**
     * 数据的反序列化;
     * json中包含message_type，message_id;
     */
    json js = json::parse(buf);
    /**
     * 通过js["msgid"]绑定一个回调操作，即一个id一个操作;
     * 解析到msgid后获取业务处理器handler，处理器是事先绑定的方法。网络模块看不到，业务模块绑定的;
     * 回调handler时，conn、js对象、time都可以传入;
     * 我们要达到的目的：完全解耦网络模块的代码和业务模块的代码;
     */
    auto msgHandler = ChatService::instance()->getHandler(
        js["msgid"].get<int>());    // js["msgid"] 依旧是json类型，需要转为int
    /* 回调消息绑定好的事件处理器，来执行相应的业务处理 */
    msgHandler(conn, js, time);
}
```





## ChatService类的设计与实现

```cpp
      #ifndef CHATSERVICE_H
      #define CHATSERVICE_H
      #include<muduo/net/TcpConnection.h>
      #include<unordered_map>
      #include<functional>
      
      #include"redis.hpp"
      
      #include"usermodel.hpp"
      #include"offlinemsgmodel.hpp"
      #include"friendmodel.hpp"
      #include"groupmodel.hpp"
      
      using namespace std;
      using namespace muduo;
      using namespace muduo::net;
      #include"json.hpp"
      using json = nlohmann::json;
      
      #include<mutex>
      
      /* 表示处理消息的事件回调方法类型 */
      using MsgHandler = std::function<void(const TcpConnectionPtr&, json&, Timestamp)>;
      /**
       * 聊天服务器业务类. 
       * 用映射关系来存储消息id和具体处理函数. 
       * 此类有一个实例就够了，所以采用单例模式. 
       */
      class ChatService
      {
      public:
          /* 获取单例对象的接口函数 */
          static ChatService* instance();
      public:
          /* 处理登录业务 */
          void login(const TcpConnectionPtr &conn, json &js, Timestamp time);
          /* 处理注册业务 */
          void reg(const TcpConnectionPtr &conn, json &js, Timestamp time);
      
          /* 添加好友业务 */
          void addFriend(const TcpConnectionPtr &conn, json &js, Timestamp time);
          /* 一对一聊天业务 */
          void oneChat(const TcpConnectionPtr &conn, json &js, Timestamp time);
      
          /* 创建群组业务 */
          void createGroup(const TcpConnectionPtr &conn, json &js, Timestamp time);
          /* 加入群组业务 */
          void addGroup(const TcpConnectionPtr &conn, json &js, Timestamp time);
          /* 群组聊天业务 */
          void groupChat(const TcpConnectionPtr &conn, json &js, Timestamp time);
      public:
          /* 获取消息对应的处理器 */
          MsgHandler getHandler(int msgid);
      public:
          /* 处理客户端异常退出 */
          void clientCloseException(const TcpConnectionPtr & conn);
      public:
          /* 业务重置方法，通常在服务器异常退出时调用 */
          void reset();
      #ifdef __CLUSTER__
      public:
          /* 从Redis消息队列中获取订阅的消息 */
          void handleRedisSubscribeMessage(int channel, string message);
      #endif
      
      private:
          ChatService();
      private:
          UserModel       _userModel;         /* 数据操作类对象 */
          OfflineMsgModel _offlineMsgModel;   /* 数据操作类对象 */
          FriendModel     _friendModel;       /* 数据操作类对象 */
          GroupModel      _groupModel;        /* 数据操作类对象 */
      private:
          /* 定义互斥锁，保证m_userConnectionMap的线程安全 */
          mutex _connMutex;
      private:
          /* 存储消息id和其对应的业务处理方法 */
          unordered_map<int, MsgHandler> _msgHandlerMap;
          /* 存储在线用户的通信连接 */
          unordered_map<int, TcpConnectionPtr> _userConnectionMap;
      #ifdef __CLUSTER__
      private:
          /* redis操作对象 */
          Redis m_redis;
      #endif
      };
      #endif
```

```cpp
      #include"chatservice.hpp"
      #include"public.hpp"
      #include<muduo/base/Logging.h>  // LOG_ERROR <<
      
      using namespace muduo;
      
      #include<vector>
      using namespace std;
      
      ChatService* ChatService::instance()
      {
          static ChatService service;
          return &service;
      }
      /**
       * 注册消息以及对应的Handler回调操作
       * 对unordered_map成员变量进行初始化
       * 这是业务模块的核心，也是对项目进行解耦的核心
       */
      ChatService::ChatService()
      {
          /* 键：消息id - 值：函数对象 */
          _msgHandlerMap.insert({LOGIN_MSG, std::bind(&ChatService::login, this, _1, _2, _3)});
          _msgHandlerMap.insert({REG_MSG, std::bind(&ChatService::reg, this, _1, _2, _3)});
          _msgHandlerMap.insert({ONE_CHAT_MSG, std::bind(&ChatService::oneChat, this, _1, _2, _3)});
          _msgHandlerMap.insert({ADD_FRIEND_MSG, std::bind(&ChatService::addFriend, this, _1, _2, _3)});
      #ifdef __CLUSTER__
          if(m_redis.connect())
          {
              m_redis.init_notify_handler(std::bind(
                  &ChatService::handleRedisSubscribeMessage, this, _1, _2));
          }
      #endif
      }
      /* 业务重置方法，通常在服务器异常退出时调用 */
      void ChatService::reset()
      {
          /* 把所有online用户的状态置为offline */
          _userModel.resetAllState();
      }
      /* 获取消息对应的处理器 */
      MsgHandler ChatService::getHandler(int msgid)
      {
          /* 记录错误日志，msgid没有对应的事件处理回调时 */
          auto it = _msgHandlerMap.find(msgid);
          if(it == _msgHandlerMap.end())
          {
              /* 没有找到处理器时，返回一个默认的处理器，空操作。
               * 如此设计的好处，即使没有找到程序也不会因此挂掉，进行空操作，继续运行。
               * 因为需要获取函数参数msgid，所以[=]按值获取。
               */
              return [=](const TcpConnectionPtr&, json&, Timestamp)
              {
                  LOG_ERROR << "msgid: " << msgid << " can't find handler";
              };
          }
          else
          {
              return _msgHandlerMap[msgid];
          }
      }
      /* 处理登录业务 */
      void ChatService::login(const TcpConnectionPtr &conn,
                              json &js, Timestamp time)
      {
          /* 从json参数获取账号、密码信息 */
          int id = js["id"].get<int>();
          string password = js["password"];
          User user = _userModel.query(id);
      
          json response;
          response["msgid"] = LOGIN_MSG_ACK;
          if(user.getId() == id && user.getPassword() == password)
          {
              if(user.getState() == "online")
              {
                  /* 该用户已经登录在线，不允许重复登陆 */
                  response["errno"] = LOGIN_REPEAT;
                  response["errmsg"] = "该用户已经登录";
              }
              else if(user.getState() == "offline")
              {
                  /* 登陆成功 */
                  {
                      /* 记录用户连接信息 */
                      lock_guard<mutex> lock(_connMutex);
                      _userConnectionMap.insert({id, conn});
                  }
      #ifdef __CLUSTER__
                  /**
                   * 集群环境下, 向redis订阅此id 
                   */
                  m_redis.subscribe(id);
      #endif
                  /* 更新用户状态信息 */
                  user.setState("online");
                  _userModel.updateState(user);
      
                  response["errno"] = LOGIN_SUCCEESS;
                  response["id"] = user.getId();
                  response["name"] = user.getName();
                  /* 查询该用户是否在离线时未收到的消息 */
                  vector<string> offlineMsgVec = _offlineMsgModel.query(id);
                  if(!offlineMsgVec.empty())
                  {
                      response["offlinemsg"] = offlineMsgVec;
                      /* 把该用户的所有离线消息从从数据中删除掉 */
                      _offlineMsgModel.remove(id);
                  }
                  /* 查询该用户的好友信息，并返回 */
                  vector<User> userVec = _friendModel.query(id);
                  if(!userVec.empty())
                  {
                      vector<string> friendJsonInfoVec;
                      for(User &user : userVec)
                      {
                          json js;
                          js["id"] = user.getId();
                          js["name"] = user.getName();
                          js["state"] = user.getState();
                          friendJsonInfoVec.push_back(js.dump());
                      }
                      response["friends"] = friendJsonInfoVec;
                  }
              }
          }
          else if(user.getId() != id)
          {
              /* 登录失败，用户不存在 */
              response["errno"] = LOGIN_NOTFOUND;
              response["errmsg"] = "用户不存在";
          }
          else if(user.getPassword() != password)
          {
              /* 登录失败，密码不匹配 */
              response["errno"] = LOGIN_WRONGPWD;
              response["errmsg"] = "密码验证失败";
          }
          conn->send(response.dump());
      }
      /* 处理注册业务 */
      void ChatService::reg(const TcpConnectionPtr &conn,
                            json &js, Timestamp time)
      {
          /* LOG_INFO << "do reg service."; */
          string name = js["name"];
          string password = js["password"];
      
          User user;
          user.setName(name);
          user.setPassword(password);
          bool state = _userModel.insert(user);
          json response;
          response["msgid"] = REG_MSG_ACK;
          if(state)
          {
              /* 注册成功 */
              response["errno"] = 0;
              response["id"] = user.getId();
          }
          else
          {
              /* 注册失败 */
              response["errno"] = 1;
          }
          conn->send(response.dump());
      }
      /* 处理客户端异常退出 */
      void ChatService::clientCloseException(const TcpConnectionPtr & conn)
      {
          /* 查找 */
          lock_guard<mutex> lock(_connMutex);
          User user;
          for(auto it = _userConnectionMap.begin(); it != _userConnectionMap.end(); ++it)
          {
              if(it->second == conn)
              {
                  /* 从map表删除用户的连接信息 */
                  user.setId(it->first);
                  _userConnectionMap.erase(it);
                  break;
              }
          }
      #ifdef __CLUSTER__
          /**
           * 集群环境下, 向redis取消订阅此id
           */
          m_redis.unsubscribe(user.getId());
      #endif
          /* 更新用户的状态信息 */
          if(user.getId() != -1)
          {
              user.setState("offline");
              _userModel.updateState(user);
          }
      }
      /* 一对一聊天业务 */
      void ChatService::oneChat(const TcpConnectionPtr &conn, json &js, Timestamp time)
      {
          int to = js["to"].get<int>();
          {
              lock_guard<mutex> lock(_connMutex);
              auto it = _userConnectionMap.find(to);
              if(it != _userConnectionMap.end())
              {
                  /* 接收方在线，转发消息 */
                  /* 服务器主动推送消息给接收方 */
                  it->second->send(js.dump());    // it->second 表示 
                  return;
              }
          }
      #ifdef __CLUSTER__
          /**
           * 集群环境下, 需要查询对方(to)是否在线;
           * 不可通过服务器connMap查询, 是通过数据库信息;
           */
          User user = _userModel.query(to);
          if(user.getState() == "online")
          {
              m_redis.publish(to, js.dump());
              return;
          }
      #endif
          /* 接收方离线，存储离线消息 */
          _offlineMsgModel.insert(to, js.dump());
      }
      /* 添加好友业务 */
      void ChatService::addFriend(const TcpConnectionPtr &conn, json &js, Timestamp time)
      {
          int userid = js["id"].get<int>();
          int friendid = js["friendid"].get<int>();
      
          /* 存储好友信息 -> include"friendmodel.hpp" */
          _friendModel.insert(userid, friendid);
      }
      
      /* 创建群组业务 */
      void ChatService::createGroup(const TcpConnectionPtr &conn, json &js, Timestamp time)
      {
          int userid = js["id"].get<int>();
          string groupName = js["groupname"];
          string groupDesc = js["groupdesc"];
      
          /* 存储新创建的群组信息 */
          Group group(-1, groupName, groupDesc);
          if(_groupModel.createGroup(group))
          {
              /* 把创建人加入群组 */
              _groupModel.addGroup(userid, group.getId(), "Creator");
          }
      }
      /* 加入群组业务 */
      void ChatService::addGroup(const TcpConnectionPtr &conn, json &js, Timestamp time)
      {
          int userid = js["id"].get<int>();
          int groupid = js["groupid"].get<int>();
          _groupModel.addGroup(userid, groupid, "Normal");
      }
      /* 群组聊天业务 */
      void ChatService::groupChat(const TcpConnectionPtr &conn, json &js, Timestamp time)
      {
          int userid = js["id"].get<int>();
          int groupid = js["groupid"].get<int>();
          vector<int> useridVec = _groupModel.queryGroupUsers(userid, groupid);
      
          bool offline = true;
          bool reallyOffline = true;
          for(int id : useridVec)
          {
              {
                  lock_guard<mutex> lock(_connMutex);
                  auto it = _userConnectionMap.find(id);
                  if(it != _userConnectionMap.end())//在本台服务器上线
                  {
                      offline = false;
                      reallyOffline = false;
                      it->second->send(js.dump());
                  }
              }
      #ifdef __CLUSTER__
              if(offline)
              {
                  /**
                   * 集群环境下, 需要判断其是否在其他服务器上在线;
                   */
                  User user = _userModel.query(id);
                  if(user.getState() == "online")
                  {
                      reallyOffline = false;
                      m_redis.publish(id, js.dump());
                  }
              }
      #endif
              if(reallyOffline)
              {
                  _offlineMsgModel.insert(id, js.dump());
              }
              reallyOffline = true;
              offline = true;
          }
      }
      #ifdef __CLUSTER__
      /**
       * 场景: 本台服务器收到了redis某一频道发来的通知消息;
       *  在前面的代码逻辑中, redis的publish都是判断用户在线之后才发的;
       *   但是用户可能在这个判断和真正收到消息这段时间内下线,
       *  所以, 本服务器处理收到redis某一频道上消息的回调时,
       *   需要再次判断此id是否在线, 如果在线则发送, 
       *   如果不在线, 需要存到离线消息数据库中; 
       */
      void ChatService::handleRedisSubscribeMessage(int userid, string msg)
      {
          lock_guard<mutex> lock(_connMutex);
          auto it = _userConnectionMap.find(userid);
          /* 该用户在线, 则直接发送 */
          if(it != _userConnectionMap.end())
          {
              it->second->send(msg);
              return;
          }
          /* 存储该用户的离线消息 */
          _offlineMsgModel.insert(userid, msg);
      }
      #endif
```

# 问题

1. 如果聊天内容带有`'`单引号，则会影响SQL语句的正确输入，需要处理转义。

