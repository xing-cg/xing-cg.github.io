---
title: MySQL数据库连接池
categories:
  - - 项目
    - 连接池
  - - 数据库
tags: 
date: 2022/6/14
updated: 
comments: 
published:
---

# 内容

C++实现的小型项目, MySQL数据库连接池, 属于常用组件;

约400到500行代码, 代码不多, 麻雀虽小, 五脏俱全: 

1. 涉及到数据库编程;
2. 单例模式
3. queue队列容器
4. `C++11`多线程编程, 线程互斥, 线程同步通信, `unique_lock`
5. 基于CAS的原子类型
6. 智能指针`shared_ptr`
7. lambda表达式
8. 生产者-消费者线程模型

# 项目背景

为了提高MySQL数据库（需要特别注意的, 它是基于C/S设计的）的访问瓶颈，除了在服务器端增加缓存服务器缓存常用的数据之外（例如Redis），还可以增加连接池，来提高MySQL Server的访问效率，在高并发情况下，大量的TCP三次握手、MySQL Server连接认证、MySQL Server关闭连接回收资源和TCP四次挥手所耗费的性能时间也是很明显的，增加连接池就是为了减少这一部分的性能损耗。

在市场上比较流行的连接池包括阿里的druid，c3p0以及apache dbcp连接池，它们对于短时间内大量的数据库增删改查操作性能的提升是很明显的，但是它们全部由Java实现。那么本项目就是为了在`C/C++`语言下，提供MySQL Server的访问效率，实现基于`C++`代码的数据库连接池模块。

# 连接池功能介绍

该项目是基于C++语言实现的连接池，主要实现若干个通用基础功能。

连接池一般包含了数据库连接所用的ip地址、port端口号、用户名和密码以及其它的性能参数，例如初
始连接量，最大连接量，最大空闲时间、连接超时时间等;

1. **初始连接量(initSize)**
   1. 表示连接池事先会和MySQL Server创建initSize个数的connection连接;
   2. 当应用发起MySQL访问时，不用再创建和MySQL Server新的连接，直接从连接池中获取一个可用的连接就可以;
   3. 使用完成后，并不去释放connection，而是把当前connection再归还到连接池当中。
2. **最大连接量(maxSize)**
   1. 总的连接数量上限是maxSize;
   2. 当并发访问MySQL Server的请求增多时，初始连接量已经不够使用了，此时会根据新的请求数量去创建更多的连接给应用去使用;
   3. 但是，不能无限制的创建连接，因为每一个连接都会占用一个socket资源，一般连接池和服务器程序是部署在一台主机上的，如果连接池占用过多的socket资源，那么服务器就不能接收太多的客户端请求了;
   4. 当这些连接使用完成后，再次归还到连接池当中来维护。
3. **最大空闲时间(maxIdleTime)**
   1. 当访问MySQL的并发请求多了以后，连接池里面的连接数量会动态增加，上限是maxSize个;
   2. 当这些连接用完再次归还到连接池当中;
   3. 如果在指定的maxIdleTime里面，这些新增加的连接都没有被再次使用过，那么新增加的这些连接资源就要被回收掉，只需要保持初始连接量initSize个连接就可以了。
4. **连接超时时间(connectionTimeout)**
   1. 当MySQL的并发请求量过大，连接池中的连接数量已经到达maxSize了，而此时没有空闲的连接可供使用，那么此时应用从连接池获取连接需要通过阻塞的方式获取连接;
   2. 阻塞时间如果超过connectionTimeout时间，那么获取连接失败，无法访问数据库;

该项目主要实现上述的连接池四大功能，其余连接池更多的扩展功能，可以在该框架的基础上进行很好的拓展;

# 设计思路

1. 需要抽象出两个类
   1. ConnectionPool - 连接池
   2. Connection - 封装数据库操作, 增删改查
2. 连接池只需要一个实例, 则ConnectionPool设计为单例模式
3. ConnectionPool需要提供获取MySQL连接Connection的接口
4. 空闲连接Connection全部维护在一个Connection队列中, 使用互斥锁保证队列的线程安全
5. 如果在获取连接时发现空闲连接队列空, 则需要动态创建新连接, 但是上限数量是maxSize, 在创建新连接的过程中, 消费者线程需要**忙等待**connectionTimeout时长, 直到队列中不空; 若时长一到仍获取未果则认定为连接失败; 可以使用带超时时间的mutex互斥锁来实现连接超时时间;
6. 使用量下降后, 如果队列中的空闲连接保持空闲超过maxIdleTime, 需要释放连接, 直到减少到initSize个连接; 这个释放的过程需要放在独立线程做 - **定时线程**, 清理队列多余的空闲连接;
7. 用户获取的连接用`shared_ptr`智能指针来管理，用lambda表达式定制连接释放的功能（不真正释放连接，而是把连接归还到连接池中）
8. 连接的生产和连接的消费采用`生产者-消费者`线程模型来设计，使用了线程间的同步通信机制条件变量和互斥锁;

# MySQL数据库编程

## 公共代码

