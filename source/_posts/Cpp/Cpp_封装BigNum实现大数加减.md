---
title: Cpp_封装BigNum实现大数加减
typora-root-url: ../..
categories:
  - [Cpp]
tags:
  - null 
date: 2021/10/20
update:
comments:
published:
---

# bignum.h

```cpp
//bignum.h
#pragma once
#include<iostream>
using namespace std;
class BigNum
{
public:
	BigNum();
	BigNum(int);
	BigNum(const char*);
	BigNum(string);
	BigNum(const BigNum& src);
	BigNum operator=(const BigNum& src);
	BigNum operator+(const BigNum& src);
	BigNum operator-(const BigNum& src);

	BigNum& operator++();
	BigNum operator++(int);
	BigNum& operator--();
	BigNum operator--(int);

	friend ostream& operator<<(ostream& out, const BigNum& src);
	friend istream& operator>>(istream& in, BigNum& des);
	friend void show_bignum(BigNum& src);
private:
	string _num;
};

```

# bignum.cpp

```cpp
//bignum.cpp
#include"bignum.h"
#include<string>
int num_digits(int n)
{
	int i = 0;
	while (n != 0)
	{
		i++;
		n /= 10;
	}
	return i;
}
int num_string(const char* s)
{
	int i = 0;
	const char* p = s;
	while (*p != 0)
	{
		i++;
		p++;
	}
	return i;
}
void swap(char* a, char* b)
{
	char tmp = *a;
	*a = *b;
	*b = tmp;
}
void reverse(char* s, int front, int tail)
{
	for (int i = front; i < (tail - front+1) / 2;++i)
	{
		swap(&s[i], &s[tail - front - i]);
	}
	return;
}
void show_bignum(BigNum& src)
{
	const char* p = src._num.data();
	int n = src._num.length();
	for (int i = n - 1;i >= 0;i--)
	{
		printf("%c", p[i]);
	}
	printf("\n");
}
BigNum::BigNum()
	:_num()
{

}
BigNum::BigNum(int n)
{
	int i = num_digits(n);
	char* tmp = new char[i+1];
	_itoa_s(n,tmp,i+1,10);
	reverse(tmp, 0, i - 1);
	_num.append(tmp);
}
BigNum::BigNum(const char* s)
{
	int i = num_string(s);
	char* tmp = new char[i + 1];
	const char* p = s;
	i = 0;
	while (*p != 0)
	{
		tmp[i++] = *p++;
	}
	tmp[i] = 0;
	reverse(tmp, 0, i - 1);
	_num.append(tmp);
}
BigNum::BigNum(string s)
{
	int i = s.length();
	char* tmp = new char[i + 1];
	const char* p = s.data();
	i = 0;
	while (*p != 0)
	{
		tmp[i++] = *p++;
	}
	tmp[i] = 0;
	reverse(tmp, 0, i - 1);
	_num.append(tmp);
}
BigNum::BigNum(const BigNum& src)
{
	int i = src._num.length();
	char* tmp = new char[i + 1];
	const char* p = src._num.data();
	i = 0;
	while (*p != 0)
	{
		tmp[i++] = *p++;
	}
	tmp[i] = 0;
	//reverse(tmp, 0, i - 1);
	_num.append(tmp);
}
BigNum BigNum::operator=(const BigNum& src)
{
	if (this == &src)
	{
		return *this;
	}
	_num.clear();
	_num.append(src._num);
	return *this;
}
BigNum BigNum::operator+(const BigNum& src)
{
	int i = 0;
	string long_s, short_s;
	if (_num.length() >= src._num.length())
	{
		long_s = _num, short_s = src._num;
	}
	else
	{
		long_s = src._num, short_s = _num;
	}
	const char* p1 = long_s.data();
	const char* p2 = short_s.data();
	char* new_s = new char[long_s.length() + 1 + 1]();
	char* p = new_s;
	int tmp_10 = 0;
	while (*p1 != 0 && *p2 != 0)
	{
		tmp_10 = (*p1 & 15) + (*p2 & 15) + (*p & 15);
		*p = (tmp_10 % 10) | (0b00110000);
		*(p + 1) = (tmp_10 / 10) | (0b00110000);
		if (i == long_s.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p2++;
		p++;
		i++;
	}
	while (i != long_s.length())
	{
		tmp_10 = (*p & 15) + (*p1 & 15);//p1、p2为\0或有效值，不影响结果。
		*p = (tmp_10 % 10)|(0b00110000);
		*(p + 1) = (tmp_10 / 10) | (0b00110000);
		if (i == long_s.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p++;
		i++;
	}
	i = new_s[long_s.length()] == '0' ? long_s.length() : long_s.length() + 1;
	reverse(new_s, 0, i - 1);
	return string(new_s);
}
BigNum BigNum::operator-(const BigNum& src)
{
	int i = 0;
	string long_s, short_s;
	if (_num.length() >= src._num.length())
	{
		long_s = _num, short_s = src._num;
	}
	else
	{
		long_s = src._num, short_s = _num;
	}
	const char* p1 = long_s.data();
	const char* p2 = short_s.data();
	char* new_s = new char[long_s.length() + 1]();
	char* p = new_s;
	int tmp_10 = 0;
	int tmp_2 = 0;
	while (*p1 != 0 && *p2 != 0)
	{
		tmp_10 = (*p1 & 15) - (*p2 & 15) - (~(*p)+1);
		*p = ((tmp_10 + 10) % 10 ) | (0b00110000);
		*(p + 1) = ((tmp_10-9) / 10);
		if (i == long_s.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p2++;
		p++;
		i++;
	}
	while (i != long_s.length())
	{
		tmp_10 = (*p1 & 15) - (~(*p) + 1);//p1、p2为\0或有效值，不影响结果。
		*p = ((tmp_10 + 10) % 10) | (0b00110000);
		*(p + 1) = ((tmp_10 - 9) / 10);
		if (i == long_s.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p++;
		i++;
	}
	i = new_s[long_s.length()-1] == '0' ? long_s.length()-1 : long_s.length();
	new_s[i] = 0;
	reverse(new_s, 0, i - 1);
	return string(new_s);
}

BigNum& BigNum::operator++()
{
	int i = 0;
	const char* p1 = _num.data();
	char* p = new char[_num.length()+2]();
	char* new_s = p;
	int tmp = 0;

	tmp = (*p1 & 15) + ('1' & 15) + (*p & 15);
	*p = (tmp % 10) | (0b00110000);
	*(p + 1) = (tmp / 10) | (0b00110000);
	if (i == _num.length() - 1)
	{
		if (*(p + 1) == (0 | (0b00110000)))
		{
			*(p + 1) = '\0';
		}
	}
	p1++;
	p++;
	i++;

	while (i != _num.length())
	{
		tmp = (*p & 15) + (*p1 & 15);
		*p = (tmp % 10) | (0b00110000);
		*(p + 1) = (tmp / 10) | (0b00110000);
		if (i == _num.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p++;
		i++;
	}
	i = new_s[_num.length()] == '0' ? _num.length() : _num.length() + 1;
	_num.clear();
	_num.append(new_s);
	return *this;
}
BigNum BigNum::operator++(int)
{
	BigNum old = *this;
	
	int i = 0;
	const char* p1 = _num.data();
	char* p = new char[_num.length() + 2]();
	char* new_s = p;
	int tmp = 0;

	tmp = (*p1 & 15) + ('1' & 15) + (*p & 15);
	*p = (tmp % 10) | (0b00110000);
	*(p + 1) = (tmp / 10) | (0b00110000);
	if (i == _num.length() - 1)
	{
		if (*(p + 1) == (0 | (0b00110000)))
		{
			*(p + 1) = '\0';
		}
	}
	p1++;
	p++;
	i++;

	while (i != _num.length())
	{
		tmp = (*p & 15) + (*p1 & 15);
		*p = (tmp % 10) | (0b00110000);
		*(p + 1) = (tmp / 10) | (0b00110000);
		if (i == _num.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p++;
		i++;
	}
	i = new_s[_num.length()] == '0' ? _num.length() : _num.length() + 1;
	_num.clear();
	_num.append(new_s);

	return old;
}
BigNum& BigNum::operator--()
{
	int i = 0;
	const char* p1 = _num.data();
	char* new_s = new char[_num.length() + 1]();
	char* p = new_s;
	int tmp_10 = 0;
	int tmp_2 = 0;

	tmp_10 = (*p1 & 15) - ('1' & 15) - (~(*p) + 1);
	*p = ((tmp_10 + 10) % 10) | (0b00110000);
	*(p + 1) = ((tmp_10 - 9) / 10);
	if (i == _num.length() - 1)
	{
		if (*(p + 1) == (0 | (0b00110000)))
		{
			*(p + 1) = '\0';
		}
	}
	p1++;
	p++;
	i++;

	while (i != _num.length())
	{
		tmp_10 = (*p1 & 15) - (~(*p) + 1);//p1、p2为\0或有效值，不影响结果。
		*p = ((tmp_10 + 10) % 10) | (0b00110000);
		*(p + 1) = ((tmp_10 - 9) / 10);
		if (i == _num.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p++;
		i++;
	}
	i = new_s[_num.length() - 1] == '0' ? _num.length() - 1 : _num.length();
	new_s[i] = 0;
	_num.clear();
	_num.append(new_s);

	return *this;
}
BigNum BigNum::operator--(int)
{
	BigNum old = *this;

	int i = 0;
	const char* p1 = _num.data();
	char* new_s = new char[_num.length() + 1]();
	char* p = new_s;
	int tmp_10 = 0;
	int tmp_2 = 0;

	tmp_10 = (*p1 & 15) - ('1' & 15) - (~(*p) + 1);
	*p = ((tmp_10 + 10) % 10) | (0b00110000);
	*(p + 1) = ((tmp_10 - 9) / 10);
	if (i == _num.length() - 1)
	{
		if (*(p + 1) == (0 | (0b00110000)))
		{
			*(p + 1) = '\0';
		}
	}
	p1++;
	p++;
	i++;

	while (i != _num.length())
	{
		tmp_10 = (*p1 & 15) - (~(*p) + 1);//p1、p2为\0或有效值，不影响结果。
		*p = ((tmp_10 + 10) % 10) | (0b00110000);
		*(p + 1) = ((tmp_10 - 9) / 10);
		if (i == _num.length() - 1)
		{
			if (*(p + 1) == (0 | (0b00110000)))
			{
				*(p + 1) = '\0';
			}
		}
		p1++;
		p++;
		i++;
	}
	i = new_s[_num.length() - 1] == '0' ? _num.length() - 1 : _num.length();
	new_s[i] = 0;
	_num.clear();
	_num.append(new_s);

	return *this;

	return old;
}
ostream& operator<<(ostream& out, const BigNum& src)
{
	out << src._num;
	return out;
}
istream& operator>>(istream& in, BigNum& des)
{
	des._num.clear();
	string s;
	in >> s;
	int i = s.length();

	char* new_s = new char[i + 1]();

	const char* p = s.data();
	i = 0;
	while (*p != 0)
	{
		new_s[i++] = *p++;
	}

	reverse(new_s, 0, i-1);
	des._num.append(new_s);

	return in;
}
```

