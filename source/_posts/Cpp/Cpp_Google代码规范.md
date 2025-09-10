---
title: Cpp_Google代码规范
categories:
  - - Cpp
tags: 
date: 2025/9/10
updated: 2025/9/10
comments: 
published:
---
文章：https://google.github.io/styleguide/cppguide.html
源码：https://github.com/google/styleguide/blob/gh-pages/cppguide.html

author: Chenggong Xing
date: 2025/9/10
# 头文件、include相关
## Self-contained Headers
自包含是什么意思：一个 `.h` 头文件应该​​独立编译​​（即 `#include` 它自身就能编译通过，不会报错）。

### 非自包含头文件
存在一些​​罕见情况​​，需要创建一些​​设计上就不是自包含​​的文件，它们的目的就是被 `#include` 到其他文件中。
要包含的非自包含头文件应以`.inc`结尾，并谨慎使用。

特点：
1. 可能没有`#ifndef #define`守卫：因为它们可能被设计成在同一个文件中多次包含（例如，用于代码生成或宏展开）。
2. 不包含自身依赖：它们假设包含它们的文件（通常是 .cc或另一个 .h）已经提供了必要的上下文（如包含了所需的头文件、定义了必要的宏等）。
3. 依赖包含位置：它们通常需要被包含在特定的位置（例如，在另一个文件的中间，而不是顶部）
## `#ifndef #define`守卫
符号名称的格式应是`<PROJECT>_<PATH>_<FILE>_H_`。
为了保证唯一性，应该基于项目源代码树中的完整路径。例如，项目`foo`中的文件`foo/src/bar/baz.h`，应该这么写：
```cpp
#ifndef FOO_BAR_BAZ_H_
#define FOO_BAR_BAZ_H_

...

#endif  // FOO_BAR_BAZ_H_
```
## 不要依赖传递`#include`
bar.h
```cpp
// bar.h
#ifndef BAR_H
#define BAR_H
class Bar {
    // ... Bar 的成员 ...
};
#endif // BAR_H
```
foo.h
```cpp
// foo.h
#ifndef FOO_H
#define FOO_H

#include "bar.h" // 当前 foo.h 需要 Bar 来声明 useBar 的参数

void useBar(const Bar& b); // 声明使用 Bar

#endif // FOO_H
```
foo.cc
```cpp
// foo.cc (错误写法：依赖 foo.h 包含 bar.h)
#include "foo.h" // 指望 foo.h 已经包含了 bar.h

void useBar(const Bar& b) {
    // ... 使用 b (Bar 对象) ...
}
```
导致的问题：如果将来`foo.h`中删除了`#include "bar.h"`，那么还需要在`foo.cc`添加`#include "bar.h"`，导致牵一发动全身。

正确写法：
```cpp
// foo.cc (正确写法：显式包含 bar.h)
#include "foo.h"
#include "bar.h" // 显式包含，因为直接使用了 Bar

void useBar(const Bar& b) {
    // ... 使用 b (Bar 对象) ...
}
```
这保证了 `foo.h`的修改（清理不必要的 `#include`）不会意外破坏 `foo.cc`的编译。每个文件都清晰地声明了自己的直接依赖。
## 避免前置声明，最好用include代替
避免使用前置声明，而是 include 需要的头文件。

​**​核心观点：尽量避免使用前置声明，优先使用 `#include` 包含所需的头文件。​**​

​**​什么是前置声明 (Forward Declaration)?​**​
- ​**​定义：​**​ 前置声明是指在代码中​**​声明​**​某个实体（如类、函数、变量、模板等）的存在，但​**​不提供其完整定义​**​。
- ​**​目的：​**​ 告诉编译器这个符号的名字和类型（对于函数和变量），但不需要知道其内部细节（如类的成员、函数的实现、变量的值等）。
- ​**​示例：​**​
    
    ```cpp
    class MyClass;      // 类的前置声明 (告诉编译器 MyClass 是一个类)
    void myFunction();  // 函数的前置声明
    extern int myVar;    // 变量的前置声明 (通常用于全局变量)
    ```
    

### ​Pros (优点/好处 - 为什么有人想用前置声明)​​
规范列举了使用前置声明可能带来的好处，但请注意，规范的整体立场是​**​不推荐​**​使用，所以这些优点更像是解释为什么开发者有时会倾向于使用它：
1. ​**​节省编译时间：​**​ `#include` 会让编译器打开并处理被包含文件的所有内容（可能又包含更多文件）。使用前置声明可以避免这些开销，特别是当包含的头文件很大或嵌套很深时。
2. ​**​减少不必要的重新编译：​**​ 如果一个头文件被修改了，所有直接或间接包含它的源文件都需要重新编译。如果一个头文件 `A.h` 包含了 `B.h`，那么修改 `B.h` 会导致包含 `A.h` 的所有文件都重编。如果 `A.h` 只用了 `B.h` 中的某个类指针或引用，并且改用前置声明 `class B;` 而不是 `#include "B.h"`，那么修改 `B.h` 的某些细节（比如 `B` 类的私有成员）可能​**​不会​**​触发 `A.h` 及其包含者的重编译。这可以加快增量编译速度。
### ​Cons (缺点/坏处 - 为什么 Google 规范不推荐使用)​

规范详细列举了前置声明的诸多弊端，这也是其建议避免使用的主要原因：
1. ​**​隐藏依赖关系：​**​ 这是最核心的问题。前置声明使得代码的依赖关系变得不清晰。源文件 `foo.cc` 使用了 `class Bar`，但只通过前置声明 `class Bar;` 引入，而没有 `#include "bar.h"`。当 `bar.h` 发生改变（比如 `Bar` 类的大小、成员函数签名变化）时，编译器可能​**​无法意识到​**​ `foo.cc` 需要重新编译，导致链接错误或更糟糕的运行时错误。这破坏了构建系统的可靠性。
2. ​**​阻碍自动化工具：​**​ 代码分析工具、重构工具、IDE 的智能提示等，需要知道符号的完整定义才能正常工作。前置声明使得这些工具难以确定符号的实际定义位置。
3. ​**​限制 API 的兼容性变更：​**​ 库的维护者如果想做一些理论上兼容的修改，可能会因为用户代码使用了前置声明而受阻。例如：
    - 加宽函数参数类型（如 `int` -> `long`）。
    - 给模板添加一个有默认值的模板参数。
    - 将符号移动到新的命名空间。  
        这些修改对于包含完整头文件的用户代码是兼容的，但对于仅使用前置声明的用户代码，可能导致编译失败或行为改变，因为前置声明没有捕捉到这些变化。
