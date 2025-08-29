---
title: 重写muduo库
categories:
  - - 项目
    - muduo
tags: 
date: 2022/5/28
updated: 2025/8/28
comments: 
published:
---
# 内容
1. muduo库的主要板块
    1. base：公共的代码文件
    2. net：网络相关的，如TcpServer、EventLoop、poller、protobuf、protorpc等等
        1. 我们主要写网络模块
# 目标
主要编写muduo库的网络模块代码，以及改进muduo库在使用上的不便。
muduo库原本属于静态库，且需要依赖boost库。我们改进它，使他与原生C++标准库结合，并把它生成为`.so`动态库。
# muduo库核心组件职责与关系
## Reactor模式的核心
1. EventLoop事件循环
    1. 职责：每个线程一个EventLoop。不断地”询问 - 处理“事件
    2. 关系：
        1. 拥有一个`Poller`：EventLoop通过Poller来获取当前活跃的事件
        2. 拥有一个`Channel`：EventLoop管理所有在其上注册的Channel
2. Poller：I/O多路复用接口
    1. 职责：阻塞地等待文件描述符上的事件，并将活跃的事件返回给EventLoop
    2. 具体实现是`EPollPoller`。是Linux下基于epoll的具体实现。通过`epoll_wait`返回活跃的事件。
3. Channel：通道
    1. 职责：是事件分发器。每个Channel负责一个文件描述符。
    2. 内部保存了该fd关注的事件，以及对应的回调函数
    3. 关系：
        1. 是Poller和回调之间的桥梁。Poller返回一个事件，EventLoop找到对应的Channel，调用Channel的`handleEvent()`方法
## 接受新连接：Acceptor
1. 职责：Acceptor是一个特殊的Channel。专门负责处理监听套接字（listening socket）上的可读事件，即新连接。
2. 关系：
    1. 继承自Channel。同样需要向EventLoop注册。
    2. 被TcpServer拥有：TcpServer在初始化时会创建Acceptor
    3. 持有newConnectionCallback：当有新连接到来时，最终会调用TcpServer预先设置好的回调函数
## 表示连接：TcpConnection
1. 职责：代表了一个已建立的TCP连接。整个连接的生命周期（建立、断开、收发数据）都由该对象管理。
2. 关系：
    1. 每个TcpConnection对象都有一个自己的Channel。用于监控其描述符上的事件。
    2. 被TcpServer管理：记录在TcpServer的map表中。
    3. 持有各种用户回调：如连接建立回调`ConnectionCallback`，消息到达回调`MessageCallback`。这些是由用户通过`TcpServer`设置的。
    4. 隶属于某个EventLoop。每个TcpConnection对象都只属于一个特定的`EventLoop`线程。其所有IO操作都在这个线程中进行。保证线程安全。
## 服务器门面：TcpServer
1. 职责：提供给用户使用的、易于理解的**服务器类**。用户只需关注其提供的几个回调函数（如连接回调、消息回调）即可编写网络程序。
2. 关系：
    - **拥有一个 `Acceptor`**：用于接受新连接。
    - **拥有一个 `TcpConnection` 的映射表**：管理所有活跃的连接。
    - **拥有一个 `EventLoopThreadPool`**：管理线程池。
    - **设置回调**：用户通过 `TcpServer` 设置的各种回调（`onConnection`, `onMessage`），最终会“传递”给每一个新创建的 `TcpConnection` 对象。
## 线程模型：`EventLoopThread`和`EventLoopThreadPool`
- **`EventLoopThread` (IO 线程)：
    - **职责**：封装了一个线程（`std::thread`），该线程的**唯一工作**就是运行一个 `EventLoop::loop()`。**“one loop per thread”** 的理念在此体现。
    - **关系**：它**创建并拥有**一个 `EventLoop` 对象（在其内部线程中）。
- **`EventLoopThreadPool` (线程池)：
    - **职责**：管理多个 `EventLoopThread`，即管理一个 `EventLoop` 池子。它提供了一种轮询（round-robin）算法来为新的 `TcpConnection` 分配一个 `EventLoop`。
    - **关系**：
        - **被 `TcpServer` 所拥有**：`TcpServer` 通过线程池来实现多线程 Reactor。
        - **拥有多个 `EventLoopThread`**：管理着多个 IO 线程。
        - **为 `Acceptor` 提供 `getNextLoop()`**：当 `Acceptor` 接受到一个新连接时，它会从线程池中取出下一个 `EventLoop`，将这个新连接分配给这个 `EventLoop` 来监控和处理。

# cmake
```cmake
cmake_minimum_required(VERSION 2.5)
project(mymuduo)

#mymuduo 最终编译成so动态库，设置动态库的路径
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)#注意不是OUTPUT_DIRECTORY.这两者有区别
#设置为调试模式 以及 声明C++11语言标准
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -std=c++11 -fPIC")#在较新的编译器后需要加-fPIC，以示生成的是动态库
#定义参与编译的源文件 起一个别名
aux_source_directory(. SRC_LIST)
#编译生成动态库mymuduo
add_library(mymuduo SHARED ${SRC_LIST})
```
# 辅助类
## noncopyable
```cpp
// noncopyable.h
#pragma once
/**
 * noncopyable 被继承以后，
 * 派生类对象无法拷贝构造、相互赋值。
 * 无参构造、析构是默认处理。
 */
class noncopyable
{
public:
    noncopyable(const noncopyable&) = delete;
    noncopyable & operator=(const noncopyable &) = delete;
protected:
    noncopyable() = default;
    ~noncopyable() = default;
};
```
## copyable

```cpp
class copyable
{
protected:
    copyable() = default;
    ~copyable() = default;
};
```
# TcpServer概览
需要封装以下属性：
1. `EventLoop`对象指针：多路分发器，相当于epoll
2. `InetAddress`：打包IP地址和端口号
## InetAddress
1. 允许拷贝
2. 成员变量是`sockaddr_in m_addr`，也可选择支持IPv6的`sockaddr_in6 m_addr6`。可用联合体表示。在本项目中，只使用IPv4的`m_addr`。

```cpp
#include<netinet/in.h>		//sockaddr_in / sockaddr_in6都在此文件下定义
union
{
    struct sockaddr_in  m_addr;
    struct sockaddr_in6 m_addr6;
};
```
## EventLoop概览
1. 不允许拷贝
2. 主要包含的成员
    1. poller（相当于epoll），存储了一个unorderedMap，有sockfd及其上面绑定的事件
    2. channel，属性有fd、events、revents等等

EventLoop就是要完成事件循环，事件循环最重要的几个动作：epoll（**由poller负责**）、epoll监听的fd及感兴趣的事件、实际`epoll_wait`后发生的事件。
这些sockfd、感兴趣的事件、发生的事件都**记录在channel中**。

要写EventLoop就要理清楚EventLoop、Channel、Poller之间的关系。Reactor模型中，这三个组件整体对应着Demultiplex。
## Channel
通道，封装了sockfd和其感兴趣的event，如EPOLLIN、EPOLLOUT事件。还绑定了poller返回的具体事件。
### 公有别名
定义通用事件回调函数、只读事件回调函数的函数对象类型别名。
```cpp
class Channel : noncopyable
{
public:
    using EventCallback = std::function<void()>;
    using ReadEventCallback = std::function<void(Timestamp)>;
}
```
### 成员函数
#### 构造 / 析构函数
```cpp
public:
    Channel(EventLoop * loop, int fd);
    ~Channel();
```
#### `handleEvent`
fd得到poller的通知后，处理事件
```cpp
public:
    void handleEvent(Timestamp receiveTime);
```
#### `setXxxCallback(EventCallback cb)`
对外提供的设置回调函数对象的接口
```cpp
public:
    void setReadCallback(ReadEventCallback cb)
    {
        m_readCallback = std::move(cb);
    }
    void setWriteCallback(EventCallback cb)
    {
        m_writeCallback = std::move(cb);
    }
    void setCloseCallback(EventCallback cb)
    {
        m_closeCallback = std::move(cb);
    }
    void setErrorCallback(EventCallback cb)
    {
        m_errorCallback = std::move(cb);
    }
```
#### `void tie(const std::shared_ptr<void>&)`
防止channel被手动remove后，还在执行回调操作
```cpp
public:
    void tie(const std::shared_ptr<void>&);
```
#### `fd`、`events`、`revents`相关
1. `int fd()`
2. `int events()`
3. `void set_revents(int revt)`：向poller提供的设置revents的接口

