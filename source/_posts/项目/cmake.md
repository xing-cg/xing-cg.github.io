---
title: cmake
categories:
  - - 项目
  - - Linux
tags: 
date: 2022/4/10
updated: 
comments: 
published:
---

# 内容

1. cmake

# 概述

是个编译构建工具，相当于对makefile以及其之下的g++/gcc包装了一层。makefile语法比较难记，而cmake比较容易，适合用来编译大型项目。

# 示例

```cmake
# 环境变量 PROJECT_SOURCE_DIR
cmake_minimum_required(VERSION 3.0.0)

project(distributed-system-framework)
set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} -g)
# 可执行文件的输出路径，PROJECT_SOURCE_DIR为项目的根目录
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

include_directories(${PROJECT_SOURCE_DIR}/include)
include_directories(${PROJECT_SOURCE_DIR}/include/utils)
include_directories(${PROJECT_SOURCE_DIR}/include/logger)
# 指明CMakelists的子目录位置
add_subdirectory(src)
```

# 尝试

首先文件名是CMakeLists.txt，严格大小写。

写一个hello world，用cmake编译。

```cmake
# ~/vscode/0411_test/CMakeLists.txt
cmake_minimum_required(VERSION 3.0.0)
project(helloworld)
# 指明项目下面的源文件都有哪些，"."指当前目录，SRC是自定义的变量名
aux_source_directory(. SRC)
# 指定二进制可执行文件的输出路径
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/bin)

# 附加编译时的参数
add_definitions("-g")
# 用${SRC}目录下的全部源文件生成"helloworld"可执行文件
add_executable(helloworld ${SRC})
```

```cpp
// ~/vscode/0411_test/main.c
#include<stdio.h>
int main()
{
    printf("\033[35mhello\n");
    printf("\033[36mhello\n");
    printf("\033[37mhello\n");
    return 0;
}
```

```bash
# 当前目录处于~/vscode/0411_test
mkdir build
cd build
cmake .. # cmake寻找上级目录中CMakeLists，把构建文件输出到当前目录（build目录）
make # cmake构建到build目录下一个Makefile文件，在此目录下make即可编译项目
# 因为CMakeLists.txt指明把可执行文件生成到项目根目录下的bin目录，所以make也到bin
cd ../bin
./helloworld
```

# 重写muduo项目下的cmake

```cmake
cmake_minimum_required(VERSION 2.5)
project(mymuduo)

#mymuduo 最终编译成so动态库，设置动态库的路径
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)#注意不是OUTPUT_DIRECTORY.这两者有区别
#设置为调试模式 以及 声明C++11语言标准
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -std=c++11 -fPIC")#在较新的编译器后需要加-fPIC，以示生成的是动态库
#定义参与编译的源文件 起一个别名
aux_source_directory(. SRC_LIST)
#编译生成动态库mymuduo
add_library(mymuduo SHARED ${SRC_LIST})
```

