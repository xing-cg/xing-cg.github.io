---
title: tRPC-Cpp开发实录
categories:
  - - 项目
tags: 
date: 2025/8/5
updated: 2025/8/6
comments: 
published: false
---
# cmake 问题
虽然官方宣称 3.14及以后版本的 cmake 的 FetchContent 特性在编译源码时可以拉取第三方依赖，但是实际编译过程中（`cmake`版本`3.16.3`，运行的项目根目录下的`./run_examples_cmake.sh`）不行。
需要安装很多第三方依赖：
1. ​​OpenSSL & ZLIB​​（基础加密和压缩库）
2. Python 开发环境​
3. XML 处理库​
4. Libevent​​（事件通知库）
5. ​​Libev​​（高性能事件循环）
6. ​​c-ares​​（异步DNS解析）
7. Jansson​​（JSON 处理库）
8. CUnit​​（单元测试框架）
9. ​Jemalloc​​（高效内存分配器）
10. `nlohmann_json`
11. gperftools（Google Performance Tools）：_tcmalloc_ 高性能内存分配器和 CPU Profiler 功能
12. `curl` 和 `curl` 的开发库

```sh
sudo apt update
sudo apt install -y libssl-dev zlib1g-dev

sudo apt install -y python3-dev cython3 python3-pip
pip3 install cython

sudo apt install -y libxml2-dev

sudo apt install -y libevent-dev

sudo apt install -y libev-dev

sudo apt install -y libc-ares-dev

sudo apt install -y libjansson-dev

sudo apt install -y libcunit1-dev

sudo apt install -y libjemalloc-dev

sudo apt install nlohmann-json3-dev # Ubuntu 20.04+

sudo apt-get install libgoogle-perftools-dev

sudo apt install curl
sudo apt-get install libcurl4-openssl-dev

sudo ldconfig
```

验证依赖安装成功：
```sh
# 检查 OpenSSL
openssl version

# 检查 Python 头文件
ls /usr/include/python3.*/Python.h

# 检查 libevent
pkg-config --modversion libevent

# 检查 curl 开发包是否安装
dpkg -l | grep libcurl-dev # Debian/Ubuntu
# 查找 libcurl 文件
find /usr -name 'libcurl*.so*' 2>/dev/null
# 或者检查标准库路径
ls /usr/lib/x86_64-linux-gnu/libcurl*       # Ubuntu
```