```cpp
public:
    int fd() const { return m_fd; }
    int events() const { return m_events; }
    int set_revents(int revt)
    {
        m_revents = revt;
    }
```
#### 判断函数：判断有没有注册事件等等
```cpp
public:
    bool isNoneEvent() const
    {
        return m_events == kNoneEvent;
    }
    bool isWriting() const
    {
        return m_events & kWriteEvent;
    }
    bool isReading() const
    {
        return m_events & kReadEvent;
    }
```
#### 使能、使不能函数
设置fd相应的事件状态
对`m_events`进行位操作之后调用`update()`，即`epoll_ctl`。
```cpp
public:
    void enableReading()
    {
        m_events |= kReadEvent;
        update();
    }
    void disableReading()
    {
        m_events &= ~kReadEvent;
        update();
    }
    void enableWriting()
    {
        m_events |= kWriteEvent;
        update();
    }
    void disableWriting()
    {
        m_events &= ~kWriteEvent;
    }
    void disableAll()
    {
        m_events = kNoneEvent;
        update();
    }
```
#### 与EventLoop相关
获取所属的Loop
```cpp
public:
    EventLoop * ownerLoop() {return m_loop;}
```
#### 删除：remove()
```cpp
public:
    void remove();
```
#### update()：相当于调用`epoll_ctl`
```cpp
private:
    void update();
```
#### handleEventWithGuard
受保护地处理事件
```cpp
private:
    void HandleEventWithGuard(Timestamp receiveTime);
```
#### for poller的index
```cpp
public:
    int index() {return m_index;}
    void set_index(int idx) {m_index = idx;}
```
### 成员变量
1. `kXxxEvent`：以下三个变量描述当前fd的状态，没有感兴趣的事件or对读事件感兴趣or对写事件感兴趣？

```cpp
private:
    static const int kNoneEvent;
    static const int kReadEvent;
    static const int kWriteEvent;
```

2. `m_xxxCallback`：四个函数对象，可以绑定外部传入的相关操作。因为channel知道发生了哪些事情（revents记录），所以channel负责调用具体事件的回调函数。

```cpp
private:
    ReadEventCallback m_readCallback;
    EventCallback	  m_writeCallback;
    EventCallback	  m_closeCallback;
    EventCallback	  m_errorCallback;
```

3. `EventLoop *m_loop`：事件循环
4. `m_fd`：fd，即Poller监听的对象
5. `m_events`：fd感兴趣的事件注册信息
6. `m_revents`：Poller操作的fd上具体发生的事件
7. `m_index`：？
8. `std::weak_ptr<void> m_tie`：防止手动调用remove channel后仍使用此channel，用于监听跨线程的对象生存状态。
    1. `shared_ptr`和`weak_ptr`配合使用可以发挥两个能力：
        1. 解决shread_ptr循环引用问题
        2. weak_ptr在多线程程序中可监听资源的生存状态，方法是尝试提升为强指针，若提升成功，则可以访问；若提升失败说明则资源被释放掉了。
    2. tie的意思是绑定，那么m_tie要和谁绑定呢？——自己。
    3. 绑定自己的工具还可以用另一个工具，`shared_from_this`，可以尝试得到当前对象的强智能指针。
9. `bool m_tied`：配合`m_tie`使用
## Poller
### 成员变量
成员变量中包含一个存储`<int, Channel*>`的map。

>poller监听的channel从何而来？EventLoop中有ChannelList以及Poller，则poller监听的肯定是EventLoop中所保存的channel。即这些channel在poller中也被保存了。

```cpp
protected:
    using ChannelMap = std::unordered_map<int, Channel*>;
    ChannelMap m_channels;
```
还有一个成员变量，`m_ownerLoop`，指明了从属于哪个loop。
```cpp
private:
    EventLoop * m_ownerLoop;
```
### 成员函数
#### 构造 / 析构函数
```cpp
public:
    Poller(EventLoop *loop);
    virtual ~Poller() = default;
```
#### `poll`：提供给系统的统一的一个IO复用接口
```cpp
public:
    using ChannelList = std::vector<Channel*>;
    virtual Timestamp poll(int timeoutMs, ChannelList * activeChannels) = 0;
```
参数：
1. timeoutMs：超时时间，毫秒为单位
2. activeChannels：当前激活的、对事件注册好的channel列表
#### 与事件的注册、注销有关的
```cpp
public:
    /* 当fd注册的事件有变更时, channel调用update, 函数内包含updateChannel(this) */
    virtual void updateChannel(Channel * channel) = 0;
    /* 当fd注册的事件要注销时，channel调用remove，函数内包含removeChannel(this) */
    virtual void removeChannel(Channel * channel) = 0;
```
参数：channel 均为`外部channel传入的this指针`
#### `newDefaultPoller(EventLoop * loop)`
提供给EventLoop的接口，以获取默认的IO复用具体实现。

>注意：我们最好不要实现到`poller.cc`文件中，不大妥当。因为`Poller`类是基类，而把获取具体实现写到抽象类文件实现中是不好的。可以单独把实现代码写到`defaultpoller.cc`中。

```cpp
public:
    static Poller* newDefaultPoller(EventLoop *loop);
```
#### `hasChannel`
判断poller是否拥有某一channel
```cpp
public:
    virtual bool hasChannel(Channel * channel) const;
```
## EpollPoller
是Poller抽象基类的一个具体实现类。
### 成员函数
#### 构造/析构
构造相当于`epoll_create`，记录在`m_epollfd`成员变量中。析构时close该fd。
```cpp
public:
    EpollPoller(EventLoop *loop);
    ~EpollPoller() override;
```
#### `poll`：重写Poller基类方法，相当于`epoll_wait`
```cpp
public:
    Timestamp poll(int timeoutMs, ChannelList *activeChannels) override;
```
#### `update/removeChannel`：重写Poller基类方法，相当于`epoll_ctl add/mod/del`
```cpp
public:
    void updateChannel(Channel *channel) override;
    void removeChannel(Channel *channel) override;
```
#### `fillActiveChannels`：填写活跃的channels连接
```cpp
private:
    void fillActiveChannels(int numEvents, ChannelList *activeChannels) const;
```
#### `update`：更新channel通道
```cpp
private:
    void update(int operation, Channel * channel);
```
### 成员属性
1. `m_epollfd` - epoll相关的方法都需要用到fd，通过epoll_create来创建。映射的是epoll底层的文件系统红黑树。
2. `m_events` - 是一个`vector<struct epoll_event`容器。

```cpp
private:
    int m_epollfd;
    using std::vector<struct epoll_event> EventList;
    EventList m_events;
```

3. `kInitEventListSize` - `EventList`初始的长度。

```cpp
private:
    static const int kInitEventListSize = 16;
```

4. 从Poller继承而来，拥有poller包含的`ChannelMap m_channels`。

```cpp
class Poller
{
// ...
protected:
    using ChannelMap = std::unordered_map<int, Channel*>;
    ChannelMap m_channels;
// ...
}
```
### 实现代码
首先声明了三个全局常量，表示channel的状态
```cpp
const int kNew      = -1;    //从未添加到epoll的channel
const int kAdded    = 1;     //已经添加到epoll的channel
const int kDeleted  = 2;     //已把该channel从epoll中删除
```
#### 构造函数
```cpp
#include"logger.h"      //LOG_FATAL
#include<errno.h>       //errno
#include<sys/epoll.h>

EpollPoller::EpollPoller(EventLoop * loop)
  : Poller(loop),
    m_epollfd(epoll_create1(EPOLL_CLOEXEC)),	// epoll_create
    m_events(kInitEventListSize)  // vector<epoll_event>
{
    if(m_epollfd < 0)
    {
        LOG_FATAL("epoll_create error: %d\n", errno);
    }
}
```
#### 析构
```cpp
#include<unistd.h>      //close
EpollPoller::~EpollPoller()
{
    close(m_epollfd);
}
```
## CurrentThread：主要用于获取tid
`__thread`相当于C++11标准中的`thread_local`修饰符。用于修饰全局变量。
修饰之前，全局变量只能被若干线程共享。修饰之后，此全局变量变成每个线程专有的属性。
```cpp
#pragma once
#include<unistd.h>          //pid_t  syscall
#include<sys/syscall.h>     //SYS_gettid
namespace CurrentThread
{
    /**
     * @brief 此变量被__thread修饰, 相当于C++11标准中的thread_local修饰符, 
     * 用于修饰全局变量。修饰之前, 全局变量只能被若干线程共享; 
     * 修饰之后, 此全局变量变成每个线程专有的属性。
     */
    extern __thread int t_cachedTid;
    /* 通过Linux系统调用SYS_gettid, 加载当前线程的tid值到t_cachedTid */
    void cacheTid()
    {
        if(t_cachedTid == 0)
        {
            t_cachedTid = static_cast<pid_t>(syscall(SYS_gettid));
        }
    }
    /* 返回当前线程的tid, 若加载过则直接返回存储过的值 */
    inline int tid()
    {
        /* 若t_cachedTid为0说明是第一次加载, 需要调用cacheTid */
        if(__builtin_expect(t_cachedTid == 0, 0))
        {
            cacheTid();
        }
        return t_cachedTid;
    }
}
```
# EventLoop
前面的EventLoop概览中提到：
1. 不允许拷贝
2. 主要包含的成员之一：poller（相当于epoll），属性有sockfd及其上面绑定的事件
3. 另一个主要的成员是channel，属性有fd、events、revents等等

