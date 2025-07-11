---
title: 协程_Go语言协程实现
categories:
  - - Go
  - - 操作系统
    - 多线程
tags: 
date: 2023/3/6
updated: 
comments: 
published:
---
# 原理

```go
func main()
{
    go sayHello()
    // continue doing other things
}
func sayHello()
{
    fmt.Println("hello")
}
```

