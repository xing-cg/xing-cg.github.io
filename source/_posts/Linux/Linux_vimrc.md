---
title: Linux_vimrc
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

# vim安装

Linux下的安装。

The Vim repository at GitHub: Vim is available through git. This is the most straightforward way to get Vim and keep it up-to-date.

You can obtain Vim for the first time with:

```
git clone https://github.com/vim/vim.git
```

And, if you don't have local changes, update to the latest version with:

```
cd vim
git pull
```

If you made some changes, e.g. to a makefile, you can keep them and merge with the latest version with:

```
cd vim
git stash
git pull
git stash pop
```

If you have local changes you may need to merge. If you are sure you can discard local changes (e.g. if you were just trying a patch), you can use:

```
git fetch --all
git reset --hard origin/master
```

## Building Vim

Build and install Vim as usual. If you are happy with the defaults:

```
cd src
make distclean  # if you build Vim before
make
sudo make install
```

# vimrc在哪

vimrc究竟在哪儿呢？这个问题对一个vim的入门用户来说，可能实在不是个问题，你可能回答：”不就是`$HOME/.vimrc`吗？”。是的，这个答案正确，不过不是全部。

首先，在Linux下的Vim中输入`:version`命令(可能你是使用Linux下的vi命令打开编辑器，不过在大多数Linux中，vi命令打开的就是Vim)，我们略过不相关的内容，关于vimrc的显示如下(可能你的显示不完全和这里相同)：

```
# Linux下的vim
:version
VIM - Vi IMproved 8.0 (2016 Sep 12, compiled Jun 26 2017 11:44:34)
Included patches: 1-678

   system vimrc file: "$VIM/vimrc"
     user vimrc file: "$HOME/.vimrc"
 2nd user vimrc file: "~/.vim/vimrc"
      user exrc file: "$HOME/.exrc"
  system gvimrc file: "$VIM/gvimrc"
    user gvimrc file: "$HOME/.gvimrc"
2nd user gvimrc file: "~/.vim/gvimrc"
...
```

在上面，我们看到列出了几个vimrc文件，有一个系统的vimrc文件，还有用户的vimrc文件，以及系统和用户gvimrc文件。出于和vi兼容的目的，vim也支持vi的exrc配置文件。

接着，我们在Windows系统中输入`:version`命令，可以看到如下输出(使用的是预编译的Vim 8.0)：

```
# Windows下的vim
VIM - Vi IMproved 9.0 (2022 Jun 28, compiled Jun 28 2022 12:46:00)
MS-Windows 32-bit GUI version with OLE support
Compiled by mool@tororo

   system vimrc file: "$VIM\vimrc"
     user vimrc file: "$HOME\_vimrc"
 2nd user vimrc file: "$HOME\vimfiles\vimrc"
 3rd user vimrc file: "$VIM\_vimrc"
      user exrc file: "$HOME\_exrc"
  2nd user exrc file: "$VIM\_exrc"
  system gvimrc file: "$VIM\gvimrc"
    user gvimrc file: "$HOME\_gvimrc"
2nd user gvimrc file: "$HOME\vimfiles\gvimrc"
3rd user gvimrc file: "$VIM\_gvimrc"
       defaults file: "$VIMRUNTIME\defaults.vim"
    system menu file: "$VIMRUNTIME\menu.vim"
```

比较一下上面两个`:version`命令的输出，我们发现：

1. Vim启动时，会先尝试执行用户vimrc，并使用所找到的第一个用户vimrc中的配置，忽略其余的用户vimrc。其次才找系统的vimrc文件。
2. 在Linux下使用的vimrc文件名为`.vimrc`，在Linux下，如果未找到名为`.vimrc`的文件，也会尝试查找名为`_vimrc`文件；在Windows下也是这样，只不过查找顺序颠倒一下，如果未找到名为`_vimrc`的文件，会去查找`.vimrc`。
3. 在Windows下因为不支持以点`.`开头的文件名，vimrc文件的名字使用`_vimrc`。有两个可选的用户vimrc文件，一个是`$HOME\_vimrc`，另外一个是`$VIM\_vimrc`。
4. 从这里可以看出，vimrc的执行先于gvimrc。所以我们可以把全部vim配置命令都放在vimrc中，不需要用gvimrc。

对于vim初学者，如果不知道`$HOME`或者`$VIM`具体是哪个目录，可以在vim中用下面的命令查看：

