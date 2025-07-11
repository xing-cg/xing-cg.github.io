---
title: Linux_vim_会话和viminfo
typora-root-url: ../..
categories:
  - [Linux, vim]
tags:
  - null 
date: 2022/8/27
update:
comments:
published:
---

所用命令涉及到的帮助入口：

```
:help mksession
:help 'sessionoptions'
:help source
:help wviminfo
:help rviminfo
:help 'viminfo'
```

很多软件都具有这样一种功能：在你下一次启动该软件时，它会自动为你恢复到你上次退出的环境，恢复窗口布局、所打开的文件，甚至是上次的设置。

vim有没有这种功能呢？答案当然是肯定的！这需要使用vim的会话(session)及viminfo的保存和恢复功能。

使用会话(session)和viminfo，可以把编辑环境保存下来，然后在下次启动vim后，可以再恢复回这个环境。在开发项目或书写文档时，如果在中途退出了vim而不能恢复原先的编辑环境的话，又要重新打开你所打开的文件，重新定义你的映射(map)、缩写(abbreviate)，重新定位你所设定的标记的位置(mark)，重新设置项目相关设置(options)等等，不是一般的麻烦！

要恢复上次的编辑环境，我们需要保存两种不同的信息，一种是会话(session)信息，另外一种是viminfo信息。

- 会话信息中保存了所有窗口的视图，外加全局设置。
- viminfo信息中保存了命令行历史(history)、搜索字符串历史(search)、输入行历史、非空的寄存器内容(register)、文件的位置标记(mark)、最近搜索/替换的模式、缓冲区列表、全局变量等信息。

# 会话

我们可以使用`:mksession [file]`命令来创建一个会话文件，如果省略文件名的话，会自动创建一个名为`Session.vim`的会话文件。

会话文件，其本质上是一个**vim脚本**，你可以使用上述命令生成一个session文件，然后再查看其中的内容，就会对session文件有一个深入的认识。

会话文件中保存哪些信息，是由`sessionoptions`选项决定的。缺省的`sessionoptions`选项包括：`blank`, `buffers`, `curdir`, `folds`, `help`, `options`, `tabpages`, `winsize`，也就是会话文件会恢复当前编辑环境的空窗口、所有的缓冲区、当前目录、折叠(fold)相关的信息、帮助窗口、所有的选项和映射、所有的标签页(tab)、窗口大小。

如果你使用windows上的vim，并且希望你的会话文件可以同时被windows版本的vim和UNIX版本的vim共同使用的话，在`sessionoptions`中加入`slash`和`unix`，前者把文件名中的`\`替换为`/`，后者会把会话文件的换行符保存成unix格式。

如果你不希望在`session`文件中保存当前路径，而是希望`session`文件所在的目录自动成为当前工作目录，那么，需要在`sessionoptions`去掉`curdir`，加入`sesdir`，这样每次载入`session`文件时，此文件所在的目录就被设为vim的当前工作目录。在你通过网络访问其它项目的session文件时，或者你的项目有多个不同版本（位于不同的目录），而你想始终使用一个session文件时，这个选项比较有用：你只需要把session文件拷贝到不同的目录，然后使用就可以了。设置此选项后，session文件中保存的是文件的相对路径，而不是绝对路径。

我们在上面使用`:mksession`命令创建了会话文件，那么怎么使用会话文件恢复编辑环境呢？很简单，你只需要使用`:source session-file`来导入会话文件。因为会话文件是一个脚本，里面保存的是Ex命令，所以`source`命令只是把会话文件中的Ex命令执行一遍。

# viminfo

使用`:wviminfo [file]`命令，可以手动创建一个`viminfo`文件。

其实，在vim退出时，每次都会保存一个`.viminfo`文件在用户的主目录。我们使用`:wviminfo`命令则是手动创建一个`viminfo`文件，因为缺省的`.viminfo`文件会在每次退出vim时自动更新，谁知道你在关闭当前软件项目后，又使用vim做过些什么呢？这样的话，`.viminfo`中的信息，也许就与你所进行的软件项目无关了。还是手动保存一个保险。

`:wviminfo`命令保存哪些内容，以及保存的数量，由`viminfo`选项决定，这个选项的值在windows上和在linux上的缺省值不同，具体含义参阅手册。

要读入你所保存的`viminfo`文件，使用`:rviminfo [file]`命令。

现在，先看一下我们当前目录，执行`:pwd`，显示`/home/xcg/vimtest`，接下来，执行下面的命令：

```vim
:cd src                            "切换到/home/xcg/vimtest/src目录 需要先在外部创建该目录
:set sessionoptions-=curdir        "在session option中去掉curdir
:set sessionoptions+=sesdir        "在session option中加入sesdir
:mksession vim90.vim               "创建一个会话文件
:wviminfo vim90.viminfo            "创建一个viminfo文件
:qa                                "退出vim
```

退出vim后，在命令行下执行`vim`，再次进入vim，这时看到的是一个空白窗口。然后执行下面的命令：

```
:source ~/vimtest/src/vim90.vim    '载入会话文件
:rviminfo vim90.viminfo            '读入viminfo文件
```

太棒了，又恢复到昨天退出时的状态了！继续工作。不过，每次都要手工修改`sessionoptions`或`viminfo`吗？多麻烦啊。别着急，现在是时候介绍vimrc了。