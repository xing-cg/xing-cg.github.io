---
title: Cpp_文件库
categories:
  - - Cpp
    - Modern
tags: 
date: 2024/5/22
updated: 
comments: 
published:
---
`C++17`之后，有新引入的文件库。屏蔽了操作系统的差异性，有统一的标准。
`<filesystem>`
示例：读取当前目录下（包括所有子目录）的Cpp、C相关的文件名，并统计其行数。

遍历搜索：
```cpp
#include <filesystem>
std::vector<std::string>
file_directories()
{
    std::vector<std::filesystem::directory_entry> entries;
    std::copy_if(std::filesystem::recursive_directory_iterator("./"),
    std::filesystem::recursive_directory_iterator(),
    std::back_inserter(entries),
    [](std::filesystem::directory_entry const& p) -> boll
    {
        if (p.is_regular_file())
            if (p.path().extension().c_str() == L".cpp" ||
                p.path().extension().c_str() == L".h" ||
                p.path().extension().c_str() == L".c")
                return true;
        return false;
    });
    std::vector<std::string> file_names(entries.size());
    // entry -> string name
    std::transform(
        entries.cbegin(),
        entries.cend(),
        file_names.begin(),
        [](std::filesystem::directory_entry const& p) -> std::string
        {
            return p.path().generic_string();
        });
    return file_names;
}
```
在VS环境下，默认的当前文件目录是`ProjectName.vcxproj`所在的位置。

统计行数：通过`<fstream>`中的`std::ifstream`构造函数填入文件名打开文件，已经处理好了RAII，无需考虑关闭。
```cpp
std::vector<int>
count_lines_in_files(std::vector<string::string> & file_names)
{
    std::vector<int> line_counts;
    char ch = 0;
    int line_count{ 0 };

    for (auto const & file_name : file_names)
    {
        std::ifstream in{ file_name };
        line_count = 0;
        while (in.get(ch))
        {
            if (ch == '\n')
            {
                if (ch == '\n')
                    ++line_count;
            }
        }
        line_counts.emplace_back(line_count);
        // in.close();  // RAII
    }
    return line_counts;
}
```
测试
```cpp
int main()
{
    auto file_names = file_directories();
    auto file_counts = count_lines_in_files(file_names);
    std::for_each(
        file_names.cbegin(),
        file_names.cend(),
        [](auto&& file_name) -> void
        {
            std::cout << file_name << std::endl;
        });
    std::for_each(
        file_counts.cbegin(),
        file_counts.cend(),
        [](auto&& file_count) -> void
        {
            std::cout << file_count << std::endl;
        });
    std::cout << "Files Count: " << file_names.size() << std::endl;
}
```
# 结合算法库
结合算法库，可以大大简化我们自己写的代码。尤其是繁杂的for循环。

`istream`虽然不是容器，但也有相应的迭代器，在`<iterator>`中有`Predefined iterators`其中的`istream_iterator`。
如果读写比较频繁，有效率更高的`istreambuf_iterator`。
```cpp
int count_lines(std::string const& file_name)
{
    std::ifstream in{ file_name };
    return std::count(
        std::istreambuf_iterator<char>(in),
        std::istreambuf_iterator<char>(),/* 不写默认是end */
        '\n');
}
```
之前的函数可以简化为：
```cpp
std::vector<int>
count_lines_in_files(std::vector<string::string> & file_names)
{
    std::vector<int> line_counts;
    for (auto const& file_name : file_names)
    {
        line_counts.emplace_back(count_lines(file_name));
    }
    return line_counts;
}
```
还能再次改进：
```cpp
std::vector<int>
count_lines_in_files(std::vector<string::string> & file_names)
{
    std::vector<int> line_counts(file_names.size());

    std::transform(
        std::execution::par,
        file_names.cbegin(),
        file_names.cend(),
        line_counts.begin(),
        count_lines);
    return line_counts;
}
```
也可以结合ranges：一行代码解决！
>`C++23`中可以通过`std::ranges::to<Container>`来输出到就地生成的容器中
```cpp
std::vector<int>
count_lines_in_files(std::vector<string::string> & file_names)
{
    return file_names | std::views::transform(count_lines) |
        std::ranges::to<std::vector>();
}
```