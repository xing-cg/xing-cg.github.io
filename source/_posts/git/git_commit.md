---
title: git_commit
categories:
  - - Linux
  - - git
tags: 
date: 2022/9/2
update: 
comments: 
published:
---

# 用vim编辑commit

1. 在git config中设置`core.editor`:
   `git config --global core.editor "vim"`
2. 在环境变量中设置`GIT_EDITOR`:
   `export GIT_EDITOR=vim`

# commit规约目标

1. 规范、统一的Commit
   1. 更加结构化的提交历史
   2. 保证每次信息都有确切的含义
2. 自动、清晰的ChangeLog
3. 简单、方便的issue tracker
   1. 方便信息搜索和过滤

# Commit规范

代码示意

```
[what][key][jira issue ID] describe[part1/n]
[why] describe
[how]
1.describe
2.
n.
```

段落含义解释

| 段落 | 含义                                                       | 备注                       |
| :--- | :--------------------------------------------------------- | :------------------------- |
| what | 描述提交（问题、改动、修改等调整）                         | 一句话结束。不超过80个字节 |
| why  | 阐述提交（问题触发原理、改动缘由、修改理由等一切合理解释） |                            |
| how  | 解释提交（问题解决办法、改动项、修改项等）                 | 分点分段                   |

字段含义解释

| 字段          | 含义                                     | 备注                                            |
| :------------ | :--------------------------------------- | :---------------------------------------------- |
| key           | 关键字，用于直接区分commit本质，极其重要 | 目的：过滤，筛选用途：可用于自动化生产ChangeLog |
| Jira issue ID | jira问题ID，ARMR-666                     |                                                 |
| describe      | 描述                                     |                                                 |
| part 1/n      | 跨仓库提交                               | 一般用于repo等大型仓库使用                      |

key字段解释

| key      | 解释     | 备注                                   |
| :------- | :------- | :------------------------------------- |
| feature  | 新特性   |                                        |
| merge    | 合并     |                                        |
| bugfix   | 修复问题 |                                        |
| config   | 配置机型 | 在TV里面，config是提交最多的一个种类型 |
| refactor | 重构     |                                        |

## 针对 bug 修改

```
[bugfix][类型][jira 编号] bug 简述
（添加一行空行）
[what]bug 详细描述
[why]原因
[how]解决方法
```

## 针对功能添加的修改

```
[feature][类型][可选项] 功能简述
（添加一行空行）
[what]功能的详细描述
实现方法
```

[可选项]——可考虑填写版本号（如果有的话）或者 jira 编号

## 公共约定

1. 第一行简述尽量用英文，可以使用中文
2. bug 有 jira 编号，必须附上
3. 第一行简述下必须空一行
4. 审核人发现log不按照本文规范书写，需要打回
