---
title: Cpp_auto
categories:
  - - Cpp
tags: 
date: 2024/3/20
updated: 
comments: 
published:
---
# 内容
1. 初识auto
# auto

如果不在返回auto的函数的返回值类型有冲突的后面加` -> int`，则会编译不通过，因为auto无法推断到底返回浮点型还是整型，则可以加` -> int`进行声明：返回值统一强转为int。

这种auto在前，返回值类型在后的用处还有一个好处就是：如果返回值类型特别复杂、特别长，则可以先写auto代替，把又臭又长的名字放在后缀，起到了使代码美观的作用（避免头重脚轻）。

```cpp
auto show(int v)
{
    if(v < 10)
        return 30.0;
    else
        return 30;
}
```

```cpp
auto show(int v) -> int
{
    if(v < 10)
        return 30.0;
    else
        return 30;
}
```
