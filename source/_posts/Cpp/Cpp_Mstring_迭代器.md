---
title: Cpp_Mstring_迭代器
categories:
  - - Cpp
tags: 
date: 2021/10/28
updated: 
comments: 
published:
---
# 迭代器

可以理解为，指向容器内部数据的一个指针。本质是对象。

```cpp
#include<vector>
int main()
{
    vector<int> v1;
    for(int i = 0;i<10;i++)
    {
        v1.push_back(i);
    }
    vector<int>::iterator it1 = v1.begin();
    for(;it1!=v1.end();it1++)
    {
        cout << 
    }
}
```

# Mstring_iterator

```cpp
//mstring_iterator.h
//begin end insert erase
#ifndef MSTRING_ITERATOR_H
#define MSTRING_ITERATOR_H
class Mstring;

class Mstring_iterator
{
public:
	Mstring_iterator(Mstring& mstr, int pos)
        :_mstr(mstr), _pos(_pos)
    {
        
    }
    Mstring_iterator(const Mstring_iterator& src)
        :_mstr(src._mstr), _pos(src._pos)
    {
        
    }
    Mstring_iterator& operator=(const Mstring_iterator& src)
    {
        if(&_mstr != &src._mstr)
        {
            return *this;
        }
        _pos = src._pos;
        return *this;
    }
    
    bool operator !=(const Mstring_iterator& src)
    {
        if(&_mstr != &src._mstr || _pos != src._pos)//不是一个字符串或没有指向同一位置
        {
            return true;
        }
        return false;
    }
    bool operator ==(const Mstring_iterator& src)
    {
        if(&_mstr == &src._mstr && _pos == src._pos)//不是一个字符串或没有指向同一位置
        {
            return true;
        }
        return false;
    }
    Mstring_iterator& operator++()
    {
        _pos++;
        return *this;
    }
    Mstring_iterator operator++(int)
    {
        int pos = _pos;
        _pos++;
        return Mstring_iterator(_mstr,pos);
    }

    Mstring_iterator& operator--()
    {
        _pos--;
        return *this;
    }
    Mstring_iterator operator--(int)
    {
        int pos = _pos;
        _pos--;
        return Mstring_iterator(_mstr,pos);
    }

    char& operator*();//由于[]重载在本文件不可见，则在这里实现不可编译，我们可以在mstring.cpp进行类外实现。
private:
    Mstring& _mstr;
    int _pos;
}
```

# Mstring

```cpp
//mstring.h
#ifndef MSTRING_H
#define MSTRING_H
#include<iostream>
#include"mstring_iterator.h"
#include<mutex>
using namespace std;

#define MSTRING_MORE_SIZE sizeof(int)
#define DEFEALT_LEN (10+MSTRING_MORE_SIZE)

class Mstring
{
public:
	typedef Mstring_iterator iterator;
	Mstring(const char* str = NULL);
	Mstring(const Mstring& src);
	Mstring& operator=(const Mstring& src);
	~Mstring();

	void push_back(char c);
	void pop_back();
	char back()const;
	char front()const;
	bool empty()const;
	int size()const;

	Mstring operator+(const Mstring& str)const;
	char& operator[](int pos);
	char operator[](int pos)const;

	iterator begin();
	iterator end();
	void insert(iterator it, char val);
	void erase(iterator it);

	friend ostream& operator<<(ostream& out, const Mstring& src);
	friend istream& operator>>(istream& in, Mstring& src);
private:
	bool full()const;
	void expand();
	void init_num();
	int& get_num();
	int get_num()const;
	const char* get_str_begin()const;
	char* get_str_begin();
	void write_copy();
	int down_num();//引用计数-1
	int up_num();//引用计数+1

	char* _str;//能否直接使用浅拷贝------怎么加引用计数
	int _len;//当前空间总长度
	int _val_len;//已经占用的长度，实际数据数量
	//static mutex* _lock;
};

#endif
```

# mstring.cpp

