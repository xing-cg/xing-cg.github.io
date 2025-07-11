---
title: Cpp_STL_list
typora-root-url: ../..
categories:
  - [Cpp, STL]
tags:
  - null 
date: 2022/2/28
update:
comments:
published:
---

# 内容

1. list
2. 区分：list是双向迭代器，vector是随机迭代器

# 构造

```c++
#include<iostream>
#include<vector>
using namespace std;
int main()
{
	vector<int> ar1;	//空vector
	vector<int> ar2 = { 12,23,34,45,56,67,78,89 }; //类似数组的初始化方式
	vector<int> ar3(10, 23); //存10个23
	vector<int> ar4({ 12,23,34,45,56,67 }); //初始化列表作为参数构造
	vector<int> ar5(ar2); //拷贝构造，按ar2的有效size
	vector<int> ar6(std::move(ar2)); //移动构造
}
```

# 与vector的区别

1. 底层实现
2. 空间利用率
3. 查询元素
    1. vector：iterator `operator[]`，`find` O(n)，`binary_search` O(logn)
    2. list：iterator `find` O(n)
4. 插入和删除
    1. vector情况较多
        1. 看是否需要增容；
        2. 在中间插入还是在最后插入；
        3. 删除：最后删除O(1)，中间删除需要内存拷贝；
        4. resize和reserve的区别
    2. list：
        1. 插入O(1)，需要内存申请，
        2. 删除O(1)，需要内存释放。
5. 迭代器：
    1. vector：随机迭代器，可检查越界。支持`++`，`--`，`+`，`+=`，`<`，`>`，`==`，`!=`；插入和删除元素可能导致迭代失效。
    2. list：插入元素不会导致迭代器失效，但是删除元素除了erase有特殊情况，可能会导致当前迭代器失效。
# 如何抉择？

什么时候该用vector？什么时候该用list？
1. 如果需要高效的随机存取，而不在乎插入和删除的效率（很少使用插入和删除操作)。选用vector。
2. 如果需要大量的插入和删除的操作，随机存取很少使用。选用list。
## 举例

例1∶比如制作了一个游戏玩家排行榜，那么就要实时刷新这个排行榜，这个时候，如果有人达到排行的条件，那么就要把他加进去，如果人数多出了，那么就把排名靠后的移除掉，这就需要大量的插入删除操作，那么这个时候就应该优先考虑list，而不是vector。
## 中间插入
想在某个值的元素前面插入一个元素，首先需要find出位置并得到迭代器it，然后在it位置进行插入。
```cpp
int main()
{
    std::list<int> lst{ 1, 2, 3, 4, 5, 6, 7, 8 };
    auto it = std::find(lst.begin(), lst.end(), 5);
    
    lst.emplace(it, 50);
    
    std::for_each(
        lst.cbegin(),
        lst.cend(),
        [](auto&& v)
        {
            std::cout << v << std::endl;
        });
}
```
>emplace:
>（1）在位置插入新元素。
>（2）与其他标准顺序容器不同，`list`和`forward_list`经过专门设计，可以在任何位置高效地插入和删除元素，即使在序列的中间也是如此。
>（3）就地构造
>返回值：
>一个指向新插入元素的迭代器。