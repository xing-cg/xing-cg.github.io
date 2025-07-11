---
title: C语言_数组_字符串
categories:
  - - C
tags: 
date: 2024/1/11
updated: 
comments: 
published:
---
# 数组

## 初始化

```c
int main()
{
    int a[8] = { 1, 2, 3, 4, 5, 6, 7, 8 };
    for(int i = 0; i < 8; ++i)
    {
        printf("%i\n", a[i]);
    }
}// 1 2 3 4 5 6 7 8
```

初始化时，方括号内的数字可以省略

```c
int main()
{
    int a[] = { 1, 2, 3, 4, 5, 6, 7, 8 };
    for(int i = 0; i < 8; ++i)
    {
        printf("%i\n", a[i]);
    }
}// 1 2 3 4 5 6 7 8
```

不给具体值，则里面装的是未定义值。

```c
int main()
{
    int a[8];
    for(int i = 0; i < 8; ++i)
    {
        printf("%i\n", a[i]);
    }
}// -858993460 -858993460 -858993460 -858993460 -858993460 -858993460 -858993460 -858993460
```

只给一个1，则里面装有1个1和7个0

```c
int main()
{
    int a[] = { 1 };
    for(int i = 0; i < 8; ++i)
    {
        printf("%i\n", a[i]);
    }
}// 1 0 0 0 0 0 0 0
```

只给一个1和2，则里面装有1个1、1个2和6个0。而且与编译版本无关，Debug和Release版本的行为一致。

```c
int main()
{
    int a[] = { 1, 2 };
    for(int i = 0; i < 8; ++i)
    {
        printf("%i\n", a[i]);
    }
}// 1 2 0 0 0 0 0 0
```

所以不要误会，初始化时只给一个0，只是恰巧全是0，如果只给一个1，那么不会全是1。

## 通用化遍历输出

1. `sizeof`+`数组名`=`整个数组所占的字节大小`
2. `sizeof`+`数组名[0]`=`数组一个单独元素的大小`。既然有一个数组被定义出来，里面一定有一个元素，则`数组名[0]`一定存在。
3. 则`整个数组所占的字节大小`/`数组一个单独元素的大小`=`数组元素个数`

```c
int main()
{
    int a[8] = { 1, 2, 3, 4, 5, 6, 7, 8 };
    for(int i = 0; i < sizeof a / sizeof a[0]; ++i)
    {
        printf("%i\n", a[i]);
    }
}// 1 2 3 4 5 6 7 8
```

## 特质

数组一旦建立之后，就不能再改动它的属性了，比如容量。

# 字符串

如果数组中放的是如ASCII的字符，则可以输出一个字符串。但是如果全部遍历的话会出现无效信息。

```c
int main()
{
    char a[8] = { 'H', 'e', 'l', 'l', 'o' };
    for(int i = 0; i < sizeof a / sizeof a[0]; ++i)
        printf("%c\n", a[i]);
}
//H
//e
//l
//l
//o
//空行  //实际为'\0'+'\n'
//空行  //实际为'\0'+'\n'
//空行  //实际为'\0'+'\n'
//光标最终位置
```

## 普通char数组加哨兵模式进行遍历输出

ASCII码标准中规定，char值为0表示NULL，因此在字符数组中可以代表哨兵位，意味着此位之后的值无效。字符数组在实际中往往不知道有效位数，因为它是由哨兵位动态确定的，因此应该用while+break进行遍历输出字符。

```c
int main()
{
    char a[8] = { 'H', 'e', 'l', 'l', 'o', '\0' };
    int i = 0;
    while(a[i] != '\0')
        printf("%c", a[i++]);
}
//Hello
```

## C语言提供的字符数组简洁化操作

### 初始化

如下，其实`"Hello"`的本质就是`{ 'H', 'e', 'l', 'l', 'o', '\0' }`。是一种语法糖

```c
int main()
{
    char a[8] = "Hello"; // "Hello"的本质是{ 'H', 'e', 'l', 'l', 'o', '\0' }
    int i = 0;
    while(a[i] != '\0')
        printf("%c", a[i++]);
}
//Hello
```

### 输出

printf中`%s`可以直接输出一个字符数组。

```c
int main()
{
    char a[8] = "Hello"; // "Hello"的本质是{ 'H', 'e', 'l', 'l', 'o', '\0' }
    printf("%s", a);
}
//Hello
```

但是注意，`{ 'H', 'e', 'l', 'l', 'o', '\0' }`和`"Hello"`不能完全等同。`"Hello"`的功能大于`{ 'H', 'e', 'l', 'l', 'o', '\0' }`，`"Hello"`可以直接被`%s`输出，而`{ 'H', 'e', 'l', 'l', 'o', '\0' }`不能。

```c
int main()
{
    printf("%s", "Hello");                             //ok
    //printf("%s", { 'H', 'e', 'l', 'l', 'o', '\0' }); //error
}
//Hello
```

## 操作字符串的C系统库函数

微软下调用有写入操作的字符串库函数时，默认不允许使用不带`_s`的函数。可以在VS如下设置即可使用不带`_s`的普通函数：右键项目-属性（Configuration Properties）-`C/C++`-Preprocessor-Preprocessor Definitions下拉-Edit-在已有内容后换行写入`_CRT_SECURE_NO_WARNINGS`-OK

1. strlen

2. strcpy

3. strcat
   ```c
   void str_cat(char des[], char src[])
   {
       int i = 0, j = 0;
       while(des[i++]);//找到des的末尾
       --j;//上面找到末尾后多移了一位，需要前移到第一个\0位置
       while(des[i++] = src[j++]);
   }
   ```

   以上虽然可以达到目的，但是我们应该尽量利用`i`和`j`之间的相对位置关系，可以更进一步地优化性能：

   ```c
   void str_cat(char des[], char src[])
   {
       int i = 0, j = 0;
       while(des[i++]);//找到des的末尾
       --j;//上面找到末尾后多移了一位，需要前移a到第一个\0位置
       while(des[i + j] = src[j++]);
   }
   ```

   以上，省略了一个`i++`的步骤，达到了性能的极致优化，而不是滥用后置`++`。

4. strcmp
   依次比较两个字符串中的字符，一旦不相等则返回，如果在一直相等的前提下，一旦遇到`\0`则返回。

   ```c
   int str_cmp(char str1[], char str2[])
   {
       int i = 0;
       while(str1[i] == str2[i] && str1[i] != '\0')
           ++i;
       return str1[i] - str2[i];
   }
   ```

5. strstr - locate sub string - 此处为朴素算法，提高版的有KMP、BM算法。
   库函数返回的是一个char指针，我们此处返回子串具体开始的下标值。

   ```c
   int str_str(char str[], char substr[])
   {
       int i = 0, j = 0;
       while(str[i] != '\0')
       {
           while(str[i + j] == str[j] && str[j] != '\0')
               ++j;
           if(str[j] == '\0')
           {
               return i;
           }
           else//str[i + j] != str[j]
           {
               j = 0;
               ++i;
           }
       }
       return -1;
   }
   ```

### 总结

字符串的算法思想，归纳出来可以总结为：很多时候需要找到最长的路线，即为临界点，从这个临界点退出循环则开始进行各种情况的分支，再一一去解决、处理。