```cpp
#include"mstring.h"

mutex* Mstring::_lock = new mutex();

Mstring::Mstring(const char* str)
{
	if (NULL == str)
	{
		_len = DEFEALT_LEN;
		_val_len = 0;
		_str = new char[_len];
		memset(_str, 0, _len);
		init_num();
		return;
	}

	_val_len = strlen(str);

	//加上引用计数占的空间
	_len = _val_len + 1 + MSTRING_MORE_SIZE;
	_str = new char[_len];
	memset(_str, 0, _len);

	for (int i = 0; i < _val_len; i++)
	{
		get_str_begin()[i] = str[i];
	}
	init_num();
}

Mstring::Mstring(const Mstring& src)
{
	_val_len = src._val_len;
	_len = src._len;
	_str = src._str;
	//让引用计数+1
	up_num();
}

Mstring& Mstring::operator=(const Mstring& src)
{
	if (&src == this)
	{
		return *this;
	}
	_val_len = src._val_len;
	_len = src._len;
	_str = src._str;
	up_num();

	return *this;
}

Mstring::~Mstring()
{
	//将引用计数-1
	down_num();

	//查看当前是否还有人引用
	if (0 == get_num())
	{
		delete[]_str;
	}
}

void Mstring::push_back(char c)
{
	//判断是否一个人独有
	if (get_num() > 1)
	{
		//写时拷贝，给分配独有的空间
		write_copy();
	}

	if (full())
	{
		expand();
	}

	get_str_begin()[_val_len] = c;
	_val_len++;
}

void Mstring::pop_back()
{
	if (get_num() > 1)
	{
		write_copy();
	}

	if (empty())
	{
		return;
	}

	_val_len--;
}

char Mstring::back()const
{
	if (empty())
	{
		return 0;
	}

	return get_str_begin()[_val_len - 1];
}

char Mstring::front()const
{
	if (empty())
	{
		return 0;
	}

	return get_str_begin()[0];
}

bool Mstring::empty()const
{
	return _val_len == 0;
}

Mstring Mstring::operator+(const Mstring& str)const
{
	char* p;
	int len = _val_len + str._val_len + 1;
	p = new char[len];
	memset(p, 0, len);

	//进行数据拷贝-----将两个字符串的数据拼接起来
	int i = 0;
	for (; i < _val_len; i++)
	{
		p[i] = get_str_begin()[i];
	}

	for (int j = 0; j < str._val_len; j++, i++)
	{
		p[i] = str.get_str_begin()[j];
	}

	return p;
}

char& Mstring::operator[](int pos)
{
	if (get_num() > 1)
	{
		write_copy();
	}
	return  get_str_begin()[pos];
}

char Mstring::operator[](int pos)const
{
	return  get_str_begin()[pos];
}

bool Mstring::full()const
{
	return _val_len == _len - 1 - MSTRING_MORE_SIZE;
}

void Mstring::expand()
{
	if (get_num() > 1)
	{
		write_copy();
	}

	_len = _len << 1;

	char* p = new char[_len];
	memset(p, 0, _len);

	for(int i = 0;i<_val_len+MSTRING_MORE_SIZE;i++)
	{
		p[i] = _str[i];
	}

	if (down_num() == 0)
	{
		delete[]_str;
	}

	_str = p;
}

int Mstring::size()const
{
	return _val_len;
}

//初始化引用计数
void Mstring::init_num()
{
	get_num() = 1;
}

//获取引用计数
int& Mstring::get_num()
{
	return *((int*)_str);
}

//引用计数-1
int Mstring::down_num()
{
	_lock->lock();
	int num = --get_num();
	_lock->unlock();
	return num;
}

//引用计数+1
int Mstring::up_num()
{
	//get_lock()->lock();
	_lock->lock();
	/*
	while(1)
	{
		if(_lock->a == 0)
		{
			_lock->a = 1;
			break;
		}
		sleep(10);
	}
	
	*/
	int num = ++get_num();
	_lock->unlock();
	/*
	_lock->a = 0;
	*/

	return num;
}


//获取引用计数
int Mstring::get_num()const
{
	return *((int*)_str);
}

//获取字符串的开头指针
char* Mstring::get_str_begin()
{
	return _str + MSTRING_MORE_SIZE;
}

//获取字符串的开头指针
const char* Mstring::get_str_begin()const
{
	return _str + MSTRING_MORE_SIZE;
}

//写时拷贝
void Mstring::write_copy()
{
	char* p = new char[_len];

	//将所有的数据全部拷贝过来
	for (int i = 0; i < _len; i++)
	{
		p[i] = _str[i];
	}

	//改变原来指向的内存的引用计数
	if (0 == down_num())
	{
		delete[]_str;
	}

	//将当前引用计数改为1
	_str = p;
	init_num();
}

ostream& operator<<(ostream& out, const Mstring& src)
{
	for (int i = 0; i < src.size(); i++)
	{
		out << src.get_str_begin()[i];
	}
	out << "     :::::num:::" << src.get_num();
	out << endl;
	return out;
}

istream& operator>>(istream& in, Mstring& src)
{
	if (src.get_num() > 1)
	{
		src.write_copy();
	}

	char tmp[1024];
	in >> tmp;
	
	src = tmp;
	return in;
}

////////////////////////////////迭代器

iterator Mstring::begin()
{
    return iterator(*this, 0);
}

iterator Mstring::end()
{
    return iterator(*this, _val_len);
}
iterator Mstring::insert(iterator it, char val) 
{
    //判断是否独有
    if (get_num() > 1)
    {
        write_copy();
    }

    //扩容
    if (full())
    {
        expand();
    }

    iterator it1 = end();
    iterator it2 = it1 - 1;

    for (; it1 != it; it1--, it2--)
    {
        *it1 = *it2;
    }
    *it = val;
    _val_len++;

    return it;
}
iterator Mstring::erase(iterator it)
{
    //判断是否独有
    if (get_num() > 1)
    {
        write_copy();
    }

    //判空
    if (empty())
    {
        return it;
    }

    iterator it1 = it;
    iterator it2 = it + 1;
    for (; it2 != end(); it1++, it2++)
    {
        *it1 = *it2;
    }

    _val_len--;
    return it;
}


char& Mstring_iterator::operator*()
{
	return _mstr[_pos];
}
```

# test

```cpp
int main()
{
    Mstring s1 = "123456";
    Mstring::iterator it = s1.begin();
    for(; it != s1.end(); it++)
    {
        cout << *it << " ";
        
    }
    cout << endl;
}
```

# vector

通过迭代器进行增删改查。

# 分类

1. 顺序迭代器
    1. `iterator`是正向迭代器
    2. `reverse_iterator`是反向迭代器
    3. `const_iterator`是常量正向迭代器
    4. `const_reverse_iterator`是常量反向迭代器
2. 插入型迭代器
    1. `insert_iterator`是随机插入迭代器，内部调用的是`insert`
    2. `back_insert_iterator`是尾插迭代器，调用的是`push_back`
    3. `front_insert_iterator`是头插迭代器，调用的是`push_front`
3. 流迭代器

# 顺序迭代器

```cpp
//正向迭代打印
template<class T>
void print(T & obj)
{
	typename T::iterator it = obj.begin();
	for (; it != obj.end();it++)
	{
		cout << *it << " ";
	}
	cout << endl;
}
//反向迭代打印
template<class T>
void r_print(T& obj)
{
	typename T::reverse_iterator it = obj.rbegin();
	for (;it != obj.rend();it++)
	{
		cout << *it << " ";
	}
	cout << endl;
}
```

