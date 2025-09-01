---
title: Cpp_STL_关联容器
categories:
  - - Cpp
    - STL
tags: 
date: 2022/3/1
updated: 
comments: 
published:
---
# 内容

1. 无序关联容器 - unordered
    1. 链式哈希表：增删查O(1)。适用于处理海量数据查重复、去重复
    2. `unordered_set`、`unordered_multiset`
    3. `unordered_map`、`unordered_multimap`
2. 有序关联容器
    1. 红黑树：增删查O($log_2 N$)
    2. `set`、`multiset`
    3. `map`、`multimap` 
3. set和map的区别
    1. set只有key。
    2. map每一个key都对应一个value。
# unordered_set

1. 插入：insert
2. 遍历：iterator
3. 删除：erase，可按key，也可按iterator

无序set、map的迭代器是forward类型，只能单向，不能双向。
插入速度快于红黑树，但可能浪费空间。如果hash效果不好，可能效率不高。
## 插入
```cpp
#include<unordered_set>
using namespace std;
int main()
{
    unordered_set<int> set1;
    for(int i = 0; i < 50; ++i)
    {
        set1.insert(rand() % 20 + 1)；
    }
    cout << set1.size() << endl;
    cout << set1.count(15) << endl;
}
//size是指set容器目前插入了的元素数，可能为小于等于50，因为插入已存在的话则不会重复
//count是指目标值在set容器中存在的数目，在unordered_set中只可能为1、0，不可能大于1。
```
要知道，vector、deque、list的插入都需要一个iterator才能插入，因为它们是顺序容器，没有其他的条件约束，因此要根据程序员的指令用迭代器找位置。

而无序集合底层是红黑树，有严格的约束，要想插入到里面，数据结构本身就会在内部给该值找好位置，因此只需要提供要插入的东西即可。类似地，以哈希表为底层的map也是这样，插入时进行hash运算，已经无形之中确定好了要插入的位置了。因此不需要提供迭代器。
## 遍历

```cpp
int main()
{
    auto it1 = set1.begin();
    for(; it1 != set1.end(); ++it1)
    {
        cout << *it1 << " ";
    }
    cout << endl;
}
```

for-each语句进行遍历输出：（实际上，for-each的底层就是用迭代器进行的）
```cpp
int main()
{
    for(int v : set1)
    {
        cout << v << " ";
    }
    cout << endl;
}
```
## 删除

删除某值的元素。

```cpp
int main()
{
    for(auto it1 = set1.begin(); it1 != set1.end(); ++it1)
    {
        if(*it1 == 30)
        {
            // 记得更新迭代器，因为如果不更新的话，it1将会失效
            it1 = set1.erase(it1);
        }
    }
}
```
如果是multiset，则有可能不止一个此值的元素，需要更新迭代器。千万要注意迭代器的`++`操作不再像之前一样无脑地在for的第三个位置`++`，而是下移到else的情况。
```cpp
int main()
{
    // set2 为 multiset
    for(auto it2 = set2.begin(); it2 != set2.end(); )
    {
        if(*it2 == 30)
        {
            // 记得更新迭代器，因为如果不更新的话，it1将会失效
            it2 = set2.erase(it1);
        }
        else
        {
            ++it2;
        }
    }
}
```
## find

去找指定值的元素，存在则返回其迭代器，不存在则返回`end()`
```cpp
int main()
{
    it1 = set1.find(20);
    if(it1 != set1.end())
    {
        set1.erase(it1);
    }
}
```
# unordered_map

```cpp
#include<unordered_map>
int main()
{
    unordered_map<int, string> map1;
    map1.insert(1000, "张三");
    map1.insert(make_pair(1001, "李四"));
    // C++11 的便捷方式：括号对
    map1.insert({1002, "王五"});
    map1.insert({1000, "老六"});
    
    cout << map1.size() << endl;
}
// 结果为3，因为老六的1000号重复了
```
## `operator[]`

访问或**插入**数据元素。返回一个对应键值对中的值的引用。
```cpp
int main()
{
    cout << map1[1000] << endl;
    map1[1000] = "张四";
}
```
如果访问一个不存在的key：
则创建一个键值对，key为访问的值，value用对应的数据结构的默认构造进行构造。同时，返回一个对应键值对中的值的引用。
```cpp
int main()
{
    cout << map1.size() << endl;
    map1[2000];
    cout << map1.size() << endl;
}
// 结果是[]访问2000后，size会在之前的基础上加1。
```
## 删除：erase