# test

```cpp
//test.cpp
#include"bignum.h"
#include<math.h>
#include<iostream>
using namespace std;
int main()
{
	BigNum bn1(99999);
	show_bignum(bn1);cout << "real:" << bn1 << endl;
	BigNum bn2(bn1);
	show_bignum(bn2);cout << "real:" << bn2 << endl;
	BigNum bn3(string("999"));
	show_bignum(bn3);cout << "real:" << bn3 << endl;
	BigNum bn4("456");
	show_bignum(bn4);cout << "real:" << bn4 << endl;
	bn4 = bn3;
	show_bignum(bn4);cout << "real:" << bn4 << endl;
	BigNum bn5 = bn1 + bn3;
	show_bignum(bn5);
	BigNum bn6(100003);
	BigNum bn7(411);
	BigNum bn8 = bn6 - bn7;
	show_bignum(bn8);
	BigNum bn9(999);
	++bn9;
	show_bignum(bn9);
	bn9++;
	show_bignum(bn9);
	--bn9;--bn9;
	show_bignum(bn9);
	++bn9;++bn9;
	show_bignum(bn9);
	bn9--;bn9--;
	show_bignum(bn9);
	BigNum bn10;
	cin >> bn10;
	show_bignum(bn10);
	cout << "real:" << bn10 << endl;
}
```

# 截图

![image-20211020232706800](../../images/Cpp%E5%B0%81%E8%A3%85BigNum%E5%AE%9E%E7%8E%B0%E5%A4%A7%E6%95%B0%E5%8A%A0%E5%87%8F/image-20211020232706800.png)

