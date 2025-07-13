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
## 服务调用者继承UserServiceRpc_Stub，实现rpc虚方法

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
## 
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