EventLoop就是要完成事件循环，事件循环最重要的几个动作：epoll（**由poller负责**）、epoll监听的fd及感兴趣的事件、实际`epoll_wait`后发生的事件。
这些sockfd、感兴趣的事件、发生的事件都**记录在channel中**。
要写EventLoop就要理清楚EventLoop、Channel、Poller之间的关系。Reactor模型中，这三个组件整体对应着Demultiplex。

由上述约束，在`.h`文件中，我们可以首先写出：

```cpp
#pragma once
#include"noncopyable.h"
class Channel;
class Poller;
class EventLoop : noncopyable
{
public:
private:
}
```

要用到函数对象。
```cpp
#include<functional>
public:
    using Functor = std::function<void()>;
```

## 成员变量

1. `ChannelList m_activeChannels` - EventLoop管理的所有的Channel的List；
   `Channel * m_currentActiveChannel` - 主要用于断言

   ```cpp
   private:
       using ChannelList = std::vector<Channel*>;
       ChannelList m_activeChannels;
   
       Channel * m_currentActiveChannel;
   ```

2. 标志(最好为atomic)

   1. `m_looping` - 事件循环状态标志 - 真则正在循环，假则将要退出循环

   2. `m_quit` - 标识退出loop循环

   3. `m_eventHandling` - 

   4. `m_callingPendingFunctors` - 标识当前loop当前是否有需要执行的回调操作

      ```cpp
      private:
          std::atomic_bool m_looping;
          std::atomic_bool m_quit;
          std::atomic_bool m_callingPendingFunctors;
      ```

3. `m_threadId` - 记录当前Loop所在线程的ID

   ```cpp
   private:
       const pid_t m_threadId;
   ```

4. `std::unique<Poller> m_poller` - EventLoop所管理的poller，去轮询监听channels上发生的事件。用`std::unique_ptr`管理

   ```cpp
   private:
       std::unique_ptr<Poller> m_poller;
   ```

5. `Timestamp m_pollReturnTime` - poller返回发生事件的channels的时间点

   ```cpp
   private:
       Timestamp m_pollReturnTime;
   ```

6. `int m_wakeupFd` - mainLoop获取到一个新用户的channel后，搭配轮询算法选择一个等待任务的subLoop，通过wakeupFd对其进行唤醒来处理channel。用`eventfd`创建。`eventfd`使用线程间的`wait/notify`事件通知机制，直接在内核唤醒，效率较高。与此处理相似的是，`libevent`使用的是`socketpair`的双向通信机制，相当于网络通信层面的机制，效率较低。

   ```cpp
   private:
       int m_wakeupFd;
   ```

7. `std::unique_ptr<Channel> m_wakeupChannel` - 把wakeupFd封装起来和其Channel关联，因为操作的往往不是fd而是其channel

   ```cpp
   private:
       std::unique_ptr<Channel> m_wakeupChannel;
   ```

8. `std::vector<Functor> m_pendingFunctors` - 存储loop需要执行的所有的回调操作。与`callingPendingFunctors`标识结合使用，如果此标识显示当前loop有需要执行的回调操作，则这些回调操作将在此vector容器中存放。**需要用mutex保护其线程安全**。

   ```cpp
   private:
       std::vector<Functor> m_pendingFunctors;
       mutable std::mutex m_mutex;
   ```

## 成员函数

1. 构造/析构
   ```cpp
   public:
       EventLoop();
       ~EventLoop();
   ```

2. `loop`/`quit` - 开始/结束事件循环

   ```cpp
   public:
       void loop();
       void quit();
   ```

3. `Timestamp pollReturnTime() const`

   ```cpp
   public:
       Timestamp pollReturnTime() const
       {
           return m_pollReturnTime;
       }
   ```

4. `runInLoop` - 在当前loop中执行cb

   ```cpp
   public:
       void runInLoop(Functor cb);
   ```

5. `queueInLoop` - 把cb放入队列中，唤醒loop所在的线程，执行cb

   ```cpp
   public:
       void queueInLoop(Functor cb);
   ```

6. `wakeup` - mainLoop唤醒subLoop所在的线程

   ```cpp
   public:
       void wakeup();
   ```

7. 更新Channel相关 - EventLoop的方法调用Poller的方法
   ```cpp
   public:
       void updateChannel(Channel *channel);
       void removeChannel(Channel *channel);
       bool hasChannel(Channel *channel);
   ```

8. `isInLoopThread` - 判断EventLoop对象是否在自己的线程里面

   ```cpp
   #include"currentthread.h"
   public:
       bool isInLoopThread() const
       {
           return m_threadId == CurrentThread::tid();
       }
   ```

9. handleRead - 处理wakeup唤醒相关的逻辑
   ```cpp
   private:
       void handleRead();
   ```

10. doPendingFunctors - 执行回调
    ```cpp
    private:
        void doPendingFunctors();
    ```

## 实现代码

### 全局变量

1. 防止一个线程创建多个loop
   ```cpp
   //__thread修饰表示这个全局变量转为了每个线程私有的属性
   __thread EventLoop *t_loopInThisThread = nullptr;
   ```

2. 默认的超时时间
   ```cpp
   const int kPollTimeMs = 10000;
   ```

### 全局函数

1. createEventfd() - 创建wakeupfd，用来通知等待任务的subLoop，处理新的Channel事件。

   ```cpp
   #include<sys/eventfd.h>
   int createEventfd()
   {
       int evtfd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
       if(evtfd < 0)
       {
           LOG_FATAL("Failed in eventfd: %d\n", errno);
       }
       return evtfd;
   }
   ```


# EventLoopThread

EventLoop组件及其内部的Chennel、Poller已经在上文讨论。要和thread结合达成最终的"one loop per thread"模型，较好的办法就是将EventLoop与thread组合封装。
## Thread

```cpp
class Thread : noncopyable
```

### 线程函数
线程最主要的组成部分就是线程函数。
```cpp
public:
    using ThreadFunc = std::function<void()>;
```

> 使用无返回值+无参数是为了便于统一线程函数的形式，具体绑定回调则使用函数对象绑定器。
### 成员变量
用C++线程库、智能指针。
Thread对象刚创建不会执行线程函数，而是在成员函数`start()`被调用时，用智能指针创建C++ 11的thread线程才开始真正执行。
1. `m_started` - 表示
2. `m_joined`
3. `std::shared_ptr<std::thread> m_thread`
4. `pid_t m_tid`
5. `ThreadFunc m_func` - 存储线程函数
6. `std::string m_name`
7. `static std::atomic_int m_numCreated` - 目前产生了线程对象的计数值
### 成员方法
#### setDefaultName
构造函数中如果没有传入name则赋"Thread %d "，%d为已创建的线程对象数目。
```cpp
private:
    void setDefaultName();
```
#### 构造 / 析构函数
```cpp
public:
    explicit Thread(ThreadFunc, const std::string & name = std::string());
    ~Thread();
```
#### start
```cpp
public:
    void start();
```
#### join
```cpp
public:
    void join();
```
#### 获取线程状态相关的标志、信息
1. started
2. tid
3. name
4. numCreated

