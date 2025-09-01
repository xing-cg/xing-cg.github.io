---
title: Cpp_右值引用_move的实现
categories:
  - - Cpp
    - 右值引用
tags: 
date: 2022/3/13
updated: 
comments: 
published:
---
# 内容

1. 右值引用的例子
2. 右值引用的实现

# 右值引用的例子

```c++
using namespace std;
class MyString
{
    char* str;
public:
    MyString(const char* p = NULL) : str(NULL)
    {
        if(p != NULL)
        {
            int len = strlen(p) + 1;
            str = new char[len];
            strcpy_s(str, len, p);
        }
    }
    MyString(const MyString & src) : str(NULL)
    {
        if(src.str != NULL)
        {
            int len = strlen(src.str) + 1;
            str = new char[len];
            strcpy_s(str, len, src.str);
        }
    }
    MyString(MyString && src) : str(src.str)
    {
        src.str = NULL;
    }
    ~MyString()
    {
        if(str != NULL)
        {
            delete[]str;
        }
        str = NULL;
    }
}
int main()
{
    MyString s1("xcgong");
    MyString s2(s1);
    MyString s3(std::move(s1));	//实质上是把s1“强转”为 右值引用 才能进行移动构造
    
    
}
```

# move的实现

```c++
template<class _Ty>
struct my_remove_reference
{
    using type = _Ty;
    using _Const_thru_ref_type = const _Ty;
}
template<class _Ty>
struct my_remove_reference<_Ty&>
{
    using type = _Ty;
    using _Const_thru_ref_type = const _Ty &;
}
template<class _Ty>
struct my_remove_reference<_Ty&&>
{
    using type = _Ty;
    using _Const_thru_ref_type = const _Ty &&;
}

template<class _Ty>
using my_remove_reference_t = typename my_remove_reference<_Ty>::type;
//my_remove_reference_t<Object> --> my_remove_reference<Object>::type;
template<class _Ty>
my_remove_reference_t<_Ty>&& my_move(_Ty && _Arg)
{
    								//强转为_Ty纯粹类型的右值引用形式
    return static_cast<my_remove_reference_t<_Ty> &&>(_Arg);
}
int main()
{
    MyString s1("xcgong");
    MyString s2(xcg::my_move(s1));
/*
	my_move  ->
	    return static_cast<my_remove_reference_t<_Ty> &&>(_Arg);
			my_remove_reference_t<_Ty>  ->
				my_remove_reference<_Ty>::type  ->
					my_remove_reference<MyString>::type  ->
						type == _Ty -> "MyString"
	--> return static_cast<MyString &&>(_Arg);
	效果: _Arg 由 MyString 转为 MyString &&	
则--MyString s2(xcg::my_move(s1))--等效于:
	MyString s2( (MyString &&)s1);
		调用：MyString(MyString && src), 即移动构造。进行资源拥有权的转移
*/
}
```

# 引用类型与模板结合

可测试`is_lvalue_reference`，这是个模板类，定义于`<type_traits>`中。
```cpp
#include <iostream>
#include <type_traits>
 
class A {};
 
int main()
{
    std::cout << std::boolalpha;
    std::cout << std::is_lvalue_reference<A>::value << '\n';
    std::cout << std::is_lvalue_reference<A&>::value << '\n';
    std::cout << std::is_lvalue_reference<A&&>::value << '\n';
    std::cout << std::is_lvalue_reference<int>::value << '\n';
    std::cout << std::is_lvalue_reference<int&>::value << '\n';
    std::cout << std::is_lvalue_reference<int&&>::value << '\n';
}
/* Output:
    false
    true
    false
    false
    true
    false
*/
```
可测试`is_rvalue_reference`，这是个模板类，定义于`<type_traits>`中。
```cpp
#include <type_traits>
#include <iostream>
 
class A {};
 
static_assert
(
    std::is_rvalue_reference_v<A> == false and
    std::is_rvalue_reference_v<A&> == false and
    std::is_rvalue_reference_v<A&&> != false and
    std::is_rvalue_reference_v<char> == false and
    std::is_rvalue_reference_v<char&> == false and
    std::is_rvalue_reference_v<char&&> != false
);
 
 
 
template <typename T>
void test(T&& x)
{
    static_assert(std::is_same_v<T&&, decltype(x)>);
    std::cout << "T\t" << std::is_rvalue_reference<T>::value << '\n';
    std::cout << "T&&\t" << std::is_rvalue_reference<T&&>::value << '\n';
    std::cout << "decltype(x)\t" << std::is_rvalue_reference<decltype(x)>::value << '\n';
}
 
int main()
{
    std::cout << std::boolalpha;
    std::cout << "A\t" << std::is_rvalue_reference<A>::value << '\n';
    std::cout << "A&\t" << std::is_rvalue_reference<A&>::value << '\n';
    std::cout << "A&&\t" << std::is_rvalue_reference<A&&>::value << '\n';
    std::cout << "char\t" << std::is_rvalue_reference<char>::value << '\n';
    std::cout << "char&\t" << std::is_rvalue_reference<char&>::value << '\n';
    std::cout << "char&&\t" << std::is_rvalue_reference<char&&>::value << '\n';
 
    std::cout << "\ntest(42)\n";
    test(42);
 
    std::cout << "\ntest(x)\n";
    int x = 42;
    test(x);
}
/*
    A	false
    A&	false
    A&&	true
    char	false
    char&	false
    char&&	true
     
    test(42)
    T	false
    T&&	true
    decltype(x)	true
     
    test(x)
    T	false
    T&&	false
    decltype(x)	false
*/
```

```cpp
template<class T>
void fac(T &&a)
{
    
}
```