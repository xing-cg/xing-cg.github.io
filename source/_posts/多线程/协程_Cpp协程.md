---
title: 协程_Cpp协程
categories:
  - - Cpp
    - Modern
  - - 操作系统
    - 多线程
tags: 
date: 2024/7/27
updated: 
comments: 
published:
---
# 协程框架的解释
怎么使用框架？
写了摄像头的驱动，驱动有sdk，用户可以调用sdk，来驱动摄像头。
摄像头是用什么形式的API写的？就不一定了，可能是C语言，可能是`C++`。`C++`可以按MFC提供，也可以按面向对象提供，也可以按类似于COM组件（接口）的形式提供，也可以按协程库的形式提供。
此协程库框架，是给API厂商使用的。不是一个已经实现了的库，更像是一个中间件。
比如我们想要提供一个读文件的函数给用户使用，readFile，从名字来看其一般是阻塞的，读完后才能返回。如果不想阻塞，则开辟一个线程，在新线程中让其readFile。
非阻塞版本可以称其为`readFile_Async`。用户从名字上就能看到，这是非阻塞的，调用后主线程可以立即抽身去处理其他事情，在子线程执行完毕后，以回调的形式通知主线程。
而在协程版本下的实现形式则不像传统线程间的回调形式（被割裂在两端），而是直接地写在了一个函数中，用`co_await`连接上下动作，此处的`co_await`的成功后的返回“相当于”回调函数。回调之后，就继续做`co_await`之后的动作即可。
协程库框架还可以灵活切换协程所在的线程，比如切换到主线程、工作线程等等。也可以配置线程池。
假如我们是一个厂商，现在使用该协程库框架开发一个readFile API：
```cpp
#include "Agave.hpp"
#include <iostream>
agave::IAsyncAction read_file_async(void)
{

}
```
# 协程工作的总体流程
协程的工作流，在形式上，是包装在一个函数中的。
在函数中，间断地进行：work状态和`co_await`状态交替。
假如，协程在t1线程上进行work状态。此时，需要`co_await`。
`co_await`表示等待着某一个其他工作的结果。比如等待网络中传来的文件，等待下载完毕。在`co_await`时，此函数就会返回，协程暂停运行。`co_await`成功后，才能继续下面的操作。
下面的操作，可能会在t2线程上进行work（也可能继续在t1，只是举个例子）。
随后，可能还会进行`co_await`等操作。

>看样子`co_await`表示的是“等待”的意思，但在协程的工作环境中，其实语义更像是"不等了"，“我先去处理别的事了，先走了”。有点儿异步的感觉。因为`co_await`语句使用时，会马上退出既有流程，直到被通知，才进行回调，从而返回work状态。

从t1线程到t2线程的转变，相当于调用了一个callback。此callback是被`co_await`间隔的。

由上述例子说明，协程和线程不冲突（或者说没关系），因为每次`co_await`成功后不一定会切换到另一个线程。
# 创建项目
先创建一个空项目。（Agave是根据`C++`标准协程库编写的，记得设置为20标准）
再copy已有的4个文件（`Agave.cpp`、`B_Object.hpp`、`BJobScheduler.cpp`、`BJobScheduler.h`）到该空项目文件目录下。
之后，右键VS项目中的`Header Files`，Add，Existing Item，选择`.hpp`和`.h`文件。
右键`Source Files`，Add，Existing Item，选择`.cpp`文件。
在`Source Files`中新建一个文件`demo.cpp`。在其中编写测试代码。
## 初试1
1. demo函数返回值是`agave::IAsyncAction`，则代表此函数是一个协程。
2. 我们在main主线程中调用demo，即在主线程中创建此协程。
3. demo协程在其中`co_await`，后面接的`read_file_async`可以看作是厂家的API。这个API的名称带有`async`，意味着它是一个异步的操作，在此`co_await XXX_async`语句后，会立即返回。但是不会进行下一步操作（输出读取完毕），而是暂时结束demo“函数”，直到`co_await`的操作（`read_file_async`）完成后，才继续demo之后的操作，相当于被callback。

```cpp
#include "Agave.hpp"
#include <iostream>

using namespace std::chrono_literals;

agave::IAsyncAction read_file_async(void)
{
    std::cout << "Read File Coroutine started on thread: " << std::this_thread::get_id() << '\n';
    std::cout << "Reading..." << std::endl;
    // 注意此处不能使用this_thread::sleep_for
    // std::this_thread::sleep_for(5s); // error
    co_await 5s; // 等待5秒，模拟文件读取
    co_return; // 相当于callback
}
agave::IAsyncAction demo()
{
    std::cout << "demo() on thread: " << std::this_thread::get_id() << '\n';
    co_await read_file_async();
    std::cout << "Reading finished on thread: " << std::this_thread::get_id() << '\n';
}
int main(void)
{
    std::cout << "Main on thread: " << std::this_thread::get_id() << '\n';
    auto action = demo();
    // 主线程等待协程运行完毕
    std::this_thread::sleep_for(8s);
    return 0;
}
```
### 关于为什么不能用`sleep_for`
经测试，如果用`sleep_for`
输出是这样：发现没有执行输出demo的第三个语句（Reading finished）
```
Main on thread: 25912
demo() on thread: 25912
Read File Coroutine started on thread: 25912
Reading...

(Program End)
```
为什么呢？且看下面的正常输出结果中对线程运行的分析。
### 输出结果
```
Main on thread: 6260 
demo() on thread: 6260
Read File Coroutine started on thread: 6260
Reading...
Reading finished on thread: 20516

```
demo协程在主线程6260启动，另一个read协程也在主线程6260启动。而读取完毕后，demo协程此时却处于20516线程（内容是通知信息，通知read操作完毕）。
>因此上面为什么在read协程函数中用`sleep_for`就不能正常打印最后一句话，可能和read协程函数调用`this_thread`睡眠导致主线程睡眠有关系。
>
>总之，测试的结果就是read协程睡眠后，再也没回到demo中去。