```cpp
int main()
{
    map1.erase(1002); // {1002, "王五"} 被删除
}
```
## find

```cpp
int main()
{
    auto it1 = map1.find(1001);
    if(it1 != map1.end())
    {
        // 如果找到了，则返回的是一个pair对象
        cout << "key: " << it1->first << ", value: " << it1->second << endl;
    }
}
```
## 遍历

```cpp
int main()
{
    for(cosnt pair<int, string> &p : map1)
    {
        //...
    }
}
```
for-each方法：
```cpp
int main()
{
    auto it = map1.begin();
    for(; it != map1.end(); ++it)
    {
        // ...
    }
}
```
# 体验无序关联容器（哈希表）的查重能力

```cpp
int main()
{
    const int ARR_LEN = 100000;
    int arr[ARR_LEN] = { 0 };
    for(int i = 0; i < ARR_LEN; ++i)
    {
        arr[i] = rand() % 10000 + 1;
    }

    unordered_map<int, int> map1;
    for(int key : arr)
    {
        auto it = map1.find(key);
        if(it == map1.end())
        {
            map1.insert({key, 1});
        }
        else
        {
            ++it->second;
        }

        //其实以上这些东西可以用一句话来代替：
        //++map1[key];
    }
    for(const pair<int, intg> &p : map1)
    {
        if(p.second > 1)
        {
            cout << "key: " << p.first << "重复" << p.second << "次" << endl;
        }
    }
}
```
在上面的10万个整数中，统计哪些数字重复了，并且统计数字重复的次数。
用数组找效率是很低的。则用哈希表。
## 去重、打印

```cpp
int main()
{
    // arr是有重复数字的数组
    unordered_set<int> set;
    for(int v : arr)
    {
        set.insert(v);
    }
    for(itn v : set)
    {
        cout << v << " ";
    }
    cout << endl;
}
```
# set
有序的map、set的底层都是红黑树。
set相当于只有键，没有值的map。
```cpp
int main()
{
    std::set<int> my_set{1, 2, 3, 4, 5, 5, 3};
    for (auto&& v : my_set)
    {
        std::cout << v << std::endl;
    }
}
```
输出1 2 3 4 5
## 插入自定义对象

由于是有序容器，所以类对象必须可以做到可比较。因此必须重载比较运算符。
而具体地，如果不重载运算符，直接插入，报错：
```
error C2676: 二进制“<”:“const _Ty”不定义该运算符或到预定义运算符可接收的类型的转换
```
意味着，插入有序容器，必须要重载`<`运算符，且只能是`<`，经测试，只重载`>`仍报错，所以至少要重载`<`运算符。

```cpp
class Student
{
public:
    Student(int id, string name)
        :_id(id = 0), _name(name = "") {}
    bool operator<(const Student &stu) const
    {
        return _id < stu._id;
    }
private:
    int _id;
    string _name;
    friend ostream &operator<<(ostream &out, const Student &stu);
};
ostream& operator<<(ostream& out, const Student& stu)
{
    out << "id: " << stu._id << " name: " << stu._name;
    return out;
}
int main()
{
    set<Student> set1;
    set1.insert(Student(1000, "zhangsan"));
    set1.insert(Student(1020, "lisi"));
}
```
## 遍历
```cpp
#include<set>
int main()
{
    set<int> set1;
    for(int i = 0; i < 20; ++i)
    {
        set1.insert(rand() % 20 + 1);
    }
    for(int v : set1)
    {
        cout << v << " ";
    }
    cout << endl;
}
//将会按序从小到大输出内容：1 2 3 5 6 8 10 12 15 16 17 19
```
实际的过程是对该红黑树进行中序遍历。
### 迭代器遍历

