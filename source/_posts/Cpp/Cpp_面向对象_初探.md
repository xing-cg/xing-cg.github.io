---
title: Cpp_面向对象_初探
categories:
  - - Cpp
tags:
  - tulun
date: 2021/10/7
update: 
comments: 
published:
---
# 内容

四大特性：抽象、封装、继承、多态
# 概述

1. 编程语言：是为了模拟现实而产生的
    1. 就比如要模拟person类：我们要描述人，就要抽象为数据
2. 面向对象就是一个模拟现实很好的思想
    1. 属性----成员属性：sex, age
    2. 行为----成员方法：work, eat
## this指针

this指针是指向本对象的指针。

`C++`和C的区别，很明显的是，`C++`会在编译时期暗中做很多事情，其次的区别就是`C++`封装。

就比如定义成员方法时，`C++`就在编译时期为我们加上了：
1. 在函数普通成员方法的第一个参数默认是指向本类型的this指针。（`Class_Name* const this`）
2. 在普通成员方法内，使用到普通成员的地方，加上"`this->`"。
3. 在调用普通成员方法的时候加上实参数this指针，方便方法内使用。
```c++
class Person
{
public:
    int _age;
    int _sex;
    char *_name;
    void work()
    {
        cout << _name << ": work" << endl;
    }
    void eat()
    {
        cout << _name << ": eat" << endl;
    }
};//分号不要丢
```
每一个成员方法，第一个参数一般默认在编译期时，传入一个(`Class_name * this`)。

在使用到成员方法、成员变量时，会默认加上`解引用`。
```c++
void work(Person *this)
{
    cout << this->_name << ": work" << endl;
    this->eat();
}
```
## 构造函数

构造函数就是：在对象进行构造的时候（其实就是定义，`Person p;`），默认调用的成员方法。
1. 如果没有实现构造函数，会自动生成默认构造函数
    1. 自己实现了之后，就不会自动生成。
    2. 默认构造函数就是：除了this指针以外，没有参数的构造函数。 
3. 构造函数名字与类名一致，无返回值。
4. 构造函数可以重载，参数列表不限制。
```c++
Person(int age, int sex, const char *name)
{
    _age = age;
    _sex = sex;
    // _name = name; // 这是浅拷贝：指针给指针赋值；
    
    // 以下是深拷贝，申请新的空间填充相同内容。
    _name = new char[strlen(name) + 1];
    memset(_name, 0, strlen(name) + 1);
    for(int i = 0; i < strlen(name); ++i)
    {
        _name[i] = name[i];
    }
    _name[i] = 0; 
}
```
## 析构函数

是对象生存周期满了之后（死亡），自动调用的函数。
```c++
~Person()
{
    delete[]_name;
}
```

1. 在栈桢上，先构造的后析构。
2. 如果没有实现析构函数，系统会默认生成一个默认的析构函数。函数体是空的。
3. 自己实现了之后，就不会自动生成。
## 拷贝构造函数

1. 用一个已存在的对象给另一个正在生成的对象初始化的时候自动调用的成员方法。
2. 如果没有自己实现，会自动生成一个浅拷贝的拷贝构造函数。
3. 如果自己实现，不会生成。

```c++
Person(const Person& src)
{
	_age = src._age;
    _sex = src._sex;
    //预防浅拷贝
    _name = new char[strlen(src._name) + 1];
    memset(_name, 0, strlen(_name) + 1);
    for(int i = 0;i < strlen(_name); ++i)
    {
        _name[i] = src._name[i];
    }
}
int main()
{
    char name1[] = {"zhangsan"};
    Person p1(32, 1, name1);
    Person p2(p1);//拷贝构造函数
}
```

注意：
1. 拷贝构造函数--传递引用的目的是防止死递归。因为传参（传已存在的对象）的时候，也会调用拷贝构造函数，就会一直递归下去。
2. 要防止浅拷贝。
## 运算符重载

### 等号运算符（赋值运算）

当使用一个已存在的对象，给另一个已存在的对象赋值的时候，自动调用的成员方法。

如果自己不实现，会自动生成一个浅拷贝的等号运算符重载函数。自己实现了则不会。

```c++
void operator=(const Person& src)
{
    // 如果是自赋值，可以直接返回
    if(this == &src)
    {
        return;
    }
    // 删掉原有的堆中成员属性，因为马上就要覆盖，地址会丢失，防止内存泄露
    delete[]_name;
    //防止浅拷贝
    _name = new char[strlen(src._name) + 1];
    memset(_name, 0, strlen(src.name) + 1);
    for(int i = 0; i < strlen(src._name); ++i)
    {
        _name[i] = src._name[i];
    }
}

//为了可以连等，可以加一个返回值类型。
Person& operator=(const Person& src)//使用引用节省内存，防止多余的重复构造
{
    //防止自赋值
    if(this == &src)
    {
        return *this;
    }
    //防止内存泄露
    delete[]_name;
    //防止浅拷贝
    _name = new char[strlen(src._name) + 1];
    memset(_name, 0, strlen(src._name) + 1);
    for(int i = 0; i < strlen(src._name); ++i)
    {
        _name[i] = src._name[i];
    }
    return *this;
}
```