```vim
:echo $VIM     # /usr/shared/vim
:echo $HOME    # /home/xcg
```

分别打印的是`/usr/share/vim`（vim目录），`/ssd_b/xcg`（家目录）。

## 选择vim配置

Windows版本的Vim在安装时，缺省会安装一个`$VIM/_vimrc`，你可以直接修改这个`_vimrc`，加入你自己的配置(使用`:e $VIM/_vimrc`即可打开此文件)。或者，你也可以在Windows中增加一个名为`HOME`的环境变量(`控制面板` -> `系统` –> `高级` –> `环境变量`)，然后把你的vimrc放在HOME环境变量所指定的目录中。从上面`:version`命令的输出看到，`$HOME/_vimrc`如果存在，就会执行这个文件中的配置，从而跳过`$VIM/_vimrc`。

1. 如果使用`vim -u filename`命令来启动Vim，则会用你指定的`filename`作为Vim的配置文件(在调试你的vimrc时有用)；
2. 如果用`vim -u NONE`命令启动Vim，则不读取任何vimrc文件：当你怀疑你的vimrc配置有问题时，可以用这种方式跳过vimrc的执行。

## 总结

Linux和Windows下vim的启动优先寻找`$HOME/.vimrc`（Windows下是`_vimrc`），即用户自己的配置。用户配置文件不存在时，使用的是系统的配置文件`$VIM/vimrc`。

# vimrc初步

所用命令的帮助入口：

```
:help compatible
:help mapleader
:help map
:help autocmd
```

vim的环境依赖会话(session)文件和viminfo文件，其中‘sessionoptions’选项和‘viminfo’选项的配置可能会根据需要进行调整。但如何保存所做的调整呢？需要vimrc文件。

当vim在启动时，如果没有找到vimrc或gvimrc，它缺省为VI兼容的模式。这意味着，只能使用VI所具备的功能，而vim中的大量扩展功能将无法使用。

vim中自带了一个vimrc例子，从这个例子开始分析。

下面以Linux下的vim为例。

示例的vimrc(名为`vimrc_example.vim`)通常位于`/usr/share/vim/vimXXX/`目录下，其中vimXXX与使用的vim版本有关。

首先把这个示例的vimrc拷贝到相应的目录，在Linux下，应该把它拷贝到你的home目录下，名字为”.vimrc”。

下面是shell命令：

```
cp /usr/share/vim/vim90/vimrc_example.vim ~/.vimrc 
```

或者你在vim中执行下面的命令，和上面的shell命令完成相同的功能：

```
:!cp $vimRUNTIME/vimrc_example.vim ~/.vimrc 
```

可以先读一下现在的vimrc，看看它都设定了什么：

```
:e ~/.vimrc 
```

这是一个注释良好的文件，不需要多解释。

> 对windows版本的vim，它已经默认的有了一个vimrc，可以在vim在使用下面的命令来查看它。
>
> ```
> :e $vim/_vimrc 
> ```
>
> 在这个文件中，它包含了上面提到的`vimrc_example.vim`。同时，它会把vim设置的更符合windows的操作习惯。比如，支持CTRL-C拷贝，CTRL-V粘贴等等。Windows下的用户，可以使用`$vim/_vimrc`来做为第一个vimrc。
>
> 顺便提一句，在unix/linux中，文件名可以以”.”开头，表明此文件是隐藏的。而在windows中，不允许文件名以”.”开头，所以，windows版本的vim，将读取`_vimrc`来做为自己的配置文件。

## 打开vimrc的快捷方式

在今后使用vim的日子里，会频繁的更改vimrc。所以需要设置一些快捷方式，使我们能快速的访问vimrc。

把下面这段内容拷贝到vimrc中：

```
1  "Set mapleader
2    let mapleader = ","
3
4  "Fast reloading of the .vimrc
5    map <silent> <leader>ss :source ~/.vimrc<cr>
6  "Fast editing of .vimrc
7    map <silent> <leader>ee :e ~/.vimrc<cr>
8  "When .vimrc is edited, reload it
9    autocmd! bufwritepost .vimrc source ~/.vimrc 
```