```cpp
public:
    bool started() const {return m_started;}
    pid_t tid() const {return m_tid;}
    const std::string & name() const {return m_name;}
    static int numCreated() {return m_numCreated;}
```
### 代码实现
```cpp
#include"thread.h"
#include"currentthread.h"
#include<semaphore.h>
std::atomic_int Thread::m_numCreated(0);
void Thread::setDefaultName(int numCreated)
{
    char buf[32] = {0};
    snprintf(buf, sizeof buf, "Thread%d", numCreated);
    m_name = buf;
}
Thread::Thread(ThreadFunc func, const std::string & name)
    : m_started(false), m_joined(false), m_tid(0),
      m_func(std::move(func)), m_name(name)
{
    int num = ++m_numCreated;
    if(m_name.empty())
    {
        setDefaultName(num);
    }
}
Thread::~Thread()
{
    /* 线程要么join, 要么detach */
    if(m_started && !m_joined)
    {
        m_thread->detach();
    }
}
void Thread::start()
{
    m_started = true;
    /* 为了tid初始化时期的线程安全, 保证tid有效 */
    sem_t sem;
    sem_init(&sem, false, 0);   //地址, 是否进程间共享, 初始值0
    //下面这句才是真正去创建一个立即执行的子线程。而且下面这个语句是子线程的生命周期。
    //从创建完m_thread后主线程、子线程分离, 主线程需要等待子线程执行完毕, 可以用信号量来控制。
    m_thread = std::make_shared<std::thread>([&](){
        m_tid = CurrentThread::tid();
        
        sem_post(&sem);
        
        m_func();
    });
	//这里是主线程的代码，只有sem的值变为>0时才能往下走。
    sem_wait(&sem);
}
void Thread::join()
{
    m_joined = true;
    m_thread->join();
}
```
## EventLoopThread
```cpp
class EventLoopThread : noncopyable
```
封装的目标：在thread线程对象上运行一个loop。
### 线程初始化时回调函数
```cpp
public:
    using ThreadInitCallback = std::function<void(EventLoop*)>;
```
### 成员变量
1. `m_loop` - 存储Eventloop对象指针

```cpp
private:
    EventLoop * m_loop;
```

2. `m_thread` - 存储线程对象

```cpp
private:
    Thread m_thread;
```

3. `bool m_exiting` - 线程正在退出的标志

```cpp
private:
    bool m_exiting;
```

4. `ThreadInitCallback m_callback` - 线程初始化调用的回调操作，在EventLoopThread构造时在第1个参数传入，默认是一个空操作。

```cpp
private:
    ThreadInitCallback m_callback;
```
### 成员函数
#### 构造 / 析构函数
构造函数参数：可以传入一个线程初始化回调函数对象；还有name。其中，回调函数对象默认构造为空操作。
```cpp
public:
EventLoopThread(const ThreadInitCallback &cb = ThreadInitCallback(),
                const std::string & name = std::string());
~EventLoopThread();
```
#### startLoop - 开启循环
```cpp
public:
    EventLoop * startLoop();
```
#### threadFunc - 线程函数
```cpp
private:
    void threadFunc();
```
### 代码实现
1. 构造函数
    1. 主要的工作就是构造EventLoopThread中的Thread对象即`m_thread`成员。Thread对象`m_thread`绑定的线程函数是用`std::bind`绑定的函数，用的是EventLoopThread类中的threadFunc函数，并且绑定了其this指针。EventLoopThread构造函数的第2个参数name将作为`m_thread`的名字。
    2. 第1个参数指定的是`线程初始化时的回调函数`，**与线程start后执行的线程函数无关**。第1个参数的默认值和第2个参数的默认值在`.h`文件中已指出。`ThreadInitCallback()`是指创建一个默认函数对象，函数执行空操作。
    3. Thread对象构造完成后，不会立即执行`线程函数threadFunc`，因为Thread构造并不意味着C++11标准库的thread创建完毕。只有调用`m_thread.start()`才会真正执行`线程函数threadFunc`。
    4. 构造函数还把传入的`线程初始化时回调函数cb`保存到了`m_callback`成员。

```cpp
#include"eventloopthread.h"
#include"eventloop.h"
EventLoopThread::EventLoopThread(const ThreadInitCallback &cb,
                const std::string &name)
  : m_loop(nullptr), m_exiting(false),
    m_thread(std::bind(&EventLoopThread::threadFunc, this), name),
    m_mutex(), m_cond(), m_callback(cb)
{

}
EventLoopThread::~EventLoopThread()
{
    m_exiting = true;
    if(m_loop != nullptr)
    {
        m_loop->quit();
        m_thread.join();
    }
}
EventLoop * EventLoopThread::startLoop()
{
    m_thread.start();   //启动底层新线程，执行回调函数，
    
    EventLoop * loop = nullptr;
    {/* 临界区m_loop */
        std::unique_lock<std::mutex> lock(m_mutex);
        while(m_loop == nullptr)
        {
            m_cond.wait(lock);
        }
        loop = m_loop;
    }/* 临界区m_loop */
    return loop;
}
/**
* @brief Thread对象实际执行的线程函数，在单独的子线程中执行
*/
void EventLoopThread::threadFunc()
{
    EventLoop loop; //构造一个独立的eventloop, 和m_thread一对一, one loop per thread的证据
    if(m_callback)	//如果m_callback(即ThreadInitCallback)不为空则执行此函数
    {
        m_callback(&loop);
    }
    {/* 临界区m_loop */
        std::unique_lock<std::mutex> lock(m_mutex);
        m_loop = &loop;
        m_cond.notify_one();	//通知主线程的startLoop(), loop已经在子线程创建好了。
    }/* 临界区m_loop */
    loop.loop();    //EventLoop loop => Poller.poll
    
    /* 执行到此处说明loop已经结束 退出循环 */
    std::unique_lock<std::mutex> lock(m_mutex);
    m_loop = nullptr;
}
```
# EventLoopThreadPool
```cpp
class EventLoopThreadPool : noncopyable
```
## 线程初始化时回调函数

```cpp
public:
    using ThreadInitCallback = std::function<void(EventLoop*)>;
```
## 成员变量
1. `m_baseLoop` - 用户最开始创建的loop
2. 标志相关
    1. `std::string m_name`
    2. `bool m_started`
    3. `int m_numThreads`
    4. `int m_next`
3. `std::vector<std::unique_ptr<EventLoopThread>> m_threads` - 包含了所有创建的线程
4. `std::vector<EventLoop*> m_loops` - 包含了所有管理着的loop的指针，通过`m_threads`中的某个thread进行`startLoop()`返回loop的指针。
## 成员函数
### 构造/析构函数
```cpp
public:
    EventLoopThreadPool(EventLoop * baseLoop, const std::string &nameArg);
    ~EventLoopThreadPool();
```
#### `setThreadNum(int)` - 供TcpServer调用
```cpp
public:
    void setThreadNum(int numThreads)
    {
        m_numThreads = numThreads;
    }
```
#### `start` - 开启事件循环线程
```cpp
public:
    void start(const ThreadInitCallback &cb = ThreadInitCallback());
```
#### `getNextLoop` - 如果工作在多线程中，baseLoop默认以轮询的方式分配channel给subLoop
```cpp
public:
    EventLoop * getNextLoop();
```
#### `getAllLoops` - 获取所有管理着的loop，存到vector中，相当于拷贝了`m_loops`

```cpp
public:
    std::vector<EventLoop*> getAllLoops();
```
#### 获取各种状态、信息
1. started
2. name

```cpp
public:
    bool started() const {return m_started;}
    const std::string name() const {return m_name;}
```
## 代码实现
```cpp
#include"eventloopthreadpool.h"
#include"eventloopthread.h"
EventLoopThreadPool::EventLoopThreadPool(EventLoop *baseLoop, const std::string &nameArg)
    : m_baseLoop(baseLoop),
      m_name(nameArg),
      m_started(false),
      m_numThreads(0),
      m_next(0)
{

}
EventLoopThreadPool::~EventLoopThreadPool()
{
    /**
     * nothing to do, bacause evey loop is on the thread stack,
     * that will destruct automatically.
     */
}
void EventLoopThreadPool::start(const ThreadInitCallback &cb)
{
    m_started = true;
    for(int i = 0; i < m_numThreads; ++i)
    {
        char buf[m_name.size() + 32];
        /* 以线程池name + 下标序列号 作为thread线程的名字 */
        snprintf(buf, sizeof buf, "%s%d", m_name.c_str(), i);
        std::string threadName(buf);
        m_threads.push_back(std::make_unique<EventLoopThread>(cb, threadName));
        m_loops.push_back(m_threads.back()->startLoop());
    }
    /* m_numThreads == 0时, 上面for循环不会执行, 执行下面的操作 */
    if(m_numThreads == 0 && cb != nullptr)
    {
        cb(m_baseLoop);
    }
}
/* 体现了对subLoop的轮询算法 */
EventLoop* EventLoopThreadPool::getNextLoop()
{
    EventLoop * loop = m_baseLoop;
    if(!m_loops.empty())
    {
        loop = m_loops[m_next];
        ++m_next;
        if(m_next >= m_loops.size())
        {
            m_next = 0;
        }
    }
    return loop;
}
std::vector<EventLoop*> EventLoopThreadPool::getAllLoops()
{
    if(m_loops.empty())
    {
        return std::vector<EventLoop*>(1, m_baseLoop);
    }
    else
    {
        return m_loops;
    }
}
```
# Acceptor
mainReactor主要的工作是处理客户端的连接请求，然后把sockfd轮询分配给subReactors。

