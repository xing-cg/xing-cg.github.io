---
title: 05_mprpc_rpc框架
categories:
  - - 项目
    - rpc
tags: 
date: 2022/9/13
updated: 2025/7/13
comments: 
published:
---
# 无RPC的代码，怎么升级使用RPC框架
在`/example/callee`中的代码
```cpp
#include <iostream>
class UserService
{
public:
    bool Login(std::string name, std::string pwd)
    {
        std::cout << "doing local service: Login" << std::endl;
        std::cout << "name: " << name << ", pwd: " << pwd << std::endl;
    }
};
int main()
{
    UserService us;
    us.Login("xcg", "123456");
}
```
如上，这是独立的代码，UserService可以进行Login处理。但是如果不使用RPC框架，则只能在本地被调用。
需要想办法，能让远程调用。
## 编写user.proto
在`/example/`中的代码
```protobuf
syntax = "proto3";
package xcg;
option cc_generic_services = true;
message ResultCode
{
    int32 errcode = 1;
    bytes errmsg = 2;
}
message LoginRequest
{
    bytes name = 1;
    bytes pwd = 2;
}
message LoginResponse
{
    ResultCode result = 1;
    bool success = 2;
}
service UserServiceRpc
{
    rpc Login(LoginRequest) returns(LoginResponse);
}
```
使用protoc编译
```bash
protoc user.proto --cpp_out=./
```
接下来，就是使用`user.pb.h`。
## 服务提供者继承UserServiceRpc，实现rpc虚方法，业务处理，响应
```cpp
// userservice.cc
#include <iostream>
#include "user.pb.h"
// 服务提供者
class UserService : public xcg::UserServiceRpc
{
public:
    bool Login(std::string name, std::string pwd)
    {
        std::cout << "doing local service: Login" << std::endl;
        std::cout << "name: " << name << ", pwd: " << pwd << std::endl;
    }
    // 重写基类UserServiceRpc的虚函数。这些方法都是框架直接调用的
    void Login(::google::protobuf::RpcController* controller,
                       const ::xcg::LoginRequest* request,
                       ::xcg::LoginResponse* response,
                       ::google::protobuf::Closure* done)
    {
        // 拿到了框架给上报的请求参数LoginRequest
        // 需要取出相应数据做本地业务
        std::string name = request->name();
        std::string pwd = request->pwd();
        // 取出数据后，调用本身已有的方法
        bool login_result = Login(name, pwd);
        // 填写响应消息
        xcg::ResultCode * result_code = response->mutable_result();
        result_code->set_errcode(0);
        result_code->set_errmsg("ok!");
        response->set_success(login_result);
        // 执行回调操作：由框架done->Run()处理消息的序列化、返回传输
        done->Run();
    }
};
```
这些重写了的虚函数，在哪里被调用呢？见[`google::protobuf::NewCallback<>`生成回调函数](#`google%20protobuf%20NewCallback<>`生成回调函数)
## 服务调用方调用UserServiceRpc_Stub里protobuf给我们自动实现好的rpc方法
比如，`UserServiceRpc_Stub`的`Login`方法，是位于`user.pb.h`中的。

而Login的调用最终会落到`UserServiceRpc_Stub`的`channel_`成员所含的方法`CallMethod`。

要实例化`UserServiceRpc_Stub`，需要一个`MprpcChannel`对象来给它构造，并且这个`MprpcChannel`要实现`CallMethod`。

见[Mprpcchannel类](#Mprpcchannel类)。
## 服务调用者主程序（如calluserservice.cc）
定义要调用的某个方法的request（如LoginRequest），填写参数。

定义要调用的某个方法的response（如LoginResponse），不填写参数。

定义`UserServiceRpc_Stub`，通过`MprpcChannel()`实例化。

之后通过stub发起远程调用`Login`。
```cpp
#include <iostream>
#include "mprpcapplication.h"
#include "user.pb.h"
#include "mprpcchannel.h"
int main(int argc, char ** argv)
{
    // 整个程序启动以后，想使用mprpc框架来享受rpc服务调用，一定需要先调用框架的初始化函数（只初始化一次）
    MprpcApplication::Init(argc, argv);

    // 填写rpc方法的请求参数
    xcg::LoginRequest request;
    request.set_name("zhang san");
    request.set_pwd("123456");
    // 定义rpc方法的响应
    xcg::LoginResponse response;

    // 底层转到RpcChannel->CallMethod来做所有rpc方法调用的参数序列化和网络发送
    xcg::UserServiceRpc_Stub stub(new MprpcChannel());

    // 发起rpc方法的调用。
    stub.Login(nullptr, &request, &response, nullptr);
    
    // 一次rpc调用完成，读调用的结果
    if (response.result().errcode() == 0)
    {
        // 成功
        std::cout << "rpc login response success: " << response.success() << std::endl;
    }
    else
    {
        std::cout << "rpc login response error: " << response.result().errmsg() << std::endl;
    }
    return 0;
}
```
## rpc服务框架的启动入口？
rpc服务框架由各个服务节点（服务提供者）启动。比如`UserService`，类外实现一个main函数，main函数中调用`MprpcApplication::Init(argc, argv)`进行框架相关的启动，比如读配置文件。
之后，各个服务节点（服务提供者）可以NotifyService发布自己节点上的服务名、方法名。
之后，各个服务节点调用`Run`，启动服务节点。

# 模拟rpc框架的使用，以分析框架需要什么东西
1. 框架需要一个基础类，进行Init初始化，使用argc、argv参数。因为rpc服务器本身有ip地址、端口号，需要zookeeper的ip地址、端口号。从配置文件读。
2. 框架需要提供一个可以发布服务的RpcProvider。

```cpp
// userservice.cc的代码

// class UserService的定义...
// ...

#include "mprpcapplication.h"
int main(int argc, char ** argv)
{
    // 框架的初始化操作
    MprpcApplication::Init(argc, argv);
        
    // provider是一个rpc网络服务对象，负责发布服务到rpc节点上
    RpcProvider provider; 
    // 把UserService发布到rpc节点上
    provider.NotifyService(new UserService());
    
    // 启动一个rpc服务发布节点
    provider.Run();
    // 之后，进程进入阻塞状态，等待远程的rpc调用请求
}
```
## MprpcApplication类
在`/src/include`中编写头文件
```cpp
#pragma once
// mprpc 框架的基础类，用于初始化框架
// 单例设计
class MprpcApplication
{
public:
    static void Init(int argc, char ** argv);
    static MprpcApplication& GetInstance()
    {
        static MprpcApplication app;
        return app;
    }
private:
    MprpcApplication()
    {

    }
    MprpcApplication(const MprpcApplication&) = delete;
    MprpcApplication(MprpcApplication&&) = delete;
};
```
## RpcProvider类
在`/src/include`中编写头文件
```cpp
#pragma once
#include "google/protobuf/service.h"
// 框架提供的专门发布RPC服务的网络对象类
class RpcProvider
{
public:
    // 这里是框架提供给外部使用的，可以发布Service
    void NotifyService(::google::protobuf::Service * service);
    // 启动RPC服务节点
    void Run();
};
```
## CMakeLists.txt
### 根目录
```cmake
cmake_minimum_required(VERSION 3.0)
project(mprpc)

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)

include_directories(${PROJECT_SOURCE_DIR}/src/include)
include_directories(${PROJECT_SOURCE_DIR}/example)
link_directories(${PROJECT_SOURCE_DIR}/lib)

# 框架的代码
add_subdirectory(src)
# 使用框架的实例代码
add_subdirectory(example)
```

src生成的代码就是框架的部分。需要生成一个对外的so库。
### src目录
```cmake
# 当前文件夹中所有文件 别名 SRC_LIST
aux_source_directory(. SRC_LIST)
# 生成动态库
add_library(mprpc SHARED ${SRC_LIST})
```
### example目录
```cmake
add_subdirectory(callee)
```
### example/callee目录
```cmake
set(SRC_LIST userservice.cc ../user.pb.cc)

add_executable(provider ${SRC_LIST})
# 声明编译时需要链接mprpc、protobuf动态库
target_link_libraries(provider mprpc protobuf)
```
# 流程

1. 编写protobuf文件，作为协议；
2. 继承rpc服务类。实现rpc方法。
    1. 获取请求体的参数
    2. 本地处理业务
    3. 写入响应体
    4. 执行回调操作，完成响应数据的序列化，进行发送响应。
3. 发布服务，启动服务
    1. 初始化框架
    2. 定义发布rpc服务的对象（provider）
    3. 启动rpc服务发布节点。
4. 服务程序进入等待状态，接受远程rpc调用请求。
# rpc框架的编写

## MprpcApplication类
主要需要实现Init函数。
主要用于使用外部命令行传入的参数，以及读取参数中的配置文件，进行初始化。

1. 可以单独封装一个MprpcConfig类。MprpcApplication类使用它进行config的读取。
2. Init只需要进行一次，所以将MprpcApplication设计为单例模式。
3. 由于Init是静态方法，Init方法使用到了类内对象MprpcConfig，因此MprpcConfig成员需要声明为static。

```cpp
// mprpcapplication.h
#pragma once
#include "mprpcconfig.h"
// mprpc 框架的基础类，用于初始化框架
// 单例设计
class MprpcApplication
{
public:
    static void Init(int argc, char ** argv);
    static MprpcApplication& GetInstance();
private:
    static MprpcConfig m_config;
    MprpcApplication()
    {

    }
    MprpcApplication(const MprpcApplication&) = delete;
    MprpcApplication(MprpcApplication&&) = delete;
};
```
### 实现
1. ShowArgsHelp函数用于提示用户正确的命令行参数格式
2. Init函数主要实现
    1. 检查参数格式
    2. 通过getopt拿到参数
    3. 调用MprpcConfig成员的LoadConfigFile方法解析配置文件的信息，即获取rpc服务器和zookeeper的ip、端口

```cpp
// mprpcapplication.cc
#include "mprpcapplication.h"
#include <iostream>
#include <unistd.h>
MprpcConfig MprpcApplication::m_config;
MprpcApplication& MprpcApplication::GetInstance()
{
    static MprpcApplication app;
    return app;
}

void ShowArgsHelp()
{
    std::cout << "format: commad -i <configfile>" << std::endl;
}
// 类外实现静态方法，不用前面加static
void MprpcApplication::Init(int argc, char ** argv)
{
    if (argc < 2)
    {
        ShowArgsHelp();
        exit(EXIT_FAILURE);
    }
    int c = 0;
    std::string config_file;
    while((c = getopt(argc, argv, "i:")) != -1)
    {
        switch (c)
        {
        case 'i':
            // 获取到配置文件的名字
            config_file = optarg;
            break;
        case '?':
            // std::cout << "invalid args!" << std::endl;
            ShowArgsHelp();
            exit(EXIT_FAILURE);
        case ':':
            // std::cout << "need <configfile>!" << std::endl;
            ShowArgsHelp();
            exit(EXIT_FAILURE);
        default:
            break;
        }
    }
    // 开始加载配置文件 rpcserver_ip rpcserver_port zookeeper_ip zookeeper_port
    m_config.LoadConfigFile(config_file.c_str());

    std::cout << "rpcserver_ip: " << m_config.Load("rpcserver_ip") << std::endl;
    std::cout << "rpcserver_port: " << m_config.Load("rpcserver_port") << std::endl;
    std::cout << "zookeeper_ip: " << m_config.Load("zookeeper_ip") << std::endl;
    std::cout << "zookeeper_port: " << m_config.Load("zookeeper_port") << std::endl;
}
```
若测试的是[注意读取配置文件可能出现的空格、注释、无效信息问题](#注意读取配置文件可能出现的空格、注释、无效信息问题) 中的配置文件，正确结果：
```bash
mrcan@ubuntu:~/mprpc/bin$ ./provider -i test.conf 
rpcserver_ip: 127.0.0.1
rpcserver_port: 8000
zookeeper_ip: 127.0.0.1
zookeeper_port: 5000
```
## MprpcConfig类
1. 提供LoadConfigFile方法，负责解析加载配置文件，将key、value插入到自身的map中
2. 提供Load方法，可以对外提供查询key读取value。
3. 需要集成一个`unordered_map`记录key、value。
4. Trim负责在解析配置文件过程中除去字符串前后的空格。

```cpp
#pragma once
#include <unordered_map>
#include <string>
class MprpcConfig
{
public:
    // 负责解析加载配置文件
    void LoadConfigFile(const char * config_file);
    // 查询配置项信息
    std::string Load(const std::string& key);
private:
    std::unordered_map<std::string, std::string> m_configMap;
    // 去掉字符串前后的空格
    void Trim(std::string &str_buf);
};
```
### 注意读取配置文件可能出现的空格、注释、无效信息问题
```conf
      # rpc node ip
  
   rpcserver_ip        =   127.0.0.1        


        # rpc node port
      rpcserver_port     =     8000  



    # zookeeper ip
            zookeeper_ip   =127.0.0.1       

# zookeeper port
  zookeeper_port= 5000    


```
如上，配置文件可能有：
1. 注释
2. 纯空行的换行符可以用gets过滤
3. 配置项，以=分隔，前面是key，后面是value，可能前后各有空格
    1. key前后的空格可通过Trim处理，后面肯定没有换行符。
    2. value只能用Trim处理前面的空格，因为后面可能存在`空格或换行符`：
        1. `5000\n`：即没有空格，有一个换行符
        2. `5000      \n`：即有空格，有换行符
        3. 所以需要先去掉末尾的换行符，再进行Trim处理。
### 实现
```cpp
#include "mprpcconfig.h"
#include <iostream>
void MprpcConfig::Trim(std::string &str_buf)
{
    // 去掉字符串前面多余的空格
    int idx = str_buf.find_first_not_of(' ');
    if (idx > 0)
    {
        // 说明字符串前面有空格
        str_buf = str_buf.substr(idx, str_buf.size() - idx);
    }
    // 去掉字符串后面多余的空格
    idx = str_buf.find_last_not_of(' ');
    str_buf = str_buf.substr(0, idx + 1);
}
void MprpcConfig::LoadConfigFile(const char * config_file)
{
    FILE *pf = fopen(config_file, "r");
    if (nullptr == pf)
    {
        std::cout << config_file << " not exist!" << std::endl;
        exit(EXIT_FAILURE);
    }
    // 1. 注释
    // 2. 多余的空格
    // 3. 正确的配置项（以=分隔的）
    while (!feof(pf))
    {
        char buf[512] = {0};
        fgets(buf, 512, pf);
        std::string str_buf(buf);
        Trim(str_buf);

        // 判断 # 开头的注释 或者纯空行
        if (str_buf[0] == '#' || str_buf.empty())
        {
            continue;
        }
        int idx = str_buf.find('=');
        if (idx == -1 || idx == 0)
        {
            // 配置项不合法
            continue;
        }
        std::string key;
        std::string value;
        key = str_buf.substr(0, idx);
        // 去掉前后的空格
        Trim(key);


        // 去掉末尾的换行符
        int lineBreakIdx = str_buf.find('\n', idx + 1);
        value = str_buf.substr(idx + 1, lineBreakIdx - (idx + 1));
        // 去掉前后的空格
        Trim(value);

        m_configMap.insert({key, value});
    }
}
std::string MprpcConfig::Load(const std::string& key)
{
    auto it = m_configMap.find(key);
    if (it == m_configMap.end())
    {
        return "";
    }
    return it->second;
}
```
## 修改MprpcApplication
向其中添加一个公有static方法`GetConfig()`以便RpcProvider获得config对象
```cpp
#pragma once
#include "mprpcconfig.h"
// mprpc 框架的基础类，用于初始化框架
// 单例设计
class MprpcApplication
{
public:
    static void Init(int argc, char ** argv);
    static MprpcApplication& GetInstance();
    // 以便RpcProvider获得config对象
    static MprpcConfig& GetConfig();
private:
    static MprpcConfig m_config;
    MprpcApplication()
    {

    }
    MprpcApplication(const MprpcApplication&) = delete;
    MprpcApplication(MprpcApplication&&) = delete;
};
```
## RpcProvider类
根据[[#模拟rpc框架的使用，以分析框架需要什么东西]]，框架需要提供一个可以发布服务的RpcProvider。
主要实现：
1. NotifyService，是框架提供给外部使用的，可以发布Service
2. Run，启动RPC服务节点
3. Stop，停止服务

需要集成一个TcpServer，和搭配一个Eventloop（相当于epoll）。
但是经过考虑，TcpServer没有必要写为RpcProvider的成员变量，因为只在Run方法中使用。而且我们提供的是一个对外的框架，最好不要把TcpServer的配置、复杂的参数暴露给用户，尽量让用户简单易用。
而EventLoop可能不仅Run使用，Stop还需要调用它的quit。所以定义为成员变量。
```cpp
#pragma once
#include "google/protobuf/service.h"
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
// 框架提供的专门发布RPC服务的网络对象类
class RpcProvider
{
public:
    // 这里是框架提供给外部使用的，可以发布Service
    void NotifyService(::google::protobuf::Service * service);
    // 启动RPC服务节点
    void Run();
private:
    muduo::net::EventLoop m_eventLoop;
    // 新的socket连接回调
    void onConnection(const muduo::net::TcpConnectionPtr&);
    // 已建立连接的用户的读写事件回调
    // std::function<void (const TcpConnectionPtr&, Buffer*, Timestamp)>
    void onMessage(const muduo::net::TcpConnectionPtr&, muduo::net::Buffer*, muduo::Timestamp);
};
```
### Run实现
Run主要实现TcpServer的配置、启动。
这属于RpcProvider的网络模块。
```cpp
#include "rpcprovider.h"
#include "mprpcapplication.h"
#include <functional>
void RpcProvider::Run()
{
    std::string ip = MprpcApplication::GetInstance().GetConfig().Load("rpcserver_ip");
    uint16_t port = atoi(MprpcApplication::GetInstance().GetConfig().Load("rpcserver_port").c_str());
    muduo::net::InetAddress address(ip, port);
    // 创建TcpServer对象
    muduo::net::TcpServer server(&m_eventLoop, address, "RpcProvider");
    // 绑定连接回调和消息读写回调方法 - 分离网络代码和业务代码
    // 需要给TcpServer提供一个返回值为void，参数有TcpConnectionPtr的函数
    // 这个函数我们在RpcProvider的成员方法提供
    // 实际上，onConnection到时候由muduo库进行调用
    // 调用的就是this对象，然后需要一个_1以预留TcpConnectionPtr& conn这个参数的位置
    // std::function<void (const TcpConnectionPtr&)>
    server.setConnectionCallback(std::bind(&RpcProvider::onConnection, this, std::placeholders::_1));
    // std::function<void (const TcpConnectionPtr&, Buffer*, Timestamp)>
    server.setMessageCallback(std::bind(&RpcProvider::onMessage, this, 
        std::placeholders::_1, std::placeholders::_2, std::placeholders::_3));

    // 设置muduo库的线程数量
    server.setThreadNum(4);

    std::cout << "RpcProvider Start Service at IP: " << ip << ", port: " << port << std::endl;

    // 启动网络服务
    server.start();
    // 启动epoll_wait，以阻塞的方式等待远程的连接。
    // 如果有连接请求，则muduo库会回调onConnection
    // 如果有收发数据请求，则muduo库会回调onMessage
    m_eventLoop.loop();
}
```
### NotifyService的实现
发布Rpc服务。
是`/example/callee/`下的如UserService程序调用的。
（集成的muduo网络库实现了：不光能调用本地的RpcProvider，而是可以远程调用网络上的任意一个RpcProvider）

怎么做到“发布”？“发布”的本质是什么？其实是记录在案，方便远端查询，并且定位某个方法。
那么我们就要在RpcProvider类中增加：`m_serviceMap`，记录注册的每一个service名字和这个service对应的所有（n个）method。
```cpp
#pragma once
#include "google/protobuf/service.h"
#include <unordered_map>
// 框架提供的专门发布RPC服务的网络对象类
class RpcProvider
{
public:
    // ...
private:
    struct ServiceInfo
    {
        google::protobuf::Service * m_service;
        std::unordered_map<std::string, const google::protobuf::MethodDescriptor*> m_methodMap;
    };
    std::unordered_map<std::string, ServiceInfo> m_serviceMap;

    // ...
};
```
当远端一个Service服务器（比如UserService）调用RpcProvider的NotifyService时，便在里面记录信息，这就是注册，就是发布。
我们可以：
1. 通过远端传入的Service指针，`service->GetDescriptor()`获得服务描述符`ServiceDescriptor *`，
2. 从而获得服务的名字`name()`和方法的数量`method_count()`、每一个方法的描述符`MethodDescriptor*`，
3. 从而获得每一个方法的名字`name()`。

* 把所有方法的名字、方法的描述符记录在`m_methodMap`中。
* 把服务描述符和`m_methodMap`封装在`ServiceInfo`结构体中。
* 之后，把服务的名字和这个`ServiceInfo`作为键值对插入到`m_serviceMap`中。

需要引入`<google/protobuf/descriptor.h>`
```cpp
#include "rpcprovider.h"
#include "mprpcapplication.h"
#include <functional>
#include <google/protobuf/descriptor.h>
void RpcProvider::NotifyService(::google::protobuf::Service * service)
{
    const google::protobuf::ServiceDescriptor * pserviceDesc = service->GetDescriptor();

    ServiceInfo service_info;
    service_info.m_service = service;
    int methodCnt = pserviceDesc->method_count();
    for(int i = 0; i < methodCnt; ++i)
    {
        // 该方法定义于ServiceDescriptor
        // const MethodDescriptor* method(int index) const; 
        // 返回的是一个MethodDescriptor*
        // 获取了服务对象指定下标的rpc服务方法的描述
        const google::protobuf::MethodDescriptor* pmethodDesc = pserviceDesc->method(i);
        std::string method_name = pmethodDesc->name();
        std::cout << i << ". method_name: " << method_name << std::endl;
        service_info.m_methodMap.insert({method_name, pmethodDesc});
    }
    
    std::string service_name = pserviceDesc->name();
    std::cout << "service_name: " << service_name << std::endl;
    m_serviceMap.insert({service_name, service_info});
}
```
### OnConnection
在有新的请求时，会触发OnConnection回调，用于处理连接后的操作。OnConnection的通知调用不是用户调用的，而是muduo库自动调用的。
我们暂时不用处理什么工作。只是判断一下连接状态是否正常，如果断开了则主动close。
```cpp
// 新的socket连接回调
void RpcProvider::onConnection(const muduo::net::TcpConnectionPtr& conn)
{
    if (!conn->connected())
    {
        conn->shutdown();
    }
}
```
rpc的连接是短连接，客户端请求、服务端响应之后，服务端就会主动关闭连接。
### OnMessage
消息是字节流的。
```
// 框架内部，RpcProvider和RpcConsumer协商好通信用的protobuf数据类型
// 请求时需要提供 service_name method_name args
// 需要处理包内的粘包问题，比如：UserService和Login以及args在一起，无法区分
// 因此需要header_size + header_str + args，先把名字部分和参数部分区分开
// 要注意需要区分args和下一个请求包的 包外粘包问题，需要在header_str中记录args_size
// 可以通过protobuf定义这个RpcHeader的message类型，根据上述设计，包含了service_name、method_name、args_size
// header_size指的是RpcHeader这个message包的字符串长度
// header_size（uint32_t 大小个字节）
```
#### 定义rpcheader.proto
在src下定义此文件（`rpcheader.proto`）
```protobuf
syntax = "proto3";

package mprpc;

message RpcHeader
{
    bytes service_name = 1;
    bytes method_name = 2;
    uint32 args_size = 3;
}
```
之后在此目录下用protoc命令生成cc和h文件。
```sh
protoc rpcheader.proto --cpp_out=./
```

#### rpcprovider.cc引入rpcheader.pb.h，做读取、解析工作
1. 解析接收到的字符流`recv_buf`中的内容。
    1. 取出前4个字节（`sizeof uint32_t`），即`header_size`。（`header_size`指的是RpcHeader这个message包的字符串长度，包含了`service_name`、`method_name`、`args_size`信息）。响应端通过string的`copy`方法读出（相应的写入方法是insert方法，是请求端填写的）。
    2. 获取了`header_size`之后，在`recv_buf`取出4字节之后的`header_size`长度的字符串，存入`rpc_header_str`。
    3. 用protobuf生成的RpcHeader的ParseFromString方法即可解析`rpc_header_str`，填入了RpcHeader对象。再调用RpcHeader对象方法`service_name()`、`method_name()`、`args_size()`即可读出解析到的数据。
    4. 之后，recv_buf的`4字节 + header_size`之后的`args_size`长度的字符串就是args的数据。可以同上述方法解析出内容。

需要引入`"rpcheader.pb.h"`
```cpp
void RpcProvider::onMessage(const muduo::net::TcpConnectionPtr& conn, muduo::net::Buffer* buffer, muduo::Timestamp timestamp)
{
    // 接收到的rpc调用请求的字符流
    // 包含了方法名、参数
    std::string recv_buf = buffer->retrieveAllAsString();
    // 从字符流中读取前4个字节的header_size（即按内存中原来的2进制读取）
    // 有相应的string方法：insert、copy，是按2进制写入、读取固定字节的数据，而不是直接把数字存为长度不一的字符串。
    uint32_t header_size = 0;
    // 从recv_buf的 0 位置开始，读取uint32_t 大小个字节，存入header_size。
    recv_buf.copy((char*)&header_size, sizeof(uint32_t), 0);
    // 根据header_size读取数据头的原始字符流
    // substr表示，略过recv_buf的前uint32_t 大小个字节，返回之后的header_size长度的字符串
    // rpc_header_str 目前保存了service_name method_name args_size
    std::string rpc_header_str = recv_buf.substr(sizeof(uint32_t), header_size);
    // 用rpcheader.pb.h生成的rpcHeader中的ParseFromString反序列化rpc_header_str
    mprpc::RpcHeader rpcHeader;
    std::string service_name;
    std::string method_name;
    uint32_t args_size;
    if (rpcHeader.ParseFromString(rpc_header_str))
    {
        // header反序列化成功
        service_name = rpcHeader.service_name();
        method_name = rpcHeader.method_name();
        args_size = rpcHeader.args_size();
    }
    else
    {
        // header反序列化失败
        std::cout << "rpc_header_str: " << rpc_header_str << "parse error!" << std::endl;
        return;
    }

    // 获取rpc方法参数的字符流数据
    std::string args_str = recv_buf.substr(sizeof(uint32_t) + header_size, args_size);

    // 打印调试信息
    std::cout << "==============================" << std::endl;
    std::cout << "header_size: " << header_size << std::endl;
    std::cout << "rpc_header_str: " << rpc_header_str << std::endl;
    std::cout << "service_name: " << service_name << std::endl;
    std::cout << "method_name: " << method_name << std::endl;
    std::cout << "args_str: " << args_str << std::endl;
    std::cout << "==============================" << std::endl;
    // ...
}
```
#### 查表获取service对象和method对象
```cpp
    // ...

    auto service_it = m_serviceMap.find(service_name);
    if (service_it == m_serviceMap.end())
    {
        std::cout << service_name << " not exist!" << std::endl;
        return;
    }
    auto method_it = service_it->second.m_methodMap.find(method_name);
    if (method_it == service_it->second.m_methodMap.end())
    {
        std::cout << method_name << " not exist in " << service_name << std::endl;
        return;
    }
    
    google::protobuf::Service *service = service_it->second.m_service;
    const google::protobuf::MethodDescriptor *method = method_it->second;
    // ...
```
#### `service->GetRequestPrototype(method).New()`生成request和response
```cpp
    // ...

    // 生成rpc方法调用的请求request和响应response
    google::protobuf::Message *request = service->GetRequestPrototype(method).New();
    if (!request->ParseFromString(args_str))
    {
        std::cout << "request parse args_str error! args_str: " << args_str << std::endl;
        return;
    }
    google::protobuf::Message *response = service->GetResponsePrototype(method).New();
    // ...
```
#### `google::protobuf::NewCallback<>`生成回调函数
```cpp
    // ...
    
    // 给下面的method方法的调用，绑定一个Closure的回调函数
    google::protobuf::Closure* done = google::protobuf::NewCallback
        <RpcProvider, const muduo::net::TcpConnectionPtr&, google::protobuf::Message*>
            (this, &RpcProvider::SendRpcResponse, conn, response);

    // 在框架上根据远端rpc请求，调用当前rpc节点上发布的方法
    service->CallMethod(method, nullptr, request, response, done);
}
```
#### 编写`RpcProvider::SendRpcResponse()`
```cpp
void RpcProvider::SendRpcResponse(const muduo::net::TcpConnectionPtr& conn, google::protobuf::Message* response)
{
    std::string response_str;
    // 将response序列化为字符流
    if (response->SerializeToString(&response_str))
    {
        // 序列化成功后，通过网络把rpc方法执行的结果发送回rpc的调用方
        conn->send(response_str);
    }
    else
    {
        std::cout << "Serialize response_str error!" << std::endl;
    }
    conn->shutdown(); // 短连接效果，由rpcprovider主动断开连接，让出服务器资源。
}
```
## Mprpcchannel类
```cpp
#pragma once
#include <google/protobuf/service.h>
#include <google/protobuf/descriptor.h>
#include <google/protobuf/message.h>
class MprpcChannel : public google::protobuf::RpcChannel
{
public:
    // 所有通过stub代理对象调用的rpc方法，都走到这里了，统一做rpc方法调用的数据数据序列化和网络发送
    void CallMethod(const google::protobuf::MethodDescriptor *method,
                    google::protobuf::RpcController *controller, const google::protobuf::Message *request,
                    google::protobuf::Message *response, google::protobuf::Closure *done);
       
};
```
在这个方法中，处理服务调用方的请求包。

这个请求包只包含rpc调用的函数参数（args），不包含服务名、方法名。服务名、方法名可以用`method`参数查到。

把请求包（只包含args）序列化，得到序列化之后的args字符流`args_str`长度`args_size`。

填写 RpcHeader（`service_name`、`method_name`、`args_size`），序列化后得到 RpcHeader的字符流`rpc_header_str`长度`header_size`

之后按照约定好的格式，把`header_size`、`rpc_header_str`、`args_str`格式化填充、拼接到`send_rpc_str`。

之后便是TCP客户端编程，把`send_rpc_str`发送到配置文件中的远端ip、端口。

等待远端响应的数据，解析响应数据。填入了response。

CallMethod结束，Channel的使命结束，Stub的使命结束。

调用栈就回到了服务调用方的主程序中，输出response。
见[服务调用者主程序](#服务调用者主程序)

>服务调用方是通过Channel得知远端服务提供方的ip、端口的。
### 实现
```cpp
#include "mprpcchannel.h"
#include "rpcheader.pb.h"
#include "mprpcapplication.h"
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <error.h>
// 约定好的包格式：（header_size)(service_name method_name args_size)(args)
void MprpcChannel::CallMethod(const google::protobuf::MethodDescriptor *method,
    google::protobuf::RpcController *controller, const google::protobuf::Message *request,
    google::protobuf::Message *response, google::protobuf::Closure *done)
{
    const google::protobuf::ServiceDescriptor *sd = method->service();
    std::string service_name = sd->name();
    std::string method_name = method->name();
    
    // 序列化(args)，之后才能获取args的序列化字符串长度 args_size
    uint32_t args_size = 0;
    std::string args_str;
    if (request->SerializeToString(&args_str))
    {
        args_size = args_str.size();
    }
    else
    {
        std::cout << "serialize request error!" << std::endl;
        return;
    }
    // 填写rpc的请求header
    mprpc::RpcHeader rpcHeader;
    rpcHeader.set_service_name(service_name);
    rpcHeader.set_method_name(method_name);
    rpcHeader.set_args_size(args_size);
    // 序列化（header_size)(service_name method_name args_size)
    uint32_t header_size = 0;
    std::string rpc_header_str;
    if (rpcHeader.SerializeToString(&rpc_header_str))
    {
        header_size = rpc_header_str.size();
    }
    else
    {
        std::cout << "serialize request error!" << std::endl;
        return;
    }

    std::string send_rpc_str;
    send_rpc_str.insert(0, std::string((char*)&header_size, 4));
    send_rpc_str += rpc_header_str;
    send_rpc_str += args_str;

    // 打印调试信息
    std::cout << "==============================" << std::endl;
    std::cout << "header_size: " << header_size << std::endl;
    std::cout << "rpc_header_str: " << rpc_header_str << std::endl;
    std::cout << "service_name: " << service_name << std::endl;
    std::cout << "method_name: " << method_name << std::endl;
    std::cout << "args_str: " << args_str << std::endl;
    std::cout << "==============================" << std::endl;

    // 使用tcp编程，完成rpc方法的远程调用
    int cliendfd = socket(AF_INET, SOCK_STREAM, 0);
    if (-1 == cliendfd)
    {
        std::cout << "Create Socket Error: " << errno << std::endl;
        exit(EXIT_FAILURE);
    }

    // 读取配置文件rpcserver的信息
    std::string ip = MprpcApplication::GetInstance().GetConfig().Load("rpcserver_ip");
    uint16_t port = atoi(MprpcApplication::GetInstance().GetConfig().Load("rpcserver_port").c_str());
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    if (-1 == inet_pton(AF_INET, ip.c_str(), &(server_addr.sin_addr.s_addr)))
    {
        std::cout << "inet_pton Error: " << errno << std::endl;
        close(cliendfd);
        exit(EXIT_FAILURE);
    }
    if (-1 == connect(cliendfd, (struct sockaddr*)&server_addr, sizeof(server_addr)))
    {
        std::cout << "connect Error: " << errno << std::endl;
        close(cliendfd);
        exit(EXIT_FAILURE);
    }
    if (-1 == send(cliendfd, send_rpc_str.c_str(), send_rpc_str.size(), 0))
    {
        std::cout << "send Error: " << errno << std::endl;
        close(cliendfd);
        exit(EXIT_FAILURE);
    }
    char recv_buff[1024] = {0};
    int recv_size = 0;
    if (-1 == (recv_size = recv(cliendfd, recv_buff, sizeof(recv_buff), 0)))
    {
        std::cout << "recv Error: " << errno << std::endl;
        close(cliendfd);
        exit(EXIT_FAILURE);
    }
    // 反序列化rpc调用的响应数据
    // std::string response_str(recv_buff, 0, recv_size); // 如此构造有问题，如果recv_buff中存在\0则断开。
    // if (!response->ParseFromString(response_str))
    if (!response->ParseFromArray(recv_buff, recv_size))
    {
        std::cout << "parse response_str error! response_str: " << recv_buff << std::endl;
        close(cliendfd);
        return;
    }
    close(cliendfd);
}
```
## MprpcController 的作用
用于记录rpc调用过程中的状态信息。
试想，如果请求过程中，或者响应过程中，某一环节除了问题，那么服务调用方就拿不到response任何内容了，那么它去看了个寂寞？也不清楚是哪个环节出错了。这时候可以在参数中传入`mprpccontroller`的指针，让对应的MprpcController对象在内部记录状态信息。
```cpp
#pragma once
#include <google/protobuf/service.h>
class MprpcController : public google::protobuf::RpcController
{
public:
    MprpcController();
    void Reset();
    bool Failed() const;
    std::string ErrorText() const;
    void SetFailed(const std::string& reason);

    // 暂时空实现
    void StartCancel();
    bool IsCanceled() const;
    void NotifyOnCancel(google::protobuf::Closure *callback);
private:
    // RPC方法执行过程中的状态
    bool m_failed;
    // RPC方法执行过程中的错误信息
    std::string m_errText;
};
```
### 实现
```cpp
#include "mprpccontroller.h"
MprpcController::MprpcController()
{
    m_failed = false;
    m_errText = "";
}
void MprpcController::Reset()
{
    m_failed = false;
    m_errText = "";
}
bool MprpcController::Failed() const
{
    return m_failed;
}
std::string MprpcController::ErrorText() const
{
    return m_errText;
}
void MprpcController::SetFailed(const std::string& reason)
{
    m_failed = true;
    m_errText = reason;
}
void MprpcController::StartCancel(){}
bool MprpcController::IsCanceled() const { return false; }
void MprpcController::NotifyOnCancel(google::protobuf::Closure *callback){}
```
### 如何使用
比如在`calluserservice.cc`下，可以：
在stub调用某方法前，定义一个MprpcController对象，之后把其指针传到调用方法的函数参数内。
在读取response前，先判断controller的状态信息。如果错误就不读了。

```cpp
#include "mprpccontroller.h"
// ...
    // 发起rpc方法的调用。
    MprpcController controller;
    stub.Login(&controller, &request, &response, nullptr);
    
    if (controller.Failed()) // rpc调用过程中有异常，response没有填入数据
    {
        std::cout << controller.ErrorText() << std::endl;
    }
    else // 一次rpc调用结束，返回了，读返回的response结果
    {
        if (response.result().errcode() == 0)
        {
            // 成功
            std::cout << "rpc login response success: " << response.success() << std::endl;
        }
        else
        {
            std::cout << "rpc login response error: " << response.result().errmsg() << std::endl;
        }
    }
    return 0;
}
```
在`mprpcchannel.cc`中，有可能出现异常的地方，均用controller记录。

别忘了在src下的CMakeLists中的`set(SRC_LIST ... )`后面添加上`mprpccontroller.cc`。
## 日志类
## 编译
框架依赖muduo库，因此需要在src目录下的`CMakeLists.txt`添加：

>如果忘记了库叫什么名字，可以在`/usr/lib`或`/usr/local/lib`下查找：

```bash
sudo find /usr -name "libmuduo*"
```

```bash
/usr/local/lib/libmuduo_http.a
/usr/local/lib/libmuduo_net.a
/usr/local/lib/libmuduo_base.a
/usr/local/lib/libmuduo_inspect.a
```

我们需要的muduo的库名：muduo_net、muduo_base
```cmake
# 当前文件夹中所有文件 别名 SRC_LIST
aux_source_directory(. SRC_LIST)
# 生成动态库
add_library(mprpc SHARED ${SRC_LIST})
# 依赖muduo库
target_link_libraries(mprpc muduo_net muduo_base pthread)
```
由于我们在编译muduo时编译成了静态库，而我们现在CMake指定生成的是动态库（`add_library(mprpc SHARED ${SRC_LIST})`）。
由于里面有静态库成分，所以编译链接时会报错，因此我们暂时先生成静态库。
```cmake
# 当前文件夹中所有文件 别名 SRC_LIST
aux_source_directory(. SRC_LIST)
# 生成静态库
add_library(mprpc ${SRC_LIST})
# 依赖muduo库
target_link_libraries(mprpc muduo_net muduo_base pthread)
```

测试：
```bash
mrcan@ubuntu:~/mprpc/bin$ ./provider -i test.conf 
rpcserver_ip: 127.0.0.1
rpcserver_port: 8000
zookeeper_ip: 127.0.0.1
zookeeper_port: 5000
RpcProvider Start Service at IP: 127.0.0.1, port: 8000
```

### `aux_source_directory`存在的问题
`~/mprpc/src`下的`CMakeLists.txt`原本是如下定义的：
```cmake
# 当前文件夹中所有文件 别名 SRC_LIST
aux_source_directory(. SRC_LIST)
# 生成静态库
add_library(mprpc ${SRC_LIST})
# 依赖muduo库
target_link_libraries(mprpc muduo_net muduo_base pthread)
```

`aux_source_directory(. SRC_LIST)`表示给当前文件夹下的所有内容起了别名`SRC_LIST`。
一开始构建、编译没什么问题，但后面发现如果我们给src目录增加`.cc`文件后再进行编译则无法找到这些新增的文件。

说明build只记录了当时`src`目录下的旧文件，而没有记录新增的文件。
所以我们还是这么写吧：
```cmake
# 当前文件夹中所有文件 别名 SRC_LIST
set(SRC_LIST mprpcapplication.cc mprpcconfig.cc rpcheader.cc rpcprovider.cc)

# 生成静态库
add_library(mprpc ${SRC_LIST})
# 依赖muduo库
target_link_libraries(mprpc muduo_net muduo_base pthread)
```

# 调试

在顶级cmake写下：`set(CMAKE_BUILD_TYPE "Debug")`
![](../../images/mprpc_rpc框架/image-20250713173928826.png)
运行：
```bash
gdb ./provider
```
![](../../images/mprpc_rpc框架/image-20250713174142619.png)
break加断点，由于我们是生成了so动态链接库，所以会有如下提示：
```gdb
break mprpcconfig.cc:第n行
```
![](../../images/mprpc_rpc框架/image-20250713174407071.png)
加完断点后，run，运行程序。（gdb调试时，如果有参数，正确启动方式是：`run arg1 arg2`）
```gdb
run -i test.conf
```
结果：
![](../../images/mprpc_rpc框架/image-20250713174720348.png)
调试过程中，可以敲入bt查看栈帧信息，l查看源代码信息。
bt
![](../../images/mprpc_rpc框架/image-20250713174846521.png)
l
![](../../images/mprpc_rpc框架/image-20250713174857557.png)
敲入n可以单步执行。
![](../../images/mprpc_rpc框架/image-20250713174922879.png)
p + 变量名，可以打印变量的值。
![](../../images/mprpc_rpc框架/image-20250713174949860.png)
q退出调试
# zookeeper
channel一开始不知道服务节点的ip、port，需要先去查找。