- 在vimrc中，双引号开头的行，将被当作注释忽略。
- 第2行，用来设置`mapleader`变量，当`mapleader`为未设置或为空时，使用缺省的”`\`”来作为mapleader。

mapleader变量是作用是什么呢？看下面的介绍。

- 第5行定义了一个映射(map)，这个映射把`<leader>ss`，映射为命令`:source ~/.vimrc<cr>`。当定义一个映射时，可以使用`<leader>`前缀。而在映射生效时，vim会把`<leader>`替换成mapleader变量的值。也就是说，我们这里定义的`<leader>ss`在使用时就变成了”`,ss`“，当输入这一快捷方式时，就会source一次`~/.vimrc`文件(也就是重新执行一遍`.vimrc`文件)。
- 第7行，定义了`<leader>ee`快捷键，当输入`,ee`时，会打开`~/.vimrc`进行编辑。
- 第9行，定义了一个自动命令，每次写入`.vimrc`后，都会执行这个自动命令，source一次`~/.vimrc`文件。

有了上面的快捷键，我们就能快速的打开vimrc文件编辑，快速重新source vimrc文件，方便多了。注意，`,ee`、`,ss`命令不是在冒号模式下输入，而是空悬模式下输入，在右下角可看到输入的命令。

## 使用同一个vimrc文件

为了实现无论在windows还是在linux中都能使用vim作为缺省编辑器。并且，想使用同一个vimrc文件。因此定义了一个`MySys()`函数，用来区分不同的平台，以进行不同的配置。

另外，在编辑vimrc文件时，最好新开一个标签页来编辑，而不是在当前窗口中。因此，定义了`SwitchToBuf()`函数，它在所有标签页的窗口中查找指定的文件名，如果找到这样一个窗口，就跳到此窗口中；否则，它新建一个标签页来打开vimrc文件。

> 注：标签页(tab)功能只有在vim 7.0版本以上才支持。

下面是示例：

```
" Platform
function! MySys()
  if has("win32")
    return "windows"
  else
    return "linux"
  endif
endfunction

function! SwitchToBuf(filename)
    "let fullfn = substitute(a:filename, "^\\~/", $HOME . "/", "")
    " find in current tab
    let bufwinnr = bufwinnr(a:filename)
    if bufwinnr != -1
        exec bufwinnr . "wincmd w"
        return
    else
        " find in each tab
        tabfirst
        let tab = 1
        while tab <= tabpagenr("$")
            let bufwinnr = bufwinnr(a:filename)
            if bufwinnr != -1
                exec "normal " . tab . "gt"
                exec bufwinnr . "wincmd w"
                return
            endif
            tabnext
            let tab = tab + 1
        endwhile
        " not exist, new tab
        exec "tabnew " . a:filename
    endif
endfunction

"Fast edit vimrc
if MySys() == 'linux'
    "Fast reloading of the .vimrc
    map <silent> <leader>ss :source ~/.vimrc<cr>
    "Fast editing of .vimrc
    map <silent> <leader>ee :call SwitchToBuf("~/.vimrc")<cr>
    "When .vimrc is edited, reload it
    autocmd! bufwritepost .vimrc source ~/.vimrc
elseif MySys() == 'windows'
    " Set helplang
    set helplang=cn
    "Fast reloading of the _vimrc
    map <silent> <leader>ss :source ~/_vimrc<cr>
    "Fast editing of _vimrc
    map <silent> <leader>ee :call SwitchToBuf("~/_vimrc")<cr>
    "When _vimrc is edited, reload it
    autocmd! bufwritepost _vimrc source ~/_vimrc
endif

" For windows version
if MySys() == 'windows'
    source $VIMRUNTIME/mswin.vim
    behave mswin
endif 
```

注意：在windows中也定义了一个”HOME”环境变量，然后把`_vimrc`放在”HOME”环境变量所指向的目录中。这样才能在windows中使用。

好了，现在我们知道如何永久更改`‘sessionoptions’`选项和`‘viminfo’`选项了，把对它们的配置放入vimrc即可。

vim自带的示例vimrc中，只定义最基本的配置。

在http://www.amix.dk/vim/vimrc.html有一个非常强大的vimrc，有人戏称为”史上最强的vimrc”，或许有些言过其实。不过，如果通读了这个vimrc，相信能从中学到很多。[这里](http://blog.csdn.net/redguardtoo/archive/2006/09/03/1172136.aspx)有一个redguardtoo修改过的版本，可以对照参考一下。但是不要照拷这两个vimrc，可能这个文件的设定并不符合你的习惯。另外，这个文件的设定，可能也不能在当前的工作环境中运行。