编译实录：我们没有安装依赖时，不管是创建build并且cd build后自己`cmake ..`，亦或是直接`./run_cmake_examples.sh`，提示上述的这些依赖缺少。
依赖安装成功后，我们没有删除第一次编译出来的半成品文件，再次编译，会提示：
```
-- Found ZLIB: /usr/lib/x86_64-linux-gnu/libz.so (found version "1.2.11") error: patch failed: include/spdlog/details/periodic_worker-inl.h:10 error: include/spdlog/details/periodic_worker-inl.h: patch does not apply error: patch failed: include/spdlog/details/periodic_worker.h:20 error: include/spdlog/details/periodic_worker.h: patch does not apply error: patch failed: include/spdlog/details/registry-inl.h:188 error: include/spdlog/details/registry-inl.h: patch does not apply error: patch failed: include/spdlog/details/registry.h:61 error: include/spdlog/details/registry.h: patch does not apply error: patch failed: include/spdlog/spdlog-inl.h:67 error: include/spdlog/spdlog-inl.h: patch does not apply error: patch failed: include/spdlog/spdlog.h:81 error: include/spdlog/spdlog.h: patch does not apply error: patch failed: include/spdlog/details/file_helper-inl.h:101 error: include/spdlog/details/file_helper-inl.h: patch does not apply error: patch failed: include/spdlog/details/file_helper.h:15 error: include/spdlog/details/file_helper-inl.h: patch does not apply error: patch failed: include/spdlog/details/file_helper.h:15 error: include/spdlog/details/file_helper.h: patch does not apply error: patch failed: include/spdlog/details/os-inl.h:160 error: include/spdlog/details/os-inl.h: patch does not apply error: patch failed: include/spdlog/details/os.h:46 error: include/spdlog/details/os.h: patch does not apply -- Build spdlog: 1.10.0 -- Build type: Release error: patch failed: cmake_third_party/rapidjson/include/rapidjson/document.h:316 error: cmake_third_party/rapidjson/include/rapidjson/document.h: patch does not apply -- Found ZLIB: /usr/lib/x86_64-linux-gnu/libz.so (found suitable version "1.2.11", minimum required is "1.2.3") CMake Warning at cmake_third_party/nghttp2/CMakeLists.txt:69 (find_package): By not providing "FindSystemd.cmake" in CMAKE_MODULE_PATH this project has asked CMake to find a package configuration file provided by "Systemd", but CMake did not find one. Could not find a package configuration file provided by "Systemd" (requested version 209) with any of the following names: SystemdConfig.cmake systemd-config.cmake Add the installation prefix of "Systemd" to CMAKE_PREFIX_PATH or set "Systemd_DIR" to a directory containing one of the above files. If "Systemd" provides a separate development package or SDK, be sure it has been installed. -- Could NOT find Spdylay (missing: SPDYLAY_LIBRARY SPDYLAY_INCLUDE_DIR) (Required is at least version "1.3.2") -- summary of build options: Package version: 1.40.0 Library version: 33:0:19 Install prefix: /usr/local/trpc-cpp/trpc-2024-07/ Target system: Linux Compiler: Build type: Release C compiler: /usr/bin/cc CFLAGS: -O3 C++ compiler: /usr/bin/c++ CXXFLAGS: -O3 -Wno-error=class-memaccess -fstack-protector -Wall -Wunused-but-set-parameter -lm -fno-omit-frame-pointer -Wno-free-nonheap-object -MD -MF -fPIC -fno-canonical-system-headers -Wno-builtin-macro-redefined -Wno-implicit-fallthrough WARNCFLAGS: -Wall -Wextra -Wmissing-prototypes -Wstrict-prototypes -Wmissing-declarations -Wpointer-arith -Wdeclaration-after-statement -Wformat-security -Wwrite-strings -Wshadow -Winline -Wnested-externs -Wfloat-equal -Wundef -Wendif-labels -Wempty-body -Wcast-align -Wclobbered -Wvla -Wpragmas -Wunreachable-code -Waddress -Wattributes -Wdiv-by-zero -Wconversion -Wformat-nonliteral -Wmissing-field-initializers -Wmissing-noreturn -Wsign-conversion -Wunused-macros -Wunused-parameter -Wredundant-decls -Wno-format-nonliteral CXX1XCXXFLAGS: -std=c++14 Python: Python: /usr/bin/python3.8 PYTHON_VERSION: 3.8.10 Library version:3.8.10 Cython: /usr/bin/cython3 Test: CUnit: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libcunit.so') Failmalloc: ON Libs: OpenSSL: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libssl.so;/usr/lib/x86_64-linux-gnu/libcrypto.so') Libxml2: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libxml2.so') Libev: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libev.so') Libc-ares: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libcares.so') Libevent(SSL): TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libevent.so;/usr/lib/x86_64-linux-gnu/libevent_openssl.so') Spdylay: FALSE (LIBS='') Jansson: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libjansson.so') Jemalloc: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libjemalloc.so') Zlib: TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libz.so') Systemd: (LIBS='') Boost::System: Boost::Thread: Third-party: http-parser: MRuby: 0 Neverbleed: 0 Features: Applications: OFF HPACK tools: OFF Libnghttp2_asio:OFF Examples: OFF Python bindings:OFF Threading: ON Only the library will be built. To build other components (such as applications and examples), set ENABLE_LIB_ONLY=OFF. error: patch failed: src/BUILD.bazel:35 error: src/BUILD.bazel: patch does not apply error: patch failed: src/idl_gen_cpp.cpp:4122 error: src/idl_gen_cpp.cpp: patch does not apply error: patch failed: src/idl_gen_cpp.h:21 error: src/idl_gen_cpp.h: patch does not apply -- Proceeding with version: 23.5.26.0 -- CMAKE_CXX_FLAGS: -Wno-error=class-memaccess -fstack-protector -Wall -Wunused-but-set-parameter -lm -fno-omit-frame-pointer -Wno-free-nonheap-object -MD -MF -fPIC -fno-canonical-system-headers -Wno-builtin-macro-redefined -Wno-implicit-fallthrough -- Could NOT find jsoncons (missing: jsoncons_DIR)
```

根据大模型的分析，上面给出的日志里，真正会导致 trpc-cpp 编译中断的只有这一类行：

```
error: patch failed: include/spdlog/… patch does not apply
```

含义：CMake 在给 **spdlog / rapidjson / flatbuffers** 这些第三方源码套用框架自带的补丁时，找不到匹配的行。

常见原因只有两种：
1. **目录之前已经被打过一次补丁**（比如上一次编译中途退出，留下改动）。（就是说，我们第一次尝试构建时，打过补丁了。第二次时，由于我们没有正确安装spdylay，报错。）
2. **你把 third_party 库换成了不同版本**，补丁上下文已改变。
    
> ❗ Systemd / Spdylay 找不到只是可选功能提示，不会让核心编译失败。

可以通过下面的命令解决：
```sh
# 在 trpc-cpp 根目录
git reset --hard HEAD          # 回到当前 commit
git submodule sync --recursive # 同步子模块引用
git clean -xfd                 # ⚠ 删除所有未跟踪文件，包括 third_party/build 产物
git submodule update --init --recursive --depth 1  # 取回框架锁定版本
mkdir -p build && cd build
cmake .. -DTRPC_BUILD_THIRD_PARTY=OFF   # 默认是 ON，只编译库，不编译示例程序，这里我们把示例程序也编译
make -j$(nproc)
```
这样可确保：
- third_party 目录回到 **框架需要的确切 commit**；
- 第一次 CMake 重新打补丁，不会再出现 “patch failed”。
## 两个特殊的、烦人的（看起来是过时了的）依赖
关于 Systemd / Spdylay 提示
1. `Could NOT find Systemd` ——只影响 `sd_notify` 等特性，需要的话装 `libsystemd-dev`。（但是后来装了装，不行）
2. `Could NOT find Spdylay` ——`Spdylay` 已被 `nghttp2` 取代，大多数场景可忽略。
3. **patch failed ≠ 代码有错，而是补丁找不到应打的位置**。 
4. 把仓库（含子模块）恢复到干净状态，再跑一次 CMake，大多可解决。
### Spdylay
### systemd
```
  By not providing "FindSystemd.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "Systemd", but
  CMake did not find one.

  Could not find a package configuration file provided by "Systemd"
  (requested version 209) with any of the following names:

    SystemdConfig.cmake
    systemd-config.cmake

  Add the installation prefix of "Systemd" to CMAKE_PREFIX_PATH or set
  "Systemd_DIR" to a directory containing one of the above files.  If
  "Systemd" provides a separate development package or SDK, be sure it has
  been installed.
```