注意：
1. 跳过自赋值
2. 防止内存泄漏
3. 防止浅拷贝
### 等号重载和拷贝构造区别
```c++
int d(a); // 拷贝构造是用一个已存在的对象对正在定义的对象进行初始化
int e = a;
```

```c++
int c = int();
c = a; // 等号重载，两个对象都已经存在
```
## 对象的生存周期

```c++
Person()
{
    _name = NULL;
}
Person fun(Person p)
{
    int a = 10;
}
int main()
{
    Person p1(31, 1, "zhangsan");
}
```

```c++
Person p5 = p3;
Person p6(p3);
//一样
```

```c++
Person p4 = 20;
/*
使用20生成临时对象-->看有没有对应的构造函数
使用临时对象拷贝构造p4
析构临时对象

如果出现上述步骤，会被直接优化成构造p4

--临时对象的生存周期只在当下语句。
--在当下栈帧上（如main中的，而不是子函数中的栈帧），临时对象如果被引用，临时对象的生存周期就会被提升为和引用一致。
--栈帧没有了，一切都失效了。
*/
```

```c++
Person fun(Person& p)
{
    int a = 10;
    Person tmp(10);
    return tmp;
}
int main()
{
    Person p3;
    p3 = fun(p3);
    return 0;
}
```

# 练习

## 使用class封装一个数组

```c++
class Arr
{
public:
    int *_arr;//指向堆空间
    int _cap;//容量
    int _cur_len;//实际
    Arr();
    Arr(int len);
    Arr(const Arr& src);
    Arr& operator=(const Arr& src);
    ~Arr();
    
    void push_back();
    void pop_back();
    int front();
    int back();
    void show();
    bool is_full();
    bool is_empty();
}
```
## 封装List

```c++
class List
{
	is_empty();
    push_back();
    pop_back();
    push_front();
    pop_front();
    
    ElemType front();
    ElemType back();
    
    int size();
    
}
```
## 单例模式实现stack/queue

```cpp
//再依赖链表实现stack、queue——单例模式
class Stack
{
    pop()
    {
        _list.pop_front();
    }
    top()
    {
        return _list.front();
    }
    List _list;
}
```
# class

```c++
class Person
{
public:
    int _age;
    int _sex;
    char *_name;
    void work()
    {
        cout << _name << ": work" << endl;
    }
    void eat()
    {
        cout << _name << ": eat" << endl;
    }
};//分号不要丢
```
## private

class中默认权限是private。

```c++
int last_name;

class Person
{
private:
    int _age;
    const int _sex;//常量必须初始化。
    char *_name;
    int& _last_name;//引用也必须初始化。
public:
    Person()
        :_sex(1),_last_name(last_name)
    {
        
    }
    Person(int age)
    {
        
    }
    Person(int age, int sex, const char *name)
    {
        _age = age;
        _sex = sex;
        _name = new char[strlen(name)+1];
        memset(name,0,strlen(name)+1);
        for(int i = 0;i<strlen(name);++i)
        {
            _name[i] = name[i];
        }
    }
    Person(const Person& src)
    {
        
    }
    void operator=(const Person& src)
    {
        //防止自赋值
        if(this==src)
        {
            return;
        }
        //防止内存泄露
        delete[]_name;
        _name = new char[strlen(src._name)+1];
        memset(_name,0,strlen(src.name)+1);
        for(int i = 0;i<strlen(src._name);++i)
        {
            _name[i] = src._name[i];
        }
    }
    const char* get_name()
    {
		return this.name;   
    }
    int get_age()
    {
        return this.age;
    }
    void set_age(int age)
    {
        _age = age;
    }
    void work()
    {
        cout << _name << ": work" << endl;
    }
    void eat()
    {
        cout << _name << ": eat" << endl;
    }
};//分号不要丢
```
### 与struct区别

struct成员的默认权限是public，class成员的默认权限是private；
## 权限选择

1. 必须要为外界提供的，放在public。其他的都放在private。
2. 一般成员属性都放在private，如果外界需要使用就提供公有接口。
## 名词区别

定义--划分内存

```c++
int a;
Person p;
```

初始化--定义的同时给值

```c++

```

赋值--定义之后给值--内存划分之后给值

```c++
_sex = 1;
```
## 初始化列表

```c++
public:
    Person()
        : _sex(1)//初始化列表。
    {
        
    }
```

