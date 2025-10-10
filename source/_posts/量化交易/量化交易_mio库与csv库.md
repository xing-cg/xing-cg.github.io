---
title: 量化交易_mio库与csv库
categories:
  - - 量化交易
tags:
date: 2025/9/24
updated: 2025/9/24
comments:
published:
---
# 创建一个共享内存映射文件
```cpp
enum class access_mode
{
    read,
    write
};
/**
 * This is the basis for all read-only mmap objects and should be preferred over
 * directly using `basic_mmap`.
 */
template<typename ByteT>
using basic_mmap_source = basic_mmap<access_mode::read, ByteT>;

/**
 * This is the basis for all read-write mmap objects and should be preferred over
 * directly using `basic_mmap`.
 */
template<typename ByteT>
using basic_mmap_sink = basic_mmap<access_mode::write, ByteT>;

/**
 * These aliases cover the most common use cases, both representing a raw byte stream
 * (either with a char or an unsigned char/uint8_t).
 */
using mmap_source = basic_mmap_source<char>;
using ummap_source = basic_mmap_source<unsigned char>;

using mmap_sink = basic_mmap_sink<char>;
using ummap_sink = basic_mmap_sink<unsigned char>;
```

`basic_mmap_source`是`basic_map<access_mode::read, ByteT>`。
相比于`basic_mmap<access_mode AccessMode, ByteT>`，
`basic_mmap_source`限制了其`AccessMode`是`read`。
`mmap_source`是`basic_mmap_source<char>`的别名。
可见，`mmap_source`就是：只读的`char`类型映射。

```cpp
/**
 * Convenience factory method that constructs a mapping for any `basic_mmap` or
 * `basic_mmap` type.
 */
template<
    typename MMap,
    typename MappingToken
> MMap make_mmap(const MappingToken& token,
        int64_t offset, int64_t length, std::error_code& error)
{
    MMap mmap;
    mmap.map(token, offset, length, error);
    return mmap;
}

/**
 * Convenience factory method.
 *
 * MappingToken may be a String (`std::string`, `std::string_view`, `const char*`,
 * `std::filesystem::path`, `std::vector<char>`, or similar), or a
 * `mmap_source::handle_type`.
 */
template<typename MappingToken>
mmap_source make_mmap_source(const MappingToken& token, mmap_source::size_type offset,
        mmap_source::size_type length, std::error_code& error)
{
    return make_mmap<mmap_source>(token, offset, length, error);
}
```

`make_mmap`是一个抽象工厂方法。
用于生成`MMap`类型的内存映射。

最终调用到了`MMap`类型内部的`map`方法。
至于是哪个方法，由`MappingToken`决定。如果`MappingToken`是 String，则调用的是`map(const String& path, ...)`。

`make_mmap_source`是具体工厂。
用于生成`MMap = mmap_source`的映射，即生成一个：只读的`char`类型映射。

# map方法
`basic_mmap`最基类的接口。
```cpp
    /**
     * Establishes a memory mapping with AccessMode. If the mapping is unsuccesful, the
     * reason is reported via `error` and the object remains in a state as if this
     * function hadn't been called.
     *
     * `path`, which must be a path to an existing file, is used to retrieve a file
     * handle (which is closed when the object destructs or `unmap` is called), which is
     * then used to memory map the requested region. Upon failure, `error` is set to
     * indicate the reason and the object remains in an unmapped state.
     *
     * `offset` is the number of bytes, relative to the start of the file, where the
     * mapping should begin. When specifying it, there is no need to worry about
     * providing a value that is aligned with the operating system's page allocation
     * granularity. This is adjusted by the implementation such that the first requested
     * byte (as returned by `data` or `begin`), so long as `offset` is valid, will be at
     * `offset` from the start of the file.
     *
     * `length` is the number of bytes to map. It may be `map_entire_file`, in which
     * case a mapping of the entire file is created.
     */
    template<typename String>
    void map(const String& path, const size_type offset,
            const size_type length, std::error_code& error);

    /**
     * Establishes a memory mapping with AccessMode. If the mapping is unsuccesful, the
     * reason is reported via `error` and the object remains in a state as if this
     * function hadn't been called.
     *
     * `path`, which must be a path to an existing file, is used to retrieve a file
     * handle (which is closed when the object destructs or `unmap` is called), which is
     * then used to memory map the requested region. Upon failure, `error` is set to
     * indicate the reason and the object remains in an unmapped state.
     * 
     * The entire file is mapped.
     */
    template<typename String>
    void map(const String& path, std::error_code& error)
    {
        map(path, 0, map_entire_file, error);
    }

    /**
     * Establishes a memory mapping with AccessMode. If the mapping is
     * unsuccesful, the reason is reported via `error` and the object remains in
     * a state as if this function hadn't been called.
     *
     * `handle`, which must be a valid file handle, which is used to memory map the
     * requested region. Upon failure, `error` is set to indicate the reason and the
     * object remains in an unmapped state.
     *
     * `offset` is the number of bytes, relative to the start of the file, where the
     * mapping should begin. When specifying it, there is no need to worry about
     * providing a value that is aligned with the operating system's page allocation
     * granularity. This is adjusted by the implementation such that the first requested
     * byte (as returned by `data` or `begin`), so long as `offset` is valid, will be at
     * `offset` from the start of the file.
     *
     * `length` is the number of bytes to map. It may be `map_entire_file`, in which
     * case a mapping of the entire file is created.
     */
    void map(const handle_type handle, const size_type offset,
            const size_type length, std::error_code& error);

    /**
     * Establishes a memory mapping with AccessMode. If the mapping is
     * unsuccesful, the reason is reported via `error` and the object remains in
     * a state as if this function hadn't been called.
     *
     * `handle`, which must be a valid file handle, which is used to memory map the
     * requested region. Upon failure, `error` is set to indicate the reason and the
     * object remains in an unmapped state.
     * 
     * The entire file is mapped.
     */
    void map(const handle_type handle, std::error_code& error)
    {
        map(handle, 0, map_entire_file, error);
    }
```
`basic_mmap<AccessMode, ByteT>`的map方法实现
1. 一个是针对 file handle type 为 文件路径的。类型为 `String&`
2. 一个是针对 file handle type 为 通用句柄的。如果是 WIN32 则 file handle type 为 HANDLE，其他为 int 。

`map(const String& path, ...)`实际上就是对 `map(const handle_type handle, ...)` 的一次包装，做的额外操作是 打开文件，获取文件句柄。然后传给后者。

```cpp
template<access_mode AccessMode, typename ByteT>
template<typename String>
void basic_mmap<AccessMode, ByteT>::map(const String& path, const size_type offset,
        const size_type length, std::error_code& error)
{
    error.clear();
    if(detail::empty(path))
    {
        error = std::make_error_code(std::errc::invalid_argument);
        return;
    }
    const auto handle = detail::open_file(path, AccessMode, error);
    if(error)
    {
        return;
    }

    map(handle, offset, length, error);
    // This MUST be after the call to map, as that sets this to true.
    if(!error)
    {
        is_handle_internal_ = true;
    }
}
```
## 构造一个mmap：最终的路径
```cpp
template<access_mode AccessMode, typename ByteT>
void basic_mmap<AccessMode, ByteT>::map(const handle_type handle,
        const size_type offset, const size_type length, std::error_code& error)
{
    error.clear();
    if(handle == invalid_handle)
    {
        error = std::make_error_code(std::errc::bad_file_descriptor);
        return;
    }

    const auto file_size = detail::query_file_size(handle, error);
    if(error)
    {
        return;
    }

    if(offset + length > file_size)
    {
        error = std::make_error_code(std::errc::invalid_argument);
        return;
    }

    const auto ctx = detail::memory_map(handle, offset,
            length == map_entire_file ? (file_size - offset) : length,
            AccessMode, error);
    if(!error)
    {
        // We must unmap the previous mapping that may have existed prior to this call.
        // Note that this must only be invoked after a new mapping has been created in
        // order to provide the strong guarantee that, should the new mapping fail, the
        // `map` function leaves this instance in a state as though the function had
        // never been invoked.
        unmap();
        file_handle_ = handle;
        is_handle_internal_ = false;
        data_ = reinterpret_cast<pointer>(ctx.data);
        length_ = ctx.length;
        mapped_length_ = ctx.mapped_length;
#ifdef _WIN32
        file_mapping_handle_ = ctx.file_mapping_handle;
#endif
    }
}
```

