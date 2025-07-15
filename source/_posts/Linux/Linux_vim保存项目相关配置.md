---
title: Linux_vim保存项目相关配置
categories:
  - - Linux
    - vim
tags: 
date: 2022/8/27
updated: 
comments: 
published:
---

本节所用命令的帮助入口：

```
:help 'path'
:help mksession
:help find
:help gf
:help CTRL-W_f
```

# 引入：path

vimrc定义了vim通常的行为。然而，每个项目都有其特殊的定义，虽然我们也可以在vimrc中对每个项目进行定制，但这样一来，vimrc会变得很大，使vim启动速度变慢；另外也会使vimrc变得难以维护。

因此，使用其它的方法来保存项目相关的信息，将以*path*选项的设置为例进行。

path选项定义了一个目录列表，在使用`gf`，`find`，以及`CTRL-W f`等vim命令时，如果使用的是相对路径，那么就会在path选项定义的目录列表中查找相应的文件。path选项以逗号分隔各目录名。我们以vim的源代码为例（源代码放在`~/src/vimXX/`目录中）。

对于这个项目，我们的*path*选项设置如下：

```
set path=.,/usr/include,,~/src/vim90/**
```

稍微解释一下各项的含义，更详细的信息，请查看*path*选项的帮助页：

- `.`：在当前文件所在目录中搜索
- `/usr/include`：在`/usr/include`目录中搜索
- `,,`：在当前工作路径中搜索
- `~/src/vim90/**`：在`~/src/vim90`的所有子目录树中进行搜索

设置了*path*选项后，怎么用呢？

我们把光标定位到`src/main.c`文件第22行的`fcntl.h`单词上，然后在`Normal`模式下按`gf`。咦，`vim`打开了`/usr/include/fcntl.h`文件！

现在我们按`CTRL-^`回到刚才的位置，光标仍旧定位在第22行的`fcntl.h`单词上，然后按`CTRL-W f`。啊哈，这次vim打开了一个水平分隔窗口，在此窗口中打开了`/usr/include/fcntl.h`。

尽管在`src/main.c`中未指定`fcntl.h`的路径，但vim会在*path*选项定义的路径中搜索此文件，很方便。

现在我们看一下`find`命令，输入：

```
:find netrw.vim
```

vim打开了`~/src/vim90/runtime/autoload/netrw.vim`文件。用这种方法打开文件真是太方便了，你不用输入文件的路径，vim会自动在*path*选项定义的路径中搜索。不过`find`命令也有缺陷，如果你只记得文件名的一部分，那么就没有办法用find命令打开这个文件了。而且find命令也不允许使用正则表达式。没关系，我们还有更好的方法来打开文件，要用到Lookupfile插件。

*path*选项介绍完了，进入正题，如何把本项目相关的配置保存起来，下次打开本项目时自动恢复这些配置呢？

我们有两种方法做到这一点。

## 项目相关的配置保存起来：方法1

我们在`~/src/vim90/`目录下建立一个文件，假定文件名为`workspace.vim`，文件内容为：

```
"set project path
set path+=~/src/vim70/** 
```

这个文件中保存了项目相关的信息，例如选项值，键映射，函数定义，自动命令，等等。我们的例子中只定义的path选项，我们没有使用`set path=…`语句，在vim手册中建议使用`set path+=…`和`set path-=…`格式。

接下来，在你的vimrc文件中加入下面的语句：

```
" execute project related configuration in current directory
if filereadable("workspace.vim")
    source workspace.vim
endif
```

以后，每次在`~/src/vim90/`目录中启动vim时，vim都会自动载入`workspace.vim`，恢复项目的配置信息。

## 方法2

回顾会话(session)和viminfo的作用。使用session文件和viminfo也可以保存项目环境的方法。如果你使用了会话文件，那么选项值，键映射，和其它很多信息都已经保存了。但会话的功能毕竟有限，不能把项目相关的配置全部保存下来，怎么办呢？

vim的作者已经想到了这个问题，并提供了解决办法。

在vim载入会话文件的最后一步，它会查找一个额外的文件并执行其中的ex命令。查找的规则是，把会话文件名的后缀去掉，然后在后面加上`x.vim`。假设你的会话文件名为`example.session`，vim就会查找是否有`examplex.vim`，如果找到，就会执行此文件中的ex命令。

好了，我们先创建我们的会话文件：

```
:cd ~/src/vim70
:set sessionoptions-=curdir        '在session option中去掉curdir
:set sessionoptions+=sesdir        '在session option中加入sesdir
:mksession vim70s.vim              '创建一个会话文件 
```

然后再编辑一个名为~/src/vim70/vim70sx.vim的文件，文件的内容为（当然，你可以在这个文件中加入更多内容）：

```
"set project path
set path+=~/src/vim70/** 
```

退出vim后，在命令行下执行”gvim &”，再次进入vim，这时看到的是一个空白窗口。然后执行下面的命令：

```
:source ~/src/vim70/vim70s.vim  '载入会话文件 
```

太棒了！原来的会话环境已经恢复，并且项目相关的配置也设置好了！
