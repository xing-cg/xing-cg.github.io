---
title: Benchmark_bitset
categories:
  - - Benchmark
tags:
date: 2025/10/21
updated: 2025/10/22
comments:
published:
---
# 场景
数据规模20000个，但有效数据很稀疏，50个。存的只是bool真假值。  
  
需要做 benchmark 调研：`std::vector<bool>`  
、`boost::dynamic_bitset`、`std::bitset`、`flat_set<short>`、`flat_unordered_set<short>`的性能差异调研。  
  
`std::bitset`是固定大小的位集。  
  
`std::vector<bool>`是可扩展的位集。1 个 bool 数据占 1 位。  
  
官方宣称，如果在编译时已知位集的大小，可以使用`std::bitset`，它提供了更丰富的成员函数集。  
  
`boost::dynamic_bitset`可以作为`std::vector<bool>`的替代方案。  
  
https://www.boost.org/doc/libs/latest/libs/dynamic_bitset/dynamic_bitset.html  
  
  
还需要注意：  
  
`std::vector<bool>`表示形式可能经过优化，`std::vector<bool>`不一定满足所有 Container 或 SequenceContainer 的要求。例如，由于 `std::vector<bool>` 是实现定义的，因此它可能不满足 LegacyForwardIterator 的要求。使用诸如 `std::search` 之类需要 LegacyForwardIterator 的算法可能会导致编译时或运行时错误 。  
  
Boost.Container 版本的 vector 并不专门针对 bool 。

## 总结
对稀疏真值位集（N=20,000，K≈50）在不同容器实现上的构建、查询与顺序扫描性能进行严谨对比：
- `std::bitset<N>`
- `boost::dynamic_bitset<>`
- `std::vector<bool>`
- `boost::container::flat_set<short>`
- `boost::unordered_flat_set<short>`
# 骨架
先创建一个结构化的基准测试工程骨架（CMake + Google Benchmark + 示例基准项）。
先创建目录：
`benchmark/bitset_bench/`

- **基准框架**：Google Benchmark（统计稳健性、自动热身、汇总）。

benchmark根目录的CMake文件：

```cmake
cmake_minimum_required(VERSION 3.16)
project(bitset_bench LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(BENCH_ENABLE_LTO "Enable link time optimization" ON)
if(BENCH_ENABLE_LTO)
  include(CheckIPOSupported)
  check_ipo_supported(RESULT ipo_supported OUTPUT ipo_error)
  if(ipo_supported)
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
  endif()
endif()

include(FetchContent)

# Google Benchmark
FetchContent_Declare(
  benchmark
  GIT_REPOSITORY https://github.com/google/benchmark.git
  GIT_TAG v1.8.3
)
FetchContent_MakeAvailable(benchmark)

add_subdirectory(bitset_bench)


```

子目录`bitset_bench`的CMake文件：
```cmake
set(BENCH_N 20000 CACHE STRING "Domain size (bitset length)")
set(BENCH_TRUE_COUNT 50 CACHE STRING "Number of true bits (sparsity)")
set(BENCH_QUERY_COUNT 100000 CACHE STRING "Random queries per iteration")

add_executable(bitset_bench main.cpp)
target_compile_features(bitset_bench PRIVATE cxx_std_20)

find_package(Threads REQUIRED)
option(ENABLE_BOOST_BENCH "Enable Boost-based benchmarks (dynamic_bitset / unordered_flat_set)" ON)

target_link_libraries(bitset_bench PRIVATE benchmark::benchmark Threads::Threads)

target_compile_definitions(bitset_bench PRIVATE
  BENCH_N=${BENCH_N}
  BENCH_TRUE_COUNT=${BENCH_TRUE_COUNT}
  BENCH_QUERY_COUNT=${BENCH_QUERY_COUNT}
)

if (ENABLE_BOOST_BENCH)
  find_package(Boost QUIET)
  if (Boost_FOUND)
    target_include_directories(bitset_bench PRIVATE ${Boost_INCLUDE_DIRS})
    target_compile_definitions(bitset_bench PRIVATE HAS_BOOST=1)
  else()
    message(WARNING "Boost not found; disabling Boost-based benchmarks. Install Boost or set -DENABLE_BOOST_BENCH=OFF.")
  endif()
endif()

if (CMAKE_CXX_COMPILER_ID MATCHES "Clang|AppleClang|GNU")
  target_compile_options(bitset_bench PRIVATE -O3 -DNDEBUG -Wall -Wextra -Wpedantic)
endif()
```
## 依赖
- CMake >= 3.16
- `C++23` 编译器（AppleClang/Clang/GCC）
    - 因为`C++23`才支持`flat_set`