4. ​**​`std::` 命名空间的前置声明导致未定义行为：​**​ C++ 标准明确规定，不允许用户代码前置声明标准库 (`std::`) 中的模板或其他实体。这样做会导致​**​未定义行为 (Undefined Behavior)​**​，程序可能编译失败、运行崩溃或产生不可预测的结果。必须 `#include` 相应的标准库头文件（如 `<vector>`, `<string>`）。
5. ​**​可能静默改变代码含义：​**​ 这是一个非常微妙且危险的陷阱。规范中的代码示例清晰地展示了这一点：
    - 在包含完整头文件 `b.h` 的情况下，`test(D*)` 调用 `f(B*)`，因为 `D*` 可以隐式转换为 `B*`。
    - 如果 `good_user.cc` 把 `#include "b.h"` 换成 `class B; class D;`（前置声明），那么 `test(D*)` 会调用 `f(void*)`！因为编译器只知道 `B` 和 `D` 是类类型，但不知道它们之间的继承关系，所以 `D*` 无法隐式转换为 `B*`，只能匹配 `f(void*)`。这种行为的改变是静默发生的，很难调试。
6. ​**​语法冗长：​**​ 如果需要前置声明一个头文件中的多个符号，写一堆 `class X; void Y();` 可能比直接写一个 `#include "that_header.h"` 更冗长。
7. ​**​可能导致次优设计：​**​ 为了能够使用前置声明（例如，在头文件中只使用类指针或引用，避免使用对象成员），开发者可能会被迫采用特定的代码结构（如多用指针、使用 Pimpl 惯用法）。这些结构有时会使代码运行速度变慢（额外的间接访问、堆分配）或增加代码的复杂性（需要管理指针生命周期、实现 Pimpl）。
### ​Decision (决策/结论)​​
- ​**​核心原则：​**​ ​**​尽可能避免使用前置声明。​**​ 优先使用 `#include` 来包含定义了你所需符号的头文件。
- ​**​关键限制：​**​ ​**​尤其要避免对另一个项目中定义的实体使用前置声明。​**​ 这里的“项目”可以理解为不同的库、模块或代码仓库。跨项目的前置声明极大地加剧了上述缺点（特别是隐藏依赖和限制 API 变更），因为项目间的协调和同步更困难。
- ​**​隐含建议：​**​ 在同一个项目内部，如果经过仔细权衡（比如某个头文件改动极其频繁且影响巨大），并且能严格确保依赖清晰、不会引入第 3 点和第 5 点的问题，或许可以​**​极其谨慎地​**​在源文件（`.cc`）中使用前置声明来减少编译依赖。但这需要非常高的警惕性。规范的整体倾向仍然是 `#include` 更安全、更推荐。
### ​总结​
Google C++ 规范认为，虽然前置声明在理论上可以带来编译速度的提升，但其带来的风险（隐藏依赖、破坏构建可靠性、阻碍工具、限制库演化、潜在未定义行为、静默语义改变）远大于收益。因此，规范强烈建议开发者优先使用 `#include` 来明确表达依赖关系，保证代码的健壮性、可维护性和工具友好性，尤其是在跨项目协作时。
### 其他注意的点
不要声明任何一个属于std命名空间的内容，包括标准库类的前置声明。要声明标准库中的实体，请包含适当的头文件。
## 在头文件中定义函数的注意事项
​**​核心观点：​**​ 尽量避免在头文件的 ​**​公共 API 声明部分​**​ 直接定义函数体。如果函数定义​**​必须​**​放在头文件中（例如短小的访问器、模板函数、`constexpr` 函数），应将其放在头文件的​**​内部实现部分​**​（如私有区、特定命名空间或注释标记之后），并确保其 ​**​ODR-safe​**​（通常通过 `inline` 关键字、模板或类内定义实现）。
### ​关键概念解释​
1. ​**​文本内联 (Textually inline)：​**​ 指函数的定义（实现代码）直接写在它的声明处。
2. ​**​内联展开 (Inline expansion)：​**​ 编译器优化技术，将函数调用处直接替换为函数体代码，避免函数调用的开销（压栈、跳转、返回）。这通常发生在函数体简单且被频繁调用时。
3. ​**​ODR (One Definition Rule - 单一定义规则)：​**​ C++ 核心规则，要求在整个程序中，任何变量、函数、类类型、枚举类型或模板，​**​最多只能有一个定义​**​（某些情况如 `inline` 函数/变量、模板、类类型定义等允许在多个翻译单元中存在定义，但必须完全相同）。
4. ​**​ODR-safe：​**​ 指在头文件中定义的实体（如函数、变量），通过使用 `inline` 关键字（或符合隐式 `inline` 的条件），使其在多个 `.cpp` 文件包含该头文件时，链接器不会因违反 ODR（出现多个相同定义）而报错。
### ​​Pros (优点/好处 - 为什么有时需要在头文件中定义函数)​​
规范承认在特定情况下，在头文件中定义函数有其合理性和优势：
1. ​**​减少样板代码 (Reduce boilerplate)：​**​ 对于非常简单的函数（如类的 `getter/setter`），直接在类声明中定义比在头文件声明、再到 `.cc` 文件定义要简洁得多。
2. ​**​潜在的优化机会 (Potential optimization)：​**​ 编译器更容易对在头文件中定义的小函数进行内联展开优化，可能生成更高效的代码（省去函数调用开销）。
3. ​**​技术必要性 (Technical necessity)：​**​ ​**​函数模板​**​和 ​**​`constexpr` 函数​**​ 通常​**​必须​**​在声明它们的头文件中定义（或者至少在同一个翻译单元中可见）。因为编译器在实例化模板或计算 `constexpr` 时需要看到完整的定义。这是语言特性决定的。
### ​​Cons (缺点/坏处 - 为什么规范限制在公共部分定义函数)​​
规范强调了在公共 API 部分（即用户一眼就能看到的地方）定义函数的弊端：
1. ​**​降低 API 可读性 (Reduced API readability)：​**​ API 头文件的主要目的是清晰地展示接口（有哪些函数、参数、返回值）。将函数实现细节混杂其中，会增加阅读和理解 API 的难度和认知负担。函数越复杂，这种干扰越大。
2. ​**​暴露实现细节 (Exposes implementation details)：​**​ 将函数体放在公共头文件中，相当于把内部实现逻辑公开了。这些细节通常对 API 使用者来说是无关紧要的（“无害但多余”），甚至可能暴露你不想让用户依赖的内部机制。
### ​​Decision (决策/规则)​​
基于优缺点分析，规范制定了明确的规则：
1. ​**​长度限制 (Length restriction)：​**​
    - 只有​**​非常短​**​的函数（规范建议大约 ​**​10 行或更少​**​），才允许直接在它的​**​公共声明点​**​（如在类定义的 `public:` 部分）定义。
    - ​**​长函数体​**​应该放在 `.cc` 文件中定义，除非有​**​性能原因​**​（编译器内联优化至关重要）或​**​技术原因​**​（如模板、`constexpr`）。