```cpp
int main()
{
    for(auto it = set1.begin(); it != set1.end(); ++it)
    {
        // 元素的<<输出操作符已经被重载，所以可以直接写如此。
        cout << *it << endl;
    }
}
//输出结果：
//id: 1000 name: zhangsan
//id: 1020 name: lisi
```
# map
存的是键值对，有序，所以也称为字典。
遍历时得到的是pair（对儿），定义于`<utility>`中。
底层结构是红黑树。
## insert、遍历
insert只能接收pair或者`{ 4, 'd' }`。然后系统需要临时构造对象后插入。
```cpp
int main()
{
    std::map<unsigned, char> my_map{{1, 'a'}, {2, 'b'}, {3, 'c'}};

    std::pair<unsigned, char> pr;
    pr.first = 4;
    pr.second = 'd';
    my_map.insert(pr);
    
    for (auto it = my_map.begin(); it != my_map.end(); ++it)
    {
        std::cout << (*it).first << " " << it->second << std::endl;
    }
}
```
ranged for让遍历更简洁：
```cpp
int main()
{
    for (auto&& v : my_map)
    {
        std::cout << v.first << " " << v.second << std::endl;
    }
}
```
## 结构化绑定
`C++17`之后，甚至能把key和value分开提取：叫做结构化绑定（structured binding）
```cpp
int main()
{
    for (auto&& [v, v2] : my_map)
    {
        std::cout << v << " " << v2 << std::endl;
    }
}
```

```cpp
struct A
{
    int a;
    int a2;
    int a3;
};
A bar(void)
{
    return {1, 2, 3};
}
int main()
{
    auto&& [a, b, c] = bar();
    std::cout << a << ',' << b << ',' << c << std::endl;
}
```

