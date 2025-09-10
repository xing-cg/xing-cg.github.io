---
title: git_开源协作工作流
categories:
  - - git
tags:
  - 
date: 2025/9/10
updated: 2025/9/10
comments:
published:
---
当在一个 GitHub 组织与他人协作开源项目时。

如果仓库 Owner 没有开放写权限。

就需要先 fork 一份到我们自己的仓库。

然后，在本地 git clone 我们自己的仓库。

当源仓库有更新时，我们的仓库落后了。

此时需要做的事情：
1. `git remote add upstream 原始仓库地址`：添加原始仓库为远程上游仓库
2. `git remote -v`：验证是否添加成功（应该能看到origin 和 upstream）

然后，从上游仓库获取最新更改：
```bash
git fetch upstream
```


此时，只是下载了源仓库的最新文件，但没有合并到我们本地仓库中。

确认我们目前处于我们自己的本地主分支。如果不是，需要切换：
```bash
git checkout main
```


然后，就可以合并上游仓库的更改了：
```bash
git merge upstream/main # 将上游仓库的main分支合并到你的当前分支
```


此时可能会遇到冲突，需要手动解决直到没有冲突后才能提交。

合并完成后，并且`git add -A`、`git commit`后，就可以将更新推送到我们的 fork 仓库了：

```bash
git push origin main # 现在将同步后的本地仓库推送到你的GitHub Fork (origin)
```

## 一步流

其实，以上的 git fetch 是分步骤去完成的。如果用 GitHub 比较熟练后，可以一条命令完成获取和合并：
```bash
git pull upstream main
```

