---
title: Cpp_模板
typora-root-url: ../..
categories:
  - [Cpp]
tags:
  - null 
date: 2021/11/2
update:
comments:
published:
---

# 内容

# vector

容器--数组

## 关于size

resize

reserve

size

max_size

## 使用模板实现vector



## 实现迭代器

```cpp
template<class T>
void print(T & obj)
{
	typename T::iterator it = obj.begin();
	for (; it != obj.end();it++)
	{
		cout << *it << endl;
	}
}
template<class T>
void r_print(T& obj)
{
	typename T::reverse_iterator it = obj.rbegin();
	for (;it != obj.rend();it++)
	{
		cout << *it << endl;
	}

}
```