```cpp
A aa{ 4, 5, 6 };
A& foo(void)
{
    return aa;
}
int main()
{
    auto&& [a, b, c] = foo();
    std::cout << a << ',' << b << ',' << c << std::endl;
    a = 9; // 外部aa中的第一个值也会变为9
}
```
### emplace插入
可以直接拿散装的参数合成pair，就地构造。不会造成多余的复制。
```cpp
int main()
{
    std::map<unsigned, char> my_map{{1, 'a'}, {2, 'b'}, {3, 'c'}};

    my_map.emplace(5, 'e');
}
```
### 在同一key重复插入
1. 如果在同一key重复插入其他的value值，不会改变原value值。应该会放弃插入。这种情况包含emplace和insert。（在`C++23`下测试）
2. 但是如果用`[]`来写入的话，是会被改的。
3. `insert_or_assign`是插入或修改。如果没有该key则插入，如果存在该key则改变。
```cpp
int main()
{
    std::map<unsigned, char> my_map{{1, 'a'}, {2, 'b'}, {3, 'c'}};

    my_map.emplace(3, 'e');
    my_map.emplace(3, 'f');  // 不会把e改为f
    my_map.insert({3, 'g'}); // 不会改为g
    my_map[3] = 'h';         // 会改为h
    my_map.insert_or_assign(3, 'i'); // 会改为i
}
```
## 查询
### `std::find`
如果清楚key、value，则可以按照具体的pair进行find。pair必须写明键值类型，否则可能会产生二义性无法推断明确的类型。
```cpp
int main()
{
    std::map<unsigned, char> my_map( {1, 'a'}, {2, 'b'}, {3, 'c'});
    std::pair<unsigned, char> pr = {3, 'c'};
    auto it = std::find(my_map.begin(), my_map.end(), pr);
}
```
如果不记得value值。用find_if。因为map的键是唯一的，只能对应一个值。
```cpp
int main()
{
    std::map<unsigned, char> my_map( {1, 'a'}, {2, 'b'}, {3, 'c'});
    auto it = std::find_if(
        my_map.begin(),
        my_map.end(),
        []() -> bool
        {
            return v.first == 3;
        });
}
```
### `map::find`
这是map自带的find，效率更高，因为利用了自身红黑树的特性。
```cpp
int main()
{
    std::map<unsigned, char> my_map( {1, 'a'}, {2, 'b'}, {3, 'c'});
    auto it = my_map.find(3);
    if (it != my_map.end())
    {
        std::cout << it->second << std::endl;
    }
```
### `[]`
可用`[]`查询，但是有副作用，如果查询一个不存在的key会默认构造一个键值对。
```cpp
int main()
{
    std::map<unsigned, char> my_map( {1, 'a'}, {2, 'b'}, {3, 'c'});
    std::cout << my_map[30] << std::endl;
}
```
## 插入需要基于Compare
map插入是需要对比key的大小关系的，对于有默认大小关系的可以自然插入。
但对于自定义类，就需要添加一个仿函数作为比较器了。仿函数中的`()`记得加上const修饰。只需要实现一个`<`关系即可。
```cpp
class A
{
public:
    A(unsigned val) : _val{ val } {}
    unsigned _val;
};
class KeyComp
{
public:
    bool operator (A const& left, A const& right) const
    {
        return left._val < right._val;
    }
};
int main()
{
    std::map<A, char, KeyComp> my_map{ {{1}, 'a'}, {{2}, 'b'}, {{3}, 'c'} };
    return 0;
}
```
除了使用仿函数作为比较器，也可以直接在自定义类中定义`<`函数。
```cpp
class A
{
public:
    A(unsigned val) : _val{ val } {}
    bool operator() (A const& other) const
    {
        return _val < other._val;
    }
    unsigned _val;
};
int main()
{
    std::map<A, char> my_map{ {{1}, 'a'}, {{2}, 'b'}, {{3}, 'c'} };
    return 0;
}
```
# multimap
一个键可以对应多个value。
如果我们使用的是普通map，可以用list作为值类型，嵌套的容器，来实现一键多值。
```cpp
int main()
{
    std::map<unsigned, std::list<char>> my_container{
        {1u, {'a', 'b', 'c'}},
        {2u, {'d', 'e', 'f'}},
        {3u, {'g', 'h', 'i'}},
        {4u, {'j', 'k', 'l'}} };
    for (auto&& v : my_container)
    {
        std::cout << "Key: " << v.first << " values: ";
        for (auto&& v2 : v.second)
        {
            std::cout << v2 << " ";
        }
        std::cout << stdd::endl;
    }
    
}
```
输出：
```
key: 1 values: a b c
key: 2 values: d e f
key: 3 values: g h i
key: 4 values: j k l
```
如果用multimap，将会更简单：
```cpp
int main()
{
    std::multimap<unsigned, char> multi_map{
        {1u, 'a'}, {1u, 'b'}, {1u, 'c'},
        {2u, 'd'}, {2u, 'e'}, {2u, 'f'},
        {3u, 'g'}, {3u, 'h'}, {3u, 'i'},
        {4u, 'j'}, {4u, 'k'}, {4u, 'l'} };
    for(auto&& v : multi_map)
    {
        std::cout << v.first << " " << v.second << std::endl;
    }
}
```
## 查询：equal_range
会返回一个pair，pair的first是查询到的对应key的第一个键值对的迭代器，second是符合要求的键值对的end迭代器。
同一个key对应的多个键值对是按顺序的。所以可以用`++it`遍历。
```cpp
int main()
{
    std::multimap<unsigned, char> multi_map{ /* ... */ };
    auto it_pair = multi_map.equal_range(3u);
    for(auto it = it_pair.first; it != it_pair.second; ++it)
    {
        std::cout << it->first << " " << it->second << std::endl;
    }
}
```
输出：
```
3 g
3 h
3 i
```
用structured binding：
```cpp
int main()
{
    std::multimap<unsigned, char> multi_map{ /* ... */ };
    auto && [it, end] = multi_map.equal_range(3u);
    for(; it != end; ++it)
    {
        std::cout << it->first << " " << it->second << std::endl;
    }
}
```
## lower_bound
返回的是参数key对应的键值对中位置最小的迭代器。比如查的是3u，则返回`{3u, g}`对应的迭代器。
```cpp
int main()
{
    std::multimap<unsigned, char> multi_map{ /* ... */ };
    auto lower_it = multi_map.lower_bound(3u);
    for (auto it = multi_map.begin(); it != lower_it; ++it)
    {
        std::cout << it->first << " " << it->second << std::endl;
    }
}
```
输出
```
1 a
1 b
1 c
2 d
2 e
2 f
```
对应的还有upper_bound。比如查的是3u，则返回`{4u, j}`对应的迭代器。即3u所有键值对末尾的下一个迭代器。
```cpp
int main()
{
    std::multimap<unsigned, char> multi_map{ /* ... */ };
    auto upper_it = multi_map.upper_bound(3u);
    // 从
    for (; upper_it != multi_map.end(); ++upper_it)
    {
        std::cout << it->first << " " << it->second << std::endl;
    }
}
```
输出
```
4 j
4 k
4 l
```