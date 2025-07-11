---
title: C语言_文件
categories:
  - - C
tags: 
date: 2024/1/17
updated: 
comments: 
published:
---
# 文件

## FILE

```c
typedef struct _iobuf
{
    void * _Placeholder;
}FILE;
_ACRTIMP _ALT FILE* __cdecl __acrt_iob_func(unsigned _Ix);
```

实际就是一个无符号整型ID值，当做了形式为指针来用了。

为什么要用指针，是为了防止单纯的整型随意被赋值，而不同类型的指针不可以随意赋值，因此可以用指针来阻挡一下，出错概率就小了。

### mode

mode是以内存视角来说的：

1. 从设备写入到内存中是input，是`"r"`；
2. 从内存中写出到磁盘是output，是`"w"`。如果文件存在，则删掉内容，从0开始写。

### fopen

```c
int main()
{
    FILE * file = fopen("e:\\aa.txt", "w");  //绝对路径
    fclose(file);
}
```

`\`是转义字符，所以，`\\`表示`\`。如果不想写`\\`，也可以只写一个`/`

### 写东西

文件中有两个流，输出流和输入流，两个流的指针位置互相独立、互不干扰。指针位置可以用`fseek`、`fsetpos`等改变。

在这里用fwrite写入文件：Write block of data to stream.

```c
size_t fwrite(const void * ptr, size_t size, size_t count, FILE * stream);
// ptr指示缓冲区的起始位置；Pointer to the array of elements to be written, converted to a const void *
// size是要写入的元素类型的大小
// count是有多少个元素
// stream是哪个文件
// 返回值是指写入了多少个元素，而不是字节数。
```

```c
int main()
{
    int val = 24;
    FILE * file = fopen("e:\\aa.txt", "w");  //绝对路径
    if(NULL == file) return 0;
    // 写入一个int。
    size_t byte_written = fwrite(&val, sizeof val, 1, file);
    fclose(file);
    file = NULL;
}
```

### 程序中读文件

如果已经有存在的文件，则就不要再执行写入的部分。需要用宏条件语句来控制程序。

fread是Read block of data from stream.

```c
size_t fread(void * ptr, size_t size, size_t count, FILE * stream);
```

```c
int main()
{
    FILE * file = NULL;
#ifdef _CREATE_FILE__
    int val = 24;
    file = fopen("e:\\aa.txt", "w");
    if(!file)
        return 0;
    size_t num_written = fwrite(&val, sizeof val, 1, file);
#else
    int val = 0;
    file = fopen("e:\\aa.txt", "r");
    size_t size_to_read = sizeof val;
    if(!file)
        return 0;
    // 返回的是元素个数，为1个
    size_t num_read = fread(&val, sizeof val, 1, file);
    if(num_read == size_to_read)
        printf("%i\n", val);
#endif // _CREATE_FILE__
    fclose(file);
    file = NULL;
    return 0;
}// 24
```

### 实际上

默认暂时在缓存区中存放内容，比如向磁盘中写入东西，直到缓冲区满时，才会真正开始写入磁盘。也可以中途调用`fflush`进行立即刷新缓存区。