1. 只有构造函数有初始化列表。
2. 必须初始化的成员需要放在初始化列表。如const修饰的成员属性。
3. 在本对象构造之前必须要完成的动作，必须放在初始化列表。
## 常对象

```c++
const char* get_name() const
{
    return this.name;   
}
int get_age()
{
    return this.age;
}
int main()
{
    const Person p1(18, 0, "xiaohua");
}
```

1. 常对象只能调用常方法——构造函数，析构函数，静态函数不影响。
2. 常方法中只能调用常方法——静态函数不影响。

为什么呢？因为，非const方法里默认的第一的参数是`Class_Name * const this`。所以不能把`Class_Name const * const this`赋给它。（常量不能赋给非常量）。
### 哪些成员方法需要写成常方法

1. 如果成员方法内不需要改动成员，并且没有对外暴露成员引用或指针，就可以直接写成常方法。
2. 如果成员内部不需要改动成员，但是会对外暴露成员引用或指针，就写两个成员方法，奇淫巧计，效果是形成重载。
```c++
int& get_age()
{
    return _age;
}
int get_age()const
{
    return _age;
}
```
3. 如果成员方法内部需要改动成员，就写成普通方法。
## 静态成员

### 变量

1. 类中每个静态成员在一个类中只有一份，这个类的对象公用。
2. 存储在数据段。
3. 必须在类外的.cpp文件中进行初始化，只能初始化一次。

```c++
//静态成员变量的类外初始化
int Person::_num = 0;//必须指明作用域
```
4. 静态成员变量的访问可以不依赖于对象--不依赖于this指针，使用类的作用域可以直接访问。
### 方法
```c++
int get_num()
{
    return _num;
}
int main()
{
    cout << p1.get_num() << endl;//报错，因为p1是常量对象，不可调用非常方法
    cout << p2.get_num() << endl;//可以
    cout << Person::get_num() << endl;//报错，因为
}
```

1. 静态成员方法没有this指针。
2. 静态成员方法的访问可以不依赖于对象--不依赖于this指针，可以使用类的作用域直接访问。
3. 静态成员方法内只能使用静态成员变量或方法，因为没有this指针可以拿来指向自己的成员。
## 应用
单例模式
1. 把构造方法写在private里
```c++
class Only
{
    //要求
    //限制只能产生一个对象
    //在项目中任意地方都可以获取到这个惟一的对象
private:
    static Only* _only;
    Only();
    static mutex _lock;//#include<mutex>
public:
    static Only* get_Only()
    {
        if(NULL == _only)
        {
            _lock.lock;
            if(NULL==_only)
            {
                _only = new Only();
                return _only;
            }
            _lock.unlock;
        }

        
    }
    
}
```
## 嵌套类

1. 按类与类之间的关系来说，组合关系：一个类是另一个类的一部分。
2. 一个类在另一个类的内部实现。
3. 权限属性依然生效。
4. 外界访问嵌套类需要用到外层类的作用域。

1. 成员对象：先构造内部类对象，再构造自己。
2. 析构时，先析构自己，再析构内部类对象。
3. 如果成员对象没有默认的构造函数，必须手动写在初始化列表。
## 代理关系

一个类的接口功能完全依赖于另一个类的接口功能。
## 类的编译顺序

成员变量必须放在成员方法的前面，否则成员方法编译的时候无法知道成员变量的存在。

1. 编译类名
2. 编译成员名 = 编译嵌套类
3. 编译成员方法体
# 缺省函数

```cpp
class Int
{
    int value;
    int sum;
public:
    Int(int x)
        :
    {
        
    }
    void Print()const
    {
        printf("value = %d, sum = %d\n",value,sum);
    }
}
```

```cpp
// 缺省的拷贝构造函数
{
    const& Int(const& Int src)
    {
        if(this != &src)
        {
            value = src.value;
            sum = src.value;
        }
        return *this;
    }
    const& Int operator=(const& Int src)
    {
        
    }
}
```

# 对象

```cpp
class CGoods
{
private:
    char Name[20];
	int Amount;
    float Price;
    float Total;
public:
    void RegisterGoods(const char *name, int amount, float price);
    int GetAmount()
    {
        return Amount;
    }
    float GetTotal()
    {
        return Total;
    }
};
void CGoods::RegisterGoods(const char *name, int amount, float price)
{
    strcpy_s(Name, 20, name);
    Amount = amount;
    Price = price;
    Total = Amount * Price;
}
int main()
{
    CGoods tea;
    CGoods book;
    tea.RegisterGoods("black_tea",12,560);
    book.RegisterGoods("Thinking in C++",100,98.9f);
    
    int x = tea.GetAmount();
    x = book.GetAmount();
}
```

