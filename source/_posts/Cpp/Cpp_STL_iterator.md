---
title: Cpp_STL_iterator
categories:
  - - Cpp
    - STL
tags: 
date: 2022/3/7
updated: 
comments: 
published:
---

# 内容

1. iterator的分类
2. iterator的属性
3. advance--使迭代器移动n步
4. 针对迭代器的类型萃取
5. SGI萃取的写法
6. distance--计算两迭代器之间的距离

# iterator

```c++
#pragma once
namespace xcg
{
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
using ptrdiff_t = int;

template<class _C, class _Ty, class _D = ptrdiff_t, class _Pointer = _Ty*,
	class _Reference = _Ty&>
	struct iterator
{
	typedef _C			iterator_category;
	typedef _Ty			value_type;
	typedef _D			difference_type;
	typedef _Pointer	pointer;
	typedef _Reference	reference;
};
template<class _Ty, class _D = ptrdiff_t>
struct _Forit : public iterator<forward_iterator_tag, _Ty, _D> {};
template<class _Ty, class _D = ptrdiff_t>
struct _Bidit : public iterator<bidirectional_iterator_tag, _Ty, _D> {};
template<class _Ty, class _D = ptrdiff_t>
struct _Ranit : public iterator<random_access_iterator_tag, _Ty, _D> {};

template<class _II, class _D>
inline void __advance(_II& i, _D n, input_iterator_tag)
{
	while (n--) ++i;
}
template<class _BI, class _D>
inline void __advance(_BI& i, _D n, bidirectional_iterator_tag)
{
	if (n >= 0)
	{
		while (n--) ++i;
	}
	else
	{
		while (n++) --i;
	}
}
template<class _RAI, class _D>
inline void __advance(_RAI& i, _D n, input_iterator_tag)
{
	i += n;
}
template<class _II, class _D>
inline void advance(_II& i, _D n)
{

}
}
```

```c++
template<class _Iterator>
struct iterator_traits
{
    iterator_traits() {}
	typedef typename _Iterator::iterator_category	iterator_category;
	typedef typename _Iterator::value_type			value_type;
	typedef typename _Iterator::difference_type		difference_type;
	typedef typename _Iterator::pointer				pointer;
	typedef typename _Iterator::reference			reference;
};
```

## advance

```c++
inline void advance(_II& i, _D n)
{
	iterator_traits<_II>();
	typedef typename iterator_traits<_II>::iterator_category cate;
	__advance(i, n, cate());
}
```

## 萃取器

首先是针对泛型iterator，其是标准的迭代器，重载了\*和->运算符。这里的萃取器可容纳的不包括裸指针。

```c++
template<class _Iterator>
struct iterator_traits
{
    iterator_traits() {}
	typedef typename _Iterator::iterator_category	iterator_category;
	typedef typename _Iterator::value_type			value_type;
	typedef typename _Iterator::difference_type		difference_type;
	typedef typename _Iterator::pointer				pointer;
	typedef typename _Iterator::reference			reference;
};
```

接下来是针对裸指针作出的模板特化。裸指针可看作是可随机访问的迭代器。

```c++
template<class T>
struct iterator_traits<T*>
{
	iterator_traits() {}
	typedef typename random_access_iterator_tag		iterator_category;
	typedef typename T		value_type;
	typedef typename int	difference_type;
	typedef typename T*		pointer;
	typedef typename T&		reference;
};
```

接着再次对裸指针的萃取器模板作出部分特化，可处理常裸指针。

```c++
template<class T>
struct iterator_traits<const T*>
{
	iterator_traits() {}
	typedef typename random_access_iterator_tag		iterator_category;
	typedef typename T				value_type;
	typedef typename int			difference_type;
	typedef typename const T*		pointer;
	typedef typename const T&		reference;
};
```

## 至此代码版本v1