- **Systemd** 是 Linux 上常用的服务管理器（`systemctl start xxx.service` 就是它）。
- `nghttp2`（tRPC-cpp 里用到的 HTTP/2 库）在 CMakeLists.txt 里写了：如果系统里装了 _libsystemd_，就启用 **sd_notify / socket activation** 等功能。
- CMake 在配置阶段会去找 `SystemdConfig.cmake` 或 `FindSystemd.cmake`；找不到就给出你看到的 **Warning**。

> ⚠ **只是警告，不会让核心编译失败。**  
> 除非你确实想让程序和 systemd 深度集成，否则可以忽略。

#### 什么时候需要安装 systemd 开发包？
只有在这些场景才需要：

| 场景                                                      | 需要 systemd 吗？ |
| ------------------------------------------------------- | ------------- |
| 纯粹作为 tRPC-cpp 的服务端/客户端库                                 | **不需要**       |
| 想用 **sd_notify()** 把“启动完成”信号发给 systemd                  | 需要            |
| 想用 **systemd socket activation**（由 systemd 预开监听端口再交给程序） | 需要            |
#### 如果确实要启用

安装开发包
```bash
sudo apt-get install libsystemd-dev   # Debian/Ubuntu 

#或者 
sudo dnf install systemd-devel        # Fedora/CentOS/RHEL
```

重新跑 `cmake ..`，它就会自动找到 `SystemdConfig.cmake`，警告消失。

> CMake 不需要你手动改 `CMAKE_PREFIX_PATH`，只要系统包管理器把 `*.cmake` 文件装进 `/usr/lib/x86_64-linux-gnu/cmake/systemd/` 之类的目录就能找到。

但是，经过实测，安装了这个之后，还是不行。

因为：
**`libsystemd-dev` 并不会自带 _SystemdConfig.cmake_**。`find_package(Systemd)` 首先按 _Config mode_ 去找 `<prefix>/cmake/systemd/SystemdConfig.cmake`，在 Ubuntu 包里根本没有这个文件，于是继续报 Warning。

如何验证systemd配置成功？
先确认 `pkg-config --modversion libsystemd` 能返回版本号

#### 如果想彻底关掉这段检查
在顶层 `cmake` 命令里加：
```sh
-DENABLE_LIB_SYSTEMD=OFF      # nghttp2 单独控制（有的版本叫 WITH_SYSTEMD）
```
## 最终能编译通过的make命令

### 加上示例的编译
如果只是嵌入自己项目，`-DENABLE_LIB_ONLY=ON`（这是默认配置）
要跑官方示例，需加 `-DENABLE_LIB_ONLY=OFF`
### make -j
关于`make -j$(nproc)`是什么意思

`make -j$(nproc)` 是在 Linux/macOS 终端里常见的一条 **并行编译** 命令，可拆成三部分理解：

| 片段         | 含义                                                                           |
| ---------- | ---------------------------------------------------------------------------- |
| `make`     | 调用 GNU Make，根据 Makefile 里的规则去编译工程。                                           |
| `-j` _N_   | **并行任务数**（jobs）。缺省时 make 按顺序编译；加 `-j` 后可让它同时开 _N_ 个编译进程。                     |
| `$(nproc)` | Shell 命令替换：`nproc` 会输出当前机器可用的逻辑 CPU 核心数（如 8、16）。`$(nproc)` 把这个数字插入到 `-j` 后面。 |

> 结果：如果你的机器有 8 核，`make -j$(nproc)` 等价于 `make -j8`，让 make 最多同时运行 8 个编译任务，从而显著加快大型项目的构建速度。

```sh
nproc        # 只显示核心数
lscpu        # 显示更详细的 CPU 信息
```

macOS中，没有 `nproc` 命令，可用 `sysctl -n hw.ncpu` 代替，或直接写固定数字：
```sh
make -j$(sysctl -n hw.ncpu)
```

## 最终`cmake ..`返回的信息