这个工作由mainReactor中的acceptor处理。处理的流程和TCP socket编程流程基本一致。需要有一个listenfd，即监听套接字，去其中的监听队列取可用的连接。即Acceptor主要就是对若干sockfd的封装。
## socket
### .h文件

```cpp
#pragma once
#include"noncopyable.h"
class InetAddress;
class Socket : noncopyable
{
public:
    explicit Socket(int sockfd)
        : m_sockfd(sockfd)
    {

    }
    ~Socket();
    int fd() const {return m_sockfd;}
    void bindAddress(const InetAddress &localAddr);
    void listen();
    int accept(InetAddress *peerAddr);
public:
    void shutdownWrite();
    /* 更改TCP选项, 直接交付, 不进行缓存 */
    void setTcpNoDelay(bool on);
    /* 更改TCP选项 */
    void setReuseAddr(bool on);
    /* 更改TCP选项 */
    void setReusePort(bool on);
    /* 更改TCP选项 */
    void setKeepAlive(bool on);
private:
    const int m_sockfd;
};
```
### .cc文件
```cpp
#include"socket.h"
#include"logger.h"
#include"inetaddress.h"
#include<unistd.h>  //close
#include<sys/socket.h>  //bind
#include<strings.h> //bzero
#include<netinet/tcp.h> //TCP_NODELAY
Socket::~Socket()
{
    close(m_sockfd);
}
void Socket::bindAddress(const InetAddress &localAddr)
{
    /* bind return 0 when success */
    if(0 != ::bind(m_sockfd, (sockaddr*)localAddr.getSockAddr(), sizeof(sockaddr_in)))
    {
        LOG_FATAL("bind sockfd: %d fail, createNonblocking or Die\n", m_sockfd);
    }
}
void Socket::listen()
{
    if(0 != ::listen(m_sockfd, 1024))
    {
        LOG_FATAL("listen sockfd: %d fail\n", m_sockfd);
    }
}
int Socket::accept(InetAddress * peerAddr)
{
    struct sockaddr_in addr;
    socklen_t len;
    bzero(&addr, sizeof addr);
    int connfd = ::accept(m_sockfd, (sockaddr*)&addr, &len);
    if(connfd >= 0)
    {
        peerAddr->setSockAddr(addr);
    }
    return connfd;
}
void Socket::shutdownWrite()
{
    if(::shutdown(m_sockfd, SHUT_WR) < 0)
    {
        LOG_ERROR("shutdown Write error\n");
    }
}
void Socket::setTcpNoDelay(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(m_sockfd, IPPROTO_TCP, TCP_NODELAY, &optval, sizeof optval);
}
void Socket::setReuseAddr(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(m_sockfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof optval);
}
void Socket::setTcpNoDelay(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(m_sockfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof optval);
}
void Socket::setKeepAlive(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(m_sockfd, SOL_SOCKET, SO_KEEPALIVE, &optval, sizeof optval);
}
```
## Acceptor
```cpp
class Acceptor : noncopyable
```
### 收到新连接时的回调

```cpp
public:
    using NewConnectionCallback = std::function<void(int fd, const InetAddress&)>;
```
### 成员变量
1. `m_loop`
2. `m_acceptSocket`
3. `m_acceptChannel`
4. `m_newConnectionCallback` - 把fd打包为channel，getNextLoop唤醒一个subLoop，把channel分发给subLoop。
5. `m_listening`
```cpp
private:
    EventLoop * m_loop;
    Socket m_acceptSocket;
    Channel m_acceptChannel;
    NewConnectionCallback m_newConnectionCallback;
    bool m_listenning;
```
### 成员函数
#### 构造/析构
```cpp
public:
    /* 此构造的三个参数本身也是TcpServer的三个参数 */
    Acceptor(EventLoop * loop, const InetAddress & listenAddr, bool reusePort);
    ~Acceptor();
```
#### listen
```cpp
public:
    void listen();
```
#### get/set
1. setNewConnectionCallback
2. listening

```cpp
public:
    void setNewConnectionCallback(const NewConnectionCallback &cb)
    {
        m_newConnectionCallback = cb;
    }
    bool listenning() const {return m_listenning;}
```
#### handleRead
```cpp
private:
    void handleRead();
```
### 代码实现
#### 构造
1. 由传入的`loop`对`m_loop`初始化；创建一个NonBlock的Tcp socketfd并用于构造`m_acceptSocket`；把loop和刚才创建好的socketfd打包构造`m_acceptChannel`；设置各种标志。
2. 根据传入的第2个参数`listenAddr`，`bindAddress`绑定地址到socket上。
3. TcpServer调用start()后，意味着acceptor要对listen sockfd进行listen。如果接收到了新用户的连接，需要执行一个回调（具体操作是把connfd->channel->subloop）。所以还要设置一个ReadCallback。

```cpp
static int createNonblockingSocket()
{
    int sockfd = ::socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, IPPROTO_TCP);
    if(sockfd < 0)
    {
        LOG_FATAL("%s:%s:%d listen socketfd create err: %d\n", __FILE__, __FUNCTION__, __LINE__, errno);
    }
    return sockfd;
}
Acceptor::Acceptor(EventLoop *loop, const InetAddress &listenAddr, bool reusePort)
  : m_loop(loop),
    m_acceptSocket(createNonblockingSocket()),    //create socket
    m_acceptChannel(loop, m_acceptSocket.fd()),
    m_listenning(false)
{
    m_acceptSocket.setReuseAddr(true);
    m_acceptSocket.setReusePort(true);
    m_acceptSocket.bindAddress(listenAddr);         //bind addr to socket
    // TcpServer::start() Acceptor::listen(), 如果有新连接需要执行回调 connfd->channel->subloop
    //baseLoop -> m_acceptChannel(listenfd) -> 
    m_acceptChannel.setReadCallback(std::bind(&Acceptor::handleRead, this));
}
```

#### handleRead - listenfd有事件发生, 即有新用户链接时的回调操作

#### accept一个connfd
把fd和peerAddr交给newConnectionCallback处理。newConnectionCallback是TcpServer中编写的。主要工作是**轮询找到subLoop, 唤醒, 分配当前新客户端的channel**。

```cpp
/* listenfd有事件发生, 即有新连接 */
void Acceptor::handleRead()
{
   InetAddress peerAddr;
   int connfd = m_acceptSocket.accept(&peerAddr);
   if(connfd > 0)
   {
       if(m_newConnectionCallback)
       {
           /* 轮询找到subLoop, 唤醒, 分配当前新客户端的channel */
           m_newConnectionCallback(connfd, peerAddr);
       }
       else
       {
           close(connfd);
       }
   }
   else
   {
       LOG_ERROR("%s:%s:%d accept err: %d", __FILE__, __FUNCTION__, __LINE__, errno);
       if(errno == EMFILE)
       {
           LOG_ERROR("(sockfd reached max limit)");
       }
       LOG_ERROR("\n");
   }
}
```
#### 析构
```cpp
Acceptor::~Acceptor()
{
    m_acceptChannel.disableAll();
    m_acceptChannel.remove();
}
```
#### listen
1. 设置listenning标志
2. 调用`m_acceptSocket`的listen
3. 调用`m_acceptChannel`的enableReading，即把`m_acceptChannel`注册到Poller中。

```cpp
void Acceptor::listen()
{
    m_listenning = true;
    m_acceptSocket.listen();                        //listen
    m_acceptChannel.enableReading();                //m_acceptChannel -> poller
}
```
# TcpServer

考虑一个问题：用户使用muduo库编写服务器程序时，为了避免用户再去困惑引入哪些头文件，我们在tcpserver.h中把该引入的头文件全引入进去，而不再只是对类前置声明了。
```cpp
#pragma once
#include"eventloop.h"
#include"acceptor.h"
#include"inetaddress.h"
#include"noncopyable.h"
class TcpServer : noncopyable
```
## 回调

所有的回调，都是用户设置到TcpServer后，TcpServer再自己设置到EventLoop中的。

以下是TcpServer类中包含的回调操作属性。（成员变量）

```cpp
private:
    ThreadInitCallback	    m_threadInitCallback;
    ConnectionCallback 	    m_connectionCallback;
    MessageCallback	        m_messageCallback;
    WriteCompleteCallback   m_writeCompleteCallback;
    HighWaterMarkCallback   m_highWaterMarkCallback;
```

### 线程初始化时的回调
直接声明在TcpServer class中。
```cpp
public:
    using ThreadInitCallback = std::function<void(EventLoop*)>;
```

