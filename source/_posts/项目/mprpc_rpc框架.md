---
title: mprpc_rpc框架
categories:
  - - 项目
    - rpc
tags: 
date: 2022/9/13
update: 
comments: 
published:
---

# rpc框架的使用

```cpp
int main(int argc, char ** argv)
{
    // 框架的初始化操作
    MprpcApplication::Init(argc, argv);
        
    // publish service
    RpcProvider provider; //provider是一个rpc网络服务对象，负责发布服务到rpc节点上
    // 把UserService对象发布到rpc节点上
    provider.NotifyService(new UserService());
    
    // 启动一个rpc服务发布节点
    provider.Run();
    
}
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