1. 简易日志工具 - 向标准输出设备输出
   ```cpp
   #include<iostream>
   #define LOG(str)                                            \
       do {                                                    \
       std::cout << __FILE__ << ":" << __LINE__ << ":"         \
                 << __TIMESTAMP__ << ":" << str << std::endl;  \
       } while(0)
   ```

## MySQLConnection - 封装MySQL数据库操作

### 成员变量

`MYSQL *m_conn` - 记录MYSQL类型的指针, 以获取这个mysql连接

```cpp
private:
    MYSQL *m_conn;
```

### 成员函数

1. 构造/析构 - 初始化/释放数据库连接

   ```cpp
   public:
       /* 初始化数据库连接 */
       MySQLConnection();
       /* 释放数据库连接资源 */
       ~MySQLConnection();
   ```

2. getConnection - 获取连接, 即获取成员`m_conn`

   ```cpp
   public:
       /* 获取连接 */
       MYSQL * getConnection();
   ```

3. connect - 连接数据库, 返回值为bool, 说明连接成功与否

   ```cpp
   public:
       /* 连接数据库 */
       bool connect(std::string ip, unsigned short port,
                    std::string user, std::string password, std::string dbname);
   ```

4. query - 查询操作, 参数是string类型的sql语句, 返回值为`MYSQL_RES`, 即MySQL结果集类型

   ```cpp
   public:
       /* 查询操作 */
       MYSQL_RES * query(std::string sql);
   ```

5. update - 更新操作, 参数是string类型的sql语句, 返回值为bool, 说明更新成功与否

   ```cpp
   public:
       /* 更新操作 */
       bool update(std::string sql);
   ```

### 代码实现

1. 构造 - 调用`mysql_init`, 实际上只是对mysql连接进行空间资源的开辟, 返回一个指针赋给`m_conn`成员, 没有真正连接, 因此传入nullptr

   ```cpp
   /* 初始化数据库连接 */
   MySQLConnection::MySQLConnection()
   {
       m_conn = mysql_init(nullptr);
   }
   ```

2. 析构 - 调用`mysql_close(m_conn)`, 对MySQL连接资源进行释放

   ```cpp
   /* 释放数据库连接资源 */
   MySQLConnection::~MySQLConnection()
   {
       if(m_conn != nullptr)
       {
           mysql_close(m_conn);
       }
   }
   ```

3. connect - 连接数据库, 内部调用`mysql_real_connect`, 传入`m_conn`, 以及server ip地址, user号, 密码, 要连接的数据库name, 服务器端口;

   ```cpp
   /* 连接数据库 */
   bool MySQLConnection::connect(std::string ip, unsigned short port,
                                 std::string user, std::string password,
                                 std::string dbname)
   {
       MYSQL *p = mysql_real_connect(m_conn, ip.c_str(), user.c_str(),
                                     password.c_str(), dbname.c_str(),
                                     port, nullptr, 0);
       if(p != nullptr)
       {
           /**
            * C/C++代码默认的编码字符是ASCII, 
            * 如果不设置, 则从MySQL上拉下来的中文无法正常显示
            */
           mysql_query(m_conn, "set name gbk");
           LOG("connect mysql success!");
       }
       else
       {
           LOG("connect mysql failed!");
       }
       return p != nullptr;
   }
   ```

4. query - 查询操作

   1. 内部调用`mysql_query`, 传入`m_conn`, `sql-string`的c风格字符串首址;

      > `mysql_query`的返回值: 
      >
      > 1. 如果查询成功，返回0;
      > 2. 如果出现错误，返回非0值。

   2. 返回值需要调用`mysql_use_result(m_conn)`获取结果集, 再return;

   ```cpp
   /* 查询操作 */
   MYSQL_RES * MySQLConnection::query(std::string sql)
   {
       if(mysql_query(m_conn, sql.c_str()) == 0)
       {
           LOG("select failed: " + sql);
           return nullptr;
       }
       return mysql_use_result(m_conn);
   }
   ```

5. update - 更新操作

   1. 内部调用`mysql_query`, 传入`m_conn`, `sql-string`的c风格字符串首址;
   2. 判断`mysql_query`的返回值, 若为非0则更新失败; 若为0则更新成功;

   ```cpp
   /* 更新操作 */
   bool MySQLConnection::update(std::string sql)
   {
       if(mysql_query(m_conn, sql.c_str()) == 0)
       {
           LOG("update failed: " + sql);
           return false;
       }
       return true;
   }
   ```

# 连接池

## MySQLConnectionPool

### 成员变量

### 成员函数

1. getConnection
   ```cpp
   /* 获取连接 */
   MYSQL * MySQLConnection::getConnection()
   {
       return m_conn;
   }
   ```

   

### 代码实现

# 压力测试

建个表

```mysql
CREATE TABLE user(
    id INT UNSIGNED PRIMARY KEY NOT NULL AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    age TINYINT NOT NULL,
    sex ENUM('male', 'female') NOT NULL
)ENGINE=INNODB DEFAULT CHARSET=utf8;
```