## 最最后的实际操作
```cpp
inline mmap_context memory_map(const file_handle_type file_handle, const int64_t offset,
    const int64_t length, const access_mode mode, std::error_code& error)
{
    const int64_t aligned_offset = make_offset_page_aligned(offset);
    const int64_t length_to_map = offset - aligned_offset + length;
#ifdef _WIN32
    const int64_t max_file_size = offset + length;
    const auto file_mapping_handle = ::CreateFileMapping(
            file_handle,
            0,
            mode == access_mode::read ? PAGE_READONLY : PAGE_READWRITE,
            win::int64_high(max_file_size),
            win::int64_low(max_file_size),
            0);
    if(file_mapping_handle == invalid_handle)
    {
        error = detail::last_error();
        return {};
    }
    char* mapping_start = static_cast<char*>(::MapViewOfFile(
            file_mapping_handle,
            mode == access_mode::read ? FILE_MAP_READ : FILE_MAP_WRITE,
            win::int64_high(aligned_offset),
            win::int64_low(aligned_offset),
            length_to_map));
    if(mapping_start == nullptr)
    {
        // Close file handle if mapping it failed.
        ::CloseHandle(file_mapping_handle);
        error = detail::last_error();
        return {};
    }
#else // POSIX
    char* mapping_start = static_cast<char*>(::mmap(
            0, // Don't give hint as to where to map.
            length_to_map,
            mode == access_mode::read ? PROT_READ : PROT_WRITE,
            MAP_SHARED,
            file_handle,
            aligned_offset));
    if(mapping_start == MAP_FAILED)
    {
        error = detail::last_error();
        return {};
    }
#endif
    mmap_context ctx;
    ctx.data = mapping_start + offset - aligned_offset;
    ctx.length = length;
    ctx.mapped_length = length_to_map;
#ifdef _WIN32
    ctx.file_mapping_handle = file_mapping_handle;
#endif
    return ctx;
}
```

## 核心操作：mmap

mmap 返回 `MAP_FAILED` 时，才写入 errno。

# CSV库中的bug
有时会返回错误码：134。


```cpp
        CSV_INLINE void MmapParser::next(size_t bytes = ITERATION_CHUNK_SIZE) {
            // Reset parser state
            this->field_start = UNINITIALIZED_FIELD;
            this->field_length = 0;
            this->reset_data_ptr();

            // Create memory map
            size_t length = std::min(this->source_size - this->mmap_pos, bytes);
            std::error_code error;
            this->data_ptr->_data = std::make_shared<mio::basic_mmap_source<char>>(mio::make_mmap_source(this->_filename, this->mmap_pos, length, error));
            this->mmap_pos += length;
            if (error) throw error;

            auto mmap_ptr = (mio::basic_mmap_source<char>*)(this->data_ptr->_data.get());
            // ...
```

`make_shared`一句，做了以下事情：
1. 用`make_mmap_source`生成一个：只读的`char`类型映射。把这个映射对象用共享指针包装。

在构造 mmap 时， error 可能 出现的情况只有 3 种：
1. `handle == invalid_handle`：`error = std::make_error_code(std::errc::bad_file_descriptor)`：错误码 9
2. `offset + length > file_size`：`error = std::make_error_code(std::errc::invalid_argument)`：错误码 22
3. mmap 失败。`error = detail::last_error()`。即 返回 上一个（最后一个）errno。实测，错误码 134。

```cpp
/**
 * Returns the last platform specific system error (errno on POSIX and
 * GetLastError on Win) as a `std::error_code`.
 */
inline std::error_code last_error() noexcept
{
    std::error_code error;
#ifdef _WIN32
    error.assign(GetLastError(), std::system_category());
#else
    error.assign(errno, std::system_category());
#endif
    return error;
}
```

# Linux 134 错误码 不是 errno 值 而是 进程退出状态码

在 Linux 系统中，系统调用通常通过 errno 返回错误。但错误码 **134** 不是标准的 errno 值。
134 更可能是**进程退出状态码**。当进程被信号终止时，退出码是 **128 加上 信号编号**。
134 减去 128 等于 6，对应的信号是 SIGABRT 信号。
这意味着进程可能调用了 `abort()`，或者触发了断言失败、内存分配错误等导致 libc 终止进程。

`glibc` 检测到堆内存损坏（如 `double free` 、缓冲区溢出）、调用 `abort()` 、某些安全机制如 `FORTIFY_SOURCE` 检测到缓冲区溢出、或者 `pthread` 线程相关错误。

当程序因调用 `mmap` 或其他原因终止并返回退出状态码 **134** 时，这 **通常不是 `mmap` 系统调用直接设置的 `errno`**，而是表示你的进程收到了一个特定的信号并被该信号终止。

**退出状态码 134 的含义：**

*   在 Unix/Linux 系统中，如果一个进程是被信号 (Signal) 终止的，它的退出状态码是 `128 + <signal_number>`。
*   `134 = 128 + 6`
*   **信号编号 6 是 `SIGABRT`。**

**因此，退出码 134 意味着进程收到了 `SIGABRT` 信号并因此终止。**
## SIGABRT 信号
**`SIGABRT` 信号的常见原因：**

`SIGABRT` 通常由程序自身主动触发，表明检测到了严重的内部错误，无法安全地继续运行。最常见的原因包括：

1.  **堆损坏 (Heap Corruption)：** 这是最常见的原因之一。标准库 (如 glibc) 在检测到堆内存被破坏时，会调用 `abort()` 发出 `SIGABRT`。具体错误包括：
    *   **Double Free：** 尝试释放一个已经释放的内存块。
    *   **Invalid Free：** 尝试释放一个不是通过 `malloc`/`calloc`/`realloc` 分配的内存地址（或已被释放）。
    *   **Heap Buffer Overflow：** 写入操作超出了动态分配的内存块的边界，破坏了堆的管理结构。
    *   **Heap Buffer Underflow：** 在动态分配的内存块起始位置之前进行写入。
    *   **Freeing a Pointer Not at the Start of a Block：** 释放的指针不是指向分配块的确切起始位置（虽然某些分配器允许，但 glibc 的 `malloc` 通常要求精确的起始地址）。
    *   **内存分配/释放函数内部的严重不一致：** 当 `malloc`, `free`, `realloc`, `calloc` 等函数内部数据结构出现严重问题时。

2.  **显式调用 `abort()` 函数：** 程序代码或库代码中显式调用了 `abort()` 函数。这通常是为了响应无法恢复的严重错误条件。

3.  **断言失败 (`assert` Macro)：** 如果程序使用了 `assert(condition)`，并且 `condition` 在运行时评估为 `false`，`assert` 会打印错误信息并调用 `abort()`。

4.  **C++ 异常未被捕获：** 在 C++ 中，如果抛出的异常没有被任何 `catch` 块捕获，默认情况下会调用 `std::terminate()`，而 `std::terminate()` 的默认行为通常是调用 `abort()`。