### 连接、读写事件的回调 - 单独写到callbacks.h文件中

>为了对各种回调函数进行管理，写到单独的头文件`callbacks.h`中。

1. ConnectionCallback - 有关连接的回调。
2. CloseCallback - 关闭连接的回调
3. WriteCompleteCallback - 消息发送完成后的回调
4. MessageCallback - 已连接用户有读写事件发生时的回调
5. HighWaterMarkCallback - 高水位回调，为了控制收发流量稳定

```cpp
#pragma once
#include<memory>
#include<functional>
class Buffer;
class TcpConnection;

using TcpConnectionPtr = std::shared_ptr<TcpConnection>;

using ConnectionCallback = std::function<void(const TcpConnectionPtr&)>;
using CloseCallback = std::function<void(const TcpConnectionPtr&)>;
using WriteCompleteCallback = std::function<void(const TcpConnectionPtr&)>;
using HighWaterMarkCallback = std::function<void(const TcpConnectionPtr&, size_t)>;

using MessageCallback = std::function<void(const TcpConnectionPtr &,
                                           Buffer *, Timestamp)>;
```

## 枚举声明

Option枚举类，直接声明在TcpServer类中。

`Option.kReusePort`/`Option.kNoReusePort`表示端口是否可重用。

```cpp
public:
    enum Option
    {
        kNoReusePort,
        kReusePort
    };
```

## 成员变量

其中，回调函数属性已经在上面给出。
1. `ConnectionMap m_connections` - 保存所有的连接

   ```cpp
   private:
       using ConnectionMap = std::unoredered_map<std::string, TcpConnectionPtr>;
       ConnectionMap m_connections;
   ```

2. `EventLoop *m_loop` - 用户实现定义的baseLoop

   ```cpp
   private:
       EventLoop * m_loop;
   ```

3. `m_IPport` - 存储IP和端口的字符串

   ```cpp
   private:
       const std::string m_IPport;
   ```

4. `std::unique_ptr<Acceptor> m_acceptor` - 运行在mainLoop，任务是监听新连接事件。

5. `std::unique_ptr<EventLoopThreadPool> m_threadPool` - 

6. 标识

   1. `std::string m_name` - TcpServer的易记忆名字
   2. `atomic_bool started` - 是否已启动

   ```cpp
   private:
       std::string m_name;
       atomic_bool started;
   ```

7. `m_nextConnId`

   ```cpp
   private:
       int m_connId;
   ```


## 成员函数

1. 构造/析构

   ```cpp
   public:
       TcpServer(EventLoop * loop, const InetAddress &listenAddr,
                 const std::string &nameArg, Option option = kNoReusePort);
       ~TcpServer();
   ```

2. 设置回调
   ```cpp
   public:
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
   ```

3. start - 开启服务器监听，即mainLoop的acceptor的listen
   ```cpp
   public:
       void start();
   ```

4. 关于Connection
   ```cpp
   private:
       void newConnection(int sockfd, const InetAddress& peerAddr);
       void removeConnection(const TcpConnectionPtr& conn);
       void removeConnectionInLoop(const TcpConnectionPtr& conn);
   ```

## 代码实现

1. 构造，有三个参数，loop指针，InetAddress，标识名称name，端口是否重用Option

   1. 对`m_loop`进行赋值，需要做非空检查

      ```cpp
      EventLoop* CheckLoopNotNull(EventLoop* loop)
      {
          if(loop == nullptr)
          {
              LOG_FATAL("%s:%s:%d mainLoop is null\n", __FILE__, __FUNCTION__, __LINE__);
          }
          return loop;
      }
      ```

   2. 对`m_IPport`进行赋值，值是调用参数`listenAddr`中的函数`toIPport()`获得的

   3. 对`m_name`赋值

   4. 对`m_acceptor`进行构造(unique_ptr)。传入参数`loop`，`listenAddr`，`option`

      1. 创建socket
      2. socket的fd和loop指针封装为Channel - `m_acceptChannel(loop, m_acceptSocket.fd())`
      3. 设置channel的ReadCallback回调
      4. 当TcpServer调用start时，acceptor将会调用listen，将调用`m_acceptChannel`的`enableReading()`函数，往其相应的loop中进而在poller中注册事件。
      5. loop等待事件，如果发生事件，调用channel的handleEvent，进而执行readCallBack。acceptor的readCallBack在构造时绑定为handleRead，工作是对channel的socket进行accept。

   5. 对`m_threadPool`进行构造(shared_ptr)。传入参数`loop`，`m_name`

   6. 设置回调

      1. `m_connectionCallBack`
      2. `m_messageCallBack`

   7. `m_nextConnId`

   ```cpp
   #include<functional>
   using namespace std::placeholders;
   
   TcpServer::TcpServer(EventLoop *loop, const InetAddress &listenAddr,
                        const std::string &nameArg, Option option = kNoReusePort)
       : m_loop(CheckLoopNotNull(loop)),
         m_IPport(listenAddr.toIPport()),
         m_name(nameArg),
         m_acceptor(new Acceptor(loop, listenAddr, option == kReusePort)),
         m_threadPool(new EventLoopThreadPool(loop, m_name)),
         m_connectionCallback(),
         m_messageCallback(),
         m_nextConnId(1)
   {
       m_acceptor->setNewConnectionCallback(std::bind(&TcpServer::newConnection,
                                                      this, _1, _2));//connfd, peerAddr
   }
   ```

2. newConnection - 运行在主线程当中，主线程的mainLoop调用此函数，选择了一个ioLoop，**在非子loop的线程中（即当前是在mainThread）执行cb，就需要唤醒子loop所在线程（subThread），执行cb**，即调用subLoop的`queueInLoop(cb)`。

   1. 根据轮询算法选择一个subLoop，即调用`m_threadPool->getNextLoop()`
   2. 唤醒subLoop
   3. 把当前connfd封装成channel分发给subloop

3. setThreadNum - 设置底层subLoop的个数
   ```cpp
   void TcpServer::setThreadNum(int numThreads)
   {
       m_threadPool->setThreadNum(numThreads);
   }
   ```

4. start - 开启服务器监听
   ```cpp
   void TcpServer::start()
   {
       //防止一个TcpServer对象被start多次;
       if(m_started++ == 0)//即使bool为1，bool++后的值也还是1
       {
           m_threadPool->start(m_threadInitCallback);
           m_loop->runInLoop(std::bind(&Acceptor::listen, m_acceptor.get()));
       }
   }
   ```


# TcpConnection

顾名思义，此类对象表示的是tcp通信中，客户端和服务器之间成功建立的一条连接。主要封装用户在服务端的数据。

1. mainLoop通过acceptor接收到新的连接时，将会把fd和loop封装到channel，继而封装到TcpConnection中，再通过轮询算法交给subLoop。
2. 更重要的是，TcpConnection中存储了一些连接事件、读写事件发生时的回调。
3. 最后，TcpConnection还还封装了Buffer。

## Buffer

基于非阻塞IO的服务端编程，Buffer是必不可少的。比如解决TCP粘包问题。

### 成员变量

1. `std::vector<char> m_buffer` - 数据数组。
2. `size_t m_readerIndex` - 数据可读的位置下标
3. `size_t m_writerIndex` - 数据可写的位置下标

```cpp
private:
    std::vector<char> m_buffer;
    size_t m_readerIndex;
	size_t m_writerIndex;
```

除此之外，还有两个静态常量。

1. `kCheapPrepend` - 记录数据包的长度
2. `kInitialSize` - 数据包的初始大小值。

```cpp
public:
    static const size_t kCheapPrepend = 8;
    static const size_t kInitialSize = 1024;
```

### 成员函数

1. 构造/析构
   ```cpp
   public:
       explicit Buffer(size_t initialSize = kInitialSize)
           : m_buffer(kCheapPrepend + initialSize),
             m_readerIndex(kCheapPrepend),
             m_writerIndex(kCheapPrepend)
       {
       }
       ~Buffer() = default;
   ```

2. readableBytes、writableBytes、prependableBytes
   ```cpp
   public:
       size_t readableBytes() const
       {
           return m_writerIndex - m_readerIndex;
       }
       size_t writableBytes() const
       {
           return m_buffer.size() - m_writerIndex;
       }
       size_t prependableBytes() const
       {
           return m_readerIndex;
       }
   ```

3. 返回指针

   1. begin - 获取buffer实际首部指针
   2. peek - 返回缓冲区数据包中可读数据起始位置
   3. beginWrite - 返回可写的数据起始位置

   ```cpp
   private:
       char * begin()
       {
           return &*m_buffer.begin();
       }
       const char * begin() const
       {
           return &*m_buffer.begin();
       }
   public:
       const char * peek() const
       {
           return begin() + m_readerIndex;
       }
       char * beginWrite()
       {
           return begin() + m_writerIndex;
       }
       const char * beginWrite() const
       {
           return begin() + m_writerIndex;
       }
   ```

