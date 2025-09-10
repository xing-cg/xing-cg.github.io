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

# 设计思路
1. 参考 MyBatis 的 Cache 设计——实现一些基础的 Cache 类，通过装饰器模式增强功能
    1. MyBatis 实现 LRUCache 的时候将 LRU 和底层的存储逻辑解耦
2. 在 tRPC 里面有实现好的**无锁 Hash** 以及 **无锁队列**
    1. 无锁 Hash：`trpc/util/concurrency/lightly_concurrent_hashmap.h`
        1. 基本结构：分片锁管理 Slot（即**分片锁 Hash**），锁内保护 **无锁链表** 结构
    2. **无锁队列**：`trpc/util/queue`
    3. 总结下来，tRPC 目前可以复用的结构有：分片锁Hash、无锁链表、无锁队列
3. LRUCache 需要管理的两个数据结构：Hash + 链表
4. FIFOCache 需要管理的两个数据结构：Hash + 队列

实现文件存放于：`trpc/util/cache/cache.h`、`trpc/util/cache/cache.h/decorators/lru_cache.h`
# ShardedLRUCache 设计与实现 - xing-cg版

基于[trpc-lru-group/trpc-cpp/tree/lru_cache_1_wu](https://github.com/trpc-lru-group/trpc-cpp/tree/lru_cache_1_wu)分支的 Cache 实现的 装饰器模式 的 分片 LRU Cache。
[PR链接：https://github.com/trpc-lru-group/trpc-cpp/pull/3](https://github.com/trpc-lru-group/trpc-cpp/pull/3)
## Motivation

* 在高并发场景下，单实例 LRU 受限于**全局互斥**，在多线程/协程下出现明显锁争用，吞吐与延迟抖动明显。
* 工业界的主流做法是 **Sharded LRU**：按 `hash(key)` 将 Key 均匀分散到多片，**片内严格 LRU**、**片间近似全局 LRU**，从而显著降低锁竞争、提升并发度。
* 本 PR 落地 `ShardedLRUCache`，作为**分片编排层**，复用已有 `LRUCache` 的逻辑与单测，降低实现和维护风险，并为后续 Fiber 互斥、异步提升、性能基准打下基础。

---

## Design

### 分片（Sharding）

* **分片数**：`num_shards = 2^ShardBits`（编译期常量，默认 `ShardBits=4` → 16 片）。
* **路由规则**：`shard_id = Hash(key) & (num_shards - 1)`（位与，要求分片数为 2 的幂）。
* **容量切分**：总容量 `total` 等分到各片：

  * `base = total / num_shards`，余数 `rem = total % num_shards`；
  * 前 `rem` 个分片 `base+1`，其余 `base`；**每片至少 1**。
* **全局 Size**：聚合各片 `Size()`（近似实时值）。

### 片内策略（strict LRU per shard）

* 每个分片内部复用 **`LRUCache<Key,Value,Mutex>`**（双向链表 + 哈希索引；命中提升到 MRU；容量满时淘汰队尾 LRU）。
* 片内有且仅有一把互斥（模板参数 `Mutex`，默认 `std::mutex`；后续支持 `FiberMutex`）。

### 并发模型

* **片间无共享互斥**，并发度≈分片数；
* **片内加锁**保护 LRU 元数据与底座存取；
* 默认底座为 `BasicCache`（并发 HashMap）。若验证存在**双重加锁**的不可忽视开销，将切换为**无锁底座 MapCache**（仅 `unordered_map`，依赖 LRU 层互斥）。

---

## Public API

```cpp
template <
  typename KeyType,
  typename ValueType,
  typename HashFn   = std::hash<KeyType>,
  typename KeyEqual = std::equal_to<KeyType>,
  typename Mutex    = std::mutex,
  std::size_t ShardBits = 4
>
class ShardedLRUCache : public Cache<KeyType, ValueType, HashFn, KeyEqual> {
 public:
  using FactoryFn = std::function<std::unique_ptr<Cache<KeyType,ValueType,HashFn,KeyEqual>>()>;

  explicit ShardedLRUCache(std::size_t capacity,
                           FactoryFn factory = [] {
                             return std::make_unique<BasicCache<KeyType,ValueType,HashFn,KeyEqual>>();
                           });

  bool Put(const KeyType& key, const ValueType& value) override;
  bool Put(const KeyType& key, ValueType&& value) override;
  std::optional<ValueType> Get(const KeyType& key) override;
  bool Remove(const KeyType& key) override;
  void Clear() override;
  std::size_t Size() override;

  static constexpr std::size_t NumShards();
  std::size_t Capacity() const;
};
```

### Usage example

```cpp
// 16-shard, total capacity = 1<<20
ShardedLRUCache<std::string, std::string> cache(1 << 20);
cache.Put("k", "v");
auto v = cache.Get("k"); // -> "v"
```

---

## Tests

新增 `sharded_lru_cache_test.cc`，覆盖点：

1. **Move 语义**：移动构造/赋值后数据一致、Size 正确。
2. **Basic Put/Get/Size**：基本操作与容量计数。
3. **Duplicate Key**：更新值不改变 Size（与底座 InsertOrAssign 语义对齐）。
4. **Remove/Remove non-existing**：存在/不存在删除路径。
5. **Clear**：清空所有分片。
6. **Per-shard LRU**：构造可控哈希（IdentityHash），让插入集中到指定分片，验证**片内严格 LRU 淘汰**、**片间互不影响**。
7. **Single shard 退化**：`ShardBits=0` 时行为等价于全局单片 LRU（与现有 `LRUCache` 用例对齐）。
8. **并发 Put/Get**：8 线程高并发写入/读取，断言 `hits == Size()`、正确命中/未命中分布。

> 本地已在 Debug/Release 下通过，支持 gtest 跑法与 CI 集成。建议在 TSan/ASan 下再跑一遍以兜底数据竞争与越界。

---

## 为什么要基于 LRUCache（装饰器）构建而不是在裸缓存上重新实现？

**选择：Sharded 编排层 + 片内复用 `LRUCache`（现有装饰器）**。

### 优势

* **逻辑复用 & 行为一致**：LRU 细节（命中提升、溢出淘汰、边界 case）已经在单分片用例中验证；避免**重复实现**导致的行为漂移。
* **测试负担小**：Sharded 层只关注“分片路由与容量切分”；LRU 行为由既有单测保障，组合起来更易维护。
* **易于扩展**：后续若要接入 `Synchronized`/`Blocking`/`TTL` 等装饰器或替换为 `AsyncLRU`（异步提升），**分片层无需改动**。
* **更清晰的模块边界**：Sharded 负责并发度与路由，LRU 负责策略；职责单一，易读易演进。

### 代价 / 缺点

* **潜在双重加锁**：若分片底座是并发容器（如 `BasicCache`），片内 LRU 已有互斥，再调用到底座的并发结构会**重复加锁**。

  * **缓解方案**：提供/切换到底座 **无锁 `MapCache`**（仅 `unordered_map`），由 LRU 的互斥统一保护。
* **轻微的虚调用/拼装开销**：LRU 作为装饰器包装底座，较“写死一体化实现”多一次间接，但在分片降低锁争用后，这点开销可忽略。
* **近似全局 LRU**：Sharded 模式全局上是“近似” LRU（片内严格、片间独立），不保证严格全局次序（与 LevelDB/RocksDB 的工程折中一致）。

> 结论：在**可维护性、扩展性、测试复用**与**可接受的性能折中**之间，基于 `LRUCache` 的分片编排是更稳妥的路径。

---

## Compatibility

* 纯新增类型，不修改已有公共接口；与现有 `Cache`/`LRUCache` 并存。
* 默认哈希 `std::hash<Key>`、比较器 `std::equal_to<Key>`；可自定义。
* 默认互斥 `std::mutex`；未来 PR 将提供 `FiberMutex` 版本。

---

## Performance (plan)

* micro-bench：`Put:Get = 1:9` 与 `5:5` 两种混合比；容量击穿与热点 Key 两种分布；对比：

  1. `LRUCache (single shard)`
  2. `ShardedLRUCache (ShardBits = 0/2/4/6)`
  3. `BasicCache` 直存（无 LRU）
* 指标：QPS、p99 延迟、锁争用（`perf lock`）、CPU 利用、cache line 失效；
* 期望：随着 `ShardBits` 提升，**锁争用显著下降**、吞吐上升；在热点 Key 工况下收益最明显。

---

## Follow-ups

* **FiberMutex 版本**：模板化互斥，针对协程运行时切换 `trpc::FiberMutex`。
* **无锁/近无锁增强**：命中提升改为异步批处理（promote-queue + 批量 move-to-head），淘汰回调/大对象析构锁外执行。
* **底座 `MapCache`**：提供无并发底座以避免双重加锁；必要时作为默认底座切换。
* **Blocking/Singleflight**：装饰器抑制缓存击穿（miss 时 per-key 等待）。
* **Benchmark & 文档**：补充完整基准与使用说明（examples/，设计文档 design.md）。