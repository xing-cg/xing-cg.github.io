---
title: MySQL_完整性约束
categories:
  - - 数据库
    - MySQL
tags: 
date: 2022/4/28
updated: 
comments: 
published:
---

# 内容

1. 主键约束
1. 自增键约束
1. 唯一键约束
1. 非空约束
1. 默认值约束
1. 外键约束

# 主键约束

`PRIMARY KEY`

只能有一个主键。

```mysql
create table user(
	id INT UNSIGNED PRIMARY KEY AUTO_INCREMENT comment '用户的主键id',
	username VARCHAR(50) UNIQUE NOT NULL COMMENT '用户名',
	age TINYINT UNSIGNED NOT NULL default 18,
	sex ENUM('male','demale')
);
# commit + ’...‘ 可以对本属性进行备注注释说明
```

创建表后，可以通过`desc user\G`，或者`desc user;`来查看字段的详细信息。前者是展开列表形式，后者是二维表格形式。

# 自增键约束

`AUTO_INCREMENT`

# 唯一键约束

`UNIQUE`

可以有多个唯一键，而且可以为空。

# 非空约束

`NOT NULL`

# 默认值约束

`DEFAULT`

# 外键约束

`FOREIGN KEY`

拿学生信息表、考试信息表来举例子。两张表中的信息是有关联的，不是独立的。

学生信息表中有张三，那么考试信息表里就可能有依赖张三的信息。

如果我们要删除张三，那么需要把另一个表中依赖张三的信息也删掉。否则就会出现无效冗余数据。

外键、存储函数、存储过程、触发器，在现如今的后台开发中几乎不直接用在mysql端了。

而是尽量把这个过程放在业务逻辑上提前处理好，让数据库持久层专于做增删改查，有利于提高效率。