5.  **某些安全检查失败：**
    *   **`_FORTIFY_SOURCE`:** 当使用 `-D_FORTIFY_SOURCE=2` (通常通过 `-O2` 或更高优化级别隐含启用) 编译时，glibc 会对一些标准库函数（如 `memcpy`, `strcpy`, `sprintf`）进行缓冲区溢出检查。如果检测到溢出，会调用 `abort()`。
    *   **Stack Canary/Stack Smashing Protection (`-fstack-protector`)：** 如果编译器启用了栈保护机制，并在函数返回时检测到栈上的保护值（canary）被修改（表明发生了栈缓冲区溢出），它会调用 `__stack_chk_fail`，后者通常会调用 `abort()`。

6.  **其他库的内部错误：** 第三方库（如加密库、图像处理库等）在遇到无法处理的严重错误时，也可能选择调用 `abort()`。


**如何排查退出码 134 (SIGABRT)：**

由于 `SIGABRT` 通常伴随着诊断信息，排查的关键在于捕获和分析这些信息：

1.  **检查程序输出 (`stdout` & `stderr`)：**
    *   **这是最重要的第一步！** `SIGABRT` 通常会在终止进程前将错误信息打印到标准错误 (`stderr`)。
    *   **仔细查看程序崩溃时的终端输出。**
    *   查找包含以下关键词的信息：
        *   `Aborted (core dumped)`
        *   `Error in ./your_program: ...`
        *   `malloc(): corrupted ...`
        *   `free(): invalid pointer ...`
        *   `double free or corruption ...`
        *   `stack smashing detected ...`
        *   `buffer overflow detected ...`
        *   `assertion failed: ...`
    *   这些信息直接指出了错误的类型和位置（文件名和行号）。

2.  **检查核心转储 (Core Dump)：**
    *   如果系统配置允许生成核心转储（检查 `ulimit -c`，通常默认为 0 即不生成），崩溃时会生成一个 `core` 或 `core.<pid>` 文件。
    *   使用调试器 (`gdb`) 加载程序和核心转储文件：
        ```bash
        gdb ./your_program core
        ```
    *   在 `gdb` 中：
        *   输入 `bt` (backtrace) 查看崩溃时的调用堆栈。这能告诉你 `abort()` 是在哪里被调用的。
        *   检查堆栈中各层函数的局部变量和参数。
        *   如果错误信息指向了具体的堆问题（如 `malloc`/`free` 错误），堆栈跟踪有助于定位是哪个分配/释放操作导致了问题。
    *   **启用核心转储：**
        *   临时：`ulimit -c unlimited` (在当前 shell 会话中生效)。
        *   永久：修改 `/etc/security/limits.conf` 或发行版特定的配置文件（如 Ubuntu 的 `/etc/systemd/coredump.conf`）。

3.  **使用内存调试工具：**
    *   **Valgrind (Memcheck)：** 这是检测内存错误（非法访问、泄漏、非法释放等）的金标准。在程序前加上 `valgrind` 运行：
        ```bash
        valgrind --leak-check=full ./your_program
        ```
        Valgrind 会详细报告内存错误的位置和原因。
    *   **AddressSanitizer (ASan)：** 编译时加入 `-fsanitize=address -g` 标志（GCC/Clang）。ASan 在运行时检测内存错误（堆栈缓冲区溢出、全局变量溢出、use-after-free 等），并提供非常精确的错误报告和堆栈跟踪，通常比 Valgrind 更快。
        ```bash
        gcc -g -fsanitize=address -o your_program your_program.c
        ./your_program
        ```
    *   **UndefinedBehaviorSanitizer (UBSan)：** 编译时加入 `-fsanitize=undefined -g`，检测未定义行为（如空指针解引用、有符号整数溢出等），这些行为有时也会间接导致内存破坏。
        ```bash
        gcc -g -fsanitize=undefined -o your_program your_program.c
        ./your_program
        ```

4.  **检查断言 (`assert`)：** 如果错误信息或堆栈跟踪指向了 `assert` 失败，检查失败的条件，理解为什么该条件在运行时未满足。

5.  **审查代码：** 结合错误信息、堆栈跟踪和工具报告，仔细审查相关代码区域。特别注意：
    *   动态内存的分配 (`malloc`, `calloc`, `realloc`, `new`) 和释放 (`free`, `delete`)。
    *   数组和指针操作，确保没有越界访问。
    *   字符串操作函数 (`strcpy`, `sprintf`, `strcat` 等)，优先使用安全版本 (`strncpy`, `snprintf`, `strncat`) 或确保边界安全。
    *   可能抛出异常的 C++ 代码是否被正确捕获。

**总结与 `mmap` 的关联：**

虽然退出码 134 本身直接指向 `SIGABRT`（通常是内存错误或断言失败），而不是 `mmap` 的 `errno`，但问题**可能**与 `mmap` 的使用间接相关：

1.  **内存覆盖：** 如果 `mmap` 映射的区域（尤其是使用 `MAP_FIXED`）意外覆盖了程序堆管理结构使用的内存区域，后续的 `malloc`/`free` 操作可能会破坏堆，最终导致 `SIGABRT`。
2.  **访问越界：** 对 `mmap` 映射的内存区域进行越界读写（特别是写操作），可能会破坏：
    *   堆内存（如果映射区域靠近堆）。
    *   栈内存（如果映射区域靠近栈）。
    *   其他映射区域或未映射区域（触发 `SIGSEGV` 或 `SIGBUS`，但严重破坏也可能间接导致后续 `SIGABRT`）。
3.  **映射区域释放后使用：** 在 `munmap` 一个区域后，如果程序仍然尝试访问该区域，会触发 `SIGSEGV`。如果这种访问发生在堆管理函数内部或破坏了堆结构，也可能最终导致 `SIGABRT`。

**因此，排查步骤应该是：**

1.  **首要关注 `SIGABRT` 的直接原因：** 仔细分析程序崩溃时的输出、核心转储（如果有）和使用内存调试工具（Valgrind/ASan）的结果。这些通常会直接指出错误根源（如 double free, buffer overflow）。
2.  **审查 `mmap` 的使用：** 在定位到大致问题区域后，检查该区域代码是否涉及 `mmap`。特别关注：
    *   `mmap` 的参数是否正确？尤其是 `addr` (是否用了 `MAP_FIXED`?)、`length`、`offset`。
    *   对映射区域的指针操作是否确保在 `[addr, addr + length - 1]` 范围内？
    *   映射区域的生存期管理是否正确？是否在 `munmap` 后还尝试访问？
    *   映射区域是否与堆/栈等重要区域重叠（可通过 `/proc/[pid]/maps` 检查）？

通过结合分析 `SIGABRT` 的详细报告和仔细审查 `mmap` 相关的代码逻辑，你应该能够找到导致程序终止的根本原因。

对应的是 **`EOWNERDEAD`**（Owner died）。以下是详细解释：

---

## 错误类型说明
1. **错误名称**：`EOWNERDEAD`
2. **含义**：  
   此错误通常与 **进程间同步机制**（如共享内存中的互斥锁）相关。当一个进程持有锁时异常终止（如崩溃），其他进程尝试获取该锁时会触发此错误。它表示锁的持有者已死亡，锁状态可能处于不一致状态，需要恢复操作。

3. **常见场景**：
   - 使用 `pthread_mutex_lock` 或共享内存中的互斥锁时。
   - 进程崩溃后，其他进程尝试获取该进程持有的锁。
   - 需要调用 `pthread_mutex_consistent` 修复锁状态后才能继续使用。

---

### 如何验证？
在 Linux 终端运行以下命令查看错误码定义：
```bash
$ errno 134
```
输出示例：
```
EOWNERDEAD 134  /* Owner died */
```

---

## 解决方法：
1. **检查锁的使用**：  
   确保使用锁的进程有健壮的退出处理逻辑（如信号处理、资源释放）。
