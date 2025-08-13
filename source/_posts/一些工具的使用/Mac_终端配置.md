---
title: Mac_终端配置
categories:
  - - 一些工具的使用
tags: 
date: 2025/8/14
updated: 2025/8/14
comments: 
published:
---
# 选择
Mac的终端有很多选择。
推荐得比较多的：
1. iTerm2，是个老牌macOS终端，精巧。并且更新频繁。缺点是不跨平台。
2. warp。rust写的。GPU渲染，看起来很现代。诟病比较多的是有时太占内存，甚至有时能吃2G。但是集成的无缝AI交互体验很好。
3. Termius。是专门给多端使用的，主要用于管理SSH。用了用，感觉不太习惯。
# 终端配置
warp默认使用的是zsh配置文件。在`/Users/mrcan/.zshrc`。
但zsh配置文件在macOS默认是没有的，需要手动创建。

```sh
# Warp Terminal & Zsh Configuration
# Colors for different file types in ls command

# Enable colors for ls on macOS
export CLICOLOR=1

# macOS BSD ls colors (LSCOLORS format)
# Format: directory:symlink:socket:pipe:executable:block:character:setuid:setgid:dir_w_sticky:dir_wo_sticky
export LSCOLORS=ExGxBxDxCxEgEdxbxgxcxd

# If you have GNU coreutils installed (brew install coreutils), uncomment these:
# alias ls='gls --color=auto'
# export LS_COLORS='di=1;34:ln=1;36:so=1;35:pi=1;33:ex=1;32:bd=1;34:cd=1;34:su=0;41:sg=0;46:tw=0;42:ow=0;43:mi=1;37;41:*.tar=1;31:*.tgz=1;31:*.zip=1;31:*.gz=1;31:*.bz2=1;31:*.xz=1;31:*.jpg=1;35:*.jpeg=1;35:*.png=1;35:*.gif=1;35:*.mp3=1;33:*.mp4=1;33:*.avi=1;33:*.mov=1;33:*.pdf=1;31:*.doc=1;31:*.txt=0;37:*.md=1;37:*.json=1;33:*.xml=1;33:*.yml=1;33:*.yaml=1;33:*.js=1;32:*.ts=1;32:*.py=1;32:*.go=1;32:*.rs=1;32:*.cpp=1;32:*.c=1;32:*.h=1;32'

# Useful aliases for colorful output
alias ll='ls -la'
alias la='ls -A'
alias l='ls -CF'

# Enable color support for grep
alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'

# Git colors (if you use git)
alias git='git --no-pager'

# Warp works great with these modern alternatives (install via brew if desired):
# brew install exa && alias ls='exa --color=always --icons'
# brew install lsd && alias ls='lsd --color=always'
# brew install bat && alias cat='bat --style=auto'
```
以上配置文件的效果：
What's now working:
•  Colored directories - Blue folders
•  Colored file types - Different colors for executables, links, etc.
•  Useful aliases - ll, la, l for different listing views
•  Colored grep - Search results will be highlighted

写入配置文件后，还需要`source ~/.zshrc`使这个配置文件立即生效。

# macOS使用brew安装开发工具，以安装git为例
通常在macOS上有两种常用的安装方式：brew install或者xcode-select --install
xcode-select and brew, which is better? what are the difference of them
## 两者的对比
Here are the key differences between installing Git via `xcode-select` and Homebrew:

### Xcode Command Line Tools (`xcode-select --install`)

**Pros:**
- **Apple-maintained**: Official Apple distribution, well-integrated with macOS
- **Comes with essential dev tools**: Includes GCC, make, and other development utilities
- **System-level installation**: Installed in `/usr/bin/git`
- **No additional dependencies**: Built into macOS
- **Required for many dev tasks anyway**: Needed for compiling software from source

**Cons:**
- **Slower updates**: Git version updates only come with macOS/Xcode updates
- **Older versions**: Often ships with older Git versions
- **Less control**: Can't easily switch versions or customize installation

### Homebrew (`brew install git`)

**Pros:**
- **Latest versions**: Usually has the most recent Git releases
- **Easy updates**: `brew upgrade git` keeps you current
- **Version management**: Can install multiple versions if needed
- **Package ecosystem**: Access to thousands of other developer tools
- **Customizable**: Various installation options and configurations

