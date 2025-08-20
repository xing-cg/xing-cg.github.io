---
title: Android_Handler
categories:
  - - Android
tags: 
date: 2022/9/8
updated: 
comments: 
published:
---

# 概念

Handler、MessageQueue、Runnable、Looper。

# Runnable和Message

Runnable和Message可以被压入某个MessageQueue中，形成一个集合。

注意，一般情况下某种类型的MessageQueue只允许保存相同类型的 Object。图中我们只是为了叙述方便才将它们混放在同一个MessageQueue中，实际源码中需要先对Runnable进行相应转换。

# Looper循环地去做某件事

比如在这个例子中，它不断地从MessageQueue中取出一个item，然后传给Handler进行处理，如此循环往复。假如队列为空，那么它会进入休眠。

# Handler是真正处理事情的地方

它利用自身的处理机制，对传入的各种Object进行相应的处理并产生最终结果。

用一句话来概括它们，就是：Looper不断获取MessageQueue中的一个Message，然后由Handler来处理。

可以看出，上面的几个对象是缺一不可的。它们各司其职，很像一台计算机中CPU的工作式：中央处理器（Looper）不断地从内存（MessageQueue）中读取指令（Message），执行指令（Handler），最终产生结果。当然，到目前为止我们还只是从逻辑的层面理清了它们的关系，接下来将从代码的角度来验证这些假设的真实可靠性。