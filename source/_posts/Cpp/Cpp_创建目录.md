---
title: Cpp_创建目录
categories:
  - [Cpp]
tags:
  - null 
date: 2022/7/29
update:
comments:
published:
---

参考文章：https://blog.csdn.net/guotianqing/article/details/109823501

# system调用

简单粗野，适用于Linux系统。

```cpp
int main(int argc, char **argv)
{
    if(argc != 2)
    {
        cout << "Usage: " << argv[0] << " dir" << endl;
        return -1;
    }
    string path(argv[1]);
    string cmd("mkdir -p " + path);
    int ret = system(cmd.c_str());
    if(ret)
    {
        cout << "create dir error: " << ret << ", :" << stderror(errno) << endl;
        return -1;
    }
    cout << "system cmd create dir ok: " << argv[1] << endl;
    return 0;
}
```

此方式，直接使用系统命令执行Linux下mkdir命令创建目录，缺点是system系统调用时，要处理各种返回值情况。

# mkdir函数

`C++`中提供了用于创建目录的函数，原型如下：

```cpp
int mkdir(const char * path, mode_t mode);
// The file permission bits of the new directory shall be initialized from mode
// If path names a symbolic link, mkdir() shall fail and set errno to [EEXIST].
// Upon successful completion, mkdir() shall return 0. Otherwise, -1 shall be returned, no directory shall be created, and errno shall be set to indicate the error.
```

可能的错误如下：

* EACCES - 访问错误
  * Search permission is denied on a component of the path prefix, or write permission is denied on the parent directory of the directory to be created.
* EEXIST - 已存在
  * The named file exists.
* ELOOP - 链接错误
  * A loop exists in symbolic links encountered during resolution of the path argument.
* EMLINK - 链接达到上限
  * The link count of the parent directory would exceed `LINK_MAX`.
* ENAMETOOLONG - 名字太长
  * The length of the path argument exceeds `PATH_MAX` or a pathname component is longer than `NAME_MAX`.
* ENOENT - 目录前缀有部分不存在
  * A component of the path prefix specified by path does not name an existing directory or path is an empty string.
* ENOSPC - 没有足够空间
  * The file system does not contain enough space to hold the contents of the new directory or to extend the parent directory of the new directory.
* ENOTDIR - 目录前缀有一部分不是目录
  * A component of the path prefix is not a directory.
* EROFS - 文件系统只读
  * The parent directory resides on a read-only file system.

目录的权限如下：

```
User:   S_IRUSR (read), S_IWUSR (write), S_IXUSR (execute)
Group:  S_IRGRP (read), S_IWGRP (write), S_IXGRP (execute)
Others: S_IROTH (read), S_IWOTH (write), S_IXOTH (execute)
```


它们组合在一起，形成了如下快捷方式：

```
Read + Write + Execute: S_IRWXU (User), S_IRWXG (Group), S_IRWXO (Others)
DEFFILEMODE: Equivalent of 0666 = rw-rw-rw-
ACCESSPERMS: Equivalent of 0777 = rwxrwxrwx
```

使用mkdir函数，替换上面程序中的核心代码：

```cpp
ret = mkdir(path.c_str(), S_IRWXU | S_IRWXG | S_IROTH | S_IXOTH);
if(ret && errno == EEXIST)
{
    cout << "dir: " << path << " already exist" << endl;
}
else if(ret)
{
    cout << "create dir error: " << ret << ", :" << strerror(errno) << endl;
    return -1;
}
else
{
    cout << "mkdir create dir succ: " << path << endl;
}
```

# boost库create_directory

```cpp
#include<boost/filesystem.hpp>
if(!boost::filesystem::is_directory(path))
{
    cout << "begin create path: " << path << endl;
    if(!boost::filesystem::create_directory(path))
    {
        cout << "create_directories failed: " << path << endl;
        return -1;
    }
}
else
{
    cout << path << " already exist" << endl;
}
```

编译时增加`-lboost_system -lboost_filesystem`选项。

注意，以上函数只能用于创建单个目录，不能创建多级目录，否则会抛异常

```
begin create signal_dir: ./c/d
terminate called after throwing an instance of 'boost::filesystem::filesystem_error'
  what():  boost::filesystem::create_directory: No such file or directory: "./c/d"
[1]    16669 abort (core dumped)  ./a.out ./c/d
```

如果需要一次创建多级目录，需要调用`create_directories`

# C++17库create_directory

原型如下：

```cpp
// 以所有权限创建单个目录，父目录必须存在，如果目录已经存在，不会报错
bool create_directory(const std::filesystem::path& p);
bool create_directory(const std::filesystem::path& p, std::error_code& ec) noexcept;
// 与上面一样，只不过创建的目录的权限是从已经存在的目录属性拷贝过来的。这个特性依赖操作系统
bool create_directory(const std::filesystem::path& p, const std::filesystem::path& existing_p);
bool create_directory(const std::filesystem::path& p, const std::filesystem::path& existing_p, std::error_code& ec) noexcept;
// 创建层级目录，为每级不存在的目录调用第一个函数，如果目录已经存在，它什么也不做，且不会报错
bool create_directories(const std::filesystem::path& p);
bool create_directories(const std::filesystem::path& p, std::error_code& ec);
// 创建成功返回true，否则false
```

使用与boost类似，示例代码如下：

```cpp
#include <filesystem>
if (std::filesystem::create_directories(path))
{
	cout << "fs create dir succ: " << path << endl;
}
```
