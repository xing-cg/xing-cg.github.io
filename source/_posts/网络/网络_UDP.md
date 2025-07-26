---
title: 网络_UDP
categories:
  - - 网络
tags: 
date: 2025/7/26
updated: 2025/7/26
comments: 
published:
---
# UDP组播的概念
单播是指将数据包从1个发送方到1个特定的接收方。
组播是指将数据包从1个发送方到1组特定的接收方。这个组是预先定义的，这些接收方共享同一个组播地址。

在IPv4中，组播地址是有范围和规定的，不能随便写。IPv4的组播地址范围是`224.0.0.0`到`239.255.255.255`。
其中，`224.0.0.0`到`224.0.0.255`是预留的，用于本地链接的多播地址（Link-Local Multicast Addresses），而其他范围的地址才可以用于全局组播。
在选择组播地址时，应该遵循规范，避免使用预留的地址或者其他可能会引起冲突的地址。
通常建议从`239.0.0.0`开始向上选择，确保不会与其他组播组冲突。

# setsockopt()
`setsockopt()`函数是系统调用，用于设置套接字选项，它允许程序员为打开的套接字设置不同的选项和参数
```c
int setsockpt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

参数：
1. `sockfd`：指定要设置选项的套接字文件描述符。
2. `level`：指定选项所在的协议层。常用的有`SOL_SOCKET`表示套接字级别的选项，`IPPROTO_IP`表示IP层的选项，`IPPROTO_TCP`表示TCP层的选项，`IPPROTO_IPV6`表示IPV6层的选项等。
3. `optname`：指定要设置的选项名。
    1. `IP_ADD_MEMBERSHIP`：加入组播组
    2. `IP_DROP_MEMBERSHIP`：退出组播组
4. `optval`：指向包含选项值的缓冲区的指针。
5. `optlen`：指定选项值的长度。

# 发送端
```cpp
// multicast_send.cpp
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <iostream>
#include <unistd.h>
const int PORT = 8888;
const char *MULTICAST_ADDR = "239.0.0.1";
const int MAX_MSG_LEN = 1024;

void sender()
{
    int sock;
    sock = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    inet_pton(AF_INET, MULTICAST_ADDR, &(addr.sin_addr.s_addr));

    std::string msg = "Hello, multicast";

    sendto(sock, msg.c_str(), msg.length(), 0, (struct sockaddr*)&addr, sizeof(addr));

    std::cout << "Message sent: " << msg << std::endl;

}
int main()
{
    sender();
}
```
# 接收端
1. 与单播不同，sockaddr需要写入IP地址为`0.0.0.0`、端口号为与服务端约定好的组播端口（本例为8888）
2. 需要绑定创建的sock 和 上面填写好的sockaddr
3. 还需要写入一个`ip_mreq`结构。
    1. 指明`imr_multiaddr`，写入与服务端约定好的组播地址（本例为`239.0.0.1`）
    2. 指明`imr_interface`，写入`0.0.0.0`（本地地址）
4. setsockopt 设置 sock 加入组播组。
    1. 在调用`setsockopt`函数时，参数`&mreq, sizeof(mreq)`的作用是传递一个`ip_mreq`结构体及其大小给内核，以便内核根据这个结构体中的信息将套接字加入到指定的组播组。
5. 用的是recv接收，而不是单播时的recvfrom。（其实recvfrom也行，但是接收方必须绑定sock和组播端口）

## `imr_multiaddr`和`imr_interface`有什么区别
```c
struct ip_mreq
{
    struct in_addr imr_multiaddr;  /* 组播组IP地址 */
    struct in_addr imr_interface;  /* 本地接口IP地址 */
};
```
在组播编程中，`struct ip_mreq` 结构体用于管理组播组成员关系，其中两个关键字段的区别如下：

|特性|`imr_multiaddr`|`imr_interface`|
|---|---|---|
|​**​作用​**​|指定要加入/离开的​**​组播组​**​|指定用于组播通信的​**​网络接口​**​|
|​**​地址类型​**​|D类IP地址（224.0.0.0-239.255.255.255）|本机真实IP地址或`INADDR_ANY`|
|​**​示例值​**​|239.0.0.1|192.168.1.100 或 `INADDR_ANY`|
|​**​必要性​**​|必须正确设置|可选（不设置时系统自动选择）|
|​**​用途​**​|标识通信的目标组|选择收发组播数据的物理/虚拟网卡|
### `imr_multiaddr`（组播组地址）
- ​**​定义​**​：表示要加入或离开的组播组
- ​**​组播地址范围​**​：224.0.0.0 到 239.255.255.255（D类IP）
- ​**​永久组播地址​**​：224.0.0.1（所有主机）、224.0.0.2（所有路由器）
- ​**​临时组播地址​**​：239.x.x.x（管理员定义范围）
### `imr_interface`（本地接口地址）
- **定义​**​：指定用于组播通信的网络接口
- ​**​特点​**​：
    - 可以是本机具体IP（192.168.1.100）
    - 也可以是特殊值：
        - `INADDR_ANY` (0.0.0.0)：让​**​系统自动选择​**​默认路由接口
        - `INADDR_LOOPBACK` (127.0.0.1)：限定为本地环回

```c
// 自动选择最优网卡（推荐）
mreq.imr_interface.s_addr = htonl(INADDR_ANY);