- Google Benchmark（自动 FetchContent）
- Boost（头文件即可，用于 `dynamic_bitset` / `unordered_flat_set`）
## 编译选项
- **固定工作负载**：默认 N=20,000，K=50，随机查询 Q=100,000（50%命中）。
    - 均可通过编译期宏覆盖：`-DBENCH_N`、`-DBENCH_TRUE_COUNT`、`-DBENCH_QUERY_COUNT`。
- **编译设置**：Release（`-O3 -DNDEBUG`），可选 LTO（默认启用）。
# Cpp程序
## 编译期选择 Index 类型
```cpp
// Use the narrowest index type that can hold [0, BENCH_N)
using IndexType = std::conditional_t<
    (kDomainSize - 1) <= static_cast<std::size_t>(std::numeric_limits<std::uint16_t>::max()),
    std::uint16_t,
    std::uint32_t>;
```
这是在“编译期”选择索引类型，尽量用更窄的无符号整数来表示区间 `[0, BENCH_N)` 的索引，以节省内存并提升性能。
逻辑：如果 `N−1` 不超过 `uint16_t` 的上限（65535），则 IndexType 取 `uint16_t`；否则取 `uint32_t`。这完全在编译期决定，没有运行时开销。
影响：用于存放真值位置、查询序列、以及基于集合的容器键类型。`N=20000` 时用 `uint16_t`；`N=200000` 时用 `uint32_t`。
假设与边界：默认假设 N≥1；若未来 N 可能超过 `2^32−1`，可把分支再扩展到 `uint64_t`。

## `make_dataset` 生成可复现的 K 个唯一索引（真值位置）的数据集
- 先构造 `0..N-1` 的索引数组（reserve(domain) 是为避免反复扩容）。（参数 domain 控制）
- 用固定种子 `mt19937` 打乱顺序（保证每次运行一致）。
- 截取前 K 个（得到 K 个索引）。（参数 take 控制）
- 再 `sort + unique` 确保“有序且唯一”。

这样返回的 `ones_sorted` （`std::vector<IndexType>`）就是升序、不重复、可复现实验的真值位置集合。随后还会用它：
- `complementIndices` 求出未命中的索引集合（全域减去已选）。
- 从命中/未命中集合各采样，合成 50% 命中率的随机查询集，并再打乱顺序。
### shuffle
`std::shuffle(all.begin(), all.end(), rng)`是 C++ 中用来​**​随机打乱​**​一个序列（比如数组或容器）元素顺序的标准方法。

| 代码部分           | 含义                           | 说明                                        |
| -------------- | ---------------------------- | ----------------------------------------- |
| `std::shuffle` | 标准库中的随机重排函数                  | 位于 `<algorithm>`头文件中                      |
| `all.begin()`  | 指向序列​**​开始​**​位置的迭代器         | 与 `all.end()`一起定义了需要被打乱的范围 `[begin, end)` |
| `all.end()`    | 指向序列​**​结尾​**​（最后一个元素之后）的迭代器 |                                           |
| `rng`          | ​**​随机数生成器​**​               | 一个提供了随机性的对象，是 `std::shuffle`随机性的来源        |
如果希望每次程序运行时打乱的顺序都相同（例如，为了复现某个测试结果），可以为随机数生成器提供一个​**​固定种子​**​：
```cpp
std::mt19937 rng(42); // 使用固定种子 42
std::shuffle(all.begin(), all.end(), rng); // 每次结果都相同
```
- ​**​区别于已弃用的函数​**​：在 C++11 之前，人们使用 `std::random_shuffle`，但它默认依赖 `rand()`函数，随机性较差且非线程安全。自 C++14 起它被弃用，C++17 中已被移除。​**​`std::shuffle`是现代 C++ 推荐的方式​**。​
- ​**​容器要求​**​：`std::shuffle`要求容器支持​**​随机访问迭代器​**​，因此它适用于 `std::vector`、`std::array`、`std::deque`和普通数组等。但不适用于 `std::list`这类不支持随机访问的容器。
### 技巧：排序-压缩-删除