```
mrcan@ubuntu:~/trpc-cpp/build$ cmake .. -DCMAKE_BUILD_TYPE=RelWithDebInfo 
-- The C compiler identification is GNU 9.4.0
-- The CXX compiler identification is GNU 9.4.0
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- CMAKE_SYSTEM_NAME:      Linux
-- CMAKE_SYSTEM_PROCESSOR: x86_64
-- The ASM compiler identification is GNU
-- Found assembler: /usr/bin/cc
-- Build tRPC-Cpp: v2024-07
[tRPC-Cpp] cloning relavant 3rd party, please waiting...
-- Found Python: /usr/bin/python3.8 (found version "3.8.10") found components: Interpreter 
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD
-- Performing Test CMAKE_HAVE_LIBC_PTHREAD - Failed
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - found
-- Found Threads: TRUE  
-- Module support is disabled.
-- Version: 9.1.0
-- Build type: RelWithDebInfo
-- CXX_STANDARD: 17
-- Performing Test has_std_17_flag
-- Performing Test has_std_17_flag - Success
-- Performing Test has_std_1z_flag
-- Performing Test has_std_1z_flag - Success
-- Required features: cxx_variadic_templates
-- Looking for C++ include unistd.h
-- Looking for C++ include unistd.h - found
-- Looking for C++ include stdint.h
-- Looking for C++ include stdint.h - found
-- Looking for C++ include inttypes.h
-- Looking for C++ include inttypes.h - found
-- Looking for C++ include sys/types.h
-- Looking for C++ include sys/types.h - found
-- Looking for C++ include sys/stat.h
-- Looking for C++ include sys/stat.h - found
-- Looking for C++ include fnmatch.h
-- Looking for C++ include fnmatch.h - found
-- Looking for C++ include stddef.h
-- Looking for C++ include stddef.h - found
-- Check size of uint32_t
-- Check size of uint32_t - done
-- Looking for strtoll
-- Looking for strtoll - found
-- JsonCpp Version: 1.9.3
-- Looking for C++ include clocale
-- Looking for C++ include clocale - found
-- Looking for localeconv
-- Looking for localeconv - found
-- Check size of lconv
-- Check size of lconv - done
-- Performing Test HAVE_DECIMAL_POINT
-- Performing Test HAVE_DECIMAL_POINT - Success
-- 
-- 3.15.8.0
-- Found ZLIB: /usr/lib/x86_64-linux-gnu/libz.so (found version "1.2.11") 
-- Performing Test protobuf_HAVE_BUILTIN_ATOMICS
-- Performing Test protobuf_HAVE_BUILTIN_ATOMICS - Success
-- Build spdlog: 1.10.0
-- Build type: RelWithDebInfo
-- Found PythonInterp: /usr/bin/python3.8 (found version "3.8.10") 
-- Found OpenSSL: /usr/lib/x86_64-linux-gnu/libcrypto.so (found suitable version "1.1.1f", minimum required is "1.0.1")  
-- Found Libev: /usr/lib/x86_64-linux-gnu/libev.so (found suitable version "4.31", minimum required is "4.11") 
-- Found Libcares: /usr/lib/x86_64-linux-gnu/libcares.so (found suitable version "1.15.0", minimum required is "1.7.5") 
-- Found ZLIB: /usr/lib/x86_64-linux-gnu/libz.so (found suitable version "1.2.11", minimum required is "1.2.3") 
CMake Warning at cmake_third_party/nghttp2/CMakeLists.txt:69 (find_package):
  By not providing "FindSystemd.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "Systemd", but
  CMake did not find one.

  Could not find a package configuration file provided by "Systemd"
  (requested version 209) with any of the following names:

    SystemdConfig.cmake
    systemd-config.cmake

  Add the installation prefix of "Systemd" to CMAKE_PREFIX_PATH or set
  "Systemd_DIR" to a directory containing one of the above files.  If
  "Systemd" provides a separate development package or SDK, be sure it has
  been installed.


-- Found Jansson: /usr/lib/x86_64-linux-gnu/libjansson.so (found suitable version "2.12", minimum required is "2.5") 
-- Found Libevent: /usr/lib/x86_64-linux-gnu/libevent.so (found suitable version "2.1.11-stable", minimum required is "2.0.8") found components: libevent openssl 
-- Found Cython: /usr/bin/cython3  
-- Found PythonLibs: /usr/lib/x86_64-linux-gnu/libpython3.8.so (found version "3.8.10") 
-- Found LibXml2: /usr/lib/x86_64-linux-gnu/libxml2.so (found suitable version "2.9.10", minimum required is "2.6.26") 
-- Found Jemalloc: /usr/lib/x86_64-linux-gnu/libjemalloc.so (found version "5.2.1-0-gea6b3e973b477b8061e0076bb257dbd7f3faa756") 
-- Could NOT find Spdylay (missing: SPDYLAY_LIBRARY SPDYLAY_INCLUDE_DIR) (Required is at least version "1.3.2")
-- Performing Test CXX_FLAG__std_c_14
-- Performing Test CXX_FLAG__std_c_14 - Success
-- Performing Test HAVE_STD_FUTURE
-- Performing Test HAVE_STD_FUTURE - Success
-- Performing Test HAVE_STD_MAP_EMPLACE
-- Performing Test HAVE_STD_MAP_EMPLACE - Success
-- Found CUnit: /usr/lib/x86_64-linux-gnu/libcunit.so (found suitable version "2.1-3", minimum required is "2.1") 
-- Looking for arpa/inet.h
-- Looking for arpa/inet.h - found
-- Looking for fcntl.h
-- Looking for fcntl.h - found
-- Looking for limits.h
-- Looking for limits.h - found
-- Looking for netdb.h
-- Looking for netdb.h - found
-- Looking for netinet/in.h
-- Looking for netinet/in.h - found
-- Looking for pwd.h
-- Looking for pwd.h - found
-- Looking for sys/socket.h
-- Looking for sys/socket.h - found
-- Looking for sys/time.h
-- Looking for sys/time.h - found
-- Looking for syslog.h
-- Looking for syslog.h - found
-- Looking for time.h
-- Looking for time.h - found
-- Check size of ssize_t
-- Check size of ssize_t - done
-- Performing Test HAVE_STRUCT_TM_TM_GMTOFF
-- Performing Test HAVE_STRUCT_TM_TM_GMTOFF - Success
-- Check size of int *
-- Check size of int * - done
-- Check size of time_t
-- Check size of time_t - done
-- Looking for _Exit
-- Looking for _Exit - found
-- Looking for accept4
-- Looking for accept4 - found
-- Looking for mkostemp
-- Looking for mkostemp - found
-- Looking for initgroups
-- Looking for initgroups - found
-- Performing Test C_FLAG__Wall
-- Performing Test C_FLAG__Wall - Success
-- Performing Test C_FLAG__Wextra
-- Performing Test C_FLAG__Wextra - Success
-- Performing Test C_FLAG__Wmissing_prototypes
-- Performing Test C_FLAG__Wmissing_prototypes - Success
-- Performing Test C_FLAG__Wstrict_prototypes
-- Performing Test C_FLAG__Wstrict_prototypes - Success
-- Performing Test C_FLAG__Wmissing_declarations
-- Performing Test C_FLAG__Wmissing_declarations - Success
-- Performing Test C_FLAG__Wpointer_arith
-- Performing Test C_FLAG__Wpointer_arith - Success
-- Performing Test C_FLAG__Wdeclaration_after_statement
-- Performing Test C_FLAG__Wdeclaration_after_statement - Success
-- Performing Test C_FLAG__Wformat_security
-- Performing Test C_FLAG__Wformat_security - Success
-- Performing Test C_FLAG__Wwrite_strings
-- Performing Test C_FLAG__Wwrite_strings - Success
-- Performing Test C_FLAG__Wshadow
-- Performing Test C_FLAG__Wshadow - Success
-- Performing Test C_FLAG__Winline
-- Performing Test C_FLAG__Winline - Success
-- Performing Test C_FLAG__Wnested_externs
-- Performing Test C_FLAG__Wnested_externs - Success
-- Performing Test C_FLAG__Wfloat_equal
-- Performing Test C_FLAG__Wfloat_equal - Success
-- Performing Test C_FLAG__Wundef
-- Performing Test C_FLAG__Wundef - Success
-- Performing Test C_FLAG__Wendif_labels
-- Performing Test C_FLAG__Wendif_labels - Success
-- Performing Test C_FLAG__Wempty_body
-- Performing Test C_FLAG__Wempty_body - Success
-- Performing Test C_FLAG__Wcast_align
-- Performing Test C_FLAG__Wcast_align - Success
-- Performing Test C_FLAG__Wclobbered
-- Performing Test C_FLAG__Wclobbered - Success
-- Performing Test C_FLAG__Wvla
-- Performing Test C_FLAG__Wvla - Success
-- Performing Test C_FLAG__Wpragmas
-- Performing Test C_FLAG__Wpragmas - Success
-- Performing Test C_FLAG__Wunreachable_code
-- Performing Test C_FLAG__Wunreachable_code - Success
-- Performing Test C_FLAG__Waddress
-- Performing Test C_FLAG__Waddress - Success
-- Performing Test C_FLAG__Wattributes
-- Performing Test C_FLAG__Wattributes - Success
-- Performing Test C_FLAG__Wdiv_by_zero
-- Performing Test C_FLAG__Wdiv_by_zero - Success
-- Performing Test C_FLAG__Wshorten_64_to_32
-- Performing Test C_FLAG__Wshorten_64_to_32 - Failed
-- Performing Test C_FLAG__Wconversion
-- Performing Test C_FLAG__Wconversion - Success
-- Performing Test C_FLAG__Wextended_offsetof
-- Performing Test C_FLAG__Wextended_offsetof - Failed
-- Performing Test C_FLAG__Wformat_nonliteral
-- Performing Test C_FLAG__Wformat_nonliteral - Success
-- Performing Test C_FLAG__Wlanguage_extension_token
-- Performing Test C_FLAG__Wlanguage_extension_token - Failed
-- Performing Test C_FLAG__Wmissing_field_initializers
-- Performing Test C_FLAG__Wmissing_field_initializers - Success
-- Performing Test C_FLAG__Wmissing_noreturn
-- Performing Test C_FLAG__Wmissing_noreturn - Success
-- Performing Test C_FLAG__Wmissing_variable_declarations
-- Performing Test C_FLAG__Wmissing_variable_declarations - Failed
-- Performing Test C_FLAG__Wsign_conversion
-- Performing Test C_FLAG__Wsign_conversion - Success
-- Performing Test C_FLAG__Wunreachable_code_break
-- Performing Test C_FLAG__Wunreachable_code_break - Failed
-- Performing Test C_FLAG__Wunused_macros
-- Performing Test C_FLAG__Wunused_macros - Success
-- Performing Test C_FLAG__Wunused_parameter
-- Performing Test C_FLAG__Wunused_parameter - Success
-- Performing Test C_FLAG__Wredundant_decls
-- Performing Test C_FLAG__Wredundant_decls - Success
-- Performing Test C_FLAG__Wheader_guard
-- Performing Test C_FLAG__Wheader_guard - Failed
-- Performing Test C_FLAG__Wno_format_nonliteral
-- Performing Test C_FLAG__Wno_format_nonliteral - Success
-- Performing Test CXX_FLAG__Wall
-- Performing Test CXX_FLAG__Wall - Success
-- Performing Test CXX_FLAG__Wformat_security
-- Performing Test CXX_FLAG__Wformat_security - Success
-- summary of build options:

    Package version: 1.40.0
    Library version: 33:0:19
    Install prefix:  /usr/local/trpc-cpp/trpc-2024-07/
    Target system:   Linux
    Compiler:
      Build type:     RelWithDebInfo
      C compiler:     /usr/bin/cc
      CFLAGS:         -O2 -g  
      C++ compiler:   /usr/bin/c++
      CXXFLAGS:       -O2 -g    -Wno-error=class-memaccess  -fstack-protector -Wall -Wunused-but-set-parameter -lm  -fno-omit-frame-pointer -Wno-free-nonheap-object  -MD -MF -fPIC  -fno-canonical-system-headers -Wno-builtin-macro-redefined  -Wno-implicit-fallthrough
      WARNCFLAGS:      -Wall -Wextra -Wmissing-prototypes -Wstrict-prototypes -Wmissing-declarations -Wpointer-arith -Wdeclaration-after-statement -Wformat-security -Wwrite-strings -Wshadow -Winline -Wnested-externs -Wfloat-equal -Wundef -Wendif-labels -Wempty-body -Wcast-align -Wclobbered -Wvla -Wpragmas -Wunreachable-code -Waddress -Wattributes -Wdiv-by-zero -Wconversion -Wformat-nonliteral -Wmissing-field-initializers -Wmissing-noreturn -Wsign-conversion -Wunused-macros -Wunused-parameter -Wredundant-decls -Wno-format-nonliteral
      CXX1XCXXFLAGS:  -std=c++14
    Python:
      Python:         /usr/bin/python3.8
      PYTHON_VERSION: 3.8.10
      Library version:3.8.10
      Cython:         /usr/bin/cython3
    Test:
      CUnit:          TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libcunit.so')
      Failmalloc:     ON
    Libs:
      OpenSSL:        TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libssl.so;/usr/lib/x86_64-linux-gnu/libcrypto.so')
      Libxml2:        TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libxml2.so')
      Libev:          TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libev.so')
      Libc-ares:      TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libcares.so')
      Libevent(SSL):  TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libevent.so;/usr/lib/x86_64-linux-gnu/libevent_openssl.so')
      Spdylay:        FALSE (LIBS='')
      Jansson:        TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libjansson.so')
      Jemalloc:       TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libjemalloc.so')
      Zlib:           TRUE (LIBS='/usr/lib/x86_64-linux-gnu/libz.so')
      Systemd:         (LIBS='')
      Boost::System:  
      Boost::Thread:  
    Third-party:
      http-parser:    
      MRuby:          0
      Neverbleed:     0
    Features:
      Applications:   OFF
      HPACK tools:    OFF
      Libnghttp2_asio:OFF
      Examples:       OFF
      Python bindings:OFF
      Threading:      ON

Only the library will be built. To build other components (such as applications and examples), set ENABLE_LIB_ONLY=OFF.
-- Check if the system is big endian
-- Searching 16 bit integer
-- Check size of unsigned short
-- Check size of unsigned short - done
-- Using unsigned short
-- Check if the system is big endian - little endian
-- Looking for byteswap.h
-- Looking for byteswap.h - found
-- Looking for sys/endian.h
-- Looking for sys/endian.h - not found
-- Looking for sys/mman.h
-- Looking for sys/mman.h - found
-- Looking for sys/resource.h
-- Looking for sys/resource.h - found
-- Looking for sys/uio.h
-- Looking for sys/uio.h - found
-- Looking for windows.h
-- Looking for windows.h - not found
-- Looking for zlibVersion in z
-- Looking for zlibVersion in z - found
-- Looking for lzo1x_1_15_compress in lzo2
-- Looking for lzo1x_1_15_compress in lzo2 - not found
-- Performing Test HAVE_VISUAL_STUDIO_ARCH_AVX
-- Performing Test HAVE_VISUAL_STUDIO_ARCH_AVX - Failed
-- Performing Test HAVE_VISUAL_STUDIO_ARCH_AVX2
-- Performing Test HAVE_VISUAL_STUDIO_ARCH_AVX2 - Failed
-- Performing Test HAVE_CLANG_MAVX
-- Performing Test HAVE_CLANG_MAVX - Success
-- Performing Test HAVE_CLANG_MBMI2
-- Performing Test HAVE_CLANG_MBMI2 - Success
-- Performing Test HAVE_BUILTIN_EXPECT
-- Performing Test HAVE_BUILTIN_EXPECT - Success
-- Performing Test HAVE_BUILTIN_CTZ
-- Performing Test HAVE_BUILTIN_CTZ - Success
-- Performing Test SNAPPY_HAVE_SSSE3
-- Performing Test SNAPPY_HAVE_SSSE3 - Failed
-- Performing Test SNAPPY_HAVE_BMI2
-- Performing Test SNAPPY_HAVE_BMI2 - Failed
-- Looking for mmap
-- Looking for mmap - found
-- Looking for sysconf
-- Looking for sysconf - found
-- Performing Test CFLAG_Wall
-- Performing Test CFLAG_Wall - Success
-- Performing Test CFLAG_Wextra
-- Performing Test CFLAG_Wextra - Success
-- Performing Test CFLAG_Wundef
-- Performing Test CFLAG_Wundef - Success
-- Performing Test CFLAG_Wcast_qual
-- Performing Test CFLAG_Wcast_qual - Success
-- Performing Test CFLAG_Wcast_align
-- Performing Test CFLAG_Wcast_align - Success
-- Performing Test CFLAG_Wshadow
-- Performing Test CFLAG_Wshadow - Success
-- Performing Test CFLAG_Wswitch_enum
-- Performing Test CFLAG_Wswitch_enum - Success
-- Performing Test CFLAG_Wdeclaration_after_statement
-- Performing Test CFLAG_Wdeclaration_after_statement - Success
-- Performing Test CFLAG_Wstrict_prototypes
-- Performing Test CFLAG_Wstrict_prototypes - Success
-- Performing Test CFLAG_Wpointer_arith
-- Performing Test CFLAG_Wpointer_arith - Success
-- Performing Test CFLAG_W4
-- Performing Test CFLAG_W4 - Failed
-- Proceeding with version: 23.5.26.0
-- Looking for strtof_l
-- Looking for strtof_l - found
-- Looking for strtoull_l
-- Looking for strtoull_l - found
-- Looking for realpath
-- Looking for realpath - found
-- CMAKE_CXX_FLAGS:   -Wno-error=class-memaccess  -fstack-protector -Wall -Wunused-but-set-parameter -lm  -fno-omit-frame-pointer -Wno-free-nonheap-object  -MD -MF -fPIC  -fno-canonical-system-headers -Wno-builtin-macro-redefined  -Wno-implicit-fallthrough
-- Found nlohmann_json: /usr/lib/cmake/nlohmann_json/nlohmann_jsonConfig.cmake (found version "3.7.3") 
-- Could NOT find jsoncons (missing: jsoncons_DIR)
-- Configuring done
-- Generating done
CMake Warning:
  Manually-specified variables were not used by the project:

    TRPC_BUILD_THIRD_PARTY

-- Build files have been written to: /home/mrcan/trpc-cpp/build
```