2. **锁恢复机制**：  
   如果使用 `pthread_mutex` 的 `robust` 属性（`PTHREAD_MUTEX_ROBUST`），需在获取锁后检查错误码：
   ```c
   int err = pthread_mutex_lock(&mutex);
   if (err == EOWNERDEAD) {
       // 1. 使锁恢复一致状态
       pthread_mutex_consistent(&mutex);
       // 2. 执行数据恢复逻辑
   }
   ```
3. **避免资源泄漏**：  
   确保进程崩溃前释放锁（例如通过 `atexit` 注册清理函数）。

---

### 其他可能的高位错误码：
Linux 错误码通常较小（如 `ENOENT=2`）。若遇到 **大于 133** 的错误码（如 134），通常是：
- **实时扩展错误**（定义在 `<errno.h>`）：如 `EOWNERDEAD` (134)、`ENOTRECOVERABLE` (135)。
- **特定于同步机制**：与进程间通信（IPC）、线程或共享内存相关。

---

### 总结：
| 错误码 | 名称         | 原因与场景                     |
|--------|--------------|-------------------------------|
| 134    | `EOWNERDEAD` | 锁的持有者进程已终止，需恢复锁状态。 |

建议检查代码中是否有未处理的进程崩溃或锁恢复逻辑缺失。

# mmap 底层 Linux 内核代码

`/usr/include/sys/mman.h`

找 `MAP_FAILED` 字眼。

```cpp
/* Return value of `mmap' in case of an error.  */
#define MAP_FAILED	((void *) -1)

__BEGIN_DECLS
/* Map addresses starting near ADDR and extending for LEN bytes.  from
   OFFSET into the file FD describes according to PROT and FLAGS.  If ADDR
   is nonzero, it is the desired mapping address.  If the MAP_FIXED bit is
   set in FLAGS, the mapping will be at ADDR exactly (which must be
   page-aligned); otherwise the system chooses a convenient nearby address.
   The return value is the actual mapping address chosen or MAP_FAILED
   for errors (in which case `errno' is set).  A successful `mmap' call
   deallocates any previous mapping for the affected region.  */

#ifndef __USE_FILE_OFFSET64
extern void *mmap (void *__addr, size_t __len, int __prot,
		   int __flags, int __fd, __off_t __offset) __THROW;
#else
# ifdef __REDIRECT_NTH
extern void * __REDIRECT_NTH (mmap,
			      (void *__addr, size_t __len, int __prot,
			       int __flags, int __fd, __off64_t __offset),
			      mmap64);
# else
#  define mmap mmap64
# endif
#endif
#ifdef __USE_LARGEFILE64
extern void *mmap64 (void *__addr, size_t __len, int __prot,
		     int __flags, int __fd, __off64_t __offset) __THROW;
#endif
```

# 详解内存映射系统调用 mmap
```cpp
#include <sys/mman.h>
void* mmap(void* addr, size_t length, int prot, int flags, int fd, off_t offset);

// 内核文件：/arch/x86/kernel/sys_x86_64.c
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
  unsigned long, prot, unsigned long, flags,
  unsigned long, fd, unsigned long, off)
```

调用 mmap 进行匿名映射的时候（比如进行堆内存的分配），是将进程虚拟内存空间中的某一段虚拟内存区域与物理内存中的匿名内存页进行映射，
调用 mmap 进行文件映射的时候，是将进程虚拟内存空间中的某一段虚拟内存区域与磁盘中某个文件中的某段区域进行映射。

![](../../images/量化交易_mio库与csv库/image-20250925191855231.png)

在文件映射与匿名映射这段虚拟内存区域中，包含了一段一段的虚拟映射区，每当我们调用一次 mmap 进行内存映射的时候，内核都会在文件映射与匿名映射区中划分出一段虚拟映射区出来，这段虚拟映射区就是我们申请到的虚拟内存。

## mmap 的 前 2 个参数：addr、 length
addr ：表示我们要映射的这段虚拟内存区域在进程虚拟内存空间中的起始地址（虚拟内存地址），但是这个参数只是给内核的一个暗示，内核并非一定得从我们指定的 addr 虚拟内存地址上划分虚拟内存区域，内核只不过在划分虚拟内存区域的时候会优先考虑我们指定的 addr，如果这个虚拟地址已经被使用或者是一个无效的地址，那么内核则会自动选取一个合适的地址来划分虚拟内存区域。我们一般会将 addr 设置为 NULL，意思就是完全交由内核来帮我们决定虚拟映射区的起始地址。

length ：要申请的这段虚拟内存有多大呢 ？如果是匿名映射，length 参数决定了我们要映射的匿名物理内存有多大，如果是文件映射，length 参数决定了我们要映射的文件区域有多大。

> addr，length 必须要按照 PAGE_SIZE（4K） 对齐。

![](../../images/量化交易_mio库与csv库/image-20250925192405212.jpeg)

如果我们通过 mmap 映射的是磁盘上的一个文件，那么就需要通过参数 fd 来指定要映射文件的描述符（file descriptor），通过参数 offset 来指定文件映射区域在文件中偏移。

在内存管理系统中，物理内存是按照内存页为单位组织的；
在文件系统中，磁盘中的文件是按照磁盘块为单位组织的。

内存页和磁盘块大小一般情况下都是 4K 大小，所以这里的 offset 也必须是按照 4K 对齐的。
## 虚拟映射对象的结构体
而在文件映射与匿名映射区中的这一段一段的虚拟映射区，其实本质上也是虚拟内存区域，它们和进程虚拟内存空间中的代码段，数据段，BSS 段，堆，栈没有任何区别，
在内核中都是 `struct vm_area_struct` 结构来表示的，下面我们把进程空间中的这些虚拟内存区域统称为 VMA。

进程虚拟内存空间中的所有 VMA 在内核中有两种组织形式：一种是双向链表，用于高效的遍历进程 VMA，这个 VMA 双向链表是有顺序的，所有 VMA 节点在双向链表中的排列顺序是按照虚拟内存低地址到高地址进行的。

另一种则是用红黑树进行组织，用于在进程空间中高效的查找 VMA，因为在进程虚拟内存空间中不仅仅是只有代码段，数据段，BSS 段，堆，栈这些虚拟内存区域 VMA，尤其是在数据密集型应用进程中，文件映射与匿名映射区里也会包含有大量的 VMA，进程的各种动态链接库所映射的虚拟内存在这里，进程运行过程中进行的匿名映射，文件映射所需要的虚拟内存也在这里。而内核需要频繁地对进程虚拟内存空间中的这些众多 VMA 进行增，删，改，查。所以需要这么一个红黑树结构，方便内核进行高效的查找。


```cpp
// 进程虚拟内存空间描述符
struct mm_struct {
    // 串联组织进程空间中所有的 VMA  的双向链表 
    struct vm_area_struct *mmap;  /* list of VMAs */
    // 管理进程空间中所有 VMA 的红黑树
    struct rb_root mm_rb;
}

// 虚拟内存区域描述符
struct vm_area_struct {
    // vma 在 mm_struct->mmap 双向链表中的前驱节点和后继节点
    struct vm_area_struct *vm_next, *vm_prev;
    // vma 在 mm_struct->mm_rb 红黑树中的节点
    struct rb_node vm_rb;
}
```

![](../../images/量化交易_mio库与csv库/image-20250925192856271.png)

## 内核代码
`mmap()` 库调用由 libc 实现，它将字节偏移量转换为页面偏移量，然后调用 `mmap_pgoff()` 系统调用。`mmap_opgoff()` 系统调用获取与文件描述符参数对应的 `struct file *`，并调用 `vm_mmap_pgoff()`。

# gdb 调试
```
[INFO] filename: TA2601.csv.xz
[New Thread 0x7ffff77ff640 (LWP 1376157)]
[Thread 0x7ffff77ff640 (LWP 1376157) exited]
[New Thread 0x7ffff77ff640 (LWP 1376167)]
[Thread 0x7ffff77ff640 (LWP 1376167) exited]
[New Thread 0x7ffff77ff640 (LWP 1376168)]
[Thread 0x7ffff77ff640 (LWP 1376168) exited]
[New Thread 0x7ffff77ff640 (LWP 1376169)]
[Thread 0x7ffff77ff640 (LWP 1376169) exited]
[New Thread 0x7ffff77ff640 (LWP 1376177)]
[Thread 0x7ffff77ff640 (LWP 1376177) exited]
[New Thread 0x7ffff77ff640 (LWP 1376178)]
terminate called after throwing an instance of 'std::error_code'