1. 类是一种数据类型，定义时系统不为类分配存储空间，所以不能对类的数据成员初始化。
2. 类中的任何数据成员也不能使用关键字extern、auto或register限定其存储类型。
3. 函数在类中声明后，在类中定义则默认为内联函数，在类外定义则默认不为内联函数，需要主动加上。
## 对象的创建与使用

对象是类的实例。声明一种数据类型只是告诉编译系统该数据类型的构造，并没有预定内存。类只是一个样板（图纸），以此样板可以在内存中开辟出同样结构的实例--对象。

创建类的对象有两种常用方法。
第一种是直接定义类的实例--对象：
```cpp
int a;
CGoods Car;
```

这个定义创建了CGoods类的一个对象Car，同时为它分配了属于它自己的存储块，用来存放数据和对这些数据实施操作的成员函数（代码）。对象只在定义它的域中有效。

第二种是用new来创建。
```cpp
CGoods s1;//对象构建在数据区，还没进入主函数就构建好了。
int main()
{
    int a = 10;
    CGoods c1;//整个对象构建在栈区，首地址为&c1
    CGoods *cp = new CGoods();//对象用new构建在堆区，在栈区用一个指针保存其首地址cp。
    delete cp;
}
```

# 运算符重载

1. 一元运算符：`*`、`++`、`--`
2. 二元运算符：`+`、`-`、`=`、`&&`、`||`
3. 三元运算符：` : ? `

```cpp
int main()
{
    string s1 = "aaaa";
    string s2 = "bbbbb";
    string s3 = s1 + s2;
    cout << s3 << endl;
}
```

```cpp
class Complex
{
public:
    Complex();
    Complex(int a, int b);
    
    Complex(const Complex& src);
    Complex& operator=(const Complex& src);
    Complex  operator+(const Complex& src) const;
    Complex  operator-(const Complex& src) const;
    ~Complex();
    friend ostream& operator <<(ostream& out, const Complex& src);
private:
    int _a;//实部
    int _b;//虚部
};
```

```cpp
Complex()
{
    
}
Complex(int a, int b)
{
    _a = a;
    _b = b;
}
Complex operator+(const Complex& src)const
{
    int a = _a + src._a;
    int b = _b + src._b;
    return Compare(a,b);
}
Complex operator-(const Complex& src)const
{
    int a = _a - src._a;
    int b = _b - src._b;
    return Compare(a,b);
}
bool operator&&(const Complex& src)const
{
    return _a && src._a;
}
void operator <<(ostream& out)
{
    out << _a << "+" << _b << "i" << endl;
}
void operator <<(ostream& out, const Complex& src)
{
    out << src._a << "+" << src._b << "i" << endl;
}

Complex& operator++()//前置++
{
    ++_a;
    return *this;//退出函数时，this指针还有效。
}
Complex operator ++(int)//后置++，int没用，只作为标志是后置
{
    int tmp = _a;
    _a++;
    return Complex(tmp, _b);//临时对象，不能返回引用
}

ostream& operator <<(ostream& out, const Complex& src)
{
	out << src._a << "+" << src._b << "i" << endl;
	return out;
}

//- -- || >>
int main()
{
    string s1 = "aaaa";
	string s2 = "bbbbb";

	string s3 = s1 + s2;
	cout << s3 << endl;


	Complex c1(1, 2);
	Complex c2(2, 3);

	Complex c3 = c1 + c2;

	if (c1 && c2)
	{
		cout << "all 0" << endl;
	}

	ostream& oo = cout;
	oo << s1 << endl;

	//c1.operator<<(cout);

	cout << c1 << c2 <<endl;


	c1++;
	//++c1;
	cout << c1 << endl;
	//c1--;
	//--c1;

	return 0;
}
```
## 输入输出重载

### `<<`重载

ostream是iostream的一个类。cout就是这个类的对象，在屏幕上输出东西。

```cpp
ostream oo = cout;//不允许，只能有一个ostream对象，默认已创建好，为cout。
//我们可以利用引用，起个别名。
ostream& oo = cout;
oo << "abc" << endl;
```
### `>>`重载

istream也是iostream中的一个类。cin就是这个类的对象，从键盘获取数据给其他对象。

```cpp
void operator <<(ostream& out)
{
    out << _a << "+" << _b << "i" << endl;
}
int main()
{
    cout << c1 << endl;//报错
    //只能写成如下形式：
    c1 << cout;
}

```

改进：

```cpp
void operator <<(ostream& out)
{
    
}
```

## 练习

BigNum

```cpp
class BigNum
{
    BigNum(int);
    BigNum(const char*);
    BigNum& operator++();
    BigNum  operator++(int);
    BigNum& operator--();
    BigNum  operator--(int);
private:
    string _num;
};
```