---
title: Cpp_封装队/栈+单例模式
categories:
  - - Cpp
tags: 
date: 2021/10/19
updated: 
comments: 
published:
---
# node.h

```cpp
//node.h
#ifndef NODE_H
#define NODE_H
#include<stdlib.h>
using namespace std;
class List;
void show(List& list);
class Node
{
public:
	Node(int val = int())//给函数形参默认值
	{
		_val = val;
		_next = NULL;
		_pre = NULL;
	}
	friend class List;
	friend void show(List& list);
private:
	int _val;
	Node* _next;
	Node* _pre;
};
#endif
```

# list

## list.h

```cpp
#include"node.h"
using namespace std;
class List
{
public:
	List();
	List(const List& src);
	~List();
	List& operator=(const List& src);
	void push_back(int val);
	void pop_back();
	void push_front(int val);
	void pop_front(int *pval);

	bool get_back(int * pval);
	bool get_front(int* pval);
	bool is_empty();
	int size();
	//void show();
	friend void show(List& list);
private:
	class Node
	{
	public:
		Node(int val = int())//给函数形参默认值
		{
			_val = val;
			_next = NULL;
			_pre = NULL;
		}
		friend class List;
		friend void show(List& list);
	private:
		int _val;
		Node* _next;
		Node* _pre;
	};
	Node* _head;
	Node* _tail;

};
```



## list.cpp

```cpp
#include"list.h"
#include<stdio.h>
/*
类内实现的成员方法默认会被建议为内联（默认加上
对于大类更多时候在类外实现，为了人看代码的方便、清晰
*/
List::List()
{
	_head = new Node();
	_tail = _head;
}
List::List(const List& src)
{
	//申请新结点
	_head = new Node();
	_tail = _head;
	//遍历src链表，将每个数据push_back插入到新链表
	Node* tmp = src._head->_next;//next是私有的，只能加友元
	while (NULL != tmp)
	{
		push_back(tmp->_val);
		tmp = tmp->_next;
	}
	
}
List::~List()
{
	while (!is_empty())
	{
		pop_back();
	}
	//删除头结点
	delete _head;
}
List& List::operator=(const List& src)
{
	if (this == &src)
	{
		return *this;
	}
	//1.清空目标链表。2.防止内存泄露
	while (!is_empty())
	{
		pop_back();
	}

	//拷贝
	Node* tmp = src._head->_next;
	while (NULL != tmp)
	{
		push_back(tmp->_val);
		tmp = tmp->_next;
	}
	return *this;
}
void List::push_back(int val)
{
	Node* node = new Node(val);
	_tail->_next = node;
	node->_pre = _tail;
	_tail = node;
}
void List::pop_back()
{
	if (is_empty())
	{
		return;
	}
	_tail = _tail->_pre;
	delete _tail->_next;
	_tail->_next = NULL;
}
void List::push_front(int val)
{
	
	Node* newnode = new Node(val);//构造好时的节点的next和pre已经为NULL了
	newnode->_pre = _head;
	//空链表的情况，即_head->_next为NULL
	if (!is_empty())
	{
		newnode->_next = _head->_next;//没必要，构造好时的节点的next和pre已经为NULL了
		_head->_next->_pre = newnode;//NULL就异常了
	}
	else
	{
		_tail = newnode;
	}
	_head->_next = newnode;
	
}
void List::pop_front(int *pval)
{
	if (is_empty())
	{
		return;
	}
	Node* tmp = _head->_next;
	if (NULL != tmp->_next)
	{
		tmp->_next->_pre = _head;
	}
	else
	{
		//如果删除的是当前链表的唯一实际节点
		_tail = _head;
	}
	_head->_next = tmp->_next;
	*pval = tmp->_val;
	delete tmp;
}

bool List::get_back(int* pval)
{
	if (is_empty())
	{
		return false;
	}
	*pval = _tail->_val;
	return true;
}
bool List::get_front(int* pval)
{
	if (is_empty())
	{
		return false;
	}
	*pval = _head->_next->_val;
	return true;
}
bool List::is_empty()
{
	return _head->_next == NULL;
}
int List::size()
{
	int i = 0;
	Node* p = _head->_next;
	while (p != NULL)
	{
		i++;
		p = p->_next;
	}
	return i;
}
void show(List& list)
{

	List::Node* p = list._head->_next;
	while (p != NULL)
	{
		printf("%d ", p->_val);
		p = p->_next;
	}
	printf("\n");
}
```