4. `retrieve`/`retrieveAll`/`retrieveAsString`/`retrieveAllAsString` - 后两个是把buffer中的数据转为string类型，多与onMessage配合使用；前两个是将`m_readerIndex`和`m_writerIndex`调整位置。

   ```cpp
   public:
       void retrieve(size_t len)
       {
           if(len < readableBytes())//只读取了可读缓冲区数据的一部分
           {
               m_readerIndex += len;
           }
           else
           {
               retrieveAll();
           }
       }
       void retrieveAll()
       {
           m_readerIndex = m_writerIndex = kCheapPrepend;
       }
       std::string retrieveAsString(size_t len)
       {
           std::string result(peek(), len);
           retrieve(len);
           return result;
       }
       std::string retrieveAllAsString()
       {
           return retrieveAsString(readableBytes());
       }
   ```

5. ensureWritableByte - 确保buffer可写空间大小足够len，不足则扩容
   ```cpp
   public:
       void ensureWritableBytes(size_t len)
       {
           if(writableBytes() < len)
           {
               makeSpace(len);
           }
       }
   ```

6. makeSpace - 扩容
   ```cpp
   private:
       void makeSpace(size_t len)
       {
           if(writableBytes()+prependableBytes()-kCheapPrepend < len)
           {
               m_buffer.resize(m_writerIndex + len);
           }
           else//move readable data to the front to make space
           {
               size_t readable = readableBytes();
               //将m_readerIndex到m_writerIndex的内容复制到kCheapPrepend处
               std::copy(begin() + m_readerIndex, begin() + m_writerIndex, begin() + kCheapPrepend);
               m_readerIndex = kCheapPrepend;
               m_writerIndex = m_readerIndex + readable;
           }
       }
   ```

7. append
   ```cpp
   public:
       void append(const char * data, size_t len)
       {
           ensureWritableBytes(len);
           std::copy(data, data+len, beginWrite());
           m_writerIndex += len;
       }
   ```

8. readFd - 从fd上读取数据
   ```cpp
   public:
       ssize_t readFd(int fd, int * saveErrno);	//在.cc文件中实现
   ```
   

### 代码实现

1. readFd - 从fd上读取数据

   1. Poller默认工作在LT模式

   2. Buffer缓冲区是有大小的，但从fd上读数据却不清楚数据有多少。

   3. 此函数使用了系统调用`readv`。

   4. struct iovec结构 - `iov_base`指向缓冲区首址；`iov_len`是缓冲区的长度。网络编程中可以使用此工具，创建一个`struct iovec iov[2]`，第一个的`iov_base`指向tcp连接底层的缓冲区，第二个的`iov_base`指向额外的缓冲区，以备使用。如果使用到额外的缓冲区，在readv完毕后，把额外缓冲区内容拼接到tcp底层缓冲区尾部即可。
      ```cpp
      struct iovec
      {
          void * iov_base;
          size_t iov_len;
      }
      ```

   
   ```cpp
   #include"buffer.h"
   #include<sys/uio.h>
   ssize_t Buffer::readFd(int fd, int * savedErrno)
   {
       char extrabuf[65536] = {0};	//栈上内存空间
       struct iovec iov[2];
       const size_t writable = writableBytes();//buffer底层缓冲区剩余的可写空间大小
       iov[0].iov_base = begin() + m_writerIndex;
       iov[0].iov_len = writable;
       iov[1].iov_base = extrabuf;
       iov[1].iov_len = sizeof extrabuf;
       const int iovcnt = (writable < sizeof extrabuf) ? 2 : 1;
       //如果可写空间大小少于64kb则可以按需写到vec[0]/vec[1];
       //如果可写空间大小大于等于64kb则只能写到vec[0]。
       //说明, 可收到的数据大小限制至少为64kb。
       const ssize_t n = ::readv(fd, iov, iovcnt);
       if(n < 0)
       {
           *savedErrno = errno;
       }
       else if(n <= writable)
       {
           m_writerIndex += n;
       }
       else// n > writable, 需要拼接extrabuf
       {
           m_writerIndex = m_buffer.size();
           append(extrabuf, n-writable);
       }
       return n;
   }
   ```
   

## TcpConnection

```cpp
class TcpConnection : noncopyable, public std::enable_shared_from_this<TcpConnection>
```

### 成员变量

1. `m_loop` - subLoop

   ```cpp
   private:
       EventLoop *m_loop;
   ```

2. `m_socket`/`m_channel` - `unique_ptr`管理

   ```cpp
   private:
       std::unique_ptr<Socket>  m_socket;
       std::unique_ptr<Channel> m_channel;
   ```

3. `m_localAddr`/`m_peerAddr` - 本地/对端地址信息

   ```cpp
   private:
       const InetAddress m_localAddr;
       const InetAddress m_peerAddr;
   ```

4. `m_inputBuffer`/`m_outputBuffer` - 读写缓冲区

   ```cpp
   private:
       Buffer m_inputBuffer;
       Buffer m_outputBuffer;
   ```

5. 各种标志

   1. `m_name`
   2. `m_state` - atomic，用枚举类变量赋值
   3. `m_reading`
   4. `m_highWaterMark` - 高水位阈值

   ```cpp
   private:
       const std::string m_name;
       
       enum StateE {kDisconnected, kConnecting, kConnected, kDisconnecting};
       std::atomic_int m_state;
       
       bool m_reading;
       
       size_t m_highWaterMark;
   ```

6. 各种回调
   ```cpp
   private:
       ConnectionCallback	    m_connectionCallback;
       MessageCallback	        m_messageCallback;
       WriteCompleteCallback   m_writeCompleteCallback;
       HighWaterMarkCallback   m_highWaterMarkCallback;
       CloseCallback           m_closeCallback;
   ```


### 成员函数

1. 构造/析构
   ```cpp
   public:
       TcpConnection(EventLoop *loop, const std::string& name, int sockfd,
                     const InetAddress& localAddr, const InetAddress& peerAddr);
       ~TcpConnection();
   ```

2. 建立/销毁连接
   ```cpp
   public:
       void connectEstablished();
       void connectDestoryed();
   ```

3. send - 发送数据

   ```cpp
   private:
       void send(const std::string &buf);
       void send(const void * data, int len);
   ```

4. shutdown - 关闭连接
   ```cpp
   public:
       void shutdown();
   private:
       void shutdownInLoop();
   ```
   
5. 设置回调
   ```cpp
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
   ```
   
6. 设置、判断标志

   ```cpp
   public:
       bool connected() const {return m_state == kConnected;}
       void setState(StateE state) {m_state = state;}
   ```

7. 获取属性

   ```cpp
   public:
       EventLoop * getLoop() const {return m_loop;}
       const std::string& name() const {return m_name;}
       const InetAddress& localAddress() const {return m_localAddr;}
       const InetAddress& peerAddress() const {return m_peerAddr;}
   ```

8. handleRead/handleWrite/handleClose/handleError
   ```cpp
   private:
       void handleRead(Timestamp receiveTime);
       void handleWrite();
       void handleClose();
       void handleError();
   ```


### 代码实现

1. 构造：重要参数 - loop、sockfd、localAddr、peerAddr

   1. 给loop赋值，name起名字
   2. 赋state为`kConnecting`、reading为`true`
   3. 以sockfd为参数构造socket，new后赋给智能指针`m_socket`
   4. 以loop、sockfd为参数构造channel，new后赋给智能指针`m_channel`
   5. 赋值localAddr、peerAddr
   6. 赋高水位阈值`m_highWaterMark`为`64*1024*1024`(64M)
   7. 给`m_channel`设置相应的回调，当poller给channel通知感兴趣的事件发生，则channel会回调相应的操作函数
   8. 对`m_socket`调用`setKeepAlive`，使TCP启动保活机制

   ```cpp
   /* 写为static，防止函数名字冲突 */
   static EventLoop* CheckLoopNotNull(EventLoop* loop)
   {
       if(loop == nullptr)
       {
           LOG_FATAL("%s:%s:%d mainLoop is null\n", __FILE__, __FUNCTION__, __LINE__);
       }
       return loop;
   }
   TcpConnection::TcpConnection(EventLoop* loop, const std::string &nameArg, int sockfd,
                                const InetAddress &localAddr, const InetAddress &peerAddr)
       : m_loop(CheckLoopNotNull(loop)), m_name(nameArg),
         m_state(kConnecting), m_reading(true),
         m_socket(new Socket(sockfd)),
         m_channel(new Channel(loop, sockfd)),
         m_localAddr(localAddr), m_peerAddr(peerAddr),
         m_highWaterMark(64*1024*1024) //64M
   {
       m_channel->setReadCallback(std::bind(&TcpConnection::handleRead, this, _1));
       m_channel->setWriteCallback(std::bind(&TcpConnection::handleWrite, this));
       m_channel->setCloseCallback(std::bind(&TcpConnection::handleClose, this));
       m_channel->setErrorCallback(std::bind(&TcpConnection::handleError, this));
       LOG_INFO("%s [%s] at fd = %d\n", __FUNCTION__, m_name.c_str(), sockfd);
       m_socket->setKeepAlive(true);   //启动TCP保活机制
   }
   ```