C++中一个非常经典的STL操作，用于​**​从容器中删除所有重复元素，只保留唯一值​**​。它通常被称为“排序-去重”或“unique-erase”惯用法。

下面这个表格可以帮你快速理解整个操作过程：

| 步骤            | 函数                                    | 作用          | 说明                                          |
| ------------- | ------------------------------------- | ----------- | ------------------------------------------- |
| ​**​1. 排序​**​ | `std::sort(all.begin(), all.end())`   | 使重复元素相邻     | ​**​（关键前提）​**​ 这行代码本身不包含排序，但你必须先做这一步，否则去重无效 |
| ​**​2. 压缩​**​ | `std::unique(all.begin(), all.end())` | 将相邻重复元素移到末尾 | 返回一个指向​**​新逻辑末尾​**​的迭代器                     |
| ​**​3. 删除​**​ | `all.erase(new_end, all.end())`       | 物理删除末尾的重复元素 | 容器大小真正变小                                    |
### 代码
```cpp
std::vector<IndexType> generateUniqueIndices(std::size_t domain,
                                             std::size_t take,
                                             std::uint32_t seed) {
  std::vector<IndexType> all;
  all.reserve(domain);
  for (std::size_t i = 0; i < domain; ++i)
    all.push_back(static_cast<IndexType>(i));
  std::mt19937 rng(seed);
  std::shuffle(all.begin(), all.end(), rng);
  if (take < all.size())
    all.resize(take);
  std::sort(all.begin(), all.end());
  all.erase(std::unique(all.begin(), all.end()), all.end());
  return all;
}
```
## Benchmark项目
1. 从0到20000构建
2. random query，一半命中，一半未命中。测试contains性能
3. 顺序扫描
## 总代码
```cpp
#include <benchmark/benchmark.h>

#include <algorithm>
#include <array>
#include <cstdint>
#include <limits>
#include <numeric>
#include <random>
#include <type_traits>
#include <utility>
#include <vector>

#include <bitset>
#ifdef HAS_BOOST
#  include <boost/dynamic_bitset.hpp>
#  include <boost/container/flat_set.hpp>
#  include <boost/unordered/unordered_flat_set.hpp>
#endif

#ifndef BENCH_N
#  define BENCH_N 20000
#endif
#ifndef BENCH_TRUE_COUNT
#  define BENCH_TRUE_COUNT 50
#endif
#ifndef BENCH_QUERY_COUNT
#  define BENCH_QUERY_COUNT 100000
#endif

static constexpr std::size_t kDomainSize = static_cast<std::size_t>(BENCH_N);
static constexpr std::size_t kTrueCount  = static_cast<std::size_t>(BENCH_TRUE_COUNT);
static constexpr std::size_t kQueryCount = static_cast<std::size_t>(BENCH_QUERY_COUNT);

// Use the narrowest index type that can hold [0, BENCH_N)
using IndexType = std::conditional_t<
    (kDomainSize - 1) <= static_cast<std::size_t>(std::numeric_limits<std::uint16_t>::max()),
    std::uint16_t,
    std::uint32_t>;

static_assert(kTrueCount <= kDomainSize, "True count cannot exceed domain size");

namespace util {

std::vector<IndexType> generateUniqueIndices(std::size_t domain, std::size_t take, std::uint32_t seed) {
  std::vector<IndexType> all;
  all.reserve(domain);
  for (std::size_t i = 0; i < domain; ++i) all.push_back(static_cast<IndexType>(i));
  std::mt19937 rng(seed);
  std::shuffle(all.begin(), all.end(), rng);
  if (take < all.size()) all.resize(take);
  std::sort(all.begin(), all.end());
  all.erase(std::unique(all.begin(), all.end()), all.end());
  return all;
}

std::vector<IndexType> generateRandomIndicesFromPool(const std::vector<IndexType>& pool, std::size_t count, std::uint32_t seed) {
  if (pool.empty()) return {};
  std::vector<IndexType> out;
  out.reserve(count);
  std::mt19937 rng(seed);
  std::uniform_int_distribution<std::size_t> dist(0, pool.size() - 1);
  for (std::size_t i = 0; i < count; ++i) {
    out.push_back(pool[dist(rng)]);
  }
  return out;
}

std::vector<IndexType> complementIndices(const std::vector<IndexType>& present, std::size_t domain) {
  std::vector<IndexType> missing;
  missing.reserve(domain - present.size());
  std::size_t p = 0;
  for (std::size_t i = 0; i < domain; ++i) {
    IndexType ii = static_cast<IndexType>(i);
    if (p < present.size() && present[p] == ii) {
      ++p;
    } else {
      missing.push_back(ii);
    }
  }
  return missing;
}

} // namespace util

// Containers under test
using BitsetFixed = std::bitset<BENCH_N>;
#ifdef HAS_BOOST
using BitsetDyn   = boost::dynamic_bitset<>;
using FlatSet     = boost::container::flat_set<IndexType>;
using UnorderedFlatSet = boost::unordered_flat_set<IndexType>;
#endif

// Build helpers
BitsetFixed build_bitset_fixed(const std::vector<IndexType>& ones) {
  BitsetFixed b; // zero-initialized
  for (IndexType idx : ones) b.set(static_cast<std::size_t>(idx));
  return b;
}

#ifdef HAS_BOOST
BitsetDyn build_bitset_dynamic(const std::vector<IndexType>& ones) {
  BitsetDyn b(kDomainSize);
  for (IndexType idx : ones) b.set(static_cast<std::size_t>(idx));
  return b;
}
#endif

std::vector<bool> build_vector_bool(const std::vector<IndexType>& ones) {
  std::vector<bool> v(kDomainSize);
  for (IndexType idx : ones) v[static_cast<std::size_t>(idx)] = true;
  return v;
}

#ifdef HAS_BOOST
FlatSet build_flat_set(const std::vector<IndexType>& ones) {
  FlatSet s;
  s.reserve(ones.size());
  for (IndexType idx : ones) s.insert(idx);
  return s;
}
#endif

#ifdef HAS_BOOST
UnorderedFlatSet build_unordered_flat_set(const std::vector<IndexType>& ones) {
  UnorderedFlatSet s;
  s.reserve(ones.size());
  for (IndexType idx : ones) s.insert(idx);
  return s;
}
#endif

// Contains helpers
template <typename T>
inline bool contains_index(const T& container, std::size_t i);

template <> inline bool contains_index<BitsetFixed>(const BitsetFixed& b, std::size_t i) { return b.test(i); }
#ifdef HAS_BOOST
template <> inline bool contains_index<BitsetDyn>(const BitsetDyn& b, std::size_t i) { return b.test(i); }
#endif
template <> inline bool contains_index<std::vector<bool>>(const std::vector<bool>& v, std::size_t i) { return v[i]; }
#ifdef HAS_BOOST
template <> inline bool contains_index<FlatSet>(const FlatSet& s, std::size_t i) { return s.find(static_cast<IndexType>(i)) != s.end(); }
template <> inline bool contains_index<UnorderedFlatSet>(const UnorderedFlatSet& s, std::size_t i) { return s.find(static_cast<IndexType>(i)) != s.end(); }
#endif

// Precomputed data
struct Dataset {
  std::vector<IndexType> ones_sorted;
  std::vector<IndexType> misses_pool;
  std::vector<IndexType> rand_queries_50_50;
};

Dataset make_dataset() {
  Dataset d;
  d.ones_sorted = util::generateUniqueIndices(kDomainSize, kTrueCount, /*seed=*/0xC0FFEEu);
  auto misses = util::complementIndices(d.ones_sorted, kDomainSize);
  // 50-50 hits/misses random queries
  auto hits  = util::generateRandomIndicesFromPool(d.ones_sorted, kQueryCount / 2, /*seed=*/0xBEEF1234u);
  auto missq = util::generateRandomIndicesFromPool(misses,       kQueryCount - hits.size(), /*seed=*/0xDEADBEEFu);
  d.rand_queries_50_50.reserve(hits.size() + missq.size());
  d.rand_queries_50_50.insert(d.rand_queries_50_50.end(), hits.begin(), hits.end());
  d.rand_queries_50_50.insert(d.rand_queries_50_50.end(), missq.begin(), missq.end());
  // Shuffle queries to avoid locality artifacts
  std::mt19937 rng(0x1234567u);
  std::shuffle(d.rand_queries_50_50.begin(), d.rand_queries_50_50.end(), rng);
  d.misses_pool = std::move(misses);
  return d;
}

static Dataset g_dataset = make_dataset();

// 1) Build benchmarks
static void BM_build_vector_bool(benchmark::State& state) {
  for (auto _ : state) {
    benchmark::DoNotOptimize(_);
    auto v = build_vector_bool(g_dataset.ones_sorted);
    benchmark::DoNotOptimize(v);
    benchmark::ClobberMemory();
  }
  state.counters["N"] = static_cast<double>(kDomainSize);
  state.counters["K"] = static_cast<double>(kTrueCount);
}

static void BM_build_bitset_fixed(benchmark::State& state) {
  for (auto _ : state) {
    benchmark::DoNotOptimize(_);
    auto b = build_bitset_fixed(g_dataset.ones_sorted);
    benchmark::DoNotOptimize(b);
    benchmark::ClobberMemory();
  }
  state.counters["N"] = static_cast<double>(kDomainSize);
  state.counters["K"] = static_cast<double>(kTrueCount);
}

#ifdef HAS_BOOST
static void BM_build_bitset_dynamic(benchmark::State& state) {
  for (auto _ : state) {
    benchmark::DoNotOptimize(_);
    auto b = build_bitset_dynamic(g_dataset.ones_sorted);
    benchmark::DoNotOptimize(b);
    benchmark::ClobberMemory();
  }
  state.counters["N"] = static_cast<double>(kDomainSize);
  state.counters["K"] = static_cast<double>(kTrueCount);
}
#endif

#ifdef HAS_BOOST
static void BM_build_flat_set(benchmark::State& state) {
  for (auto _ : state) {
    benchmark::DoNotOptimize(_);
    auto s = build_flat_set(g_dataset.ones_sorted);
    benchmark::DoNotOptimize(s);
    benchmark::ClobberMemory();
  }
  state.counters["N"] = static_cast<double>(kDomainSize);
  state.counters["K"] = static_cast<double>(kTrueCount);
}
#endif

#ifdef HAS_BOOST
static void BM_build_unordered_flat_set(benchmark::State& state) {
  for (auto _ : state) {
    benchmark::DoNotOptimize(_);
    auto s = build_unordered_flat_set(g_dataset.ones_sorted);
    benchmark::DoNotOptimize(s);
    benchmark::ClobberMemory();
  }
  state.counters["N"] = static_cast<double>(kDomainSize);
  state.counters["K"] = static_cast<double>(kTrueCount);
}
#endif

// 2) Random contains() queries (50% hits, 50% misses), container built outside timed loop
template <typename Container>
static void BM_contains_random(benchmark::State& state, const char* name, const Container& container) {
  (void)name; // name is useful if we later add custom labels
  const auto& queries = g_dataset.rand_queries_50_50;
  for (auto _ : state) {
    std::size_t sum = 0;
    for (std::size_t i = 0; i < queries.size(); ++i) {
      sum += contains_index<Container>(container, static_cast<std::size_t>(queries[i]));
    }
    benchmark::DoNotOptimize(sum);
  }
  state.counters["Q"] = static_cast<double>(queries.size());
  state.counters["N"] = static_cast<double>(kDomainSize);
  state.counters["K"] = static_cast<double>(kTrueCount);
}

static void BM_contains_random_vector_bool(benchmark::State& state) {
  auto container = build_vector_bool(g_dataset.ones_sorted);
  BM_contains_random(state, "vector_bool", container);
}

static void BM_contains_random_bitset_fixed(benchmark::State& state) {
  auto container = build_bitset_fixed(g_dataset.ones_sorted);
  BM_contains_random(state, "bitset_fixed", container);
}

#ifdef HAS_BOOST
static void BM_contains_random_bitset_dynamic(benchmark::State& state) {
  auto container = build_bitset_dynamic(g_dataset.ones_sorted);
  BM_contains_random(state, "bitset_dynamic", container);
}
#endif

#ifdef HAS_BOOST
static void BM_contains_random_flat_set(benchmark::State& state) {
  auto container = build_flat_set(g_dataset.ones_sorted);
  BM_contains_random(state, "flat_set", container);
}
#endif

#ifdef HAS_BOOST
static void BM_contains_random_unordered_flat_set(benchmark::State& state) {
  auto container = build_unordered_flat_set(g_dataset.ones_sorted);
  BM_contains_random(state, "unordered_flat_set", container);
}
#endif

// 3) Sequential scan across full domain: count number of true bits
template <typename Container>
static void BM_sequential_scan_count_true(benchmark::State& state, const char* name, const Container& container) {
  (void)name;
  for (auto _ : state) {
    std::size_t cnt = 0;
    for (std::size_t i = 0; i < kDomainSize; ++i) {
      cnt += contains_index<Container>(container, i);
    }
    benchmark::DoNotOptimize(cnt);
  }
  state.counters["N"] = static_cast<double>(kDomainSize);
  state.counters["K"] = static_cast<double>(kTrueCount);
}

static void BM_sequential_scan_vector_bool(benchmark::State& state) {
  auto container = build_vector_bool(g_dataset.ones_sorted);
  BM_sequential_scan_count_true(state, "vector_bool", container);
}

static void BM_sequential_scan_bitset_fixed(benchmark::State& state) {
  auto container = build_bitset_fixed(g_dataset.ones_sorted);
  BM_sequential_scan_count_true(state, "bitset_fixed", container);
}

#ifdef HAS_BOOST
static void BM_sequential_scan_bitset_dynamic(benchmark::State& state) {
  auto container = build_bitset_dynamic(g_dataset.ones_sorted);
  BM_sequential_scan_count_true(state, "bitset_dynamic", container);
}
#endif

#ifdef HAS_BOOST
static void BM_sequential_scan_flat_set(benchmark::State& state) {
  auto container = build_flat_set(g_dataset.ones_sorted);
  BM_sequential_scan_count_true(state, "flat_set", container);
}
#endif

#ifdef HAS_BOOST
static void BM_sequential_scan_unordered_flat_set(benchmark::State& state) {
  auto container = build_unordered_flat_set(g_dataset.ones_sorted);
  BM_sequential_scan_count_true(state, "unordered_flat_set", container);
}
#endif

BENCHMARK(BM_build_vector_bool);
BENCHMARK(BM_build_bitset_fixed);
// Boost-enabled benchmarks
#ifdef HAS_BOOST
BENCHMARK(BM_build_bitset_dynamic);
BENCHMARK(BM_build_flat_set);
BENCHMARK(BM_build_unordered_flat_set);
#endif

BENCHMARK(BM_contains_random_vector_bool);
BENCHMARK(BM_contains_random_bitset_fixed);
#ifdef HAS_BOOST
BENCHMARK(BM_contains_random_bitset_dynamic);
BENCHMARK(BM_contains_random_flat_set);
BENCHMARK(BM_contains_random_unordered_flat_set);
#endif

BENCHMARK(BM_sequential_scan_vector_bool);
BENCHMARK(BM_sequential_scan_bitset_fixed);
#ifdef HAS_BOOST
BENCHMARK(BM_sequential_scan_bitset_dynamic);
BENCHMARK(BM_sequential_scan_flat_set);
BENCHMARK(BM_sequential_scan_unordered_flat_set);
#endif

BENCHMARK_MAIN();



```
# 矩阵模板
```csv
impl,N,K,Q,benchmark,min(ns),median(ns),mean(ns),stddev(ns),notes
std::bitset,20000,50,100000,build,,,,,
std::bitset,20000,50,100000,contains_random,,,,,
std::bitset,20000,50,100000,sequential_scan,,,,,
boost::dynamic_bitset,20000,50,100000,build,,,,,
boost::dynamic_bitset,20000,50,100000,contains_random,,,,,
boost::dynamic_bitset,20000,50,100000,sequential_scan,,,,,
std::vector<bool>,20000,50,100000,build,,,,,
std::vector<bool>,20000,50,100000,contains_random,,,,,
std::vector<bool>,20000,50,100000,sequential_scan,,,,,
boost::container::flat_set<short>,20000,50,100000,build,,,,,
boost::container::flat_set<short>,20000,50,100000,contains_random,,,,,
boost::container::flat_set<short>,20000,50,100000,sequential_scan,,,,,
boost::unordered_flat_set<short>,20000,50,100000,build,,,,,
boost::unordered_flat_set<short>,20000,50,100000,contains_random,,,,,
boost::unordered_flat_set<short>,20000,50,100000,sequential_scan,,,,,

```