2. ​**​位置隔离 (Location isolation)：​**​
    - 即使函数定义​**​必须​**​放在头文件中（例如，它是一个模板成员函数），也​**​不应该​**​放在公共 API 部分（如 `public:` 或文件顶部）。
    - 应该将定义放在头文件的​**​内部实现区域​**​：
        - 类的 `private:` 部分（即使函数本身是 `public` 的）。
        - 一个包含 `internal` 字样的命名空间内（例如 `namespace myproject_internal { ... }`）。
        - 在明确的注释标记之后（例如 `// Implementation details follow` 或 `// Implementation details only below here`）。
    - ​**​目的：​**​ 将实现细节与公共接口​**​物理分离​**​，提高公共头文件的可读性和整洁度。
3. ​**​ODR 安全 (ODR safety)：​**​
    - 任何在头文件中定义的函数（或变量），​**​必须​**​确保它是 ​**​ODR-safe​**​ 的。这意味着当多个 `.cpp` 文件包含该头文件时，链接器不会报“多重定义”错误。
    - 实现 ODR-safe 的常用方法：
        - 显式使用 `inline` 关键字修饰函数/变量定义。
        - 函数是​**​函数模板​**​。
        - 函数是​**​类成员函数​**​，并且是​**​在类定义内部直接定义的​**​（这是隐式 `inline` 的）。
        - 函数是 `constexpr` (C++11 起，`constexpr` 函数在头文件中定义默认是 `inline` 的)。
        - 变量是 `inline` 变量 (C++17 起) 或 `constexpr` 变量。
### ​代码示例解析
```cpp
template <typename T>
class Foo
{
public:
  // 短函数 (getter)，直接在公共声明点定义 -> 允许 (短 + 隐式 inline)
  int bar() { return bar_; }

  // 长函数声明。定义不能放在这里污染公共接口。
  void MethodWithHugeBody();

private:
  int bar_;
};

// Implementation details only below here
// **************** 内部实现区域分隔线 ****************

// 长函数定义放在这里 (头文件内部实现区域)
template <typename T>
void Foo<T>::MethodWithHugeBody()
{
  ... // 可能很长的实现代码
}
```
- `bar()`：是一个简单的 `getter` 函数，非常短（一行）。它直接在类定义的 `public:` 部分定义。这是允许的，因为它短小，并且作为类内定义的成员函数，它是​**​隐式 `inline`​**​ 的（满足 ODR-safe）。
- `MethodWithHugeBody()`：声明在 `public:` 部分（它是公共接口）。但它的​**​定义​**​被移到了类定义之后、用注释明确标记的​**​内部实现区域​**​。这样保证了公共接口的清晰。因为它是一个​**​模板成员函数​**​，所以它的定义​**​必须​**​在头文件中（技术必要性），并且模板本身保证了 ODR-safe。
### ​​总结​
Google C++ 规范主张保持头文件（尤其是公共 API 部分）的​**​简洁和声明性​**​。函数实现细节应尽量放在 `.cc` 文件中。如果必须在头文件中定义函数（短函数、模板、`constexpr`），应将其放在专门的内部区域，并确保其 ODR-safe。这样做的主要目的是​**​提高代码的可读性、可维护性，并清晰地分离接口与实现​**​。

## include的名称和顺序
1. 不要用`./`、`../`，应该列为项目源目录的后代，比如`project/src/base/logging.h`应包含为`#include "base/logging.h"`
### 尖括号
仅当库要求你这么做时，才去使用尖括号路径包含标头。
1. C和`C++`标准库头文件。例如`<stdlib.h>`、`<string>`
2. POSIX、Linux、Windows 系统头文件，例如`<unistd.h>`、`<windows.h>`
3. 第三方库，在极少数情况下：如`<Python.h>`
### include顺序
在 `dir/foo.cc` 或 `dir/foo_test.cc` ，其主要目的是实现或测试 `dir2/foo2.h` ，包含顺序如下：
1. `dir2/foo2.h`
2. 一个空白行
3. C 系统头文件，以及尖括号中的任何其他头文件 `.h` 扩展名，例如 `<unistd.h>` ， `<stdlib.h>` 、 `<Python.h>` 。
4. 一个空白行
5. C++ 标准库头文件（不带文件扩展名），例如， `<algorithm>` ， `<cstddef>` 。
6. 一个空白行
7. 其他库的`.h`文件。
8. 一个空白行
9. 项目自己的 `.h` 文件。

在每个部分内部，包含顺序应按字母顺序排序。

使用这个顺序后，如果相关头文件如 `dir2/foo2.h` 省略了任何必要的includes，那么`dir/foo.cc` 的构建 或 `dir/foo_test.cc` 将中断。因此，此规则确保构建中断首先显示给处理这些文件的人，而不是其他包中的无辜者。

示例
```cpp
#include "foo/server/fooserver.h"

#include <sys/types.h>
#include <unistd.h>

#include <string>
#include <vector>

#include "base/basictypes.h"
#include "foo/server/bar.h"
#include "third_party/absl/flags/flag.h"
```
#### 例外
有时，特定于系统的代码需要条件包含。这样的代码可以将条件包含放在其他包含之后。当然，请保持特定于系统的代码较小且本地化（localized）。例：
```cpp
#include "foo/public/fooserver.h"

#ifdef _WIN32
#include <windows.h>
#endif  // _WIN32
```
### C头文件和C++头文件可以互换
C 头文件（例如 `stddef.h）` 基本上可以与 C++ 对应物互换 （`cstddef`）。任何一种风格都是可以接受的，但最好与现有代码保持一致。
# namespace相关
除了少数例外，都要把代码放到命名空间中。
命名空间的名字应该是基于项目名（多为路径）的唯一名称。


在 includes、 gflags 定义/声明、前置声明之后，放置命名空间，将整个源文件包装。

在命名空间结尾处，加注释标记。
```cpp
namespace mynamespace {

}  // namespace mynamespace
```

单行嵌套命名空间声明是新代码中的首选，但不是必需的。
## using `::foo::Bar`的使用
```cpp
#include "a.h"

ABSL_FLAG(bool, someflag, false, "a flag");

namespace mynamespace {

using ::foo::Bar;

...code for mynamespace...    // Code goes against the left margin.

}  // namespace mynamespace
```

`using ::foo::Bar;`
这条语句的意思是：在 `mynamespace` 这个命名空间内部，允许我直接使用 `Bar` 这个名字来指代 `::foo::Bar`。
它​​不是​​将整个 `foo` 命名空间引入 `mynamespace`（那是 `using namespace foo;` 的作用），而是​​只引入 `Bar` 这一个特定的名字。

为什么这样写？​​
1. 代码简洁性：在 `mynamespace` 内部频繁使用 `::foo::Bar` 时，每次都写全名很冗长。`using ::foo::Bar;` 允许直接使用 `Bar` ，使代码更简洁易读。
2. 明确来源：使用 ::foo::Bar而不是 foo::Bar是一种防御性编程。它明确指定了 foo命名空间位于全局命名空间，避免了可能存在的嵌套命名空间 `mynamespace::foo` 的歧义（如果存在的话）。`::`确保了引用的绝对是全局的 `foo`。
3. 作用域限制：这个 `using` 声明只在 `mynamespace` 内部有效。它不会污染全局命名空间或其他命名空间。这是一种相对安全的引入方式。
### using和（不要）using namespace的区别
using是引入命名空间下的一个符号。
using namespace是引入整个命名空间。