# stack

## stack.h

```cpp
#include"list.h"
#include<mutex>
class Stack
{
public:
	static Stack* get_stack();
	
	//~Stack();
	//Stack(const Stack& src);
	//Stack& operator=(const Stack& src);
	void push(int val);
	void pop(int* pval);
	int top();
	bool is_empty();
	friend void show(Stack& stack);
private:
	Stack();
	List _list;
	static mutex* _lock;
	static Stack* _stack;
};
```

## stack.cpp

```cpp
#include"stack.h"
#include<stdio.h>
Stack* Stack::_stack = NULL;
mutex* Stack::_lock = new mutex();
Stack* Stack::get_stack()
{
	if (NULL == _stack)
	{
		_lock->lock();
		if (NULL == _stack)
		{
			_stack = new Stack();
		}
		_lock->unlock();
	}
	return _stack;
}
Stack::Stack()
	:_list()
{

}
/*
Stack::~Stack()
{

}
Stack::Stack(const Stack& src)
	:_list(src._list)
{
	
}
Stack& Stack::operator=(const Stack& src)
{
	if (this == &src)
	{
		return *this;
	}
	_list.operator=(src._list);
	return *this;
}
*/
void Stack::push(int val)
{
	Stack::_list.push_front(val);
}
void Stack::pop(int* pval)
{
	Stack::_list.pop_front(pval);
}
int Stack::top()
{
	int val;
	_list.get_back(&val);
	return val;
}
bool Stack::is_empty()
{
	return _list.is_empty();
}
void show(Stack& stack)
{
	show(stack._list);
}
```

# queue

## queue.h

```cpp
#ifndef QUEUE_H
#define QUEUE_H

#include"list.h"
#include<mutex>
class Queue
{
public:
	static Queue* get_Queue();
	//~Queue();
	//Queue(const Queue& src);
	//Queue& operator=(const Queue& src);
	void push(int val);
	void pop(int* pval);
	int top();
	bool is_empty();
	friend void show(Queue& queue);
private:
	Queue();
	List _list;
	static mutex* _lock;
	static Queue* _queue;
};

#endif // !QUEUE_H
```

## queue.cpp

```cpp
#include"queue.h"
#include<stdio.h>
Queue* Queue::get_Queue()
{
	if (NULL == _queue)
	{
		_lock->lock();
		if (NULL == _queue)
		{
			_queue = new Queue();
		}
		_lock->unlock();
	}
	return _queue;
}
Queue::Queue()
	:_list()
{
}
/*
Queue::~Queue(){}
Queue::Queue(const Queue& src)
	:_list(src._list)
{

}

Queue& Queue::operator=(const Queue& src)
{
	if (this == &src)
	{
		return *this;
	}
	_list.operator=(src._list);
	return *this;
}
*/
Queue* Queue::_queue = NULL;
mutex* Queue::_lock = new mutex();
void Queue::push(int val)
{
	Queue::_list.push_back(val);
}
void Queue::pop(int* pval)
{
	Queue::_list.pop_front(pval);
}
int Queue::top()
{
	int val;
	_list.get_front(&val);
	return val;
}
bool Queue::is_empty()
{
	return _list.is_empty();
}
void show(Queue& queue)
{
	show(queue._list);
}
```

# main

```cpp
//test
#if 0
#include"queue.h"
int main()
{

	Queue* queue1 = Queue::get_Queue();

	for (int i = 0;i < 5;i++)
	{
		queue1->push(i);
	}
	show(*queue1);

	int val;
	queue1->pop(&val);
	show(*queue1);
	return 0;
}
#endif


#include"stack.h"
int main()
{

	Stack * stack1 = Stack::get_stack();
	for (int i = 0;i < 5;i++)
	{
		stack1->push(i);
	}
	show(*stack1);

	int val;
	stack1->pop(&val);
	show(*stack1);
	return 0;
}
#if 0 
#endif
#if 0
#include"list.h"
int main()
{

	List list1;
	for (int i = 0;i < 5;i++)
	{
		list1.push_back(i);
	}
	for (int i = 0;i < 5;i++)
	{
		list1.push_front(i);
	}
	show(list1);

	List list2 = list1;
	show(list2);

	list1.pop_back();
	show(list1);
	show(list2);
	return 0;
}
#endif
```