// 指定具体接口（有多个网卡时）
inet_pton(AF_INET, "192.168.1.100", &mreq.imr_interface);
```

不能指定`127.0.0.1`换回接口。当接收外部组播时，即使发送方是在本机上，但指定环回地址会导致接收方不能接收本地的组播数据。
### 正确示例
```c
struct ip_mreq mreq;

// 1. 设置组播组（固定D类地址）
inet_pton(AF_INET, "239.0.0.100", &mreq.imr_multiaddr);

// 2 以下考虑到了多网卡的场景，三选一：
// 2.1 监听所有可用网卡上的组播流量（让系统选择最优接口）
mreq.imr_interface.s_addr = INADDR_ANY;
// 或者指定特定网卡，需要写本地具体ip地址
// 2.2 有线网卡
const char* iface_ip = "192.168.1.100"; 
inet_pton(AF_INET, iface_ip, &mreq.imr_interface);
// 2.3 无线网卡
const char* wifi_ip = "10.0.0.5"; 
inet_pton(AF_INET, wifi_ip, &mreq.imr_interface);

// 3. 加入组播组
setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));
```

## 为什么接收端要绑定？
在组播接收端，`bind`操作是​**​必须​**​的，因为：
1. 组播数据包是通过UDP传输的，而UDP是无连接的。
2. 操作系统需要知道将哪些端口上的数据传递给应用程序。

如果你不调用`bind`，那么你的套接字就没有与本地端口关联，操作系统就不会将收到的数据包递送到该套接字。因此，你无法收到任何组播数据。

`recvfrom`函数是用来接收数据并获取发送者的地址，但是它并不能替代`bind`。实际上，`recvfrom`通常是在已经绑定的套接字上使用的。

因此，正确的步骤是：
1. 创建套接字。
2. 绑定到指定的端口（通常还会指定地址为`INADDR_ANY`，即`0.0.0.0`）。
3. 加入组播组（通过`setsockopt`设置IP_ADD_MEMBERSHIP）。
4. 使用`recvfrom`（或者`recv`）接收数据。

所以，即使你想使用`recvfrom`来获取发送者的信息，你仍然需要先绑定端口。

如果你尝试不绑定，那么你将会看到`recvfrom`调用会失败，并返回一个错误（例如“Transport endpoint is not connected”或者“Invalid argument”）。

因此，结论是：不能省略`bind`步骤，即使你打算使用`recvfrom`。
### `bind()`的核心作用 - ​声明端口所有权​
- 作用：告知内核："我负责处理本机所有网卡上指定端口的UDP流量"

底层原理：
```c
// 内核网络栈伪代码
void udp_input(struct sk_buff *skb) {
    // 查找绑定对应端口的套接字
    sock = udp_v4_lookup(skb->dport); 
    if (sock) deliver_to_socket(sock, skb);
    else discard_packet(); // ⚠️ 未绑定则丢弃包
}
```
### `setsockopt()`的作用 - ​订阅组播频道​
- 作用：告诉网络驱动："我对 特定组播地址 的流量感兴趣"
- 触发内核进行：
    - IGMP加入报文发送（通知路由器）
    - 设置网卡组播过滤器
## 代码
```cpp
// multicast_recv.cpp
#include <netinet/in.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <iostream>
#include <unistd.h>
const int PORT = 8888;
const char *MULTICAST_ADDR = "239.0.0.1";
const int MAX_MSG_LEN = 1024;
void receiver()
{
    int sock = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    inet_pton(AF_INET, "0.0.0.0", &(addr.sin_addr.s_addr));
    // 绑定sock 和 addr
    bind(sock, (struct sockaddr*)&addr, sizeof(addr));

    // 本地的IP接口加入到组播地址中
    // IPv4 multicast request.
    struct ip_mreq mreq;
    // 指定组播地址
    inet_pton(AF_INET, MULTICAST_ADDR, &(mreq.imr_multiaddr.s_addr));
    inet_pton(AF_INET, "0.0.0.0", &(mreq.imr_interface.s_addr));

    setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, sizeof(mreq));

    char buffer[MAX_MSG_LEN + 1] = {0};
    int len = recv(sock, buffer, MAX_MSG_LEN, 0);

    buffer[len] = '\0';

    std::cout << "Received message: " << buffer << std::endl;
    close(sock);
}
int main()
{
    receiver();
}
```
# 问题
## 为什么发送端发给的是8888，接收端也要从本地的8888接收？
1. ​**​组播地址是逻辑分组​**​（如239.0.0.1）
    - 只负责标识组播组，不包含端口信息
    - 相当于"广播频道"，但无法单独区分不同应用
2. ​**​端口号标识具体服务​**​
    - 接收端通过绑定特定端口（8888）声明："我是这个组播组中负责8888端口的应用"
    - 发送端必须将数据发送到该组播地址+指定端口的组合

想象一个无线电系统：
- 组播地址 = 频道（如FM 239.0）
- 端口号 = 子频道（如"新闻子频道8888"）
- 发送端必须在指定频道(FM 239.0)用指定子频道(8888)发射信号
- 接收端必须调到相同频道(FM 239.0)+相同子频道(8888)才能接收

## 端口重用问题
- 同一台机器上的不同程序不能同时绑定相同端口
- 如需多进程接收，需要设置`SO_REUSEPORT`选项
## 防火墙配置​​
- 接收端必须开放对应的UDP端口（示例中是8888）
- Linux检查：`sudo ufw allow 8888/udp`