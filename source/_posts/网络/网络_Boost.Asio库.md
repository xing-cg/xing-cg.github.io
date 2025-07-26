---
title: 网络_Boost.Asio库
categories:
  - - 网络
  - - Cpp
tags: 
date: 2025/7/26
updated: 2025/7/26
comments: 
published:
---
# `Boost::Asio`库
`Boost::Asio`是一个用于网络和异步编程的`C++`库，它提供了一套**跨平台的网络编程接口**，能够方便地创建客户端和服务器应用程序。`Boost::Asio`支持同步和异步操作，可以处理多种类型的网络协议，包括TCP、UDP、SSL等。
## 安装
```sh
sudo apt-get install libboost-all-dev
```
## cmake配置
在Linux系统上，除了链接boost库以外，还需要链接pthread库。
```cmake
cmake_minimum_required(VERSION 3.10.0)
project(boost-asio-demo VERSION 0.1.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 11)

find_package(Boost REQUIRED COMPONENTS system)

include_directories(${Boost_INCLUDE_DIRS})

add_executable(server server.cpp)
target_link_libraries(server ${Boost_LIBRARIES} pthread)

add_executable(client client.cpp)
target_link_libraries(client ${Boost_LIBRARIES} pthread)

```
# demo
## 服务器
```cpp
#include <iostream>
#include <boost/asio.hpp>
int main(int argc, char *argv[])
{
    boost::asio::io_service service;
    boost::asio::ip::tcp::acceptor acceptor(service, 
        boost::asio::ip::tcp::endpoint(boost::asio::ip::tcp::v4(), 8888));
    std::cout << "Server started, listening on port 8888" << std::endl;

    while (1)
    {
        boost::asio::ip::tcp::socket socket(service);
        acceptor.accept(socket);
        std::cout <<"Connection" << std::endl;

        std::string msg = "Hello From Boost.Asio Server\n";
        boost::system::error_code err;
        boost::asio::write(socket, boost::asio::buffer(msg), err);
    }
}
```
## 客户端
```cpp
#include <iostream>
#include <boost/asio.hpp>
int main(int argc, char *argv[])
{
    boost::asio::io_service service;
    boost::asio::ip::tcp::endpoint endpoint(boost::asio::ip::address::from_string("127.0.0.1"), 8888);
    boost::asio::ip::tcp::socket socket(service);
    socket.connect(endpoint);
    
    boost::asio::streambuf buf;
    boost::asio::read_until(socket, buf, "\n");
    
    std::istream in(&buf);
    std::string msg;
    std::getline(in, msg);
    std::cout << "Received msg from server: " << msg << std::endl;
}
```
## 测试
```
mrcan@ubuntu:~/boost-asio-demo/build$ ./server 
Server started, listening on port 8888
Connection
```

```
mrcan@ubuntu:~/boost-asio-demo/build$ ./client 
Received msg from server: Hello From Boost.Asio Server
```