Thread 7 "cmpmd5xz_O0" received signal SIGABRT, Aborted.
[Switching to Thread 0x7ffff77ff640 (LWP 1376178)]
0x00007ffff788bedc in __pthread_kill_implementation () from /lib64/libc.so.6
```


```
(gdb) thread apply all bt full

Thread 7 (Thread 0x7ffff77ff640 (LWP 1376178) "cmpmd5xz_O0"):
#0  0x00007ffff788bedc in __pthread_kill_implementation () from /lib64/libc.so.6
No symbol table info available.
#1  0x00007ffff783eb46 in raise () from /lib64/libc.so.6
No symbol table info available.
#2  0x00007ffff7828833 in abort () from /lib64/libc.so.6
No symbol table info available.
#3  0x00007ffff7cb2e64 in __gnu_cxx::__verbose_terminate_handler () at ../../../../libstdc++-v3/libsupc++/vterminate.cc:95
        terminating = true
        t = <optimized out>
#4  0x00007ffff7cc505a in __cxxabiv1::__terminate (handler=<optimized out>) at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:48
No locals.
#5  0x00007ffff7cb29ee in std::terminate () at ../../../../libstdc++-v3/libsupc++/eh_terminate.cc:58
No locals.
#6  0x00007ffff7cc52f8 in __cxxabiv1::__cxa_throw (obj=<optimized out>, tinfo=0x5555555c44b0 <typeinfo for std::error_code>, dest=0x0) at ../../../../libstdc++-v3/libsupc++/eh_throw.cc:98
        globals = <optimized out>
        header = 0x7ffff296ece0
#7  0x0000555555563b1c in csv::internals::MmapParser::next(unsigned long) [clone .cold] ()
No symbol table info available.
#8  0x0000555555595976 in csv::CSVReader::read_csv(unsigned long) ()
No symbol table info available.
#9  0x00007ffff7cf2ea4 in std::execute_native_thread_routine (__p=0x55555591ad20) at ../../../../../libstdc++-v3/src/c++11/thread.cc:104
        __t = <optimized out>
#10 0x00007ffff788a19a in start_thread () from /lib64/libc.so.6
No symbol table info available.
#11 0x00007ffff790f240 in clone3 () from /lib64/libc.so.6
No symbol table info available.

