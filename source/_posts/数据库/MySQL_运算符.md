---
title: MySQL_运算符
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

1. 算数运算符
1. 逻辑运算符
1. 比较运算符

# 算术运算符

| 运算符     | 作用           |
| ---------- | -------------- |
| `+`        | 加法           |
| `-`        | 减法           |
| `*`        | 乘法           |
| `/`、`DIV` | 除法，返回商   |
| `%`、`MOD` | 除法，返回余数 |

* 注意
  * 计算时，需要注意当时的数据类型取值范围是否合适，以免越界导致业务出错

## 示例

```mysql
update user set age=age+1;
```

# 逻辑运算符

| 运算符      | 作用   |
| ----------- | ------ |
| `NOT`或`!`  | 逻辑非 |
| `AND`或`&&` | 逻辑与 |
| `OR`或`||`  | 逻辑或 |

# 比较运算符

| 运算符     | 作用                             |
| ---------- | -------------------------------- |
| `=`        | 等于                             |
| `<>`或`!=` | 不等于（`<>`在未来可能会被淘汰） |
| `<=>`      | NULL安全的等于(NULL-safe)        |
| <          | 小于                             |
| <=         | 小于等于                         |
| >          | 大于                             |
| >=         | 大于等于                         |

| 运算符            | 作用           |
| ----------------- | -------------- |
| `BETWEEN`         | 存在于指定范围 |
| `IN`              | 存在于指定集合 |
| `IS NULL`         | 为NULL         |
| `IS NOT NULL`     | 不为NULL       |
| `LIKE`            | 通配符匹配     |
| `REGEXP`或`RLIKE` | 正则表达式匹配 |

* 注意
  * 判空不要写`=NULL`，而是要写`IS NOT NULL`或`IS NULL`；判空经常出现在左连接、右连接中的外键查询。
  * 通配符与索引的关系？
    * `LIKE`通配符不一定会用到索引，需要看通配符加的地方，如果通配符在中间、末尾则能用到；但是如果加到最开始则不会用到。

## 示例

```mysql
select * from user where age between 20 and 22;
select * from user where score in (99.0, 100.0);
```

```mysql
select * from user where score IS NOT NULL;
```

```mysql
select * from user where name like 'zhang%';	#zhang开头的任何字符串
select * from user where name like 'zhang_';	#下划线是占位通配符，只能匹配后面只有一个字符的
```





# 综合示例

下面这个查询语句用到了多个运算符：`sex='M'`、`and`、`score>=90.0`

```mysql
select * from where sex='M' and score>=90.0;
```