**严禁**：不要用`using namespace xxx;`，这会污染命名空间。
## 内联namespace（不要）
不要用内联namespace。

```cpp
namespace outer {
inline namespace inner {
  void foo();
}  // namespace inner
}  // namespace outer
```
这样的效果是：表达式 `outer::inner::foo()` 和 `outer::foo()` 是可以互换的。

内联命名空间主要用于跨版本的 ABI 兼容性。
## 仅在显式标记的内部空间中使用命名空间别名
命名空间别名：
```cpp
// Remove uninteresting parts of some commonly used names in .cc files.
namespace sidetable = ::pipeline_diagnostics::sidetable;
```

```cpp
// In a .h file, an alias must not be a separate API, or must be hidden in an
// implementation detail.
namespace librarian {

namespace internal {  // Internal, not part of the API.
namespace sidetable = ::pipeline_diagnostics::sidetable;
}  // namespace internal

inline void my_inline_function() {
  // Local to a function.
  namespace baz = ::foo::bar::baz;
  ...
}

}  // namespace librarian
```
## 使用名称带有internal的命名空间隔离API内部细节
标记为 internal 的代码是供​​库或模块自身的开发者​​在实现公共 API 功能内部细节时使用的。
​​库开发者（比如 Absl 维护者）可以在 Abseil 库自己的代码里使用 container_internal里的东西，但外部用户（non-absl code）绝对不应该在他们的代码里使用它。​
它​​严格禁止​​被库的​​外部用户​​（即使用这个库的程序员）在他们的代码中直接引用或依赖。

>absl指的是Abseil库，是 Google 开源的一套 C++ 核心库组件，提供了许多基础数据结构、工具和设施，旨在构建更健壮、更高效的 C++ 代码。

请注意，嵌套内部命名空间中的库之间仍然存在冲突的风险，因此通过添加库的文件名，为命名空间中的每个库提供唯一的内部命名空间。例如，`gshoe/widget.h` 将使用 `gshoe::internal_widget` 而不是 `gshoe::internal`。

## Internal Linkage（文件内部链接）: 匿名namespace和static
限制符号（变量、函数、类等）的作用域，使其仅在单个 `.cc` 文件内可见​​。
1. ​​内部链接 (Internal Linkage)：​​
    - 指符号的链接属性，使得该符号​**​仅在定义它的翻译单元（通常就是一个 `.cc` 或 `.cpp` 源文件）内可见和可用​**​。
    - 其他文件（翻译单元）即使知道该符号的名字，也无法访问或链接到它。
    - 如果另一个文件定义了一个同名符号，它们是完全独立的两个实体，互不影响。
2. ​​目的：​​
    - ​**​封装与隔离：​**​ 将只在单个文件内部使用的辅助函数、变量、常量或类型隐藏起来，避免污染全局命名空间。
    - ​**​避免命名冲突：​**​ 防止不同文件中的辅助符号（如 `helperFunction()`）因同名而发生链接错误或意外覆盖。
    - ​**​编译优化：​**​ 编译器知道这些符号不会被外部引用，可能进行更好的优化。
    - ​**​代码清晰：​**​ 明确标识出哪些符号是文件内部的实现细节。

​实现内部链接的两种方式​：匿名 namespace 或 static 修饰
1. 未命名命名空间 (Unnamed Namespaces / Anonymous Namespaces)：​**​
    - 语法：`namespace { ... }`
    - 效果：将定义在 `{ ... }` 内部的​**​所有符号​**​（类、函数、变量、类型别名等）赋予内部链接。这些符号的作用域被限制在​**​当前文件内​**​。
    - ​**​格式要求：​**​ 像命名空间一样格式化，结尾注释写 `} // namespace`（空名）。
    - ​**​现代 C++ 首选方式：​**​ 这是 C++ 标准推荐的方式，适用于所有类型的符号。

匿名命名空间结尾也要有注释，`// namespace`
```cpp
// myfile.cc
namespace { // 开始未命名命名空间
    int helperVariable = 42; // 内部链接，仅本文件可见

    void helperFunction() { // 内部链接，仅本文件可调用
        // ... do something ...
    }

    class InternalClass { // 内部链接，仅本文件可用
        // ...
    };
}  // namespace (结束，无名)
```
    
2. ​`static` 关键字：​​
    - 语法：在函数或变量的声明前加 `static`。
    - 效果：将​**​函数或全局变量​**​赋予内部链接。​**​不能用于类定义或类型别名。​**​
    - ​**​传统方式：​**​ 在 C 和早期 C++ 中常用，但在现代 C++ 中，对于文件作用域的符号，未命名命名空间通常是更好的选择。
    
    ```cpp
    // myfile.cc
    static int helperVariable = 42; // 内部链接 (static 变量)
    static void helperFunction() {  // 内部链接 (static 函数)
        // ... do something ...
    }
    // static 不能用于类：static class InternalClass {}; // 错误！
    ```
### ​强调​：鼓励在cc文件中使用，禁止在h文件中使用
1. ​**​在 `.cc` 文件中使用：​**
    - ​**​强烈鼓励：​**​ 对于 `.cc` 文件中定义的、​**​不需要被其他 `.cc` 或 `.h` 文件引用​**​的任何符号（辅助函数、内部状态变量、实现类等），都应该使用​**​未命名命名空间​**​或 `static` (仅限函数/变量) 来赋予它们​**​内部链接​**​。这是最佳实践。
2. ​**​禁止在 `.h` 文件中使用：​**​
    - ​**​绝对不要​**​在头文件 (`.h`) 中使用未命名命名空间或 `static` 声明函数/变量。
    - ​**​原因：​**​
        - ​**​违反 ODR (单一定义规则)：​**​ 头文件会被多个 `.cc` 文件包含。如果头文件里有 `static int globalVar;`，那么每个包含该头文件的 `.cc` 文件都会获得一个​**​独立的、名为 `globalVar` 的副本​**​。这通常不是想要的效果，且可能导致内存浪费或逻辑错误。
        - ​**​未命名命名空间同理：​**​ 每个包含该头文件的 `.cc` 文件都会有一个​**​独立的、内容相同但彼此隔离​**​的未命名命名空间副本。这同样违反 ODR 的意图（期望全局唯一），并可能导致奇怪的链接或运行时行为。
        - ​**​头文件的目的是声明接口：​**​ 头文件应该声明那些需要被其他文件​**​使用​**​的符号（通常是外部链接）。内部实现细节不应该出现在公共头文件里。