Thread 1 (Thread 0x7ffff7e8a740 (LWP 1376141) "cmpmd5xz_O0"):
#0  0x00007ffff78976c9 in _int_free () from /lib64/libc.so.6
No symbol table info available.
#1  0x00007ffff789a2c5 in free () from /lib64/libc.so.6
No symbol table info available.
#2  0x00005555555a1d7a in std::_Sp_counted_ptr_inplace<csv::internals::RawCSVData, std::allocator<void>, (__gnu_cxx::_Lock_policy)2>::_M_dispose() ()
No symbol table info available.
#3  0x0000555555596315 in csv::CSVReader::read_row(csv::CSVRow&) ()
No symbol table info available.
#4  0x000055555559a5b1 in csv::CSVReader::iterator::operator++() ()
No symbol table info available.
#5  0x0000555555565e4c in main (argc=3, argv=0x7fffffffcee8) at cmp_md5_xz.cpp:198
        row = @0x7fffffffb688: {data = {<std::__shared_ptr<csv::internals::RawCSVData, (__gnu_cxx::_Lock_policy)2>> = {<std::__shared_ptr_access<csv::internals::RawCSVData, (__gnu_cxx::_Lock_policy)2, false, false>> = {<No data fields>}, _M_ptr = 0x7ffff296e240, _M_refcount = {_M_pi = 0x7ffff296e230}}, <No data fields>}, data_start = 9999776, fields_start = 1972448, row_length = 32}
        __for_range = @0x7fffffffb7f0: {_format = {possible_delimiters = {<std::_Vector_base<char, std::allocator<char> >> = {_M_impl = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<char, std::allocator<char> >::_Vector_impl_data> = {_M_start = 0x5555555e0ac0 ",|\t;^U", _M_finish = 0x5555555e0ac1 "|\t;^U", _M_end_of_storage = 0x5555555e0ac5 "U"}, <No data fields>}}, <No data fields>}, trim_chars = {<std::_Vector_base<char, std::allocator<char> >> = {_M_impl = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, <std::_Vector_base<char, std::allocator<char> >::_Vector_impl_data> = {_M_start = 0x0, _M_finish = 0x0, _M_end_of_storage = 0x0}, <No data fields>}}, <No data fields>}, header = 0, no_quote = false, quote_char = 34 '"', col_names = {<std::_Vector_base<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > >> = {_M_impl = {<std::allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >> = {<std::__new_allocator<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > >> = {<No data fields>}, <No data fields>}, <std::_Vector_base<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::allocator
<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > >::_Vector_impl_data> = {_M_start = 0x0, _M_finish = 0x0, _M_end_of_storage = 0x0}, <No data fields>}}, <No data fields>}, variable_column_policy = csv::VariableColumnPolicy::IGNORE_ROW}, col_names = {<std::__shared_ptr<csv::internals::ColNames, (__gnu_cxx::_Lock_policy)2>> = {<std::__shared_ptr_access<csv::internals::ColNames, (__gnu_cxx::_Lock_policy)2, false, false>> = {<No data fields>}, _M_ptr = 0x5555555d87b0, _M_refcount = {_M_pi = 0x5555555d87a0}}, <No data fields>}, parser = {_M_t = {<std::__uniq_ptr_impl<csv::internals::IBasicCSVParser, std::default_delete<csv::internals::IBasicCSVParser> >> = {_M_t = {<std::_Tuple_impl<0, csv::internals::IBasicCSVParser*, std::default_delete<csv::internals::IBasicCSVParser> >> = {<std::_Tuple_impl<1, std::default_delete<csv::internals::IBasicCSVParser> >> = {<std::_Head_base<1, std::default_delete<csv::internals::IBasicCSVParser>, true>> = {_M_head_impl = {<No data fields>}}, <No data fields>}, <std::_Head_base<0, csv::internals::IBasicCSVParser*, false>> = {_M_head_impl = 0x5555564b4e80}, <No data fields>}, <No data fields>}}, <No data fields>}}, records = {_M_t = {<std::__uniq_ptr_impl<csv::internals::ThreadSafeDeque<csv::CSVRow>, std::default_delete<csv::internals::ThreadSafeDeque<csv::CSVRow> > >> = {_M_t = {<std::_Tuple_impl<0, csv::internals::ThreadSafeDeque<csv::CSVRow>*, std::default_delete<csv::internals::ThreadSafeDeque<csv::CSVRow> > >> = {<std::_Tuple_impl<1, std::default_delete<csv::internals::ThreadSafeDeque<csv::CSVRow> > >> = {<std::_Head_base<1, std::default_delete<csv::internals::ThreadSafeDeque<csv::CSVRow> >, true>> = {_M_head_impl = {<No data fields>}}, <No data fields>}, <std::_Head_base<0, csv::internals::ThreadSafeDeque<csv::CSVRow>*, false>> = {_M_head_impl = 0x5555555e00e0}, <No data fields>}, <No data fields>}}, <No data fields>}}, n_cols = 32, _n_rows = 61639, header_trimmed = true, read_csv_worker = {_M_id = {_M_thread = 140737345746496}}}
        __for_begin = {daddy = 0x7fffffffb7f0, row = {data = {<std::__shared_ptr<csv::internals::RawCSVData, (__gnu_cxx:
:_Lock_policy)2>> = {<std::__shared_ptr_access<csv::internals::RawCSVData, (__gnu_cxx::_Lock_policy)2, false, false>> = {<No data fields>}, _M_ptr = 0x7ffff296e240, _M_refcount = {_M_pi = 0x7ffff296e230}}, <No data fields>}, data_start = 9999776, fields_start = 1972448, row_length = 32}, i = 0}
        __for_end = {daddy = 0x0, row = {data = {<std::__shared_ptr<csv::internals::RawCSVData, (__gnu_cxx::_Lock_policy)2>> = {<std::__shared_ptr_access<csv::internals::RawCSVData, (__gnu_cxx::_Lock_policy)2, false, false>> = {<No data fields>}, _M_ptr = 0x0, _M_refcount = {_M_pi = 0x0}}, <No data fields>}, data_start = 0, fields_start = 0, row_length = 0}, i = 0}
        line_number = 0
        tmd = {_M_t = {_M_impl = {<std::allocator<std::_Rb_tree_node<std::pair<long const, MarketDataDepth5> > >> = {<std::__new_allocator<std::_Rb_tree_node<std::pair<long const, MarketDataDepth5> > >> = {<No data fields>}, <No data fields>}, <std::_Rb_tree_key_compare<std::less<long> >> = {_M_key_compare = {<std::binary_function<long, long, bool>> = {<No data fields>}, <No data fields>}}, <std::_Rb_tree_header> = {_M_header = {_M_color = std::_S_red, _M_parent = 0x555555769380, _M_left = 0x5555555db2a0, _M_right = 0x555556437c70}, _M_node_count = 78287}, <No data fields>}}}
        bmd = {_M_t = {_M_impl = {<std::allocator<std::_Rb_tree_node<std::pair<long const, MarketDataDepth5> > >> = {<std::__new_allocator<std::_Rb_tree_node<std::pair<long const, MarketDataDepth5> > >> = {<No data fields>}, <No data fields>}, <std::_Rb_tree_key_compare<std::less<long> >> = {_M_key_compare = {<std::binary_function<long, long, bool>> = {<No data fields>}, <No data fields>}}, <std::_Rb_tree_header> = {_M_header = {_M_color = std::_S_red, _M_parent = 0x55555673b310, _M_left = 0x5555564b5450, _M_right = 0x555556f83590}, _M_node_count = 61614}, <No data fields>}}}
        bfs = {<std::basic_istream<char, std::char_traits<char> >> = {<std::basic_ios<char, std::char_traits<char> >> = {<std::ios_base> = {_vptr.ios_base = 0x7ffff7e76dc8 <vtable for std::basic_ifstream<char, std::char_traits<char> >+64>, static boolalpha = std::_S_boolalpha, static dec = std::_S_dec, static fixed = std::_S_fixed, static hex = std::_S_hex, static internal = std::_S_internal, static left = std::_S_left, static oct = std::_S_oct, static right = std::_S_right, static scientific = std::_S_scientific, static showbase = std::_S_showbase, static showpoint = std::_S_showpoint, static showpos = std::_S_showpos, static skipws = std::_S_skipws, static unitbuf = std::_S_unitbuf, static uppercase = std::_S_uppercase, static adjustfield = std::_S_adjustfield, static basefield = std::_S_basefield, static floatfield = std::_S_floatfield, static badbit = std::_S_badbit, static eofbit = std::_S_eofbit, static failbit = std::_S_failbit, static goodbit = std::_S_goodbit, static app = std::_S_app, static ate = std::_S_ate, static binary = std::_S_bin, static in = std::_S_in, static out = std::_S_out, static trunc = std::_S_trunc, static __noreplace = std::_S_noreplace, static beg = std::_S_beg, static cur = std::_S_cur, static end = std::_S_end, _M_precision = 6, _M_width = 0, _M_flags = 4098, _M_exception = std::_S_goodbit, _M_streambuf_state = std::_S_goodbit, _M_callbacks = 0x0, _M_word_zero = {_M_pword = 0x0, _M_iword = 0}, _M_local_word = {{_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}}, _M_word_size = 8, _M_word = 0x7fffffffbd10, _M_ios_locale = {static none = 0, static ctype = 1, static numeric = 2, static collate = 4, static time = 8, static monetary = 16, static messages = 32, static all = 63, _M_impl = 0x7ffff7e80080 <(anonymous namespace)::c_locale_impl>}}, _M_tie = 0x0, _M_fill = 32 ' ', _M_fill_init = true, _M_streambuf = 0x7fffffffbbe0, _M_ctype = 0x7ffff7e7faa0 <(anonymous namespace)::ctype_c>, _M_num_put = 0x7ffff7e7fa30 <(anonymous namespace)::num_put_c>, _M_num_get = 0x7ffff7e7fa40 <(anonymous namespace)::num_get_c>}, _vptr.basic_istream = 0x7ffff7e76da0 <vtable for std::basic_ifstream<char, std::char_traits<char> >+24>, _M_gcount = 0}, _M_filebuf = {<std::basic_streambuf<char, std::char_traits<char> >> = {_vptr.basic_streambuf = 0x7ffff7e76ca8 <vtable for std::basic_filebuf<char, std::char_traits<char> >+16>, _M_in_beg = 0x555556437d30 "θ,\023V(\037\220X\267m#\323\322\326Z\317oUJ\262fٜ\254>;P*\267\263\003\303\322\030\226\226\276\005U\247i\257)\230\001/\037\314G8\314\342\201:>\0275\357ɐ\220\3269\253\207ld\223>\027\251s5\004߰\001\355\342=\\\264S\026\350v&\034\231<\243E\335\r'^\266M[\341\334O\242n\233N\bF\3634!\243J\3338\025`\304ɀ\260\314Ĭ\277.\343E\365xI[0\243\355ޔ\375\021Y\017\223\257\222\271\rf", _M_in_cur = 0x555556437d30 "θ,\023V(\037\220X\267m#\323\322\326Z\317oUJ\262fٜ\254>;P*\267\263\003\303\322\030\226\226\276\005U\247i\257)\230\001/\037\314G8\314\342\201:>\0275\357ɐ\220\3269\253\207ld\223>\027\251s5\004߰\001\355\342=\\\264S\026\350v&\034\231<\243E\335\r'^\266M[\341\334O\242n\233N\bF\3634!\243J\3338\025`\304ɀ\260\314Ĭ\277.\343E\365xI[0\243\355ޔ\375\021Y\017\223\257\222\271\rf", _M_in_end = 0x555556437d30 "θ,\023V(\037\220X\267m#\323\322\326Z\317oUJ\262fٜ\254>;P*\267\263\003\303\322\030\226\226\276\005U\247i\257)\230\001/\037\314G8\314\342\201:>\0275\357ɐ\220\3269\253\207ld\223>\027\251s5\004߰\001\355\342=\\\264S\026\350v&\034\231<\243E\335\r'^\266M[\341\334O\242n\233N\bF\3634!\243J\3338\025`\304ɀ\260\314Ĭ\277.\343E\365xI[0\243\355ޔ\375\021Y\017\223\257\222\271\rf", _M_out_beg = 0x0, _M_out_cur = 0x0, _M_out_end = 0x0, _M_buf_locale = {static none = 0, static ctype = 1, static numeric = 2, static collate = 4, static time = 8, static monetary = 16, static messages = 32, static all = 63, _M_impl = 0x7ffff7e80080 <(anonymous namespace)::c_locale_impl>}}, _M_lock = {__data = {__lock = 0, __count = 0, __owner = 0, __nusers = 0, __kind = 0, __spins = 0, __elision = 0, __list = {__prev = 0x0, __next = 0x0}}, __size = '\000' <repeats 39 times>, __align = 0}, _M_file = {_M_cfile = 0x7ffff2fefbc0, _M_cfile_created = true}, _M_mode = 12, _M_state_beg = {__count = 0, __value = {__wch = 0, __wchb = "\000\000\000"}}, _M_state_cur = {__count = 0, __value = {__wch = 0, __wchb = "\000\000\000"}}, _M_state_last = {__count = 0, __value = {__wch = 0, __wchb = "\000\000\000"}}, _M_buf = 0x555556437d30 "θ,\023V(\037\220X\267m#\323\322\326Z\317oUJ\262fٜ\254>;P*\267\263\003\303\322\030\226\226\276\005U\247i\257)\230\001/\037\314G8\314\342\201:>\0275\357ɐ\220\3269\253\207ld\223>\027\251s5\004߰\001\355\342=\\\264S\026\350v&\034\231<\243E\335\r'^\266M[\341\334O\242n\233N\bF\3634!\243J\3338\025`\304ɀ\260\314Ĭ\277.\343E\365xI[0\243\355ޔ\375\021Y\017\223\257\222\271\rf", _M_buf_size = 8192, _M_buf_allocated = true, _M_reading = false, _M_writing = false, _M_pback = 0 '\000', _M_pback_cur_save = 0x0, _M_pback_end_save = 0x0, _M_pback_init = false, _M_codecvt = 0x7ffff7e7fa10 <(anonymous namespace)::codecvt_c>, _M_ext_buf = 0x0, _M_ext_buf_size = 0, _M_ext_next = 0x0, _M_ext_end = 0x0}}
        bin = {<boost::iostreams::detail::filtering_stream_base<boost::iostreams::chain<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char> >, boost::iostreams::public_>> = {<boost::iostreams::access_control<boost::iostreams::detail::chain_client<boost::iostreams::chain<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char> > >, boost::iostreams::public_, boost::iostreams::detail::pub_<boost::iostreams::detail::chain_client<boost::iostreams::chain<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char> > > > >> = {<boost::iostreams::detail::pub_<boost::iostreams::detail::chain_client<boost::iostreams::chain<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char> > > >> = {<boost::iostreams::detail::chain_client<boost::iostreams::chain<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char> > >> = {_vptr.chain_client = 0x5555555c3760 <vtable for boost::iostreams::filtering_stream<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char>, boost::iostreams::public_>+24>, chain_ = 0x7fffffffb8b0}, <No data fields>}, <No data fields>}, <std::basic_istream<char, std::char_traits<char> >> = {<std::basic_ios<char, std::char_traits<char> >> = {<std::ios_base> = {_vptr.ios_base = 0x5555555c37b8 <vtable for boost::iostreams::filtering_stream<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char>, boost::iostreams::public_>+112>, static boolalpha = std::_S_boolalpha, static dec = std::_S_dec, static fixed = std::_S_fixed, static hex = std::_S_hex, static internal = std::_S_internal, static left = std::_S_left, static oct = std::_S_oct, static right = std::_S_right, static scientific = std::_S_scientific, static showbase = std::_S_showbase, static showpoint = std::_S_showpoint, static showpos = std::_S_showpos, static skipws = std::_S_skipws, static unitbuf = std::_S_unitbuf, static uppercase = std::_S_uppercase, static adjustfield = std::_S_adjustfield, static basefield = std::_S_basefield, static floatfield = std::_S_floatfield, static badbit = std::_S_badbit, static eofbit = std::_S_eofbit, static failbit = std::_S_failbit, static goodbit = std::_S_goodbit, static app = std::_S_app, static ate = std::_S_ate, static binary = std::_S_bin, static in = std::_S_in, static out = std::_S_out, static trunc = std::_S_trunc, static __noreplace = std::_S_noreplace, static beg = std::_S_beg, static cur = std::_S_cur, static end = std::_S_end, _M_precision = 6, _M_width = 0, _M_flags = 4098, _M_exception = std::_S_goodbit, _M_streambuf_state = std::_S_goodbit, _M_callbacks = 0x0, _M_word_zero = {_M_pword = 0x0, _M_iword = 0}, _M_local_word = {{_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}}, _M_word_size = 8, _M_word = 0x7fffffffb900, _M_ios_locale = {static none = 0, static ctype = 1, static numeric = 2, static collate = 4, static time = 8, static monetary = 16, static messages = 32, static all = 63, _M_impl = 0x7ffff7e80080 <(anonymous namespace)::c_locale_impl>}}, _M_tie = 0x0, _M_fill = 32 ' ', _M_fill_init = true, _M_streambuf = 0x5555555db450, _M_ctype = 0x7ffff7e7faa0 <(anonymous namespace)::ctype_c>, _M_num_put = 0x7ffff7e7fa30 <(anonymous namespace)::num_put_c>, _M_num_get = 0x7ffff7e7fa40 <(anonymous namespace)::num_get_c>}, _vptr.basic_istream = 0x5555555c3790 <vtable for boost::iostreams::filtering_stream<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char>, boost::iostreams::public_>+72>, _M_gcount = 0}, chain_ = {<boost::iostreams::detail::chain_base<boost::iostreams::chain<boost::iostreams::input, char, std::char_traits<char>, std::allocator<char> >, char, std::char_traits<char>, std::allocator<char>, boost::iostreams::input>> = {pimpl_ = {px = 0x7ffff00ff8d0, pn = {pi_ = 0x5555555dcbd0}}}, <No data fields>}}, <No data fields>}
        bof = {<std::basic_ostream<char, std::char_traits<char> >> = {<std::basic_ios<char, std::char_traits<char> >> = {<std::ios_base> = {_vptr.ios_base = 0x7ffff7e76e88 <vtable for std::basic_ofstream<char, std::char_traits<char> >+64>, static boolalpha = std::_S_boolalpha, static dec = std::_S_dec, static fixed = std::_S_fixed, static hex = std::_S_hex, static internal = std::_S_internal, static left = std::_S_left, static oct = std::_S_oct, static right = std::_S_right, static scientific = std::_S_scientific, static showbase = std::_S_showbase, static showpoint = std::_S_showpoint, static showpos = std::_S_showpos, static skipws = std::_S_skipws, static unitbuf = std::_S_unitbuf, static uppercase = std::_S_uppercase, static adjustfield = std::_S_adjustfield, static basefield = std::_S_basefield, static floatfield = std::_S_floatfield, static badbit = std::_S_badbit, static eofbit = std::_S_eofbit, static failbit = std::_S_failbit, static goodbit = std::_S_goodbit, static app = std::_S_app, static ate = std::_S_ate, static binary = std::_S_bin, static in = std::_S_in, static out = std::_S_out, static trunc = std::_S_trunc, static __noreplace = std::_S_noreplace, static beg = std::_S_beg, static cur = std::_S_cur, static end = std::_S_end, _M_precision = 6, _M_width = 0, _M_flags = 4098, _M_exception = std::_S_goodbit, _M_streambuf_state = std::_S_failbit, _M_callbacks = 0x0, _M_word_zero = {_M_pword = 0x0, _M_iword = 0}, _M_local_word = {{_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}, {_M_pword = 0x0, _M_iword = 0}}, _M_word_size = 8, _M_word = 0x7fffffffbb08, _M_ios_locale = {static none = 0, static ctype = 1, static numeric = 2, static collate = 4, static time = 8, static monetary = 16, static messages = 32, static all = 63, _M_impl = 0x7ffff7e80080 <(anonymous namespace)::c_locale_impl>}}, _M_tie = 0x0, _M_fill = 32 ' ', _M_fill_init = true, _M_streambuf = 0x7fffffffb9d8, _M_ctype = 0x7ffff7e7faa0 <(anonymous namespace)::ctype_c>, _M_num_put = 0x7ffff7e7fa30 <(anonymous namespace)::num_put_c>, _M_num_get = 0x7ffff7e7fa40 <(anonymous namespace)::num_get_c>}, _vptr.basic_ostream = 0x7ffff7e76e60 <vtable for std::basic_ofstream<char, std::char_traits<char> >+24>}, _M_filebuf = {<std::basic_streambuf<char, std::char_traits<char> >> = {_vptr.basic_streambuf = 0x7ffff7e76ca8 <vtable for std::basic_filebuf<char, std::char_traits<char> >+16>, _M_in_beg = 0x0, _M_in_cur = 0x0, _M_in_end = 0x0, _M_out_beg = 0x0, _M_out_cur = 0x0, _M_out_end = 0x0, _M_buf_locale = {static none = 0, static ctype = 1, static numeric = 2, static collate = 4, static time = 8, static monetary = 16, static messages = 32, static all = 63, _M_impl = 0x7ffff7e80080 <(anonymous namespace)::c_locale_impl>}}, _M_lock = {__data = {__lock = 0, __count = 0, __owner = 0, __nusers = 0, __kind = 0, __spins = 0, __elision = 0, __list = {__prev = 0x0, __next = 0x0}}, __size = '\000' <repeats 39 times>, __align = 0}, _M_file = {_M_cfile = 0x0, _M_cfile_created = true}, _M_mode = 0, _M_state_beg = {__count = 0, __value = {__wch = 0, __wchb = "\000\000\000"}}, _M_state_cur = {__count = 0, __value = {__wch = 0, __wchb = "\000\000\000"}}, _M_state_last = {__count = 0, __value = {__wch = 0, __wchb = "\000\000\000"}}, _M_buf = 0x0, _M_buf_size = 8192, _M_buf_allocated = false, _M_reading = false, _M_writing = false, _M_pback = 0 '\000', _M_pback_cur_save = 0x0, _M_pback_end_save = 0x0, _M_pback_init = false, _M_codecvt = 0x7ffff7e7fa10 <(anonymous namespace)::codecvt_c>, _M_ext_buf = 0x0, _M_ext_buf_size = 0, _M_ext_next = 0x0, _M_ext_end = 0x0}}
        tstpath = {static preferred_separator = 47 '/', _M_pathname = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x5555555e0f90 "/home/fdata/raw/tick/CZCE/depth5/20250821/TA2601.csv.xz"}, _M_string_length = 55, {_M_local_buf = "T", '\000' <repeats 14 times>, _M_allocated_capacity = 84}}, _M_cmpts = {_M_impl = {_M_t = {<std::__uniq_ptr_impl<std::filesystem::__cxx11::path::_List::_Impl, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {_M_t = {<std::_Tuple_impl<0, std::filesystem::__cxx11::path::_List::_Impl*, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {<std::_Tuple_impl<1, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {<std::_Head_base<1, std::filesystem::__cxx11::path::_List::_Impl_deleter, true>> = {_M_head_impl = {<No data fields>}}, <No data fields>}, <std::_Head_base<0, std::filesystem::__cxx11::path::_List::_Impl*, false>> = {_M_head_impl = 0x5555555e0ff0}, <No data fields>}, <No data fields>}}, <No data fields>}}}}
        benpath = {static preferred_separator = 47 '/', _M_pathname = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x5555555d8f10 "/home/fdata/raw/tick/MDC/depth5/20250821/TA2601.csv.xz"}, _M_string_length = 54, {_M_local_buf = "R", '\000' <repeats 14 times>, _M_allocated_capacity = 82}}, _M_cmpts = {_M_impl = {_M_t = {<std::__uniq_ptr_impl<std::filesystem::__cxx11::path::_List::_Impl, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {_M_t = {<std::_Tuple_impl<0, std::filesystem::__cxx11::path::_List::_Impl*, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {<std::_Tuple_impl<1, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {<std::_Head_base<1, std::filesystem::__cxx11::path::_List::_Impl_deleter, true>> = {_M_head_impl = {<No data fields>}}, <No data fields>}, <std::_Head_base<0, std::filesystem::__cxx11::path::_List::_Impl*, false>> = {_M_head_impl = 0x5555555d8f70}, <No data fields>}, <No data fields>}}, <No data fields>}}}}
        f = @0x5555555e4c30: {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x5555555e4c40 "TA2601.csv.xz"}, _M_string_length = 13, {_M_local_buf = "TA2601.csv.xz\000\000", _M_allocated_capacity = 7146703741821010260}}
        __for_range = @0x7fffffffb5f0: {_M_t = {_M_impl = {<std::allocator<std::_Rb_tree_node<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > >> = {<std::__new_allocator<std::_Rb_tree_node<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > >> = {<No data fields>}, <No data fields>}, <std::_Rb_tree_key_compare<std::less<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > > >> = {_M_key_compare = {<std::binary_function<std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >, bool>> = {<No data fields>}, <No data fields>}}, <std::_Rb_tree_header> = {_M_header = {_M_color = std::_S_red, _M_parent = 0x5555555e1510, _M_left = 0x5555555e5520, _M_right = 0x5555555e4350}, _M_node_count = 241}, <No data fields>}}}
        __for_begin = {_M_node = 0x5555555e4c10}
        __for_end = {_M_node = 0x7fffffffb5f8}
        exchange = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7fffffffbef0 "CZCE"}, _M_string_length = 4, {_M_local_buf = "CZCE\000\000\000\000@d\032\001\000\000\000", _M_allocated_capacity = 1162041923}}
        date = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x7fffffffbed0 "20250821"}, _M_string_length = 8, {_M_local_buf = "20250821\000\355\372\367\377\177\000", _M_allocated_capacity = 3544957636396068914}}
        bendir = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x5555555d8650 "/home/fdata/raw/tick/MDC/depth5/20250821/"}, _M_string_length = 41, {_M_local_buf = "P", '\000' <repeats 14 times>, _M_allocated_capacity = 80}}
        tstdir = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x5555555d86e0 "/home/fdata/raw/tick/CZCE/depth5/20250821/"}, _M_string_length = 42, {_M_local_buf = "<\000\000\000\000\000\000\000p\276\377\377\377\177\000", _M_allocated_capacity = 60}}
        mktopen = 20250821090000000
        eod = 20250821150000000
        son = 20250821210000000
        temp_path = {static preferred_separator = 47 '/', _M_pathname = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x5555555d8810 "/tmp/tempfile_1759759482756864030"}, _M_string_length = 33, {_M_local_buf = "!\000\000\000\000\000\000\000\000\364\374\367\377\177\000", _M_allocated_capacity = 33}}, _M_cmpts = {_M_impl = {_M_t = {<std::__uniq_ptr_impl<std::filesystem::__cxx11::path::_List::_Impl, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {_M_t = {<std::_Tuple_impl<0, std::filesystem::__cxx11::path::_List::_Impl*, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {<std::_Tuple_impl<1, std::filesystem::__cxx11::path::_List::_Impl_deleter>> = {<std::_Head_base<1, std::filesystem::__cxx11::path::_List::_Impl_deleter, true>> = {_M_head_impl = {<No data fields>}}, <No data fields>}, <std::_Head_base<0, std::filesystem::__cxx11::path::_List::_Impl*, false>> = {_M_head_impl = 0x5555555d8840}, <No data fields>}, <No data fields>}}, <No data fields>}}}}
        tempfile = {_M_dataplus = {<std::allocator<char>> = {<std::__new_allocator<char>> = {<No data fields>}, <No data fields>}, _M_p = 0x5555555d86b0 "/tmp/tempfile_1759759482756864030"}, _M_string_length = 33, {_M_local_buf = "!\000\000\000\000\000\000\000>)\375\367\377\177\000", _M_allocated_capacity = 33}}
```