这样的线程模式是不好的：
1. demo协程在主线程启动无所谓，没事。
2. 但是read协程以及read操作在主线程上就不妥了，应该将read操作另起一个线程
3. 在read完毕callback后，执行demo中的第三句，应该给主线程通知才对，不应该通知给一个无关的线程。我们此例通知给了一个无关的线程，是因为read协程函数中`co_await 5s`会导致它另起一个时间线程。导致返回到demo中时，跑到了这个时间线程中去通知了。
## 初试2
更改模式，应该在read协程函数中另起线程，调用的是`co_await agave::resume_background()`。意为唤醒一个后台的线程。默认是即刻构造一个`std::jthread`。我们后期也可以拓展功能，将这个接口连接到一个线程池中，从线程池中拿取线程。
在这种情况下，就可以使用`sleep_for`了。
```cpp
#include "Agave.hpp"
#include <iostream>

using namespace std::chrono_literals;

agave::IAsyncAction read_file_async(void)
{
    co_await agave::resume_background(); // 另起线程
    std::cout << "Read File Coroutine started on thread: " << std::this_thread::get_id() << '\n';
    std::cout << "Reading..." << std::endl;
    
    std::this_thread::sleep_for(5s);
    
    co_return; // 相当于callback
}
agave::IAsyncAction demo()
{
    std::cout << "demo() on thread: " << std::this_thread::get_id() << '\n';
    co_await read_file_async();
    std::cout << "Reading finished on thread: " << std::this_thread::get_id() << '\n';
}
int main(void)
{
    std::cout << "Main on thread: " << std::this_thread::get_id() << '\n';
    auto action = demo();
    // 主线程等待协程运行完毕
    std::this_thread::sleep_for(8s);
    return 0;
}
```
输出结果：
```
Main on thread: 29580
demo() on thread: 29580
Read File Coroutine started on thread: 15800
Reading...
Reading finished on thread: 15800

```
但是此时demo的read操作完毕消息还是没有发送给主线程，而是发送给了read另起的线程中。
如何回到主线程呢？不同的操作系统具体实现不一样，但是我们可以自己写一个统一的接口`resume_mainThread`。在demo中的`co_await read_file_async`语句的下一条写下`co_await resume_mainThread()`。之后的输出操作便的在主线程中操作了。
## 返回带值
不返回东西时返回值类型写为`agave::IAsyncAction`。
返回东西时，返回值类型写为`agave::IAsyncOperation<T>`。模板参数类型是要携带的值类型。
对应地，`co_await`不能裸着用了，要用一个变量接收。
```cpp
agave::IAsyncOperation<int> read_file_async(void)
{
    co_await agave::resume_background(); // 另起线程
    std::cout << "Read File Coroutine started on thread: " << std::this_thread::get_id() << '\n';
    std::cout << "Reading..." << std::endl;
    
    std::this_thread::sleep_for(5s);
    
    co_return 50; // callback + 携带值
}

agave::IAsyncAction demo()
{
    std::cout << "demo() on thread: " << std::this_thread::get_id() << '\n';
    
    auto result = co_await read_file_async();
    
    std::cout << "Reading finished: " << result << ", on thread: " << std::this_thread::get_id() << '\n';
}
```
# 协程取消
1. 调用`co_await agave::get_cancellation_token()`获取一个此协程的取消变量。随时可以查看`can.is_canceled()`来判断是否有取消信号。
2. 在其他线程中，通过`auto async_var = read_file_async()`拿到协程的标识，提供这个标识`async_var.cancel()`，来给该协程发送取消信号。

```cpp
#include "Agave.hpp"
#include <iostream>

using namespace std::chrono_literals;

agave::IAsyncAction read_file_async(void)
{
    co_await agave::resume_background(); // 另起线程
    auto can = co_await agave::get_cancellation_token();
    std::cout << "Read File Coroutine started on thread: " << std::this_thread::get_id() << '\n';
    for (int i = 0; i < 5; ++i)
    {
        std::this_thread::sleep_for(1s);
        if (can.is_canceled())
        {
            std::cout << "Canceled on thread: " << std::this_thread::get_id() << '\n';
            break;
        }
        else
        {
            std::cout << "Reading... " << i << std::endl;
        }
    }

    co_return; // 相当于callback
}
agave::IAsyncAction demo()
{
    std::cout << "demo() on thread: " << std::this_thread::get_id() << '\n';
    auto async_var = read_file_async();
    std::this_thread::sleep_for(3s);
    async_var.cancel();
    co_await async_var;

    std::cout << "Reading finished on thread: " << std::this_thread::get_id() << '\n';

}
int main(void)
{
    std::cout << "Main on thread: " << std::this_thread::get_id() << '\n';
    auto action = demo();
    // 主线程等待协程运行完毕
    std::this_thread::sleep_for(8s);
    return 0;
}
```
输出：
```
Main on thread: 18768
demo() on thread: 18768
Read File Coroutine started on thread: 19036
Reading... 0
Reading... 1
Canceled on thread: 19036
Reading finished on thread: 19036

```
# Async的机制怎么实现
Async换句话来说是让函数（或任务）的结果在“成功之后”返回（但实际上函数调用后当即返回了）。如果是在C语言下做，是函数返回一个标号，我们在外部轮询一个范围的标号，如果有某个标号上有消息，再去获取它。
# 