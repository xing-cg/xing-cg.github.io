---
title: Windows_wchar和wmain
categories:
  - - Windows
  - - 操作系统
tags: 
date: 2024/6/5
updated: 
comments: 
published:
---
# 字符集
Character Set

char - 1 Byte - 8 bit - ASCII

参考：ASCII和ANSI [Ascii Codes - C++ Tutorials (cplusplus.com)](https://legacy.cplusplus.com/doc/ascii/)
## ANSI码

ANSI码仅在前128（0-127）个与ASCII码相同，之后的字符全是某个国家语言的所有字符。值得注意的是，两个字节最多可以存储的字符数目是2的16次方，即65536个字符，这对于一个语言的字符来说，绝对够了。还有ANSI编码其实包括很多编码：中国制定了GB2312编码，用来把中文编进去另外，日本把日文编到Shift_JIS里，韩国把韩文编到Euc-kr里，各国有各国的标准。受制于当时的条件，不同语言之间的ANSI码之间不能互相转换，这就会导致在多语言混合的文本中会有乱码。
## Unicode编码

为了解决不同国家ANSI编码的冲突问题，Unicode应运而生：如果全世界每一个符号都给予一个独一无二的编码，那么乱码问题就会消失。这就是Unicode，就像它的名字都表示的，这是一种所有符号的编码。

Unicode标准也在不断发展，但最常用的是用两个字节表示一个字符（如果要用到非常偏僻的字符，就需要4个字节）。现代操作系统和大多数编程语言都直接支持Unicode。

但是问题在于，原本可以用一个字节存储的英文字母在Unicode里面必须存两个字节（规则就是在原来英文字母对应ASCII码前面补0），这就产生了浪费。那么有没有一种既能消除乱码，又能避免浪费的编码方式呢？答案就是UTF-8！
## UTF-8编码

这是一种变长的编码方式：它可以使用1~4个字节表示一个符号，根据不同的符号而变化字节长度，当字符在ASCII码的范围时，就用一个字节表示，保留了ASCII字符一个字节的编码做为它的一部分，如此一来UTF-8编码也可以是为视为一种对ASCII码的拓展。值得注意的是unicode编码中一个中文字符占2个字节，而UTF-8一个中文字符占3个字节。从unicode到uft-8并不是直接的对应，而是要过一些算法和规则来转换。

在计算机内存中，统一使用Unicode编码，当需要保存到硬盘或者需要传输的时候，就转换为UTF-8编码。

用记事本编辑的时候，从文件读取的UTF-8字符被转换为Unicode字符到内存里，编辑完成后，保存的时候再把Unicode转换为UTF-8保存到文件。
# Unicode对应的字符类型 - wchar

Unicode对应的字符类型为`wchar_t`。打印sizeof为2，即占大小为2字节、16位。
```cpp
#include<iostream>
int main()
{
    std::cout << sizeof(wchar_t);
    return 0;
}
```
1. 初始化一个Unicode字符串。需要在"双引号"前加`L`，表示使用Unicode字符集串。
2. 如果要正常打印Unicode字符串，需要使用`std::wcout`。
3. Cpp中也有Unicode字符串对应的类，是`std::wstring`。
```cpp
#include<iostream>
int main()
{
    // wchar_t str[] = "Hello";  // 错误，"Hello"默认为ANSI字符集串
    wchar_t str[] = L"Hello";
    std::cout  << sizeof str << std::endl; // 输出 12. Hello 5个 加 1个哨兵位，乘以2
    std::wcout << str;                     // 输出 Hello
    
    // Cpp中的Unicode字符串
    std::wstring test = str;
    std::wcout << test;
}
```
## C中的wchar库
新标准写法为cwchar
[cwchar (wchar.h) - C++ Reference (cplusplus.com)](https://legacy.cplusplus.com/reference/cwchar/)

C语言标准库中也提供了对wchar的支持，输出用`wprintf`。区别为"双引号"前要加`L`。
相应于普通的strcat、strcmp、strcpy、strlen等，都可用wcs开头的wcscat等代替。
>Cpp中的宽字符输出流为：`std::wcout`
```c
#include<cwchar>
int main()
{
    wchar_t str[] = L"Hello";
    wprintf(L"%s\n", str);      // 记得加L.  输出Hello
}
```
# wmain

```cpp
int main(int ac, char * av[]) // main入口 + 普通的char * []
{
    for(int i = 0; i < ac; ++i)
    {
        std::cout << av[i] << std::endl; // cout
    }
}
```

```cpp
int wmain(int ac, wchar_t * av[]) // wmain入口 + wchar_t * []
{
    for(int i = 0; i < ac; ++i)
    {
        std::wcout << av[i] << std::endl;// wcout
    }
}
```
# 第一个Windows程序

不遵循ANSI、ISO标准，是Windows平台下程序的入口。
使用Windows.h库，是Windows系统自带的SDK。

```cpp
#include <iostream>
#include <Windows.h>

int wWinMain(
        HINSTANCE hInstance,
        HINSTANCE hPrevInstance,
        LPWSTR lpCmdLine,
        int nShowCmd)
{
    return 0;
}
```

>编译之前，要配置好VS项目的属性，右键项目选择Properties，左边栏选择Linker，然后选择System。右边详细条目"SubSystem"需下拉选择Windows。

如果直接return 0，不会出现窗口，程序直接退出。
如果想要出现窗口，则需要加语句：
```cpp
{
    ::MessageBox(nullptr, L"First App", L"cap", MB_OK);
}
```
>MessageBox API: 
>`int MessageBoxW(HWND hWnd, LPCWSTR lpText, LPCWSTR lpCaption, UINT uType)`
>hWnd表示窗口句柄。
>lpText表示显示文本。
>lpCaption表示窗口标题。
>uType表示显示的按钮是什么类型。我们在此例中使用`MB_OK`这个值。有关于此的参考文档，可以进入[learn.microsoft.com](https://learn.microsoft.com)，在输入框搜索"messagebox"。搜索结果查看`MessageBox function (winuser.h) - Win32 apps`
>
>LPCWSTR表示Long Pointer Const Wide String。
>关于LP：Windows 3.2版本以前由于内存分段的原因，要区分长短指针。但是现在内存不分段了，是平坦模式，因此长短指针不区分了。
>因此只需要关注CWSTR：比如`L"Hello First Windows Program"`

运行效果：出现一个如下的窗口，程序会阻塞于此，当点击OK按钮时，程序才会继续执行。
![](../../images/Windows_wchar和wmain/image-20240623234329264.png)
# 窗口程序

多个程序共享一个窗口。

## 弱化流程——基于消息驱动

解决抢资源的问题，以前是锁，是资源抢占形式，不太好。现在，让多个人共享同一种资源，最好的办法说白了就是排队。排队需要优先级，比如时间先后顺序。那就需要一个消息队列（消息循环）。
这些消息可能是并行的，需要串行化后进入消息队列，从队列中不断取消息，把消息一个一个地投放到目标窗口。
## Windows下的面向对象

Windows编程，Windows SDK或Win32，是一个面向对象的程序设计方法。虽然1985年`C++`面向对象语言才出来，在这之前大多数是用C语言开发，但这不是说用不了面向对象的思想。

假设整个应用程序抽象为一个对象，应用程序可能有多个窗口，每个窗口是一个窗口对象，此窗口对象的数据结构存在于操作系统的内部（因此也叫内核对象 (Kernel Object)），每个窗口对象有一个对应的Handle，即句柄，字面意义上是把手，形容可以用此把手提起一个箱子。

与Linux思想不同，Linux是一切皆文件，而Windows是一切皆窗口，在其中，窗口也视为一种内核对象。

权威教材描述：
1. Programming Windows - Charles Petzold
2. Windows核心编程（Windows via `C/C++`）
## wWinMain解析

1. HINSTANCE：应用程序的句柄类型，H代表句柄。
2. nShowCmd：预设的值以及含义见[ShowWindow function (winuser.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-showwindow)中的Parameters。可以用于控制窗口的显示、隐藏等。

应用程序创建窗口首先需要一个对象。

WNDCLASSEX是一个结构体，用于定义窗口类。因为当时没有面向对象语言，需要把该类注册给操作系统，让操作系统在内部生成内核对象。
[About Window Classes - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/winmsg/about-window-classes)

[WNDCLASSEXA (winuser.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/winuser/ns-winuser-wndclassexa)

1. 第一步先初始化窗口类对象的成员值。
    1. lpfnWndProc：是WNDPROC类型，需要指定一个函数指针。[WNDPROC - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nc-winuser-wndproc)。
2. 注册该窗口类对象。之后操作系统就知道了该窗口类的存在。
    1. [RegisterClassExW function (winuser.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-registerclassexw)
    2. 如果注册成功，返回一个非零的类原子值，唯一标识该类。If the function succeeds, the return value is a class atom that uniquely identifies the class being registered. This atom can only be used by the [CreateWindow](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-createwindowa), [CreateWindowEx](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-createwindowexa), [GetClassInfo](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-getclassinfoa), [GetClassInfoEx](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-getclassinfoexa), [FindWindow](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-findwindowa), [FindWindowEx](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-findwindowexa), and [UnregisterClass](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/nf-winuser-unregisterclassa) functions and the `IActiveIMMap::FilterClientWindows` method.
    3. If the function fails, the return value is zero. To get extended error information, call [GetLastError](https://learn.microsoft.com/en-us/windows/desktop/api/errhandlingapi/nf-errhandlingapi-getlasterror).
3. 操作系统以名字——`lpszClassName`访问，如果没有定义该变量，就不认识了。
4. 用注册成功的类来创建窗口，调用CreateWindowEx，返回窗口句柄。
    1. CreateWindowEx的参数来定义创建的窗口的属性。
        1. [CreateWindowExW function (winuser.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-createwindowexw)
        2. 第一个参数dwExStyle：[Extended Window Styles (Winuser.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/winmsg/extended-window-styles)。表示样式。我们暂用`WS_EX_OVERLAPPEDWINDOW`。
        3. 第二个参数lpClassName：窗口类名。指定用哪个窗口类来创建窗口的实例，用窗口类注册的名字索引。
        4. 第三个参数lpWindowName：窗口的标题。
        5. 第四个参数dwStyle：和第一个参数dwExStyle相比是基础的，Ex表示拓展。我们暂用`WS_OVERLAPPEDWINDOW`。具体值为：`WS_OVERLAPPED | WS_CAPTION（标题栏） | WS_SYSMENU（菜单） | WS_THICKFRAME | WS_MINIMIZEBOX（最小化按钮） | WS_MAXIMIZEBOX（最大化按钮）`。
        6. 第5、6个参数X、Y：表示窗口的起始坐标。
        7. 第7、8个参数nWidth、nHeight：分别表示宽度、高度。
        8. 第9个参数hWndParent：表示父窗口句柄，若无则填NULL或nullptr。
        9. 第10个参数hMenu：Menu栏句柄，若无则填NULL或nullptr。
        10. 第11个参数hInstance：整个应用程序的句柄。
        11. 第12个参数lpParam：特殊、额外的参数。若无则填NULL或nullptr。
5. 显示窗口、更新窗口。创建完窗口后，就交由Windows系统管理了。
6. 开始消息循环。调用GetMessage从当前的应用程序队列中获得消息，队列是操作系统内部提供的。[GetMessage function (winuser.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-getmessage)
    1. 第一个参数lpMsg，放入一个MSG类型的指针，用于获取消息后填充给它。MSG是个结构体。
        1. MSG结构体成员：hwnd、message、wParam、lParam、time、pt
    2. 第二个参数hWnd，指定应用程序的某个窗口。如果填nullptr表示指定所有窗口。
    3. 第三、四参数wMsgFilterMin、wMsgFilterMax，指定关注消息的号段。如果全填0，表示关注所有消息。
    4. 返回BOOL。正常则返回非零，继续循环；如果收到的消息是`WM_QUIT（0x0012）`，则返回0退出循环。
7. 循环退出则程序结束。
```cpp
int wWinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPWSTR lpCmdLine, int nShowCmd)
{
    std::wstring app_name{ L"First App" };

    WNDCLASSEX wcex = { 0 };
    wcex.cbSize = sizeof(wcex);
    wcex.style = CS_HREDRAW | CS_VREDRAW;
    // 绑定CALLBACK，见后面解释
    wcex.lpfnWndProc = &window_procedure;
    // wcex.... = ...;
    wcex.lpszClassName = app_name.c_str();
    // wcex.... = ...;
    

    if (!::RegisterClassEx(&wcex))
    {
        return -1;
    }
    // 定义一个窗口对象的句柄
    // HWND: 窗口类型的句柄，也叫内核对象
    HWND hWnd = ::CreateWindowEx(
        WS_EX_OVERLAPPEDWINDOW,
        app_name.c_str(),
        L"App",
        WS_OVERLAPPEDWINDOW,
        CW_USEDEFAULT, 0, CW_USEDEFAULT, 0,
        nullptr,
        nullptr,
        hInstance,
        nullptr);
    if (!hWnd)
    {
        int i = ::GetLastError();
        return -1;
    }
    // 显示窗口，nShowCmd可以指定窗口以最小/最大/正常状态显示
    ShowWindow(hWnd, nShowCmd);
    // 更新窗口，绘制窗口。刚显示出来可能是无效的，需要在显示后绘制。
    UpdateWindow(hWnd);

    MSG msg;
    while (::GetMessage(&msg, nullptr, 0, 0))
    {
        // 翻译消息，比如输入法通过‘wo’生成‘我’
        ::TranslateMessage(&msg);
        // 发送消息，调用刚才注册的callback窗口处理函数
        ::DispatchMessage(&msg);
    }

    return 0;
}
```
其中，CALLBACK函数如下：message是消息类型，为unsigned 
```cpp
// 按照WNDPROC的原型定义，返回默认的DefWindowProc
LRESULT CALLBACK window_procedure(HWND wnd, UINT message, WPARAM wparam, LPARAM lparam)
{
    return ::DefWindowProc(wnd, message, wparam, lparam);
}
```
以上这个是什么也不处理，直接返回一个默认的操作。
以下使用switch进行处理：
```cpp
LRESULT CALLBACK window_procedure(HWND wnd, UINT message, WPARAM wparam, LPARAM lparam)
{
    PAINTSTRUCT ps;
    switch (message)
    {
    case WM_CREATE:           // 相当于constructor，往往先于消息循环。
        break;
    case WM_PAINT:
        // Handle of Device Context
        // case语句中不能直接声明、定义变量，因为case中的语句会影响程序编译的行为，而程序不确定会运行到定义变量语句，case是没有对应的栈的，没有了确定性，因此需要在此段语句外加大括号，形成局部的栈，就可以存放临时变量了。相当于把这几条语句用函数包装了。
    {    
        HDC hdc = ::BeginPaint(wnd, &ps);
        ::Rectangle(hdc, 100, 100, 200, 200);  // 四个数字是坐标参数
        ::EndPaint(wnd, &ps);
    }
        break;
    case WM_CLOSE:
        ::DestroyWindow(wnd);
        return 0;  // 返回的LRESULT，表示WM_CLOSE消息已处理
        break;
    case WM_DESTORY:          // 相当于destructor
        ::PostQuitMessage(0); // 执行后会向消息队列投入WM_QUIT消息，下次GetMessage得到WM_QUIT将返回0，while循环退出。
        break;
    default:
        break;
    }
    return ::DefWindowProc(wnd, message, wparam, lparam);
}
```
### BeginPaint
BeginPaint 函数准备指定的窗口进行绘画，并使用有关绘画的信息填充[PAINTSTRUCT](https://learn.microsoft.com/en-us/windows/desktop/api/winuser/ns-winuser-paintstruct)结构。
![](../../images/Windows_wchar和wmain/image-20240731013725180.png)
参数：
1. hWnd，`HWND`类，要重绘的窗口的句柄。
2. lpPaint，`LPPAINTSTRUCT`类，指向将接收绘画信息的`PAINTSTRUCT`结构的Long指针。

返回值：
1. 如果成功，返回值是指定窗口的DC的句柄。
2. 如果失败，返回NULL，表示没有可用的显示设备上下文（Display Device Context）。
# 总结

以上虽然是对Windows系统中的窗口程序进行了解析，但是其实其他的操作系统中的窗口UI程序也是同样的流程和道理。

总之，不同的应用程序抢相同的资源时，程序范式有两种：
1. 红绿灯，等别人做完再做。锁机制，在多任务系统，尤其是带UI的系统会有卡顿感。
2. 基于消息循环，从消息发送到消息处理也就是100ms之内，本质上也是一种同步，但粒度更小，就可以等效于一段时间内大家都在同时做。
# 再记几个消息类型

```cpp
{
    // ...
    case WM_LBUTTONDOWN:
    {
        int x_coord = lparam & 0xffff;
        int y_coord = lparam >> 16;
        wchar_t text[50] = L"";
        // 打印到text字符数组中
        swprintf(text, 50, L"%i, %i", x_coord, y_coord);
        ::MessageBox(nullptr, text, L"BtnTest", MB_OK);
    }
        break;
    // ...
}
```

1. WM_LBUTTONDOWN：鼠标左键按钮按下动作。[WM_LBUTTONDOWN message (Winuser.h) - Win32 apps | Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/inputdev/wm-lbuttondown)
    1. 消息里的属性有：wparam，lparam。w代表字，l代表long，现代已经全部升级为32位的long了。lparam中，低16位是X坐标，高16位是Y坐标。如果要从32位数中取出16位，需要做与掩码的按位与运算。比如取X：`int x_coord = lparam & 0xffff;`，取Y：`int y_coord = lparam >> 16;`

# 自定义消息类型

Windows中默认的message都是有相应预定义好的值的。比如在`WinUser.h`中：
```cpp
#define WM_MOUSEMOVE      0x0200
#define WM_LBUTTONDOWN    0x0201
#define WM_LBUTTONUP      0x0202
```

我们也可以根据需要自定义一个消息类型，在`WinUser.h`中，又一个给出的`WM_USER`值，它的注释说明了，小于`0x0400`的值已被系统占用，自定义类型值需要从`0x0400`开始：
```cpp
/* 
 * NOTE: All Message Numbers below 0x0400 are RESERVED.
 * Private Window Messages Start Here:
 */
#define WM_USER          0x0400 
```
所以，比较简便的自定义消息类型的预定义语句可以这么写：
```cpp
#define MYMSG            WM_USER + 10
```

使用：通过左击鼠标，用PostMessage来发送MYMSG到wnd的消息队列。下次取出MYMSG消息后做相应的动作。
```cpp
{
    // ...
    case WM_LBUTTONDOWN:
        ::PostMessage(wnd, MYMSG, wparam, lparam);
        break;
    case MYMSG:
        ::MessageBox(nullptr, L"My Message", L"MyMsg", MB_OK);
        break;
    // ...
}
```
也可以用SendMessage，区别是不经过消息队列，直接发送消息。效果是在SendMessage后，立即调用回调函数，处理完毕后，再回到原先的SendMessage下一条位置继续。
```cpp
{
    // ...
    case WM_LBUTTONDOWN:
        ::SendMessage(wnd, MYMSG, wparam, lparam);
        break;
    case MYMSG:
        ::MessageBox(nullptr, L"My Message", L"MyMsg", MB_OK);
        break;
    // ...
}
```

# 带参数

利用SendMessage、PostMessage中可以带wparam、lparam，回调函数可以使用这些参数。

发送消息时携带参数：
```cpp
struct MyParam
{
    int x_coord;
    int y_coord;
}
// ...
{
    // ...
    case WM_LBUTTONDOWN:
    {
        int x_coord = lparam & 0xffff;
        int y_coord = lparam >> 16;
        MyParam* param = new MyParam{ x_coord, y_coord };
        ::SendMessage(wnd, MYMSG, reinterpret_cast<WPARAM>(param), lparam);
    }
        break;
    // ...
}
```

处理消息时解析参数：
```cpp
{
    // ...
    case MYMSG:
    {
        MyParam* param = reinterpret_cast<MyParam*>(wparam);
        std::wstring text = std::format(L"{0}, {1}", x_coord, y_coord);
        ::MessageBox(nullptr, text.c_str(), L"Param Test", MB_OK);
        delete param;
    }
        break;
    // ...
}
```

# 总结

Windows SDK的开发，聚焦的点主要就在于window_procedure里的代码逻辑和消息的处理。