**Cons:**
- **Additional dependency**: Requires Homebrew to be installed first
- **PATH management**: Need to ensure Homebrew's path comes before system paths
- **User-space installation**: Installed in `/opt/homebrew/bin/git` (Apple Silicon) or `/usr/local/bin/git` (Intel)

### Which is Better?

**For most developers, Homebrew is better** because:
1. You get newer Git features and security fixes faster
2. You'll likely want Homebrew for other development tools anyway
3. Easy to keep everything updated

**Use Xcode Command Line Tools if:**
- You prefer minimal system modifications
- You don't need the latest Git features
- You're doing iOS/Mac development (you'll need Xcode tools anyway)

**Recommendation**: Install Xcode Command Line Tools first (many things depend on it), then install Git via Homebrew for the best of both worlds.
## brew的安装
默认macOS是没有brew的。需要先安装。而安装brew还需要安装xcode command line tools（在安装brew时如果检测到没有这个，会自动帮你装）。
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

安装完之后，它会提示你，加入PATH：
```bash
# Run these commands in your terminal to add Homebrew to your PATH:
echo >> /Users/mrcan/.zprofile
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/mrcan/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```
之后，就可以`brew help`测试了。

>但是，这里有个坑，就是第二句，实际最后`.zprofile`文件中保存成了`eval \"$(/opt/homebrew/bin/brew shellenv)\"`。
>导致下一次打开终端报错：

```
(eval):1: no such file or directory: export HOMEBREW_PREFIX=/opt/homebrew; export HOMEBREW_CELLAR=/opt/homebrew/Cellar; export HOMEBREW_REPOSITORY=/opt/homebrew; fpath[1,0]=/opt/homebrew/share/zsh/site-functions; PATH=/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/System/Cryptexes/App/usr/bin:/usr/bin:/bin:/usr/sbin:/sbin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/local/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/bin:/var/run/com.apple.security.cryptexd/codex.system/bootstrap/usr/appleinternal/bin; export PATH; [ -z  ] || export MANPATH=:; export INFOPATH=/opt/homebrew/share/info:;
```

>应该存成`eval "$(/opt/homebrew/bin/brew shellenv)"`才对。
>修改之后，`source ~/.zprofile`即可生效。
>The issue was in your .zprofile file where the quotes around the eval command were escaped (with backslashes), which caused the shell to interpret them incorrectly.
>This command properly sets up your Homebrew environment variables (like HOMEBREW_PREFIX, PATH, etc.) that you saw in the error message. The error occurred because the shell couldn't properly execute the command due to the escaped quotes.
>Your shell should now work properly without that error. You can restart your terminal or run source `~/.zprofile` to apply the changes.
## brew安装git
```bash
brew install git
git --version
```
# git配置
## 生成公钥
```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```
默认生成到了`~/.ssh/id_rsa`or`~/.ssh/id_ed25519`

实际测试，默认保存到了`/Users/mrcan/.ssh/id_ed25519`，这是私钥，公钥保存到了`/Users/mrcan/.ssh/id_ed25519.pub`

之后，把公钥上传到GitHub即可。
## 发现不能读取repo
```bash
ssh -T git@github.com
```
这是通用的检测方法。发现不行。
原因是我们没有把github.com添加到可信名单中。
```bash
ssh-keyscan -H github.com >> ~/.ssh/known_hosts
```
现在，再次
```bash
ssh -T git@github.com
```
回复：
```
Hi xing-cg! You've successfully authenticated, but GitHub does not provide shell access.
```
现在还需要在本地配置自己的user.email
```bash
git config --global user.email "myemail@xxx.com"
```
现在就可以正常git管理repo了。
# 安装npm
macOS默认没有npm。
可以用brew安装node.js。
```bash
brew install node
node --version
```
之后，在有`package.json`的目录中，就可以直接`npm install`安装依赖了。
## hexo博客依赖安装
hexo博客目录下只`npm install`还不够，还需要安装`hexo-cli`
```bash
npm install -g hexo-cli
```
# 远程连接云服务器
```
ssh-keyscan -H mrcan.work >> ~/.ssh/known_hosts
```
之后才能进行连接请求：
```
ssh root@mrcan.work
```