# 其他Scoping相关
- [Nonmember, Static Member, and Global Functions](https://google.github.io/styleguide/cppguide.html#Nonmember,_Static_Member,_and_Global_Functions)
- [Local Variables](https://google.github.io/styleguide/cppguide.html#Local_Variables)
- [Static and Global Variables](https://google.github.io/styleguide/cppguide.html#Static_and_Global_Variables)
- [thread_local Variables](https://google.github.io/styleguide/cppguide.html#thread_local)
# 类相关
- [Doing Work in Constructors](https://google.github.io/styleguide/cppguide.html#Doing_Work_in_Constructors)
- [Implicit Conversions](https://google.github.io/styleguide/cppguide.html#Implicit_Conversions)
- [Copyable and Movable Types](https://google.github.io/styleguide/cppguide.html#Copyable_Movable_Types)
- [Structs vs. Classes](https://google.github.io/styleguide/cppguide.html#Structs_vs._Classes)
- [Structs vs. Pairs and Tuples](https://google.github.io/styleguide/cppguide.html#Structs_vs._Tuples)
- [Inheritance](https://google.github.io/styleguide/cppguide.html#Inheritance)
- [Operator Overloading](https://google.github.io/styleguide/cppguide.html#Operator_Overloading)
- [Access Control](https://google.github.io/styleguide/cppguide.html#Access_Control)
- [Declaration Order](https://google.github.io/styleguide/cppguide.html#Declaration_Order)
# 函数相关
- [Inputs and Outputs](https://google.github.io/styleguide/cppguide.html#Inputs_and_Outputs)
- [Write Short Functions](https://google.github.io/styleguide/cppguide.html#Write_Short_Functions)
- [Function Overloading](https://google.github.io/styleguide/cppguide.html#Function_Overloading)
- [Default Arguments](https://google.github.io/styleguide/cppguide.html#Default_Arguments)
- [Trailing Return Type Syntax](https://google.github.io/styleguide/cppguide.html#trailing_return)
# C++特性相关
- [Rvalue References](https://google.github.io/styleguide/cppguide.html#Rvalue_references)
- [Friends](https://google.github.io/styleguide/cppguide.html#Friends)
- [Exceptions](https://google.github.io/styleguide/cppguide.html#Exceptions)
- [noexcept](https://google.github.io/styleguide/cppguide.html#noexcept)
- [Run-Time Type Information (RTTI)](https://google.github.io/styleguide/cppguide.html#Run-Time_Type_Information__RTTI_)
- [Casting](https://google.github.io/styleguide/cppguide.html#Casting)
- [Streams](https://google.github.io/styleguide/cppguide.html#Streams)
- [Preincrement and Predecrement](https://google.github.io/styleguide/cppguide.html#Preincrement_and_Predecrement)
- [Use of const](https://google.github.io/styleguide/cppguide.html#Use_of_const)
- [Use of constexpr, constinit, and consteval](https://google.github.io/styleguide/cppguide.html#Use_of_constexpr)
- [Integer Types](https://google.github.io/styleguide/cppguide.html#Integer_Types)
- [Floating-Point Types](https://google.github.io/styleguide/cppguide.html#Floating-Point_Types)
- [Architecture Portability](https://google.github.io/styleguide/cppguide.html#Architecture_Portability)
- [Preprocessor Macros](https://google.github.io/styleguide/cppguide.html#Preprocessor_Macros)
- [0 and nullptr/NULL](https://google.github.io/styleguide/cppguide.html#0_and_nullptr/NULL)
- [sizeof](https://google.github.io/styleguide/cppguide.html#sizeof)
- [Type Deduction (including auto)](https://google.github.io/styleguide/cppguide.html#Type_deduction)
- [Class Template Argument Deduction](https://google.github.io/styleguide/cppguide.html#CTAD)
- [Designated Initializers](https://google.github.io/styleguide/cppguide.html#Designated_initializers)
- [Lambda Expressions](https://google.github.io/styleguide/cppguide.html#Lambda_expressions)
- [Template Metaprogramming](https://google.github.io/styleguide/cppguide.html#Template_metaprogramming)
- [Concepts and Constraints](https://google.github.io/styleguide/cppguide.html#Concepts)
- [C++20 modules](https://google.github.io/styleguide/cppguide.html#modules)
- [Coroutines](https://google.github.io/styleguide/cppguide.html#coroutines)
- [Boost](https://google.github.io/styleguide/cppguide.html#Boost)
- [Disallowed standard library features](https://google.github.io/styleguide/cppguide.html#Disallowed_Stdlib)
- [Nonstandard Extensions](https://google.github.io/styleguide/cppguide.html#Nonstandard_Extensions)
- [Aliases](https://google.github.io/styleguide/cppguide.html#Aliases)
- [Switch Statements](https://google.github.io/styleguide/cppguide.html#Switch_Statements)
# 命名专题
## 文件名（my_awesome_class）全小写，单词间可以用下划线连接

可接受的文件名示例：
```
my_useful_class.cc
my-useful-class.cc
myusefulclass.cc
myusefulclass_test.cc // _unittest and _regtest are deprecated.
```
首选：全小写，单词间可以用下划线连接
### 代码文件扩展名
`C++` 文件应具有 `.cc` 文件扩展名，头文件应具有 `.h` 扩展名。依赖于在特定点以文本方式包含的文件应以 `.inc` 结尾（另请参阅**自包含的标头** (Self-contained Headers)）。

## 类型名（MyAwesomeClass）
所有类型的名称（类、结构、类型别名、枚举和类型模板参数）都具有相同的命名约定。**类型名称应以大写字母开头，每个新单词都有一个大写字母**。没有下划线：`MyExcitingClass`、`MyExcitingEnum`。

```cpp
// classes and structs
class UrlTable { ...
class UrlTableTester { ...
struct UrlTableProperties { ...

// typedefs
typedef hash_map<UrlTableProperties *, std::string> PropertiesMap;

// using aliases
using PropertiesMap = hash_map<UrlTableProperties *, std::string>;

// enums
enum class UrlTableError { ...
```

## Concept 名称（同类型名称规则）
## 变量名称（snake_case）
1. 变量（包括函数参数）的名称是 `snake_case`（全部小写，单词之间带有下划线）。如：`a_local_variable`。
2. 类的数据成员 （但不是结构体）还有尾随下划线。例如：`a_class_data_member_`。
3. 结构体数据成员：`a_struct_data_member`。
### Common 变量名称（snake_case）
For example:  例如：
```cpp
std::string table_name;  // OK - snake_case.
```

```cpp
std::string tableName;   // Bad - mixed case.
```
### 类数据成员（`snake_case_`，右端尾随下划线）
包括静态、非静态。都按照`snake_case_`。

**静态常量**数据成员是例外，遵循“常量命名”规则。

```cpp
class TableInfo {
    public:
    ...
    static const int kTableVersion = 3;  // OK - constant naming.
    ...
    
    private:
    std::string table_name_;             // OK - underscore at end.
    static Pool<TableInfo>* pool_;       // OK.
};
```

### 结构体数据成员（同Common 变量名称，snake_case，右端没有尾随下划线）
结构体的数据成员，包括静态和非静态，其命名方式与普通非成员变量类似。右端没有类数据成员那样的尾随下划线。

```cpp
struct UrlTableProperties {
    std::string name;
    int num_entries;
    static Pool<UrlTableProperties>* pool;
};
```

### 拓展：结构体与类如何选择？
结构体和类关键字在 C++ 中的行为几乎相同。
#### 何时使用结构体
何时使用结构体作为抽象：
1. 仅对于**携带数据的被动对象**，
2. 并且它可能具有关联的常量。
3. 所有字段都必须是公共的。
4. **结构体不得包含暗示不同字段间关系的不变量**。
5. struct 也可以拥有函数，但这些函数的职责应仅限于​​数据本身的简单操作​​，
    1. 例如初始化（构造函数）、清理（析构函数）、或打印数据。
    2. 它们​​不应包含复杂的业务逻辑​​，也不能试图去保护和维持数据之间的隐藏关系（不变量）。

什么是具有关联的常量？
struct内部除了数据成员，还可以定义与这些数据成员紧密相关的常量，例如枚举值或静态常量。这些常量用于描述或分类该数据结构，不破坏其**被动数据**的本质。
示例：
```cpp
struct Configuration {
    // 数据成员
    int resolutionWidth;
    int resolutionHeight;
    // 关联常量 - 用于描述或分类数据
    enum Quality { Low, Medium, High };
    Quality currentQuality;
};
```

**结构体不得包含暗示不同字段间关系的不变量**是什么意思？
**含义​**​：这是最核心的一条限制。“不变量”指的是对象在其生命周期内必须始终保持为真的​**​一种状态或关系​**​。
例如，在一个 `class`中，`age`字段必须大于 0 就是一个典型的不变量。
由于`struct`的字段都是公开的，使用者可能直接修改任意字段。如果字段之间存在某种隐含的依赖或关系（即“不变量”），就很容易破坏这种关系，导致数据状态不一致。

反面示例：
```cpp
// 一个“账户” struct，其字段间存在强不变性约束：余额不能为负。
struct BadBankAccount {
    double balance; // 余额
    double overdraftLimit; // 透支额度
    // 问题：用户可以直接修改 balance 为任意值，例如 -10000，
    // 这直接破坏了“余额不能低于透支额度”的业务逻辑（不变量）。
};
```

​**​STL 中的例外​**​：在标准模板库中，`struct`常被用于​**​无状态的类型​**​，如traits、模板元函数和仿函数，这是因为其默认的公有访问性更为方便。

除了以上描述的 struct 的适用场景，其余情况都用类。

**技术无差别，约定成俗​**​：从编译器角度看，`struct`和 `class`的唯一区别就是默认访问权限。所有其他的区别都是​**​程序员之间形成的约定​**​，旨在让代码更易读、更易维护。
### 常量名称（kMyConstantVar）
先说关键点：是否要用 k 前导，关键看对象的**存储期限类型**、以及**是否是常量**，如果两者都满足，才适用于用 k 前导。

一个对象，**其值在程序期限内是固定的（编译期或生命周期内固定）**，以前导小写 `k` 命名，后跟大小写混合，以大写分隔单词，不带下划线。
```cpp
const int kDaysInAWeek = 7;
```
如果后面的字是无法区分大小写的，那就用下划线作为分隔。
```cpp
const int kAndroid8_0_0 = 24; // Android 8.0.0
```
#### k 代表的含义和 const 修饰无关
const 只保证 对象 的值 在一段期间是常量（比如一次函数调用期间），但不保证在整个程序周期内 不变。

如果你确定整个程序周期内这个对象不变，那就可以用 k ，否则不要用。
```cpp
void ComputeFoo(absl::string_view suffix) {
    // 两种方式都可以接受 (Either of these is acceptable)
    const absl::string_view kPrefix = "prefix"; // 使用 kPrefix
    const absl::string_view prefix = "prefix";  // 使用 prefix
    ...
}
```
#### 非常容易出错的点
试图将一个​**​运行时才能确定的值​**​放入一个按规范应代表“编译期或生命周期内固定”的 `k`常量中。这是错误的，因为 **`k`命名的变量暗示其值是不可变的且每次调用都相同**。
```cpp
void ComputeFoo(absl::string_view suffix) {
    // 错误示例 (Bad) - kCombined 的值会随着每次调用 ComputeFoo 时传入的 suffix 不同而不同！
    const std::string kCombined = absl::StrCat(kPrefix, suffix);
    ...
}
```
#### 拓展：Storage Duration是什么？
https://en.cppreference.com/w/cpp/language/storage_duration.html#Storage_duration
存储期限是对象的属性，它定义了包含该对象的存储的最小潜在生存期。存储期限由用于创建对象的构造决定，并且是以下内容之一：
1. static storage duration：静态存储期限
    1. static 关键字
2. thread storage duration (also known as thread-local storage duration)：线程本地存储期限
    1. thread_local 关键字
3. automatic storage duration：自动存储期限
    1. auto 关键字（until `C++11`）
4. dynamic storage duration：动态存储期限
    1. 与 new 、 delete 相关

静态 、线程本地、自动存储期限与声明引入的对象和临时对象相关联。
动态存储期限与 new 表达式创建的对象或隐式创建的对象相关联。

存储说明符：auto、register、static、thread_local、extern、mutable。

## 函数名称（MyFunction）
遵循大写字母开头，每个新单词都有一个大写字母分隔。
```cpp
AddTableEntry()
DeleteUrl()
OpenFileOrDie()
```
## 命名空间名称（snake_case）
全部小写，单词之间带有下划线。

## 枚举器（Enumerator）名称（用 kEnumName 而不是 ENUM_NAME）
2009 年 1 月之前，枚举类用宏的风格命名枚举值。这导致枚举值和宏之间的名称冲突出现问题。因此，更改为首选常量样式命名。新代码应使用常量样式命名。
```cpp
// ok!
enum class UrlTableError {
  kOk = 0,
  kOutOfMemory,
  kMalformedInput,
};
```

```cpp
// no!
enum class AlternateUrlTableError {
  OK = 0,
  OUT_OF_MEMORY = 1,
  MALFORMED_INPUT = 2,
};
```
## 模板参数名称（类型模板参数按类型名称；非类型模板参数按变量、常量名称）
1. 类型模板参数 应遵循 类型名称 风格
2. 非类型模板参数 应遵循 变量或常量的 风格
## 宏名称（全大写、下划线分隔）
一般来说，不应使用宏。但是，如果绝对需要它们，则应使用所有大写字母和下划线来命名它们，并带有特定于项目的前缀。
```cpp
#define MYPROJECT_ROUND(x) ...
```
## 别名（Aliases）
别名的名称遵循与任何其他新名称相同的原则，应用于定义别名的上下文，而不是在原始名称出现的位置。
## 命名规则的例外情况
If you are naming something that is analogous to an existing C or C++ entity then you can follow the existing naming convention scheme.
如果您要命名类似于现有 C 或 C++ 实体的内容，则可以遵循现有的命名约定方案。
1. `bigopen()`
    1. function name, follows form of open()
2. uint
    1. typedef
3. bigpos
    1. struct or class, follows form of pos
4. sparse_hash_map
    1. STL-like entity; follows STL naming conventions
5. LONGLONG_MAX
    1. a constant, as in INT_MAX
# 注释格式
使用`//`或 `/* */` 语法，只要你保持和现有代码一致。最好首选`//`。
## 文件注释
每个文件都以许可证样板（license boilerplate）开头。

如果一个源文件（例如 `.h` 文件）声明了多个面向用户的外部抽象（常见的函数、相关的类等），应包含一个描述这些抽象集合的注释。注释应包含足够的信息，以便未来的作者知道哪些内容不适合放在这里。然而，关于各个抽象的详细文档应属于这些抽象本身，而不是文件级别。

例如，如果你为 `frobber.h` 编写文件注释，你不需要在 `frobber.cc` 或 `frobber_test.cc` 中包含文件注释。另一方面，如果你在 `registered_objects.cc` 中编写了一组没有相关头文件的类，你必须要在 `registered_objects.cc` 中包含文件注释。
### 法律声明和作者行
每个文件都应包含许可证模板。选择适合项目所用许可证的模板（例如，Apache 2.0、BSD、LGPL、GPL）。

如果对带有作者行的文件进行了重大修改，可以考虑删除作者行。新文件通常不应包含版权声明或作者行。

## 结构体和类注释
每个**非显而易见**的类或结构体声明都应该有一个相应的注释，描述它的用途以及如何使用它。

```cpp
// Iterates over the contents of a GargantuanTable.
// Example:
//    std::unique_ptr<GargantuanTableIterator> iter = table->NewIterator();
//    for (iter->Seek("foo"); !iter->done(); iter->Next()) {
//      process(iter->key(), iter->value());
//    }
class GargantuanTableIterator {
    ...
};
```
### 类注释
类注释应该向读者提供足够的信息，让他们知道如何以及何时使用该类，以及正确使用该类所需的任何额外注意事项。如果类有任何同步假设，应记录这些假设。如果类的实例可以被多个线程访问，需要特别小心地记录多线程使用相关的规则和不变量。

类注释通常是一个好地方，用来可以放一小段示例代码，展示该类的简单和专注的使用方式。

当函数足够分离（例如， `.h` 和 `.cc` 文件）时，描述类使用的注释应与其接口定义放在一起；关于类操作和实现的注释应伴随类方法的实现。
## 函数注释
声明注释描述函数的使用（当使用不明显时）；函数定义处的注释描述其操作。
### 函数声明
几乎每个函数声明前都应该有注释，描述函数的作用和使用方法。只有当函数简单且显而易见时（例如，类中简单访问明显属性的方法），才可省略这些注释。
在 `.cc` 文件中声明的私有方法和函数也不例外。

函数注释应以"此函数"（_This function_）为隐含主语，并以动词短语（verb phrase）开头；例如，"打开文件"（_Opens the file_），而不是"打开文件"（_Open the file_）。
通常，这些注释不描述函数如何执行任务。相反，这些细节应留给函数定义中的注释。

在函数声明注释中应提及的事项类型：
1. 输入和输出是什么。如果函数参数名称用反引号括起来，那么代码索引工具可能能够更好地展示文档。
2. 对于类成员函数：对象是否在方法调用持续时间之外记住引用或指针参数。这对于构造函数的指针/引用参数来说非常常见。
3. 对于每个指针参数，是否允许其为空，如果为空会发生什么。
4. 对于每个输出或输入/输出参数，该参数中的任何状态会发生什么（例如，状态是被追加还是被覆盖？）。
5. 如果一个函数的使用存在性能影响。

例子
```cpp
// Returns an iterator for this table, positioned at the first entry
// lexically greater than or equal to `start_word`. If there is no
// such entry, returns a null pointer. The client must not use the
// iterator after the underlying GargantuanTable has been destroyed.
//
// This method is equivalent to:
//    std::unique_ptr<Iterator> iter = table->NewIterator();
//    iter->Seek(start_word);
//    return iter;
std::unique_ptr<Iterator> GetIterator(absl::string_view start_word) const;
```
#### override的注释
对于函数重写（override）。应关注重写之后的细节，而不是重复原抽象函数的注释。在许多情况下，重写不需要额外的文档说明，因此无需注释。

#### 构造、析构的注释
在注释构造函数和析构函数时，要让读你代码的人知道构造函数和析构函数的用途，因此仅说明“销毁此对象”之类的注释没有用处。
应记录构造函数如何使用它们的参数（例如，如果它们接管指针的所有权），
以及析构函数执行了哪些清理工作。

如果这些很平凡，就可以省略注释。
析构函数没有头部注释是非常常见的。
### 函数定义
如果一个函数的工作方式有任何复杂之处，函数定义应该有一个解释性注释。
例如，在定义性注释中，你可以描述任何你使用的编程技巧，概述你经过的步骤，或者解释为什么你选择以这种方式实现函数而不是使用一个可行的替代方案。
例如，你可能会提到为什么函数的前半部分必须获取锁，但后半部分不需要。

请注意，不要仅仅重复函数声明中、 .h 文件或其他地方的注释。简要概括函数的作用是可以的，但注释的重点应该是它如何实现这一点。
## 变量注释
一般来说，变量的实际名称应该足够描述性，以便清楚地表明其用途。在某些情况下，需要更多的注释。
### 类数据成员
每个类数据成员（也称为实例变量或成员变量）的目的必须明确。如果类型和名称未能清楚地表达任何不变式（特殊值、成员之间的关系、生命周期要求），则必须进行注释。然而，如果类型和名称已经足够（ `int num_events_;`），则不需要注释。
特别是，当哨兵值（如 nullptr 或-1）的存在和含义不明显时，应添加注释来描述它们。例如：
```cpp
private:
    // Used to bounds-check table accesses. -1 means
    // that we don't yet know how many entries the table has.
    int num_total_entries_;
```
### 全局变量
所有全局变量都应该有注释说明它们是什么、用途是什么，以及（如果不清楚）为什么需要是全局的。例如：
```cpp
// The total number of test cases that we run through in this regression test.
const int kNumTestCases = 6;
```
## 实现（Implementation）注释
在你的实现中，你应该在代码中那些棘手、不明显、有趣或重要的部分添加注释。
### 解释性注释
复杂的代码块前应有注释。
## 函数参数注释
当函数参数的含义不明显时，可以考虑以下补救措施：
- 如果参数是一个字面常量，并且该常量在多个函数调用中被多次使用，隐含地假设它们是相同的，你应该使用常量名称来明确这种约束，并确保其成立。
- 考虑将函数签名更改为用 `enum` 参数替换 `bool` 参数。这将使参数值具有自描述性。
- 对于具有多个配置选项的函数，可以考虑定义一个类或结构体来包含所有选项，并传递该类或结构体的实例。这种方法有几个优点。
    - 在调用点通过名称来引用选项，这可以明确它们的含义。
    - 还减少了函数参数的数量，这使得函数调用更容易阅读和编写。
    - 此外，当你添加另一个选项时，你不必更改调用点
- 用命名变量替换大型或复杂的嵌套表达式。
- 作为最后的手段，使用注释来澄清调用点处参数的含义。

错误示例：
```cpp
// What are these arguments?
const DecimalNumber product = CalculateProduct(values, 7, false, nullptr);
```

正确示例：
```cpp
ProductOptions options;
options.set_precision_decimals(7);
options.set_use_cache(ProductOptions::kDontUseCache);
const DecimalNumber product =
    CalculateProduct(values, options, /*completion_callback=*/nullptr);
```
## 标点符号、拼写、语法
[Punctuation, Spelling, and Grammar](https://google.github.io/styleguide/cppguide.html#Punctuation_Spelling_and_Grammar)
## TODO注释
使用 `TODO` 注释来标记临时的代码、短期解决方案或足够好但并非完美的代码。

`TODO` 应包含大写的字符串 `TODO` ，随后是错误 ID、名称、电子邮件地址或其他标识符，这些标识符应能提供关于 `TODO` 所引用问题的最佳背景信息。

```cpp
// TODO: bug 12345678 - Remove this after the 2047q4 compatibility window expires.
// TODO: example.com/my-design-doc - Manually fix up this code the next time it's touched.
// TODO(bug 12345678): Update this list after the Foo service is turned down.
// TODO(John): Use a "\*" here for concatenation operator.
```

如果你的 `TODO` 形式为 "在未来某个日期做某事"，请确保你包含一个非常具体的日期（"在 2005 年 11 月前修复"）或一个非常具体的事件（"当所有客户端都能处理 XML 响应时移除此代码"）。

# 文档格式化
google 给 emacs 创建的设置文件：https://raw.githubusercontent.com/google/styleguide/gh-pages/google-c-style.el

## 行长度
代码中的每一行文本长度不应超过 80 个字符。
## 编码（关于非 ASCII 字符）
非 ASCII 字符应当罕见；必须使用 UTF-8 格式。

十六进制编码也是可以的，并且当它增强可读性时被鼓励使用——例如， `"\xEF\xBB\xBF"` ，或者更简单的方式， `"\uFEFF"` ，是 Unicode 零宽非断空格字符，如果直接以 UTF-8 形式包含在源代码中将是不可见的。

在可能的情况下，避免使用 `u8` 前缀。它在 C++20 开始与 C++17 中的语义有显著不同，产生 `char8_t` 数组而不是 `char` ，并且将在 C++23 中再次改变。

你不应该使用 `char16_t` 和 `char32_t` 字符类型，因为它们用于非 UTF-8 文本。出于类似的原因，你也应该避免使用 `wchar_t` （除非你正在编写与 Windows API 交互的代码，后者广泛使用 `wchar_t` ）。
## 缩进（Spaces vs. Tabs）
仅使用空格，每次缩进 2 个空格。
使用空格进行缩进。不要在代码中使用制表符。你应该设置你的编辑器在按下制表键时输出空格。
## 函数声明和定义
函数名与返回类型放在同一行。

如果参数能适应，参数也放在同一行。

如果参数列表不能适应单行，像在函数调用中换行参数那样换行参数列表。

```cpp
ReturnType ClassName::FunctionName(Type par_name1, Type par_name2) {
  DoSomething();
  ...
}
```
如果你有太多文本无法放在一行上：
```cpp
ReturnType ClassName::ReallyLongFunctionName(Type par_name1, Type par_name2,
                                             Type par_name3) {
  DoSomething();
  ...
}
```
或者即使第一参数也无法在一行内放下：
```cpp
ReturnType LongClassName::ReallyReallyReallyLongFunctionName(
    Type par_name1,  // 4 space indent
    Type par_name2,
    Type par_name3) {
  DoSomething();  // 2 space indent
  ...
}
```

一些需要注意的：
1. 选择好的参数名
2. 只有当参数在函数定义中未使用时，才可省略参数名。
3. 如果无法将返回类型和函数名放在同一行，则在这两者之间断行。
4. 如果你在函数声明或定义的返回类型后换行，不要缩进。
5. 左括号始终与函数名位于同一行。
6. 函数名和左括号之间永远不加空格。
7. 括号和参数之间永远不加空格。
8. 花括号始终位于函数声明的最后一行的末尾，而不是下一行的开头。
9. 闭花括号可以单独位于最后一行，或者与开花括号位于同一行。
10. 闭括号和开花括号之间应该有一个空格。
11. 所有参数应该尽可能对齐。
12. 默认缩进为 2 个空格。
13. 包装参数的缩进为 4 个空格。

在上下文中显而易见的不用参数可以省略名称：
```cpp
class Foo {
 public:
  Foo(const Foo&) = delete;
  Foo& operator=(const Foo&) = delete;
};
```

对于可能不太明显的未使用参数，应在函数定义中注释掉变量名：
```cpp
class Shape {
 public:
  virtual void Rotate(double radians) = 0;
};

class Circle : public Shape {
 public:
  void Rotate(double radians) override;
};

void Circle::Rotate(double /*radians*/) {}
```
错误示例：
```cpp
// Bad - if someone wants to implement later, it's not clear what the
// variable means.
void Circle::Rotate(double) {}
```

属性和扩展为属性的宏出现在函数声明或定义的最开始，在返回类型之前：
```cpp
  ABSL_ATTRIBUTE_NOINLINE void ExpensiveFunction();
  [[nodiscard]] bool IsOk();
```
## lambda表达式
## 浮点字面量
## 函数调用
## 花括号初始化列表格式
## 循环和分支语句
## 指针和引用表达式和类型
## 布尔表达式
## 返回值
## 变量和数组初始化
## 预处理指令
## 类格式
## 构造函数初始化列表
## 命名空间格式化
## 水平空格
水平空格的使用取决于位置。绝不要在行的末尾放置尾随空格。
## 垂直空白
少用垂直空白；不必要的空行会使整体代码结构更难看清。只在有助于读者理解结构的地方使用空行。

不要在已经通过缩进清晰分隔的地方添加空行，例如代码块的开始或结束处。应使用空行将代码分隔成紧密相关的块，类似于散文中的段落分隔。在一个语句或声明中，通常只在需要保持在行长度限制内，或需要将注释附加到部分内容时插入换行。
# 例外
- [Existing Non-conformant Code](https://google.github.io/styleguide/cppguide.html#Existing_Non-conformant_Code)
- [Windows Code](https://google.github.io/styleguide/cppguide.html#Windows_Code)
# cpplint 检测风格错误
使用 `cpplint.py` 来检测风格错误。[cpplint.py](https://raw.githubusercontent.com/cpplint/cpplint/HEAD/cpplint.py)
`cpplint.py` 是一个读取源文件并识别许多风格错误的工具。它并不完美，既有误报也有漏报，但它仍然是一个有价值的工具。