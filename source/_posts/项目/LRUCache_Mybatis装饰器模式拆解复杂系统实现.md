---
title: LRUCache_Mybatis装饰器模式拆解复杂系统实现
categories:
  - - 项目
tags: 
date: 2025/8/19
updated: 2025/8/19
comments: 
published: false
---
用 **接口 + 组合 + 装饰器** 拆解复杂度，让**策略（LRU）** 与**存储（HashMap/分片/锁）** 解耦。
# 1 定义统一接口（ICache）
```cpp
struct ICache
{
    virtual ~ICache() = default;
    virtual bool Put(const Key& k, Value v) = 0;
    virtual bool Get(const Key& k, Value* out) = 0;
    virtual bool Erase(const Key& k) = 0;
    virtual void Clear() = 0;
    virtual size_t Size() const = 0;
};
```
# 2 基础存储（PerpetualCache）

- 仅做 **unordered_map** 存取；不带淘汰。
- 可做**分片(shard)**：`ShardedMapCache` 来降低锁竞争（每片一把锁）。
- **线程/协程锁**：用`MutexTraits<Thread/Fiber>` 选择 `std::mutex` 或 `trpc::FiberMutex`。
# 3 LruCache 装饰器
- 内部只维护“**key 的 LRU 队列**”（双向链表 + 哈希定位），**不存 value**。
- `Put/Get/Erase` 都会先调 **delegate**（底层存储），再更新自己的 LRU-Key 结构；溢出时**只拿到 eldest key 并调用 `delegate->Erase(eldestKey)`**。
- 这样你可以把 **LRU 改成 FIFO/Random/Segmented-LRU**，无需改存储实现。
# 4 其它装饰器（按需叠加）

- **`SynchronizedCache<MutexTag>`**：最外层一把大锁（模板参数决定 `Mutex` 或 `FiberMutex`），快速满足“线程安全 API”。
- **`BlockingCache`**：miss 时对单 key 加锁，防止击穿（查不到要去 DB 拉，别并发打爆 DB的场景）。
- **`TtlCache / ScheduledCache`**：在装饰器里做过期判断/惰性清理。
- **`LoggingCache`**：打点、埋点、调试日志。
- **`SerializedCache`**：若需要隔离对象别名或跨线程安全，可在这里做（可选）。

组合示例（从内到外）：  
```cpp
auto cache = make_shared<SynchronizedCache<Fiber>>(
    make_shared<BlockingCache>(
        make_shared<LruCache>(
            make_shared<ShardedMapCache>(...)))));
```