```c++
#pragma once
namespace xcg
{
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag : public input_iterator_tag {};
struct bidirectional_iterator_tag : public forward_iterator_tag {};
struct random_access_iterator_tag : public bidirectional_iterator_tag {};
using ptrdiff_t = int;

template<class _C, class _Ty, class _D = ptrdiff_t, class _Pointer = _Ty*,
	class _Reference = _Ty&>
	struct iterator
{
	typedef _C			iterator_category;
	typedef _Ty			value_type;
	typedef _D			difference_type;
	typedef _Pointer	pointer;
	typedef _Reference	reference;
};

template<class _Iterator>
struct iterator_traits
{
	iterator_traits() {}
	typedef typename _Iterator::iterator_category	iterator_category;
	typedef typename _Iterator::value_type			value_type;
	typedef typename _Iterator::difference_type		difference_type;
	typedef typename _Iterator::pointer				pointer;
	typedef typename _Iterator::reference			reference;
};

template<class T>
struct iterator_traits<T*>
{
	iterator_traits() {}
	typedef typename random_access_iterator_tag		iterator_category;
	typedef typename T		value_type;
	typedef typename int	difference_type;
	typedef typename T*		pointer;
	typedef typename T&		reference;
};
template<class T>
struct iterator_traits<const T*>
{
	iterator_traits() {}
	typedef typename random_access_iterator_tag		iterator_category;
	typedef typename T				value_type;
	typedef typename int			difference_type;
	typedef typename const T*		pointer;
	typedef typename const T&		reference;
};

template<class _Ty, class _D = ptrdiff_t>
struct _Forit : public iterator<forward_iterator_tag, _Ty, _D> {};
template<class _Ty, class _D = ptrdiff_t>
struct _Bidit : public iterator<bidirectional_iterator_tag, _Ty, _D> {};
template<class _Ty, class _D = ptrdiff_t>
struct _Ranit : public iterator<random_access_iterator_tag, _Ty, _D> {};

template<class _II, class _D>
inline void __advance(_II& i, _D n, input_iterator_tag)
{
	while (n--) ++i;
}
template<class _BI, class _D>
inline void __advance(_BI& i, _D n, bidirectional_iterator_tag)
{
	if (n >= 0)
	{
		while (n--) ++i;
	}
	else
	{
		while (n++) --i;
	}
}
template<class _RAI, class _D>
inline void __advance(_RAI& i, _D n, input_iterator_tag)
{
	i += n;
}
template<class _II, class _D>
inline void advance(_II& i, _D n)
{
	iterator_traits<_II>();
	typedef typename iterator_traits<_II>::iterator_category cate;
	__advance(i, n, cate());
}
}
```

## SGI的做法

```c++
//以下三个函数是SGI比较重要的写法模式
template<class _II>
inline typename iterator_traits<_II>::iterator_category
iterator_category(const _II&)
{
	typedef typename iterator_traits<_II>::iterator_category cate;
	return cate();
}
template<class _II>
inline typename iterator_traits<_II>::value_type*
value_type(const _II&)
{
	return static_cast<typename iterator_traits<_II>::value_type*>(0);
}
template<class _II>
inline typename iterator_traits<_II>::difference_type*
difference_type(const _II &)
{
	return static_cast<typename iterator_traits<_II>::difference_type*>(0);
}
```

```c++
template<class _II, class _D>
inline void advance(_II& i, _D n)
{
    //这里是SGI的写法，使代码更加灵活。
	__advance(i, n, iterator_category(i));
}
```

## distance

```c++
template<class _II>
inline typename iterator_traits<_II>::difference_type
__distance(_II _F, _II _L, input_iterator_tag)
{
	typename iterator_traits<_II>::difference_type n = 0;
	while (_F != _L)
	{
		++_F;
		++n;
	}
	return n;
}
template<class _RAI>
inline typename iterator_traits<_RAI>::difference_type
__distance(_RAI _F, _RAI _L, input_iterator_tag)
{
	typename iterator_traits<_RAI>::difference_type n = 0;
	return _L - _F;
}

template<class _II>
inline typename iterator_traits<_II>::difference_type
distance(_II _F, _II _L, input_iterator_tag)
{
	return __distance(_F, _L, iterator_category(_F));
}
```

