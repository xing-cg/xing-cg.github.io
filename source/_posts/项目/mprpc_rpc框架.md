---
title: mprpc_rpc框架
categories:
  - - 项目
    - rpc
tags: 
date: 2022/9/13
update: 2025/7/13
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
接下来，就是使用user.pb.h。
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
## 服务调用者继承UserServiceRpc_Stub，实现rpc虚方法，

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

## mprpcapplication.h

# 调试

在顶级cmake写下：`set(CMAKE_BUILD_TYPE "Debug")`

注意点：

1. gdb调试时，如果有参数，正确启动方式是：`run arg1 arg2`
2. 
# CMakeLists.txt
## 根目录
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
