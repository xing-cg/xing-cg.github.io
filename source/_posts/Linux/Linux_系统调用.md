---
title: Linux_系统调用
categories:
  - - Linux
  - - 操作系统
tags: 
date: 2021/10/17
updated: 2025/7/17
comments: 
published:
---
# 文件操作

打开，读或写，关闭

open的返回值：int型，作为文件描述符。

read

write

close

C语言中的打开文件函数：`fopen`，库函数，返回值为`FILE *`，有`stdin`、`stdout`、`stderr`。

# mybash

```c
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<assert.h>
#include<string.h>
#include<sys/wait.h>
#include<errno.h>
#include<pwd.h>
char * get_cmd(char buff[], char* myargv[])
{
    if(NULL==buff || NULL==myargv)
    {
        return NULL;
    }
    char *s = strtok(buff," ");
    int i = 0;
    while(s != NULL)
    {
        myargv[i++] = s;
        s = strtok(NULL," ");
    }
    return myargv[0];
}
void print_info()
{
    //getuid -->unistd.h
    int id = getuid();
    char *s = "$";
    if(id == 0)//root is 0
    {
        s = "#";
    }
    //#inlcude<pwd.h>
    //getpwuid
    struct passwd* ptr = getpwuid(id);//get info of user through uid
    if(ptr == NULL)
    {
        printf("$");
        fflush(stdout);
        return;
    }
    //gethostname-->unistd.h
    char hostname[64] = {0};
    if(gethostname(hostname,64)==-1)
    { 
        printf("$");
        fflush(stdout);
        return;
    }
    //chdir & getcwd --> #include<unistd.h>
    char pwd_buff[128] = {0};
    if(getcwd(pwd_buff, 128)==NULL)
    {    
        printf("$");
        fflush(stdout);
        return;
    }
    //ptr->pw_name:用户名字。（xcg）				   -->getpwuid-->pwd.h
    //hostname:主机名。（ubuntu）					   -->gethostname-->unistd.h
    //pwd_buff:当前目录完整信息（/home/xcg/c219/1024） -->getcwd-->unistd.h
    //s:管理员或普通标识（#为管理员，$为普通）			-->getuid-->unistd.h
    //最后效果：xcg @ ubuntu : /home/xcg/c219/1024 $
    printf("%s@%s:%s%s ", ptr->pw_name, hostname, pwd_buff,s);
    fflush(stdout);
    /*
    char **p = engv;
    while(*p != NULL)
    {
        if(strncmp(*p,"PWD",3)==0)
        {
            char *s = &((*p)[4]);
            printf("%s",s);
            break;
        }
        p++;
    }
    printf("$ ");
    */
}
int main(int argc, char* argv[], char* engv[])
{
    while(1)
    {
        print_info();//打印[用户名字@主机信息:当前目录$]
        
        //键盘获取[命令、命令参数]字符串
        char buff[128] = {0};
        fgets(buff,128,stdin);
        buff[strlen(buff)-1] = 0;//remove the \n
        //存放解析的命令参数
        char * myargv[10] = {0};
        char * cmd = get_cmd(buff,myargv);
        if(NULL == cmd)
        {
            continue;
        }
        else if(strcmp(cmd,"exit") == 0)
        {
            break;
        }
        else if(strcmp(cmd,"cd")==0)
        {
            //chdir()
            if(myargv[1] == NULL)
            {
                continue;
            }
            if(chdir(myargv[1]) == -1)//chdir-->unistd.h
            {
                perror("cd err"); //#include<errno.h>
            }
            continue;//redo the bash
        }
        else
        {
            //fork + exec
            pid_t pid = fork();
            if(pid == -1)
            {
                printf("error\n");
                continue;
            }
            if(pid == 0)
            {
                execvp(cmd,myargv);
                printf("execvp error\n");
                exit(1);
            }
            wait(NULL);
        }
    }
}

```

# pwd和ls

pwd可以输出当前目录的路径信息。

用unistd.h下的getcwd函数获取到buff字符串中即可。

```c
#include<unistd.h>
int main()
{
    char buff[128] = {0};
    if(getcwd(buff,128)==NULL)
    {
        return;
    }
    printf("%s\n",buff);
    exit(0);
}
```

ls是打印当前目录下的所有文件名。

即需要扫描目录。

## 目录流

目录实际上是一个抽象的概念，因为实际上并没有实际的文件来存储目录，而是一个目录流。就类似于“219班”这个概念，实际上并没有219班这个人，只是抽象出来的一个可以代表一些文件的集合名词。

与目录操作有关的函数在dirent.h头文件中声明，使用一个名为DIR的结构作为目录操作的基础。指向这个结构体的指针（DIR \*）被称为目录流，用于完成各种目录的操作。与操作普通文件的文件流（FILE \*）非常相似。

涉及到的函数有：

1. opendir
   1. ![image-20211031173750225](../../images/Linux_%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/image-20211031173750225.png)
2. readdir
   1. ![image-20211031173910841](../../images/Linux_%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/image-20211031173910841.png)
   2. 返回的指针的指向的结构体是dirent。包含的内容有：
      ![image-20211031174007909](../../images/Linux_%E7%B3%BB%E7%BB%9F%E8%B0%83%E7%94%A8/image-20211031174007909.png)
3. telldir
4. seekdir
5. closedir

```c
#include<unistd.h>
#include<string.h>
#include<dirent.h>
int main()
{
    char buff[128] = {0};
    if(getcwd(buff,128))==NULL)
    {
        return 0;
    }
    DIR * p = opendir(buff);//打开通过getcwd获取的当前目录字符串所指定的目录，获取一个目录流
    if(p==NULL)
    {
        return 0;
    }
    struct dirent * p_dirent = NULL;
    while((p_dirent = readdir(p))!=NULL)
    {
        printf("%s  ",p_dirent->d_name);
    }
    closedir(p);
    return 0;
}
```

```bash

```