1. 最后一句`Manually-specified variables were not used by the project: TRPC_BUILD_THIRD_PARTY`
    1. 这表示 `TRPC_BUILD_THIRD_PARTY` 这个开关在新版本的 tRPC-Cpp 里已经被移除或改名，CMake 忽略了它。默认就会拉取并编译必须的第三方库，所以**不影响后续编译**。
    2. 可以删掉这个参数，或用官方 CMakeLists.txt 里真正存在的等价选项（有的版本叫 `TRPC_USE_EXTERNAL`、`TRPC_FETCH_DEPS` 等，需看源码注释）。
2. `Could NOT find Systemd`、`Could NOT find Spdylay`、`Could NOT find jsoncons`，可以忽略。
    1. 仅当你真的要用 `systemd socket-activation`、`Spdy` 协议或 `jsoncons` 库时才需手动装 dev 包；否则忽略即可。
3. 其余都是 `Performing Test … Success/Failed`，这是编译器特性探测，正常现象。无需处理。

**最关键的一点：** 这一次没有再出现 _patch failed_ 之类的致命错误，并且配置阶段以`-- Configuring done`、`-- Generating done`、`Build files have been written to …` 正常结束。
说明 **CMake 已经成功生成 Makefile**，可以进入编译阶段了。

### 下一步怎么做？
```bash
#仍在 build 目录
make -j$(nproc)     # 并行编译核心库
sudo make install   # 如果你需要把 libtrpc*.so 安装到 /usr/local`
```

## 总结
| 步骤                 | 命令                                                                                                                | 说明                                                                                            |
| ------------------ | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 1. 安装系统依赖          | `sudo apt update && sudo apt install -y ...`                                                                      | 这些库都是 tRPC-Cpp 默认启用或示例常用的。`libgoogle-perftools-dev` 提供 tcmalloc；如果不用可稍后关掉。                    |
| 2. 获取源码（含子模块）      | `git clone https://github.com/tencent/trpc-cpp.git cd trpc-cpp git submodule update --init --recursive --depth 1` | 保证 third_party 版本和补丁完全匹配。                                                                     |
| 3. _可选_：清理旧残留      | `git reset --hard && git clean -xfd`                                                                              | 避免之前打补丁失败留下冲突。                                                                                |
| 4. 生成构建目录          | `mkdir build && cd build`                                                                                         | 分离源码与构建产物。                                                                                    |
| 5.**全功能模式**（连示例一起） | `cmake .. DCMAKE_BUILD_TYPE=RelWithDebInfo -DENABLE_LIB_ONLY=OFF`                                                 | `RelWithDebInfo` → 既有优化又保留调试符号；`ENABLE_LIB_ONLY=OFF` 编译示例和工具。如果是ON则：**纯库模式**（只编译 libtrpc*.so） |
| 6. 关闭可选组件（按需）      | 后追加：`-DWITH_SYSTEMD=OFF` - 不找 systemd<br><br>`-DWITH_SPDYLAY=OFF` - 不编译 Spdy 支持                                   | 可减少依赖或解决缺库。                                                                                   |
| 7. 并行编译            | `make -j$(nproc)`                                                                                                 | Ninja 会自动并行；若用 Makefile 就是 `make -j$(nproc)`。                                                 |
| 8. _可选_ 安装         | `sudo make install`                                                                                               | 默认装到 `/usr/local/trpc-cpp/trpc-2024-07/`；可用 `-DCMAKE_INSTALL_PREFIX=/opt/trpc` 修改。            |
| 9. 运行示例            | `./examples/helloworld/build/helloworld_svr & ./examples/helloworld/build/helloworld_cli`                         | 验证框架功能链路。                                                                                     |
在 v2024-07 版本里，示例可执行文件会生成到各自子目录的 `build/` 下，例如`examples/helloworld/build/helloworld_svr`。