2. 析构
   ```cpp
   TcpConnection::~TcpConnection()
   {
       LOG_INFO("%s [%s] at fd = %d, state = %d\n",
                 __FUNCTION__, m_name.c_str(), m_channel->fd(), m_state.load());
   }
   ```

3. handleRead - 调用`m_inputBuffer`的`readFd`, 读取channel上的消息; 如果有数据则调用`m_messageCallback`; 如果返回值为0说明客户端断开, 调用`handleClose`; 如果出错则`handleError`;
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
   ```

4. handleWrite - 调用`m_outputBuffer`的`writeFd`, 写到channel上对应fd的缓冲区(调用`peek`, 找到缓冲区数据包中可读数据起始位置, 把从此位置起共`readableBytes()`数据写到fd); 如果成功则调用`m_loop->queueInLoop(std::bind(m_writeCompleteCallback, shared_from_this()))`; 最后, 判断连接的状态如果是`Disconnecting`则`shutdownInLoop`
   ```cpp
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
   ```

5. handleClose - 调用`setState(kDisconnected)`, `m_channel->disableAll()`, `m_connectionCallback`, `m_closeCallback`
   ```cpp
   void TcpConnection::handleClose()
   {
       LOG_INFO("%s: fd = %d, state: %d\n", __FUNCTION__, m_channel->fd(), m_state.load());
       setState(kDisconnected);
       m_channel->disableAll();
   
       TcpConnectionPtr connPtr(shared_from_this());
       m_connectionCallback(connPtr);
       m_closeCallback(connPtr);
   }
   ```

6. handleError - 调用`getsockopt`, 调查fd的错误, 如果连getsockopt也失败则存储全局errno

   * `int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen)`: 操作套接字选项时，必须指定选项所在的级别和选项的名称, `SOL_SOCKET`表示在套接字API级别, 参见`getprotoent(3)`; 参数`optval`和`optlen`对于getsockopt(), 标识一个缓冲区，请求选项的值将在其中返回。`optlen`是一个值结果参数，最初包含`optval`指向的缓冲区的大小，并在返回时修改以指示返回值的实际大小。 如果不提供或返回选项值，则`optval`可能为NULL。

   ```cpp
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
   ```

7. send - 用户会给TcpServer注册onMessageCallback, 已建立连接的用户有读写事件时, 尤其是读事件, onMessage会响应; 处理完客户端发来的事件后(onMessageCallback), 服务端会send给客户端回发消息; 

   > 在TcpConnection的成员中, 有两个Buffer成员; 
   >
   > 1. inputBuffer - 接收数据的缓冲区 - 即recv操作需要暂存的区域
   > 2. outputBuffer - 发送数据的缓冲区 - 即send操作需要暂存的区域
   >
   > 其中, outputBuffer存在的意义?
   >
   > 1. 应用层可能需要处理很多数据, 数据从传输层到网络层到数据链路层的传输往往比应用层发送得快; 需要用缓冲区暂存; 
   > 2. 为了防止应用层与底层传输的数据量差距悬殊导致数据丢失, 设置了一个高水位阈值; 

   > 收发数据的方式: 本项目的数据收发统一使用json或protobuf格式化的字符串进行, 所以此send函数的参数为了方便起见, 直接规定为string类型; 

   1. 判断当前连接的状态是否为connected; 
   2. 判断此loop是否在本thread中, 如果是则调用sendInLoop; 否则runInLoop, 绑定的函数也是sendInLoop; 

   ```cpp
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
   ```

8. sendInLoop - 写数据操作, 结合了sendInLoop和handleWrite
   ```cpp
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

9. connectEstablished

   1. setState为kConnected;
   2. 调用`m_channel->tie`, 让`m_channel`绑定一个TcpConnection, 方便后期观察TcpConnection是否还有效, 若已失效将不进行相应的操作, 已然无意义;

   ```cpp
   void TcpConnection::connectEstablished()
   {
       setState(kConnected);
       m_channel->tie(shared_from_this());
       m_channel->enableReading();	//向poller注册channel的epollin事件
       //有新连接建立, 调用connectionCallback
       m_connectionCallback(shared_from_this());
   }
   ```

10. connectDestroyed

    1. 判断state是否为connected, 若是则setState为kDisconnected, 并调用`m_channel->disableAll()`, 调用`connectionCallback`
    2. 最后`m_channel->remove()`, 把channel从poller中删除掉

    ```cpp
    void TcpConnection::connectDestoryed()
    {
        if(m_state == kConnected)
        {
            setState(kDisconnected);
            m_channel->disableAll();
            m_connectionCallback(shared_from_this());
        }
        m_channel->remove();
    }
    ```

11. shutdown/shutdownInLoop

    * 关闭写端, 将会触发epollhup, 调用closeCallback, 即TcpConnection中的handleClose方法,
      1. setState(kDisconnected)
      2. m_channel->disableAll()
      3. connectionCallback, closeCallback

    ```cpp
    void TcpConnection::shutdown()
    {
        if(m_state == kConnected)
        {
            setState(kDisconnecting);
            m_loop->runInLoop(std::bind(&TcpConnection::shutdownInLoop, this));
        }
    }
    /**
     * 关闭写端, 将会触发epollhup, 
     * 会调用closeCallback, 
     * 即TcpConnection中的handleClose方法,
     * handleClose将会: 
     *  1. setState(kDisconnected);
     *  2. m_channel->disableAll();
     *  3. connectionCallback, closeCallback
     */
    void TcpConnection::shutdownInLoop()
    {
        if(!m_channel->isWriting())//说明outputBuffer中的数据已经全部发送完成
        {
            m_socket->shutdownWrite();//关闭写端, 将会触发epollhup, 会调用closeCallback, 即TcpConnection中的handleClose方法
        }
    }
    ```

# TcpServer收尾

1. newConnection - 运行在主线程当中，主线程的mainLoop调用此函数，选择了一个ioLoop，**在非子loop的线程中（即当前是在mainThread）执行cb，就需要唤醒子loop所在线程（subThread），执行cb**，即调用subLoop的`queueInLoop(cb)`。

   1. 根据轮询算法选择一个subLoop，即调用`m_threadPool->getNextLoop()`
   2. 唤醒subLoop
   3. 把当前connfd封装成channel分发给subloop

   ```cpp
   void TcpServer::newConnection(int sockfd, const InetAddress &peerAddr)
   {
       EventLoop *ioLoop = m_threadPool->getNextLoop();
       char buf[64] = {0};
       snprintf(buf, sizeof buf, "-%s#%d", m_IPport.c_str(), m_nextConnId)
   }
   ```

2. removeConnection/removeConnectionInLoop
   ```cpp
   void TcpServer::removeConnection(const TcpConnectionPtr &conn)
   {
       m_loop->runInLoop(std::bind(&TcpServer::removeConnectionInLoop, this, conn));
   }
   void TcpServer::removeConnectionInLoop(const TcpConnectionPtr &conn)
   {
       LOG_INFO("%s [%s] - connection %s\n",
                __FUNCTION__, m_name.c_str(), conn->name().c_str());
       m_connections.erase(conn->name());
       EventLoop *ioLoop = conn->getLoop();
       ioLoop->queueInLoop(std::bind(&TcpConnection::connectDestoryed, conn));
   }
   ```


# 流程

1. TcpServer -> Acceptor -> 有一个新用户连接，通过accept函数得到connfd
2. 用户给TcpServer设置回调 -> TcpServer给TcpConnection设置回调 -> TcpConnection把回调传给Channel -> Channel注册到Poller中 -> Poller通知Channel调用回调
3. mainLoop中的acceptor是一个特殊的socketfd, 它只有一个回调`ReadCallback`, 负责监听新用户的连接, 返回socket, 将这个socket打包成TcpConnection, 再注册相应的回调; 
