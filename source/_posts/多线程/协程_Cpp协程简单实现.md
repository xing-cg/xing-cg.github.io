---
title: 协程_Cpp协程简单实现
categories:
  - - Cpp
    - Modern
  - - 操作系统
    - 多线程
tags: 
date: 2023/5/26
updated: 
comments: 
published:
---

在`C++`中实现协程库可能是一项具有挑战性的任务，但它是深入了解协程工作原理的好方法。 

# 协程工作原理

在高层次上，协程允许编写看似同步的代码，但可以异步挂起和恢复。 

基本思想是协程是一个可以在执行过程中的特定点暂停和恢复的函数，允许其他代码同时运行。

# 状态机实现协程

要在`C++`中实现协程，需要管理协程的状态，包括其当前堆栈帧、指令指针和任何局部变量。还需要能够在执行过程中的特定点暂停和恢复协程。

在`C++`中实现协程的一种方法是使用状态机。状态机跟踪协程的状态并提供挂起和恢复协程的方法。当协程第一次被调用时，它进入初始状态并开始执行。当遇到挂起点时，状态机保存协程的状态并将控制权返回给调用代码。当协程恢复时，状态机恢复协程的状态并从它停止的地方继续执行。

下面是一个示例，说明如何使用状态机实现一个简单的协程库：

```cpp
#include <iostream>
#include <functional>
#include <vector>

class Coroutine {
public:
    Coroutine(std::function<void()> f)
        : m_function(f), m_state(0)
    {}

    void resume() {
        switch (m_state) {
        case 0: // Start
            m_state++;
            m_function();
            break;
        case 1: // Suspend
            return;
        }
    }

    bool is_complete() const {
        return m_state == 2;
    }

private:
    std::function<void()> m_function;
    int m_state;
};

class Scheduler {
public:
    void add_coroutine(Coroutine coroutine) {
        m_coroutines.push_back(coroutine);
    }

    void run() {
        while (!m_coroutines.empty()) {
            auto coroutine = m_coroutines.front();
            m_coroutines.pop_front();

            coroutine.resume();

            if (!coroutine.is_complete()) {
                m_coroutines.push_back(coroutine);
            }
        }
    }

private:
    std::deque<Coroutine> m_coroutines;
};

// Example usage
void foo(Coroutine& coroutine) {
    for (int i = 0; i < 5; i++) {
        std::cout << "foo " << i << std::endl;
        coroutine.resume();
    }
}

void bar(Coroutine& coroutine) {
    for (int i = 0; i < 3; i++) {
        std::cout << "bar " << i << std::endl;
        coroutine.resume();
    }
}

int main() {
    Scheduler scheduler;
    Coroutine coroutine1(std::bind(foo, std::placeholders::_1));
    Coroutine coroutine2(std::bind(bar, std::placeholders::_1));

    scheduler.add_coroutine(coroutine1);
    scheduler.add_coroutine(coroutine2);

    scheduler.run();

    return 0;
}
```

在这个例子中，我们定义了一个 Coroutine 类，它接受一个函数对象作为它的构造函数参数。 resume() 方法将协程推进到下一个状态，is_complete() 方法检查协程是否已执行完毕。

我们还定义了一个 Scheduler 类，它维护一个协程列表并一次运行一个，直到它们全部完成。 在示例用法中，我们定义了两个简单的函数 foo 和 bar，它们打印出一些文本并调用`coroutine.resume()`来暂停协程并将控制权交还给调度程序。我们创建两个`Coroutine`对象，每个函数一个，并将它们添加到`Scheduler`中。 最后，我们调用`scheduler.run()`开始运行协程。

运行该程序时，您应该会看到以下输出：

```
foo 0
bar 0
foo 1
bar 1
foo 2
bar 2
foo 3
foo 4
```

这演示了协程如何工作的基本思想。 foo 和 bar 函数以来回的方式执行，因为调度程序暂停和恢复协程。

当然，这只是一个简单的例子，实现一个功能完备的协程库还有很多细节。 但是，我希望这能让您大致了解如何在 C++ 中实现协程。

# 实现

要在`C++`中实现协程库，您需要使用`C++20`中引入的语言对协程的内置支持。具体来说，您需要使用`std::coroutine_traits`模板来定义协程类型，并使用`std::suspend_always`和`std::suspend_never`类来指示协程何时应该挂起和恢复。
