---
title: tRPC-Cpp开发实录_LRUCache支持
categories:
  - - 项目
tags: 
date: 2025/8/11
updated: 2025/8/11
comments: 
published: false
---
# 需求描述
- 背景：LRU Cache （Least Recently Used，最近最少使用）是业务日常开发中常用的一个 Local Cache 工具，提供 Set/Get 等接口操作 Cache 数据。当缓存容量满了之后，LRU 算法根据数据的历史访问记录顺序来进行数据淘汰，核心思想是”如果数据最近被访问过，那么将来被访问的概率也更高“。业务期望框架能集成一个LRU Cache工具，方便日常开发。
- 需求：希望你能参考当前已有类库的接口，为tRPC-Cpp设计一套灵活、易用的接口，更好地扩展tRPC的生态。具体来来说，在实现时，你需要注意下面的要点：
    - 增加LRU Cache特性支持，**涵盖服务端与客户端**，包括协议、编解码、代理等实现与测试。需要支持如下特性：
        - 提供多线程安全的接口。
        - 支持高性能的 Put/Get 操作。
        - 支持LRU淘汰策略。
        - 支持自定义的Key和Value。
        - 支持指定线程Mutex和FiberMutex （框架提供自适应的同步原语）。
    - 你应该尽可能复用tRPC-Cpp的各个模块，具体来说，你应该继承 ServiceProxy，实现一个比如HttpSseProxy，提供设计良好的接口，在实现接口时，应该复用框架的服务发现以及拦截器模块。
- 参考资料：
    - LevelDB，git：[https://github.com/google/leveldb](https://github.com/google/leveldb)
    - RocksDB，git：[https://github.com/facebook/rocksdb](https://github.com/facebook/rocksdb)
    - CacheLib，git：[https://github.com/facebook/CacheLib](https://github.com/facebook/CacheLib)

指导原则（20250805会议记录）：
1. 要处理锁的问题、高并发、多线程安全相关的东西
2. 性能要求高，可能涉及到无锁的结构，涉及到分片锁
3. 提供多线程安全的接口，因为有可能多个请求同时去处理访问Cache
    1. 增删改查，比如提供增加、查询的操作，一些常见的KV结构，需要做热点处理的时候，比如put，需要先查一遍，查到了直接返回，没查到，需要从db或者其他的存储服务里把这个KV拿出来。
4. Cache的淘汰：细分为：随机淘汰、顺序淘汰？惰性淘汰、主动淘汰？实践中需要考虑。
5. 要制定一个KV。一般的LRU的实现会存储二进制的字节，当然我们这里可以简单一点，直接存对象，比如用模板的方式存数据、字符串、业务自定义的数据结构、对象。
    1. 实现上的问题是，做哈希的时候，要实现哈希的接口。可以让标准的容器或者第三方容器存起来。
    2. 多线程问题，有可能是线程环境，有可能是Fiber协程环境。所以如果有无锁的情况下，可能要把锁这个东西做一个模板。
6. 业内常用的LRU实现，LevelDB、RocksDB、CacheLib，它们的接口做得非常周全，可以基于它们的思路做一些简化。因为我们只是做内存，序列化我们其实不需要。