如何运行`run_cmake.sh`示例？在 **项目根目录** 执行`bash examples/helloworld/run_cmake.sh`
# Bazel 编译问题
## 小提示
```
WARNING: Download from https://mirrors.tencent.com/github.com/nghttp2/nghttp2/releases/download/v1.40.0/nghttp2-1.40.0.tar.gz failed: class java.io.FileNotFoundException GET returned 404 Not Found
```
不影响核心编译。
## 使用`g++-9`编译错误

```
mrcan@ubuntu:~/trpc-cpp$ ./build.sh 
INFO: Analyzed 1185 targets (0 packages loaded, 0 targets configured).
ERROR: /home/mrcan/trpc-cpp/trpc/auth/BUILD:44:8: Compiling trpc/auth/auth_center_follower_factory_test.cc failed: (Exit 1): gcc failed: error executing CppCompile command (from target //trpc/auth:auth_center_follower_factory_test) /usr/bin/gcc -U_FORTIFY_SOURCE -fstack-protector -Wall -Wunused-but-set-parameter -Wno-free-nonheap-object -fno-omit-frame-pointer '-std=c++14' -MD -MF ... (remaining 42 arguments skipped)

Use --sandbox_debug to see verbose messages from the sandbox and retain the sandbox build root for debugging
In file included from /usr/include/c++/9/bits/move.h:55,
                 from /usr/include/c++/9/bits/stl_pair.h:59,
                 from /usr/include/c++/9/bits/stl_algobase.h:64,
                 from /usr/include/c++/9/bits/char_traits.h:39,
                 from /usr/include/c++/9/string:40,
                 from ./trpc/auth/auth_center_follower_factory.h:16,
                 from trpc/auth/auth_center_follower_factory_test.cc:14:
/usr/include/c++/9/type_traits: In instantiation of 'struct std::__and_<std::is_copy_constructible<testing::Matcher<std::any&> >, std::__not_<std::is_constructible<testing::Matcher<std::any&>, const testing::Matcher<std::any&>&> >, std::__not_<std::__is_in_place_type<testing::Matcher<std::any&> > > >':

// ... （略去一长串的 Any 相关的报错）

Use --verbose_failures to see the command lines of failed build steps.
INFO: Elapsed time: 50.781s, Critical Path: 19.27s
INFO: 134 processes: 35 internal, 99 linux-sandbox.
ERROR: Build did NOT complete successfully

```
实测`g++`、`gcc` 10.1版本以上解决了any的问题。
```sh
mrcan@ubuntu:~/trpc-cpp$ export CC=gcc-10 CXX=g++-10
mrcan@ubuntu:~/trpc-cpp$ bazel build //trpc/config:config_test
```
根据编译错误信息，问题核心在于 `std::any` 与 `testing::Matcher` 的交互触发了 GCC 9.4.0 标准库的模板元编程缺陷。以下是系统性解决方案：
1. ​**​类型系统冲突：`std::any` 的构造函数在实例化 `testing::Matcher<const std::any&>` 时，无法正确推导其类型特性（如 `is_copy_constructible`），导致模板元编程失败。
2. ​**​GCC 9.4.0 的已知限制​**​：该版本的 `libstdc++` 在嵌套模板类型推导中存在边界情况，尤其涉及 Google Test 的 Matcher 时会触发