# 测试执行
```bash
rm -rf ~/Work/benchmark/build
cmake -S ~/Work/benchmark -B ~/Work/benchmark/build -DCMAKE_BUILD_TYPE=Release -DENABLE_BOOST_BENCH=ON

cmake --build ~/Work/benchmark/build -j

~/Work/benchmark/build/bitset_bench/bitset_bench --benchmark_min_time=0.5 --benchmark_repetitions=15 --benchmark_report_aggregates_only=true
```

​​-S​​：用于指定​​源代码目录​​，即顶级 CMakeLists.txt文件所在的目录。例如，-S .表示使用当前目录作为源码目录 
。
​​-B​​：用于指定​​构建目录​​，即所有中间构建文件（如 Makefile、CMakeCache.txt）和最终编译输出文件存放的目录。如果指定的目录不存在，CMake 会自动创建它。

最常见的用法是 `cmake -S . -B build`。这个命令的含义是：使用当前目录（`.`）的源码，并在当前目录下创建一个名为 `build`的文件夹来存放所有构建文件

除了 `-S`和 `-B`，CMake 还提供了许多其他实用选项。下表进行了归纳：

| 选项                            | 说明                                             | 示例                               |
| ----------------------------- | ---------------------------------------------- | -------------------------------- |
| ​**​`-D <var>=<value>`​**​    | 设置或覆盖 CMake 缓存变量。这是最常用的选项之一，用于传递配置参数。          | `-DCMAKE_BUILD_TYPE=Release`     |
| ​**​`-G <generator-name>`​**​ | 指定生成器，即选择使用哪种底层构建系统（如 Makefile 或 Ninja）。       | `-G "Ninja"`                     |
| ​**​`-j [<jobs>]`​**​         | 在运行 `cmake --build`时使用，指定并行编译的作业数，以加快构建速度。     | `cmake --build build -j 8`       |
| ​**​`--target <tgt>`​**​      | 在运行 `cmake --build`时使用，指定要构建的具体目标（如某个可执行文件或库）。 | `cmake --build build --target m` |
# 生成compile_commands.json并在工作区创建符号链接
确保 IDE 能正确解析头文件与跳转。

在 `benchmark/CMakeLists.txt`中启用`CMAKE_EXPORT_COMPILE_COMMANDS`。

然后，重新运行CMake。

```bash
cmake -S . -B ./build -DCMAKE_BUILD_TYPE=Release -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

之后，在Work根目录创建符号链接：
```bash
ln -sf /Users/mrcan/Work/benchmark/build/compile_commands.json /Users/mrcan/Work/compile_commands.json
```


作用：

- 在 benchmark/CMakeLists.txt 增加了：
    - set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
- 重新生成了编译数据库：
    - 生成位置：`/Users/mrcan/Work/benchmark/build/compile_commands.json`
    - 在工作区根建立符号链接：`/Users/mrcan/Work/compile_commands.json`
- C++ 标准与依赖：
    - 已设置 `-std=c++20`，并通过 `benchmark::benchmark` 目标自动注入 `-I.../benchmark-src/include` 搜索路径。

