# Redis 面试复习笔记

> 适用：后端开发面试准备 · 系统复习 · 面试速查
> 版本参考：Redis 7.x（6.0 IO 线程、7.0 listpack 全面替代 ziplist）

---

# 一、Redis 概述

## 一、背景（为什么需要）

传统关系型数据库（MySQL）处理高并发读请求时面临两个瓶颈：

1. **磁盘 I/O 瓶颈**：即使有索引，磁盘寻道 + 读取仍需 ms 级延迟，热点数据反复读盘浪费资源
2. **连接与 CPU 瓶颈**：每秒万级请求下，DB 连接池和 CPU 迅速饱和

Redis 的设计答案：把热点数据放在内存里，用最简单的数据结构和单线程模型，把每次操作压缩到 μs 级。

## 二、核心概念（是什么）

Redis = Remote Dictionary Server。本质是一个**基于内存的全局哈希表服务器**：

```
SET name "Tom" 的本质：
  全局 dict → dictEntry { key: SDS("name"), value: redisObject("Tom") }

HSET user:1 name Tom age 18 的本质：
  全局 dict → dictEntry { key: SDS("user:1"), value: redisObject(Hash) }
  其中 redisObject.ptr → listpack 或 hashtable，存储 field-value 对
```

**四层架构**：

```
表面：SET / GET / HSET / LPUSH / ZADD ...        ← 用户接口
对象层：redisObject（type + encoding + ptr）      ← 统一抽象
数据结构层：SDS / listpack / dict / quicklist / skiplist / intset  ← 物理存储
引擎层：RDB / AOF / 混合持久化 / 主从 / Sentinel / Cluster ← 可靠+高可用
```

Redis ≠ 简单的 Map。它是：**全局哈希表 + 10 种底层数据结构 + 网络事件驱动处理器 + 持久化引擎 + 高可用复制系统**。

**与 Memcached 对比**：

| 维度 | Redis | Memcached |
|------|-------|-----------|
| 数据结构 | 5 种基础 + Bitmap/HLL/Geo/Stream | 仅 String（二进制 blob） |
| 持久化 | RDB + AOF + 混合 | 不支持 |
| 集群 | 官方 Cluster 去中心化（16384 slot） | 客户端一致性哈希 |
| 线程模型 | 命令单线程 + 6.0 IO 多线程 | 原生多线程 |
| 单个 value 上限 | 512MB | 1MB |
| Lua/事务 | 支持 | 不支持 |

## 三、底层原理

### 一条命令的执行全过程

```
客户端 TCP 发送 RESP 命令（*3\r\n$3\r\nSET...）
  → epoll_wait 监听到可读事件
  → RESP 协议解析（解析为 argc/argv）
  → 命令表查找（server.commands 字典找到对应的 redisCommand）
  → 执行命令函数（dict 操作：dictAdd / dictFind）
  → 如果开启 AOF → 追加写命令到 AOF 缓冲区
  → 返回 RESP 响应（+OK\r\n）
```

整个路径极短：网络 I/O → RESP 解析 → 命令执行 → dict 操作 → 返回。CPU 计算时间通常 < 5μs。

### Redis 为什么快：十大原因

```
存储层（3项）：
  ① 纯内存操作：DRAM ~100ns，比 SSD 快 1000 倍
  ② 高效数据结构：SDS/dict/skiplist/listpack/quicklist 针对性优化
  ③ 紧凑编码策略：小数据用 intset/listpack/embstr，几乎零元数据开销

执行层（3项）：
  ④ 单线程命令执行：无锁 + 无上下文切换 + CPU Cache 热
  ⑤ C 语言直接系统调用：无虚拟机/GC 开销
  ⑥ 渐进式 rehash + lazy free：避免大操作阻塞主线程

网络层（4项）：
  ⑦ IO 多路复用 epoll：O(1) 事件分发 + ET 边缘触发
  ⑧ RESP 协议极简：文本协议，解析成本极低
  ⑨ Pipeline：N 个命令合并为 1 次网络往返
  ⑩ Redis 6.0+ IO 线程：网络读写并行处理
```

### 为什么命令执行是单线程

**核心论点：性能瓶颈是网络 I/O 和内存带宽，不是 CPU。**

以 GET 为例：
```
网络往返（同机房）：  ~500μs  ← 99% 耗时
RESP 协议解析：         ~2μs
dict 哈希查找：         ~1μs
RESP 序列化：           ~1μs
网络写回：            ~100μs
总耗时 ≈ 600μs，CPU 计算 < 5μs（< 1%）
```

**单线程的收益**：
- 无锁：所有数据结构操作天然互斥，无 futex 系统调用开销
- 无上下文切换：CPU L1/L2 Cache 保持热度，每次切换损失 ~1-2μs
- 原子性天然保证：单条命令无需额外同步
- 代码极简：`while(1) + epoll_wait + processCommand()`

**单线程的代价与应对**：
- 无法利用多核 → 单机多实例（绑核）+ Redis Cluster
- 耗时命令阻塞 → KEYS→SCAN, DEL→UNLINK, FLUSHALL→FLUSHALL ASYNC
- 网络成为瓶颈 → 6.0 IO Threads

> Redis 不是"整个进程单线程"。fork 子进程（RDB/AOF rewrite）+ bio 线程（异步关闭文件/unlink）+ 6.0 IO 线程 = 多线程/多进程协同。

### Redis 6.0 多 IO 线程

**被多线程的（可并行）**：socket read()、RESP 协议解析、socket write()
**保持单线程的（绝不并行）**：命令执行、数据结构修改、Lua 脚本、MULTI/EXEC、过期删除

```
架构流程：
  epoll_wait → 主线程分配 clients 到 IO 线程池（Round-Robin）
  → IO 线程并行 read + 解析 RESP
  → 主线程等待全部完成 → 串行执行所有命令
  → IO 线程并行 write 响应
  → 主线程继续 epoll_wait
```

> 为什么命令执行不改多线程？一旦 dict/expires 等核心结构需加锁，锁竞争开销可能超过单线程执行。事务/Lua/复制顺序都会极其复杂。这是"花 10% 代价获 90% 收益"的精妙折中。

### 版本演进关键节点

| 版本 | 年份 | 关键变化 |
|------|------|---------|
| 1.0 | 2009 | 基础 KV，String/List/Set |
| 2.6 | 2012 | Lua 脚本 |
| 3.0 | 2015 | **Redis Cluster** 正式发布 |
| 4.0 | 2017 | **混合持久化**、UNLINK、LFU、Modules |
| 5.0 | 2018 | **Stream** 数据结构 |
| 6.0 | 2020 | **多线程网络 I/O**、ACL、RESP3 协议 |
| 7.0 | 2022 | **listpack 全面替代 ziplist**、Redis Functions、Shard Pub/Sub |

## 四、关键设计

### 为什么 Redis 不用多线程执行命令？

三个致命问题：
1. **锁竞争**：dict/expires/key-space 每个操作都需要加锁，在 Redis 的 μs 级延迟下，futex 开销不可忽略
2. **复杂性爆炸**：事务（MULTI/EXEC）、Lua 脚本、主从复制、AOF 持久化的顺序保证全部需要重构
3. **收益有限**：瓶颈在网络 I/O 而非 CPU，6.0 只把可并行的网络 I/O 多线程化已经能大幅提升吞吐

### 为什么选择单线程 + IO 多路复用？

Redis 是 I/O 密集型而非 CPU 密集型。单线程避免了并发控制的所有复杂性，用 epoll 解决了 C10K/C100K 连接管理问题。这是"以正确的方式做正确的事"——把有限的复杂度花在数据结构和持久化上。

## 五、面试高频问题

**Q1：Redis 为什么快？**

十大原因分三层。存储层：纯内存操作 + 高效数据结构 + 紧凑编码。执行层：单线程无锁 + C 语言 + 渐进式 rehash。网络层：epoll + RESP 极简协议 + Pipeline + 6.0 IO 线程。核心是瓶颈在网络不在 CPU。

**Q2：Redis 是单线程的吗？**

命令执行是单线程的（核心），但整个进程不是。fork 子进程做 RDB/AOF rewrite，bio 后台线程做异步删除，6.0 后 IO 线程做网络读写。说"Redis 是单线程的"在面试中需要补充这些例外。

**Q3：Redis 6.0 多线程改了哪里？**

只改了网络 I/O 和 RESP 协议解析。命令执行仍是主线程串行。原因：瓶颈在高带宽场景下是网络 I/O 而非 CPU，多线程并行处理网络读写即可大幅提升吞吐，不需要改动核心命令执行模型。

**Q4：Redis 和 Memcached 有什么区别？**

Redis 支持多种数据结构（不仅是 String）、持久化（RDB/AOF）、高可用（主从/哨兵/Cluster）、Lua 脚本和事务。Memcached 仅支持 String blob，无持久化，集群靠客户端哈希。几乎所有现代项目都选 Redis。

## 六、总结（速记版）

- **本质**：内存全局 dict + redisObject 统一封装 + 多编码按规模切换
- **十大快因**：内存 + 紧凑编码 + 高效结构 + 单线程无锁 + epoll + RESP + Pipeline + IO 线程 + C 语言 + 渐进 rehash
- **单线程理论**：瓶颈在网络 I/O 不在 CPU，单次 GET CPU < 5μs，网络 ~500μs
- **6.0 IO 线程**：只多线程化网络读写和协议解析，命令执行保持单线程

---

# 二、核心数据结构深度解析

## 一、背景（为什么需要）

Redis 能支撑丰富的数据类型（String/Hash/List/Set/ZSet），靠的是一套精巧的底层数据结构体系。同一个逻辑类型，在不同数据规模下自动切换底层实现——这是 Redis 内存效率和查询效率兼顾的核心设计。

## 二、核心概念（是什么）

### redisObject：Value 的统一抽象层

```c
typedef struct redisObject {
    unsigned type:4;       // 逻辑类型：STRING/LIST/SET/ZSET/HASH
    unsigned encoding:4;   // 底层编码：INT/EMBSTR/RAW/LISTPACK/HT/QUICKLIST/SKIPLIST
    unsigned lru:24;       // LRU: 秒级时间戳 / LFU: 高16位衰减时间 + 低8位对数计数
    int refcount;          // 引用计数，0-9999 整数共享对象
    void *ptr;             // 指向底层结构，int 编码时 ptr 直接存整数
} robj;  // 共 16 字节
```

**type vs encoding（高频混淆点）**：

```
TYPE key           → 返回 type（用户视角的逻辑类型）
OBJECT ENCODING key → 返回 encoding（Redis 内部的物理编码）

示例：
  SET age 18          → type=string, encoding=int        （直接存整数）
  SET name "Tom"      → type=string, encoding=embstr      （≤44B，连续内存）
  SET desc "很长的..." → type=string, encoding=raw        （>44B，分开分配）
  HSET small a 1      → type=hash, encoding=listpack      （小 Hash）
  HSET big a huge...  → type=hash, encoding=hashtable     （大 Hash）
```

> 设计思想：类似 OOP 的"接口 + 实现"——用户通过 type 定义的接口操作，Redis 根据数据规模通过 encoding 选择最佳实现。小数据紧凑省内存，大数据查询结构保性能。

### 五种数据结构编码总览

| 类型 | 小编码 | 大编码 | 转换阈值 |
|------|--------|--------|---------|
| String | int / embstr (≤44B) | raw | 44B |
| Hash | listpack | hashtable (dict) | 512 字段 / 64B 值 |
| List | quicklist（内部 listpack 节点） | quicklist | 节点 8KB |
| Set | intset（有序数组） | hashtable (dict) | 512 元素 |
| ZSet | listpack | skiplist + dict 双结构 | 128 元素 / 64B 值 |

## 三、底层原理

### SDS：String 的底层核心

**为什么不用 C 字符串？** C 字符串四个致命问题：
1. `strlen` 需要遍历到 `\0` → O(N)
2. `\0` 当结束符 → 不能存 Protobuf/图片/序列化数据（非二进制安全）
3. `strcat` 不检查空间 → 缓冲区溢出
4. 每次修改长度都需 realloc → 频繁内存分配

**SDS 结构（Redis 7.0 有 5 种 header：sdshdr5/8/16/32/64）**：

```c
struct sdshdr8 {
    uint8_t len;     // 实际数据长度 → O(1) 获取！
    uint8_t alloc;   // 已分配空间（不含 header 和 \0）
    unsigned char flags;  // header 类型标记
    char buf[];      // 柔性数组，紧跟 header
};
```

**SDS 四大优势**：
1. O(1) 获取长度：直接读 len，对频繁比较 key 的场景收益巨大
2. 二进制安全：以 len 判结尾而非 `\0`，可存任意二进制
3. 空间预分配：扩容时多分一倍（最大 1MB），减少后续 realloc
4. 惰性释放：缩短不立即回收，避免长-短-长往复的反复分配

**三种 encoding 的设计思路**：

| 编码 | 条件 | 内存分配 | 适用 |
|------|------|---------|------|
| int | value 是 long 范围整数 | ptr 直接存整数，零额外内存 | 计数器、ID |
| embstr | 字符串 ≤ 44 字节 | redisObject + SDS 一次 malloc | 短字符串、session token |
| raw | 字符串 > 44 字节 | redisObject 和 SDS 分开 malloc | 长文本、JSON |

**44 字节由来**：jemalloc 最小分配单元 64B - redisObject(16B) - sdshdr8(3B) - `\0`(1B) = 44B。

### dict：全局哈希表与渐进式 rehash

**Redis 数据库本质**：

```c
typedef struct redisDb {
    dict *dict;       // 存所有 key-value（主表）
    dict *expires;    // 存过期 key 的到期时间戳
    // ...
} redisDb;
```

**dict 双表设计**：

```c
typedef struct dict {
    dictEntry **ht_table[2];  // 两个哈希表！为渐进式 rehash
    unsigned long ht_used[2];
    long rehashidx;           // -1=不在 rehash, >=0=正在 rehash 的 bucket 索引
} dict;

typedef struct dictEntry {
    void *key;                // SDS
    union { void *val; uint64_t u64; int64_t s64; } v;
    struct dictEntry *next;   // 链地址法解决冲突
} dictEntry;
```

**渐进式 rehash 完整流程**：

```
触发扩容：
  负载因子 = used / size >= 1（BGSAVE/BGREWRITEAOF 时 >= 5，减少 COW 压力）
  → ht_table[1] 分配为 > used*2 的最小 2 的幂次
  → rehashidx = 0

迁移节奏：
  每次增删改查时顺带迁移 rehashidx 指向的 bucket（每次最多 1 个 bucket）
  serverCron 定时任务在空闲时推进（每次最多 1ms）

rehash 期间读写规则：
  查询：先 ht_table[0] → 未找到 → ht_table[1]
  新增：直接写入 ht_table[1]（让旧表只减不增，加速收敛）
  修改/删除：两张表都可能涉及

完成：
  ht_used[0] == 0 → 释放 ht_table[0] → ht_table[1] 变成 ht_table[0] → rehashidx = -1
```

**为什么新增写 ht_table[1]？** 旧表只减不增才能最终变空。如果新旧都写就永远迁不完。

**为什么 BGSAVE 时阈值提高到 5？** fork 后父进程写触发 COW（写时复制）。rehash 会产生大量新内存分配 → 更多 COW → 内存压力。提高阈值减少 rehash 频率，保护 BGSAVE 期间的内存。

### ZSet：skiplist + dict 双结构（最精妙的设计）

**为什么需要双结构？**

```
只用 dict：
  ZSCORE O(1) ✓
  ZRANGE 无法实现 ✗（dict 无序）

只用 skiplist：
  ZRANGE O(logN) ✓
  ZSCORE 需 O(N) ✗（遍历跳表找 member）

→ 两者配合：dict 管 O(1) 查 score，skiplist 管 O(logN) 排序范围查询
→ member 共享（指针引用），不复制数据
```

**跳表参数**：

- 随机层高算法：晋升概率 P=0.25（不是 0.5！）
- 最大 64 层
- 期望指针数 1.33 个/节点（P=0.5 则是 2 个/节点）→ 省 33% 内存
- 每层有 span 字段记录跨度，查找时累加得 rank（支持 ZRANK 命令）

**为什么跳表不选红黑树/B+树？**
- 跳表 vs 红黑树：实现简单（~200 行 vs 500+ 行）、范围查询直观（底层链表直接遍历）、插入只影响局部
- 跳表 vs B+ 树：B+ 树为磁盘 I/O 优化（节点 = 磁盘页），Redis 内存场景不需要

### 其他数据结构底层

**List → quicklist**：
- quicklist = 双向链表 + listpack 节点（Redis 3.2+）
- 普通链表每元素 16B 指针（prev+next）+ 独立分配 → 存大量小元素浪费严重
- listpack 紧凑但单一连续块修改代价高
- quicklist 折中：外层链表保灵活，内层 listpack 保紧凑

**Set → intset / hashtable**：
- 全整数且 ≤ 512 元素 → intset（有序数组，二分 O(logN)）
- 否则 → hashtable（dict，只存 key，value=NULL）
- intset 内部 int16 → int32 → int64 自动升级（不降级）

**Hash → listpack / hashtable**：
- 小 → listpack：连续内存，field-value 交替排列，O(N) 扫描
- 大 → hashtable：O(1) 查询
- 5 个 field 的小 Hash：hashtable 元数据 ~230B+，listpack 元数据 ~20B — 差 10 倍！

### 数据结构设计总哲学

```
小数据 → 紧凑结构（listpack / intset / embstr）
  连续内存、无指针、几乎零元数据开销
  O(N) 扫描在数据量小时完全够用

大数据 → 查询结构（hashtable / skiplist / quicklist）
  有指针开销，但 O(1) 或 O(logN) 保证性能

这就是 Redis "按数据规模动态选择最优编码"的核心设计哲学。
```

## 四、关键设计

### 什么是 listpack？为什么替代 ziplist？

**ziplist 的致命缺陷——连锁更新（cascading update）**：

```
ziplist 每个 entry 存储"前一个 entry 的长度"（previous_entry_length）。
如果前一个 entry 长度 < 254B → 该字段占 1B。
如果前一个 entry 长度 >= 254B → 该字段占 5B。

场景：ziplist 中所有 entry 都是 253B（previous_entry_length=1B）
→ 在头部插入一个 254B 的 entry → 下一个 entry 的 previous_entry_length 从 1B 变成 5B
→ 下一个 entry 膨胀 4B 后可能也 >= 254B → 下下个 entry 也要更新
→ 连锁反应！最坏 O(N²) 的重新分配和 memmove
```

**listpack 的解决方案**：不再记录"前一个 entry 的长度"，改为记录**当前 entry 的长度**（在 entry 末尾）。读取时从后往前扫描就能解析出每个 entry，彻底消除了连锁更新问题。

Redis 7.0 起所有原来用 ziplist 的场景（小 Hash、小 ZSet、Stream）全部迁移到 listpack。

### 为什么 intset 升级后不降级？

设计取舍：降级需要扫描所有元素判断是否可以降级 + 重新分配内存。Redis 认为"元素被删除后大概率还会再添加"，与其频繁升降级，不如保持当前编码。这是一种"惰性"策略——简单、稳定。

## 五、面试高频问题

**Q1：type 和 encoding 有什么区别？**

type 是用户视角的逻辑类型（STRING/LIST/SET 等），encoding 是 Redis 内部的物理编码（int/embstr/listpack/skiplist 等）。同一个 type 在不同数据规模下会用不同 encoding。例如 Hash：小数据时用 listpack，数据量大时转 hashtable。

**Q2：dict 的渐进式 rehash 怎么做的？**

维护 ht[0] 和 ht[1] 两个哈希表。触发扩容后，每次命令执行顺带迁移 1 个 bucket。rehash 期间：读先旧后新，写只进新表。核心价值：把 O(N) 一次性迁移拆成 N 次 O(1) 小迁移，避免阻塞。

**Q3：ZSet 为什么用 skiplist + dict 双结构？**

dict 负责 O(1) 查 score（ZSCORE），skiplist 负责 O(logN) 范围查询（ZRANGE/ZRANK）。只用其中之一都无法同时高效支持两类操作。member 通过指针共享，不额外复制数据。

**Q4：为什么跳表的晋升概率是 0.25 而不是 0.5？**

P=0.5 时每节点平均 2 个指针（等比数列求和），P=0.25 时约 1.33 个指针，省 33% 内存。Redis 是内存数据库，每字节都计。

**Q5：SDS 比 C 字符串好在哪里？**

O(1) 获取长度（读 len 字段）、二进制安全（以 len 而非 `\0` 判结尾）、空间预分配（减少 realloc）、惰性释放（避免反复分配）。5 种 header 按长度选最小类型。

**Q6：listpack 为什么替代 ziplist？**

ziplist 记录"前一个 entry 长度"，插入/删除可能触发连锁更新（cascading update），最坏 O(N²)。listpack 改为记录"当前 entry 长度"，从后往前解析，彻底消除连锁更新。

## 六、总结（速记版）

- **redisObject**：16B 对象头，type(4b) + encoding(4b) + lru(24b) + refcount + ptr
- **SDS**：O(1) 长度 + 二进制安全 + 预分配 + 5 种 header → int/embstr/raw 三种编码
- **dict**：双表 + 渐进式 rehash + 写进新表 + 读先旧后新 + BGSAVE 时阈值提高到 5
- **skiplist**：P=0.25 随机层高 + 64 层 + span 算 rank + dict 双结构服务 ZSet
- **设计哲学**：小数据紧凑省内存（listpack/intset/embstr），大数据查询结构保性能

---

# 三、IO 模型与网络层

## 一、背景（为什么需要）

单线程 Redis 需要管理成千上万个并发 TCP 连接。传统的 BIO（一个连接一个线程）和 NIO（轮询）都无法胜任。IO 多路复用是单线程高并发的标准答案。

## 二、核心概念

### IO 多路复用演进

| 特性 | select | poll | epoll |
|------|--------|------|-------|
| fd 上限 | 1024（默认） | 无限制 | 无限制 |
| 内核行为 | 每次遍历全部 fd | 每次遍历全部 fd | **只返回就绪 fd** |
| 用户行为 | 拷贝全部 + 遍历 | 拷贝全部 + 遍历 | **直接拿就绪列表** |
| 注册机制 | 每次重新设置 | 每次重新设置 | **一次注册，永久有效** |
| 时间复杂度 | O(N) | O(N) | **O(1)** |

## 三、底层原理

### epoll 原理三步

```
1. epoll_create → 内核创建 eventpoll 对象（红黑树 + 就绪链表）
2. epoll_ctl → fd 加入红黑树 + 注册回调（fd 就绪时自动加入就绪链表）
3. epoll_wait → 直接返回就绪链表中的 fd → O(1)
```

**ET（边缘触发）vs LT（水平触发）**：Redis 用 ET——fd 就绪只通知一次，必须循环读到 EAGAIN 为止，避免同一 fd 的重复事件通知。Nginx 同样用 ET：事件驱动 + 非阻塞 I/O 是高性能网络服务的标准范式。

### RESP 协议（REdis Serialization Protocol）

极简文本协议，解析成本极低：

```
客户端发送：  *3\r\n$3\r\nSET\r\n$4\r\nname\r\n$3\r\nTom\r\n
服务端返回：  +OK\r\n（简单字符串）
             :1\r\n（整数）
             $5\r\nhello\r\n（批量字符串）
             *2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n（数组）
             -ERR unknown command\r\n（错误）
```

**RESP2 vs RESP3（Redis 6.0+）**：
- RESP2：所有返回值是以上 5 种类型之一
- RESP3：新增了 Map、Set、Boolean、Double、Big number、Push 等类型
- RESP3 的核心改进：支持服务端**推送**（Push 类型），不需要客户端先发请求

## 四、关键设计

### 为什么 Redis 不用 HTTP 协议？

HTTP 协议头开销大（每请求数百字节）、解析复杂、不支持 Pipeline（需要 HTTP/2）。RESP 协议每个请求开销仅几个字节，解析只需简单的字符匹配，适合 Redis 的 μs 级操作。

### RESP3 为什么引入 Push 类型？

传统的 RESP2 是"请求-响应"模型，客户端必须先发请求。RESP3 的 Push 类型允许服务端主动推送消息——这对 Pub/Sub 场景和客户端缓存失效（server-assisted client-side caching）至关重要。客户端收到 Push 消息后可以主动失效本地缓存。

## 五、面试高频问题

**Q1：epoll 为什么比 select/poll 快？**

select/poll 每次调用都要把全部 fd 集合从用户态拷到内核态，内核遍历全部 fd 找就绪的，O(N)。epoll 用红黑树一次性注册 fd + 回调机制，fd 就绪时自动加入就绪链表，epoll_wait 直接返回就绪列表，O(1)。在高并发（数万连接）下差距巨大。

**Q2：Redis 为什么用 ET 模式？**

ET（边缘触发）在 fd 就绪时只通知一次，强制应用循环读写直到 EAGAIN。好处是避免同一个 fd 的重复事件通知，减少系统调用次数。代价是应用层代码需要非阻塞循环读写。

**Q3：Redis 6.0 IO 线程和 epoll 是什么关系？**

epoll 负责检测哪些 fd 可读写，IO 线程负责并行执行 read()/write() 和 RESP 解析。二者协作：主线程 epoll_wait → 分配就绪 fd 到 IO 线程 → IO 线程并行读写 → 主线程串行执行命令。

**Q4：Redis 支持并发操作吗？**

支持。但需要区分两个层面：

**连接层面——并发**：epoll + 6.0 IO 线程可同时处理数万 TCP 连接，多客户端可以同时发送请求，网络读写是并行的。

**执行层面——串行**：所有命令在主线程排队串行执行。这意味着：
- 单个命令天然原子——不会有另一个命令插在它中间修改同一个 key
- 多个客户端并发发送的命令，服务端按到达顺序依次执行
- 不存在多线程的数据竞争问题（dict/skiplist 等结构无需加锁）

**多步操作需要原子性怎么办？** 用 Lua 脚本（整个脚本作为一个命令执行）或 MULTI/EXEC（命令之间不插入其他命令）。二者在执行期间都不会被其他客户端命令打断。

> 简单记忆：Redis 是"并发连接 + 串行执行"。并发接收请求，排队执行命令。

## 六、总结（速记版）

- **epoll 三步**：epoll_create（红黑树+就绪链表）→ epoll_ctl（注册回调）→ epoll_wait（O(1)）
- **ET 模式**：fd 就绪只通知一次，循环读到 EAGAIN
- **RESP**：极简文本协议，5 种数据类型，解析成本极低
- **Pipeline**：N 命令 = 1 次 RTT，非原子（区别于 MULTI/EXEC）
- **RESP3**：支持 Push 类型（服务端推送）+ 更多数据类型
- **并发模型**：并发连接（epoll + IO 线程）+ 串行执行（单线程），Lua/MULTI 保证多步原子性


---

# 四、过期机制与内存淘汰

## 一、背景（为什么需要）

Redis 是内存数据库，内存有限。需要一套机制决定：哪些 key 该被删除（过期），以及内存满后删什么（淘汰）。这两个机制共同决定了 Redis 的内存水位和可用性。

## 二、核心概念

### 过期时间存储

```
redisDb 中有两个 dict：
  dict       ← 存所有 key-value 数据
  expires    ← 存过期 key → 到期时间戳（毫秒）

EXPIRE key 60 的本质：在 expires 中插入 expires[key] = now_ms + 60000
```

value 本身不带 TTL，TTL 独立存储在 expires 字典中。

### 8 种内存淘汰策略

```
noeviction（默认）     → 不淘汰，写操作返回错误

volatile-lru           → 有过期时间的 key 中，淘汰最近最少使用
volatile-lfu           → 有过期时间的 key 中，淘汰使用频率最低（4.0+）
volatile-random        → 有过期时间的 key 中，随机淘汰
volatile-ttl           → 有过期时间的 key 中，淘汰剩余 TTL 最短的

allkeys-lru（常用）    → 所有 key 中，淘汰最近最少使用
allkeys-lfu（常用）    → 所有 key 中，淘汰使用频率最低
allkeys-random         → 所有 key 中，随机淘汰
```

生产建议：纯缓存场景用 `allkeys-lru` 或 `allkeys-lfu`；部分 key 需永久保留用 `volatile-lru`。

## 三、底层原理

### 过期删除策略：惰性删除 + 定期删除（组合策略）

**为什么不用定时删除？**
```
定时删除 = 每个有 TTL 的 key 都设一个定时器，到期立刻删除。

问题：
  1. 百万 key = 百万定时器，内存开销巨大
  2. 大量 key 同时过期 → CPU 全在处理定时器回调 → 正常命令被阻塞
  3. 定时器精度和维护成本在单线程模型下不可接受

结论：CPU 开销太大。"CPU 换内存"——宁可少量内存被过期 key 多占一会。
```

**惰性删除（Lazy Expire）**：

每次访问 key 时检查：
```
1. 检查 expires 字典 → key 是否有 TTL？
2. 如果有 → expireAt < 当前时间？
3. 过期 → 删除 key + 删除 expires 中的记录 → 返回 nil
4. 未过期 → 正常返回 value

优点：CPU 友好，不额外消耗 CPU
缺点：冷 key 过期后从未被访问 → 永远占用内存（内存泄漏！）
```

**定期删除（Active Expire）**：

serverCron 每 100ms 执行一次（每秒 10 次）：
```
1. 从 expires 字典中随机抽取 20 个 key
2. 删除其中已过期的 key
3. 如果过期比例 > 25%（20 个中 > 5 个过期）
   → 说明过期 key 浓度高 → 继续循环
4. 否则 → 停止本轮
5. 超时保护：每轮最多执行 25ms → 避免占用过多 CPU
```

**为什么过期比例 > 25% 继续？** 如果过期比例低（如 1%），说明大部分 key 没过期，继续扫描的收益很小。25% 是经验阈值，在"多清理"和"少浪费 CPU"之间取得平衡。

**为什么最多 25ms？** 单线程模型下，过期扫描无时间限制会导致大量 key 同时过期时扫描数百 ms，所有客户端请求被阻塞。25ms 限制保证 CPU 时间让给正常命令。

### 近似 LRU（不是精确 LRU！）

精确 LRU 需要双向链表维护访问顺序（每次 GET/SET 移动节点到链表头），百万级 key 维护成本太高。

**Redis 的近似 LRU**：
```
每次淘汰时：
  1. 随机采样 maxmemory-samples 个 key（默认 5 个）
  2. 从这 5 个中找出 lru 时间戳最小的（最久未使用）
  3. 删除这个 key

采样越大越精确：
  samples=5：  精确度 ~70%
  samples=10： 精确度 ~90%
```

Redis 3.0 后维护了淘汰候选池（eviction pool），进一步提高近似精度。

### LFU 实现（Redis 4.0+）

LFU 解决 LRU 的痛点：一个 key 被访问一次和一万次在 LRU 中没区别。LFU 利用 redisObject 的 24 位 lru 字段：

```
LRU 模式：24 位存秒级时间戳（最后访问时间）
LFU 模式：
  高 16 位：最后衰减时间（Last Decrement Time，分钟级）
  低 8 位：对数计数器（Logarithmic Counter，0-255）

对数计数器：
  counter=0 时每次访问都 +1
  counter 越大，递增概率越低（不会轻易到 255）
  → 少量访问 counter 小，大量访问 counter 逐渐增大但有上限

衰减机制：
  随时间流逝 counter 自动减小
  → 曾经热但最近不热的 key 能被淘汰
```

## 四、关键设计

### 为什么是"惰性 + 定期"而不是"定时"？

定时删除是"精确但昂贵"的方案。Redis 选择了"不精确但便宜"的方案——用少量内存换取 CPU 稳定性。这是 Redis 设计哲学的核心体现：**在内存和 CPU 之间做 trade-off，通常选择 CPU 友好的方案**。

### 为什么不实现精确 LRU？

精确 LRU 的维护成本（全局链表 + 每次访问移动节点）在 Redis 的 μs 级延迟要求下不可接受。近似 LRU 通过随机采样 + 淘汰候选池，以很小的精度损失（默认 ~30%）换取了几乎零维护成本。

## 五、面试高频问题

**Q1：key 的过期删除策略是什么？**

惰性删除 + 定期删除组合。惰性删除在每次访问时检查，过期就删；定期删除每 100ms 随机抽取 20 个 key，过期比例 > 25% 继续扫描，单次最多 25ms。为什么不用定时删除？百万 key 需要百万定时器，大量同时过期时 CPU 阻塞。

**Q2：key 过期后真的被立即删除了吗？**

不一定。可能同时逃过定期删除（没被随机抽到）和惰性删除（没被访问），过期后仍占内存一段时间。这是"牺牲少量内存换 CPU 稳定性"的设计。

**Q3：Redis 的 LRU 是精确的吗？**

不是。Redis 用近似 LRU：随机采样 maxmemory-samples 个 key（默认 5 个），淘汰其中最久未使用的。精度约 70%，samples=10 可到 90%。省去了精确 LRU 需要的全局链表维护。

**Q4：LFU 比 LRU 好在哪里？**

LRU 只看"最近有没有被访问"（时间维度），无法区分"频繁访问"和"偶尔访问"。LFU 用 8 位对数计数器 + 时间衰减机制，同时关注频率和时间，能更准确识别真正的热 key。

## 六、总结（速记版）

- **过期策略**：惰性（访问时删）+ 定期（每 100ms 随机 20 个，>25% 继续，≤25ms）
- **淘汰策略**：8 种，常用 allkeys-lru/lfu
- **近似 LRU**：随机采样淘汰，默认 5 个约 70% 精度，不维护全局链表
- **LFU**：8 位对数计数器 + 时间衰减，区分"频繁"和"偶尔"

---

# 五、持久化

## 一、背景（为什么需要）

内存数据易失。重启、宕机、掉电 → 数据全丢。持久化是将内存数据保存到磁盘，实现可恢复。三种方式各有 trade-off。

## 二、核心概念

| 方式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **RDB** | 某时刻内存快照 → 二进制文件 | 文件紧凑、恢复快 | 丢最后快照后的数据 |
| **AOF** | 追加每条写命令 → 文本日志 | 最多丢 1 秒 | 文件大、恢复慢 |
| **混合持久化** | RDB 头 + AOF 增量 | 恢复快 + 数据完整 | 文件略大 |

## 三、底层原理

### RDB：fork + COW（写时复制）

**BGSAVE 执行流程**（生产只用 BGSAVE，禁用 SAVE）：
```
1. 主线程 fork() 创建子进程
2. fork 后父子进程共享同一份物理内存（只读）
3. 子进程遍历内存数据，写入 RDB 文件
4. 父进程（Redis 主线程）继续处理客户端请求：
   ├── 读操作：直接读共享内存 → 不触发 COW
   └── 写操作：OS 复制被修改的内存页（COW）→ 父进程修改新页
5. 子进程完成 → 原子 rename 替换旧 RDB → 退出
```

**COW（Copy-On-Write）本质**：
- fork 后不立即复制内存，父子共享同一物理页
- 父进程修改某页时，OS 才复制该页（通常 4KB）
- 父进程改新页，子进程读旧页（fork 时刻的快照数据）

**RDB 的代价**：
- 两次快照之间的数据可能丢失（取决于 save 配置）
- fork 时如果内存很大（如 32GB），页表复制可能耗时百 ms
- 写密集时大量 COW → 内存可能达 2 倍

### AOF：命令日志

**三种刷盘策略**：
```
appendfsync always    → 每次写都 fsync → 最安全、最慢
appendfsync everysec  → 每秒 fsync 一次 → 最多丢 1 秒（生产推荐）
appendfsync no        → OS 决定何时刷 → 性能最好、最不安全
```

**AOF Rewrite（重写）——不阻塞主线程！**

原始 AOF 随时间无限增长。重写用当前内存数据生成最简命令集：
```
原始 AOF（多次 INCR counter）：
  SET counter 0, INCR counter, INCR counter, INCR counter, INCR counter, INCR counter

重写后：SET counter 5   ← 一条等效命令替代多条历史命令
```

**Rewrite 流程**：
```
1. fork 子进程
2. 子进程根据当前内存数据生成最简命令 → 写入新 AOF 文件
3. 父进程继续服务 + 同时将写命令写入旧 AOF + AOF 重写缓冲
4. 子进程完成后，父进程将 AOF 重写缓冲追加到新 AOF 末尾
5. 原子 rename 新文件替换旧文件
```

### 混合持久化（Redis 4.0+，生产推荐）

```
文件格式：
┌──────────────────────┐
│ RDB 二进制快照数据    │ ← 加载极快
├──────────────────────┤
│ Rewrite 期间的 AOF 增量 │ ← 数据完整（最多丢 1 秒）
└──────────────────────┘

配置：aof-use-rdb-preamble yes
```

混合持久化 = RDB 的恢复速度 + AOF 的数据完整性。生产环境标配。

## 四、关键设计

### 为什么 BGSAVE 用 fork 而不是多线程？

POSIX fork 的 COW 机制天然提供了"零成本快照"：fork 瞬间父子共享内存，后续只在写时复制。如果用多线程：需要全局锁冻结所有数据 → 复制 → 解锁，实现复杂且阻塞时间长。

### 为什么 AOF Rewrite 需要 AOF 重写缓冲？

子进程 rewrite 期间父进程仍在处理写命令。如果不记录这段期间的写命令 → 新 AOF 文件缺少 rewrite 期间的增量数据 → 数据不完整。AOF 重写缓冲解决了"fork 快照和 rewrite 完成的时差"问题。

### 为什么混合持久化不是默认开启的？

历史原因（4.0 才引入）和兼容性考虑。但生产环境强烈建议开启。

## 五、面试高频问题

**Q1：RDB 和 AOF 区别？怎么选？**

RDB 是快照（fork+COW），文件小恢复快，但丢最后快照后的数据。AOF 是命令日志，everysec 最多丢 1 秒，但有 rewrite 可压缩。生产推荐混合持久化：RDB 头 + AOF 增量，兼顾两者优势。

**Q2：BGSAVE 过程中主线程能处理写请求吗？**

能。fork 后父子共享内存，父进程写操作触发 COW——被修改的内存页被 OS 复制，父进程修改新页，子进程读旧页。子进程看到的一直是 fork 时刻的数据快照，不受父进程写操作影响。

**Q3：fork 操作为什么会阻塞？**

fork 本身不复制数据（COW），但需要复制页表。如果内存很大（如 32GB），页表条目数多，复制可能耗时数十到数百 ms。这期间主线程是阻塞的。减少单个实例内存、使用物理机（非容器）可缓解。

**Q4：AOF Rewrite 期间主线程会阻塞吗？**

不会。fork 子进程生成新 AOF 期间，主线程继续服务。子进程完成后，主线程追加 AOF 重写缓冲并 rename，这两个操作很快（ms 级），几乎无感知。

## 六、总结（速记版）

- **RDB**：fork + COW 快照，文件紧凑恢复快，丢最后快照后数据
- **AOF**：追加命令，everysec 丢 1 秒，rewrite 压缩（不阻塞）
- **混合（4.0+）**：RDB 头 + AOF 增量，恢复快 + 数据完整 = 生产首选
- **COW**：fork 不复制内存，写时才复制被修改的页

---

# 六、高可用架构

## 一、背景（为什么需要）

单机 Redis 三个问题：
1. 容量有限（单机内存上限）
2. 并发有限（单机 QPS ~10 万）
3. 单点故障（宕机 → 服务不可用）

高可用体系分三层：主从复制（数据冗余 + 读写分离）→ Sentinel（自动故障转移）→ Cluster（数据分片 + 水平扩展）。

## 二、核心概念

### 主从复制

**全量同步（FULL RESYNC）——首次连接**：
```
1. Slave → PSYNC ? -1（首次，不知道 Master runid 和 offset）
2. Master 返回 FULLRESYNC <runid> <offset>
3. Master 执行 BGSAVE 生成 RDB（同时新写命令暂存到 replication buffer）
4. Master 发送 RDB 给 Slave
5. Slave 清空自身旧数据，加载 RDB
6. Master 发送 replication buffer 中的增量命令 → Slave 执行追上
```

**增量同步（Partial Resync）——断线重连**：
```
1. Slave → PSYNC <runid> <offset>（带着之前记住的 Master ID 和复制位置）
2. Master 检查：runid 匹配？offset 在 repl_backlog_buffer 范围内？
3. 是 → 只发送 offset 之后的命令（增量同步）
4. 否 → 退化为全量同步
```

**repl_backlog_buffer**：Master 维护的环形缓冲区（默认 1MB），记录最近写命令。大小决定 Slave 断线多久还能增量同步。建议生产环境调大（如 64MB）。

### Sentinel 哨兵

**四大职责**：监控（PING）→ 通知（Pub/Sub）→ 故障转移（Failover）→ 配置提供者（客户端获取 Master 地址）。

**SDOWN vs ODOWN**：
- SDOWN（主观下线）：单个哨兵认为 Master 不可用（PING 超时 > down-after-milliseconds）
- ODOWN（客观下线）：≥ quorum 个哨兵一致认为 Master 不可用 → 才触发故障转移

为什么两步？单哨兵可能因自身网络问题误判。多哨兵一致降低误判概率。

**故障转移流程**：
```
1. 判 ODOWN → quorum 个哨兵都认为 Master 下线
2. 选 Leader Sentinel → 哨兵间用 Raft 协议选举 Leader
3. 选新 Master → 从 Slave 中按优先级：slave-priority > offset > runid
4. Leader 向新 Master 发 SLAVEOF NO ONE（提升）
5. Leader 通知其他 Slave 复制新 Master
6. 发布通知 → 客户端获取新 Master 地址
```

### Redis Cluster

**16384 个 Hash Slot**：
```
slot = CRC16(key) % 16384

为什么是 16384？
  1. 16384 = 2^14 → 心跳包用位图表示 slot 分布 = 16384 bits = 2KB
     如果用 65536(2^16) → 8KB → 心跳包太大
  2. 集群节点数通常 ≤ 1000 → 16384 足够均匀分配
  3. 2KB 心跳包网络传输性价比最高
```

**请求路由：MOVED vs ASK**：
- MOVED：slot 在其他节点 → 永久重定向 + 客户端更新 slot 表
- ASK：slot 正在迁移 → 临时重定向（只本次，下次还会问原节点）

**Gossip 协议**：节点间定期（每秒）随机 PING 几个节点，交换自身信息 + 已知节点状态。去中心化：任何节点都知道集群完整拓扑。

**Hash Tag**：`{user:1001}:name` 和 `{user:1001}:age` → 强制同 slot → 支持跨 key 操作（MGET/MSET/事务/Lua）。

## 三、底层原理

### 主从复制为什么是异步的？

```
Master 写成功 → 立即返回客户端 "OK"
              → 异步发送给 Slave
```

风险：Master 返回 OK 后宕机 → 该命令未同步到 Slave → 数据丢失。

**缓解**：
```
min-replicas-to-write 1     # 至少 1 个 Slave 在线才允许写
min-replicas-max-lag 10     # Slave 复制延迟不超过 10 秒
→ 牺牲可用性换数据安全性
```

### Cluster 脑裂问题

**场景**：网络分区 → 旧 Master 被隔离但没宕机 → Client 继续向旧 Master 写入 → 新 Master 被选出 → 网络恢复后旧 Master 降级为 Slave → 清空数据同步新 Master → 分区期间写入的数据丢失。

**缓解**：同主从复制——`min-replicas-to-write` 和 `min-replicas-max-lag`。如果 Master 感知不到足够 Slave → 拒绝写入。

### Cluster 故障转移

```
Master 故障检测：
  1. PING 超时 → PFAIL（疑似下线）
  2. Gossip 传播 PFAIL → 超半数主节点确认 → FAIL（客观下线）

Failover：
  1. 故障 Master 的 Slave 发起选举
  2. 向其他 Master 请求投票 → 获过半数票 → 提升为新 Master
  3. 接管故障 Master 的 slot
```

## 四、关键设计

### 为什么 Sentinel 是独立进程而不是内嵌在 Redis 里？

职责分离。哨兵需要独立视角判断 Redis 是否存活。如果哨兵和 Redis 在同一进程，Redis 卡住时哨兵也卡住了——无法判断是进程死了还是只是暂时繁忙。独立进程 + 多个哨兵 = 可信的故障判断。

### 为什么 Cluster 用 Gossip 而不是中心化元数据？

中心化（如 etcd/Consul）需要单独的元数据集群，增加运维成本。Gossip 去中心化：任何节点都能回答 slot 分布，无单点，节点增减自动感知。代价是最终一致性（有短暂窗口 slot 信息不一致），但对 Redis 场景可以接受。

## 五、面试高频问题

**Q1：主从复制全量和增量有什么区别？**

全量同步（FULL RESYNC）：首次连接或 runid/offset 不匹配时，Master 用 BGSAVE 发 RDB + replication buffer 增量。增量同步（Partial Resync）：断线重连时，如 Slave 的 offset 在 repl_backlog_buffer 范围内，只发 offset 之后的命令。

**Q2：哨兵如何判断 Master 挂了？**

SDOWN（主观下线）：单哨兵 PING 超时。ODOWN（客观下线）：≥ quorum 个哨兵都认为挂了。两步判断防止单哨兵网络误判。

**Q3：哨兵之间如何选举 Leader？**

Raft 协议。率先发现 ODOWN 的哨兵向其他哨兵请求投票，获过半数票（≥ N/2+1）的成为 Leader 执行故障转移。这就是哨兵需要奇数个（3/5）的原因。

**Q4：Cluster 中客户端怎么知道 key 在哪个节点？**

客户端连接任意节点发送请求。如果 key 的 slot 在其他节点 → 返回 MOVED <slot> <IP:Port> → 客户端更新本地 slot 映射表 → 向目标节点重发。客户端内置了 slot → node 缓存。

**Q5：Hash Tag 是用来干什么的？**

`{ }` 包裹 key 的一部分，只有 `{ }` 内的部分参与 CRC16 计算，强制多个 key 映射到同一个 slot。用途：跨 key 操作（MGET/MSET）、事务（MULTI/EXEC）、Lua 脚本。

**Q6：脑裂问题怎么解决？**

网络分区导致旧 Master 被隔离但没宕机，Client 继续写入旧 Master → 网络恢复后数据丢失。缓解方案：`min-replicas-to-write` + `min-replicas-max-lag`，Master 感知不到足够 Slave → 拒绝写入 → 牺牲可用性换一致性。

## 六、总结（速记版）

- **主从复制**：全量（RDB+buffer）+ 增量（PSYNC+backlog），异步复制，可能丢数据
- **Sentinel**：SDOWN → ODOWN(≥quorum) → Raft 选 Leader → 选新 Master（优先级 > offset > runid）
- **Cluster**：16384 slot（CRC16 + %），MOVED（永久）vs ASK（临时），Gossip 通信，Hash Tag 强制同 slot
- **脑裂缓解**：min-replicas-to-write + min-replicas-max-lag

---

# 七、分布式锁

## 一、背景（为什么需要）

分布式系统中，多实例竞争同一资源（如库存扣减、幂等处理），需要一把跨进程的锁。Redis 是最常用的分布式锁实现。

## 二、核心概念

### 基础实现

```redis
SET lock_key unique_value NX PX 30000
```

- NX：key 不存在才设置（互斥 = 拿到锁）
- PX 30000：30 秒后过期（防死锁）
- unique_value：每个客户端生成唯一 ID（UUID），用于释放时验证身份

## 三、底层原理

### 释放锁为什么需要 Lua 脚本

**错误做法——直接 DEL**：
```
线程A 拿到锁 → 业务执行超时 → 锁自动过期
→ 线程B 拿到锁 → 线程A 业务完成，执行 DEL
→ DEL 删除的是线程B 的锁！← 误删！
```

**正确做法——Lua 原子释放**：
```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

先验证 value 是否匹配（锁是不是自己加的），匹配才删除。Lua 保证 GET + DEL 的原子性。

### Redisson Watchdog 自动续期

**问题**：业务执行时间 > 锁过期时间 → 锁自动释放 → 其他线程趁虚而入。

**Watchdog 方案**（Redisson 内置实现）：
```
1. 加锁时默认过期 30 秒
2. 启动 watchdog 后台线程（基于 Netty 定时器）
3. 每 10 秒检查：业务还在执行？
   ├── 是 → 续期为 30 秒
   └── 否 → 停止续期
4. 业务完成 → 主动释放锁 → watchdog 停止
5. 进程崩溃 → watchdog 也停止 → 锁自然过期
```

> 好处：锁的生命周期跟随业务执行时间，只在进程崩溃时才依赖过期时间兜底。

### Redisson 可重入锁原理

Redisson 的可重入锁基于 **Redis Hash + 计数器** 实现：

```
加锁时：
  lock_key → Hash {
    field: 客户端 UUID:线程 ID
    value: 重入次数（第一次=1，同线程重复加锁时 +1）
  }

释放时：
  value - 1 → 如果 value > 0 → 继续持有锁（重入场景）
            → 如果 value = 0 → 删除 lock_key（真正释放）

底层：一段 Lua 脚本保证原子性检查 + 更新 + 续期
```

> 对比 SET NX 的简单实现：SET NX 不支持可重入——同一线程再次加锁会被 NX 拒绝。Redisson Hash + 计数器天然支持重入计数。

### Redlock 算法（多节点）

**为什么需要 Redlock？** 单节点 Redis 故障 → 所有锁丢失。

**Redlock 流程**（N 个独立 Redis 节点，通常 5 个）：
```
1. 获取当前时间 T1
2. 依次向 N 个节点申请锁（相同 key + unique_value + 过期时间）
3. 获取当前时间 T2，耗时 = T2 - T1
4. 成功条件：≥ N/2+1 个节点加锁成功 且 耗时 < 过期时间
5. 锁有效时间 = 过期时间 - 耗时
6. 失败 → 向所有节点释放锁
```

**Redlock 争议**：Martin Kleppmann 认为在时钟漂移和 GC STW 场景下不安全。业界共识：Redis 分布式锁适合"效率"场景（如防止重复计算），不适合"正确性"场景（如金融扣款）。后者用 ZooKeeper。

## 四、关键设计

### 为什么 SET NX PX 是一个命令而不是两个？

Redis 2.6 之前需先 SETNX 再 EXPIRE——两个命令不原子。如果 SETNX 后客户端崩溃，EXPIRE 未执行 → 锁永远不会释放（死锁）。SET NX PX 将两个操作合并为一个原子命令。

### 为什么释放锁需要 Lua？

GET + 比较 + DEL 是三个 Redis 命令，非原子。如果 GET 返回匹配但 DEL 前锁恰好过期并被其他客户端获取 → 误删。Lua 脚本在 Redis 中原子执行——整个脚本作为一个命令，不会被其他命令插入。

### Redlock 为什么有争议？

Redlock 假设节点独立、网络延迟可预测、时钟同步。但现实中的 GC STW、时钟漂移、网络分区打破了这些假设。这就是 Martin Kleppmann 论证 Redlock 不安全的根本原因——它试图用不可靠的基础组件构建可靠系统。

## 五、面试高频问题

**Q1：Redis 分布式锁怎么实现？**

SET key uuid NX PX 30000。NX 互斥 + PX 防死锁 + uuid 验证身份。释放用 Lua 脚本（GET 验证后 DEL）防误删。长时间任务用 Redisson Watchdog 每 10 秒自动续期。

**Q2：释放锁为什么用 Lua？**

GET + 比较 + DEL 三步需要原子性。不用 Lua 的话：A 的锁过期 → B 拿到锁 → A 的 DEL 误删了 B 的锁。Lua 保证"检查是自己加锁 + 删除"的原子性。

**Q3：锁过期但业务没执行完怎么办？**

Redisson Watchdog 自动续期。后台线程每 10 秒检查业务是否还在执行，是则重置过期时间为 30 秒。业务完成主动释放，进程崩溃则 watchdog 也停止，锁自然过期。

**Q4：Redlock 算法原理？争议是什么？**

向 N 个独立 Redis 节点依次加锁，≥ N/2+1 成功才算获得锁。争议在于时钟漂移和 GC STW 场景下无法保证安全性。业界共识：Redis 分布式锁适合"效率"场景，一致性强需求用 ZK。

**Q5：Redisson 可重入锁怎么实现？**

基于 Redis Hash + 计数器。lock_key 是一个 Hash，field=客户端 UUID:线程 ID，value=重入次数。加锁时 value+1（首次=1），释放时 value-1，value=0 时真正删除锁。

## 六、总结（速记版）

- **基础**：SET NX PX 原子命令，uuid 验证身份
- **释放**：Lua 原子 GET+DEL，防误删
- **续期**：Redisson Watchdog，每 10s 续到 30s
- **可重入**：Hash + 计数器，field=客户端:线程ID，value=重入次数
- **多节点**：Redlock（过半成功），但有时钟漂移争议
- **选型**：Redis 适合"效率"场景，强一致用 ZK

---

# 八、缓存设计

## 一、背景（为什么需要）

缓存是把"频繁访问的热点数据"放在离计算更近的地方。Redis 做缓存的核心挑战：如何在内存有限的前提下，保证缓存和数据库之间的一致性，同时防止缓存被穿透、击穿、雪崩。

## 二、核心概念

### 缓存三大问题

| 问题 | 特征 | 一句话方案 |
|------|------|-----------|
| **穿透** | key 根本不存在（DB 也没有） | 布隆过滤 / 缓存空值 / 参数校验 |
| **击穿** | 单个热 key 恰好过期 | 互斥锁重建 / 逻辑过期 |
| **雪崩** | 大量 key 同时过期或 Redis 宕机 | 随机 TTL / 高可用 / 多级缓存 |

### Cache Aside 模式（最常用）

```
读：
  cache miss → 查 DB → 写缓存 → 返回

写：
  更新 DB → 删除缓存（不是更新缓存！）
```

**为什么删缓存而不是更新缓存？** 并发更新场景下，线程 A 和 B 的写入顺序不可控。删除缓存后，下一次读自然会重建正确的值。

## 三、底层原理

### 缓存穿透防御

**布隆过滤器原理**：
```
Bit 数组 + k 个 Hash 函数：
  初始化：所有存在的 key → k 次 hash → 对应 bit 位置 1
  查询：待查 key → k 次 hash → 检查对应 bit 位
    任何一位=0 → 一定不存在（100% 准确，直接拒绝）
    全部=1 → 可能存在（允许通过，需查 DB 确认）

特点：无漏判（一定不存在能准确判断）、有误判（可能存在实际不存在）、不支持删除
```

RedisBloom 模块：`BF.ADD key item` → `BF.EXISTS key item`。

**缓存空值**：DB 查不到 → Redis 缓存 null（短 TTL 如 5 分钟）→ 下次请求直接返回 null。缺点：攻击者用不同 key 发起攻击 → Redis 中大量 null → 占用内存。

### 缓存一致性极低概率脏数据

```
1. 线程 A 读缓存（miss）→ 查 DB（旧值 v1）
2. 线程 B 写 DB（新值 v2）→ 删缓存
3. 线程 A 将旧值 v1 写入缓存 ← 脏数据！

概率极低但不为零：线程 A 在步骤 1-3 之间，线程 B 恰好完成步骤 2。
```

**解决方案梯度**：

| 方案 | 做法 | 复杂度 | 可靠性 |
|------|------|--------|--------|
| 延迟双删 | 删缓存 → 写 DB → sleep(500ms) → 再删缓存 | 低 | 中 |
| MQ 异步重试 | 删缓存失败 → MQ → 消费者重试删除 | 中 | 高 |
| Canal + binlog（大厂常用） | Canal 监听 MySQL binlog → 解析变更 → MQ → 消费者异步删 Redis | 高 | 最高 |

**Canal 方案**：
```
MySQL binlog → Canal 伪装的 Slave 监听 → 解析变更
→ 发 MQ → 消费者异步更新/删除 Redis
→ 业务代码零侵入，异步解耦
→ 有秒级延迟（最终一致性）
```

## 四、关键设计

### 为什么接受最终一致性而不是强一致性？

分布式系统中，强一致性的代价是性能和可用性。CAP 理论告诉我们三者不可兼得。缓存场景下，99% 的业务可以接受短暂不一致（如商品详情缓存落后几秒），不值得为了一致性牺牲响应时间。Canal 方案用秒级延迟换零侵入和极高可靠性，是业界最佳实践。

### BigKey 与 HotKey

**BigKey**：String > 10KB，集合 > 5000 元素或 > 10MB。
- 危害：读写阻塞主线程（DEL O(N)）、集群内存倾斜、主从复制延迟
- 发现：`redis-cli --bigkeys`（SCAN + 类型检测）
- 解决：拆分大 key 为小 key + UNLINK 异步删除 + 压缩 + HSCAN/SSCAN 分页读取

**HotKey**：短时间内被大量请求访问的 key。
- 危害：单节点 CPU 打满、热 key 过期 → 缓存击穿、带宽瓶颈
- 发现：`redis-cli --hotkeys`（需 LFU 淘汰策略）
- 解决：本地缓存（最有效） + 读写分离 + key 复制（hot_key_1/2/3 随机选）

## 五、面试高频问题

**Q1：缓存穿透怎么解决？**

布隆过滤器 + 缓存空值 + 参数校验。布隆过滤器用少量内存挡住绝大部分无效 key（无漏判）。缓存空值作为兜底，但短 TTL 防内存浪费。接口层校验 ID 格式在网关层即拦截。

**Q2：缓存击穿怎么解决？**

互斥锁：查 Redis miss → 尝试 SET lock NX PX → 拿到锁的线程查 DB 重建缓存，没拿到的 sleep 后重试。需要双检（拿锁后再查一次 Redis，可能其他线程已重建）。逻辑过期方案：value 里存过期时间，过期后返回旧值 + 异步更新。

**Q3：缓存雪崩怎么解决？**

针对"大量 key 同时过期"：TTL 加随机值（base+random(0,300s)）、热 key 永不过期、分类设置。针对"Redis 宕机"：Sentinel/Cluster 高可用 + 本地缓存兜底 + 限流熔断降级。

**Q4：Cache Aside 为什么写操作是"删缓存"而不是"更新缓存"？**

并发写场景下，两个线程的更新顺序不可控，直接更新缓存可能导致缓存值和 DB 值永久不一致。删除缓存后，下一次读会从 DB 读到最新值再重建缓存，自然保证了最终一致性。

**Q5：BigKey 有什么危害？怎么处理？**

危害：DEL 大 key 阻塞主线程（O(N)）、内存倾斜、主从复制延迟。处理：业务拆分（大 Hash 拆多个小 Hash）、UNLINK 异步删除（4.0+ 后台线程释放）、压缩 value、SCAN 分页读取。

**Q6：本地缓存和 Redis 缓存有什么区别？**

| 维度 | 本地缓存（Caffeine/Guava） | Redis 缓存 |
|------|--------------------------|-----------|
| 存储位置 | 应用进程内（堆内存） | 独立进程（独立内存空间） |
| 访问速度 | **纳秒级**（直接内存访问） | 微秒级（网络 RTT + 序列化） |
| 跨实例共享 | 不支持（各实例各自一份） | **支持**（所有实例共享同一份） |
| 容量 | 受 JVM 堆限制（GB 级共享） | 独立扩展（可达数十/数百 GB） |
| 一致性 | 各实例缓存各自维护，不一致 | 集中存储，所有实例看到同一数据 |
| 高可用 | 进程重启即丢失 | 主从 + Sentinel + Cluster 保证 |
| 数据淘汰 | 应用控制 + GC | 8 种策略 + TTL 精确控制 |

**典型组合——二级缓存**：
```
L1（本地缓存 Caffeine）：
  热点数据、极小 TTL（1-5s）、允许短暂不一致
  → 纳秒级命中，挡住 90%+ 的 Redis 请求

L2（Redis）：
  稍大 TTL、跨实例共享数据、需要高可用
  → 微秒级命中，挡住 90%+ 的 DB 请求
```

> 核心选择逻辑：需要跨实例共享 → Redis；只给自己加速且能容忍不一致 → 本地缓存。高并发场景通常两层都用。

**Q7：高并发场景，Redis 单节点 + MySQL 单节点能支撑多大并发？**

**单节点能力上限（经验值）**：

| 组件 | QPS 上限 | 瓶颈 |
|------|---------|------|
| Redis（GET/SET） | **8w-12w QPS** | 网络带宽（10G 网卡 ~100w，千兆 ~8w） |
| Redis（复杂命令/ZRANGE） | 3w-5w QPS | CPU 单核 |
| MySQL（简单主键查询） | 3k-5k QPS | 磁盘 IO / 连接池 |
| MySQL（复杂查询/关联） | 300-800 QPS | CPU / 磁盘 IO |

**组合后的实际并发（读多写少典型场景）**：

```
假设：读:写 = 9:1，缓存命中率 95%

5% 读 miss → 查 MySQL：
  读 QPS × 5% ≤ MySQL 读上限 → 读 QPS ≤ 5k / 5% = 10w

1% 写 → 写 MySQL：
  写 QPS × 10% ≤ MySQL 写上限 → 写 QPS ≤ 500 / 10% = 5k

实际能支撑 ≈ 5w-8w QPS（总）
瓶颈在 MySQL 写能力（单节点 ~500 写 QPS）
```

**关键经验**：
- 纯缓存读场景（99% 命中率 + 逻辑过期兜底）→ **10w+ QPS**，瓶颈在 Redis 网络
- 读写混合场景 → **3w-5w QPS**，瓶颈在 MySQL 写
- 复杂 SQL / 无索引查询 → **2000-5000 QPS**，MySQL 直接成为瓶颈
- Redis 基本不会成为瓶颈（除非大 key/热 key），瓶颈永远在 DB

> 面试回答模板："Redis 单节点轻松 10w QPS，MySQL 单节点约 3k-5k 查询 QPS。组合后，读多写少场景加上缓存可支撑 5w-8w QPS。真正的瓶颈在 MySQL 而非 Redis。"

## 六、总结（速记版）

- **穿透**（key 不存在）→ 布隆过滤 + 缓存空值 + 参数校验
- **击穿**（热 key 过期）→ 互斥锁重建 + 逻辑过期
- **雪崩**（大量过期/宕机）→ 随机 TTL + 高可用 + 多级缓存
- **一致性**：Cache Aside（读写删）+ 延迟双删 + Canal/binlog 兜底
- **大 Key**：拆分 + UNLINK 异步删；**热 Key**：本地缓存最有效
- **本地缓存 vs Redis**：纳秒 vs 微秒、不共享 vs 共享、L1+L2 组合
- **并发量级**：Redis 10w+，MySQL 3k-5k，组合后 5w-8w QPS，瓶颈在 DB

---

# 九、工程实践

## 一、背景（为什么需要）

面试中除了原理，还会考察"会不会用"——Pipeline 怎么用、事务和 Lua 怎么选、限流怎么实现、线上变慢怎么排查、连接池怎么配。

## 二、核心概念

### Pipeline

把 N 个命令打包一次发送 → N 次 RTT 变成 1 次 RTT。

限制：
- Pipeline 中命令不能依赖前一个命令的结果（都是批量发送的）
- 不是原子操作（不同于 MULTI/EXEC）——中途可能被其他客户端命令插入
- 一次不要太多命令（会占用内存），建议分批，每批控制在合理数量

### 事务 vs Lua

| 对比 | MULTI/EXEC | Lua |
|------|-----------|-----|
| 原子性 | 命令之间不插入其他命令 | 整个脚本期间不插入 |
| 条件判断 | 不支持（先入队后执行，无法 if/else） | 支持 |
| 回滚 | 不支持（运行时错误也不回滚） | 不支持 |
| 使用场景 | 简单批量操作 | 需要条件判断的原子操作 |

**为什么 Redis 事务不支持回滚？** Redis 认为运行时错误是编程错误（不应在生产发生），为了简单和性能放弃回滚。语法错误会在入队时直接拒绝。

## 三、底层原理

### 滑动窗口限流（ZSet + Lua）

```lua
-- KEYS[1]: 限流 key, ARGV[1]: 窗口大小(秒), ARGV[2]: 阈值
local now = tonumber(redis.call('TIME')[1])
local start = now - tonumber(ARGV[1])
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, start)  -- 清除窗口外
local count = redis.call('ZCARD', KEYS[1])           -- 窗口内计数
if count < tonumber(ARGV[2]) then
    redis.call('ZADD', KEYS[1], now, now .. '-' .. math.random())
    return 1  -- 放行
else
    return 0  -- 限流
end
```

### SCAN——非阻塞遍历（生产禁 KEYS！）

```
KEYS *  → O(N) 遍历所有 key，单线程执行 → 阻塞所有客户端 → 生产禁用！
SCAN 0 MATCH user:* COUNT 100 → 返回 [下个游标, [key列表]]

特点：
  - 游标分批，不阻塞主线程（每次只扫描少量 bucket）
  - COUNT 只是建议值，不保证返回数量
  - rehash 期间可能返回重复 key（客户端需去重）
  - 可能遗漏 key（rehash 期间概率极低）
```

变体：SSCAN（Set）、HSCAN（Hash）、ZSCAN（ZSet）。

### 慢查询日志

```
slowlog-log-slower-than 10000  # 超过 10000μs（10ms）记录
slowlog-max-len 128            # 最多 128 条
SLOWLOG GET 10  → 最近 10 条慢查询
```

常见慢查询原因：
- KEYS */FLUSHALL → 生产禁用
- DEL 大 key → 用 UNLINK 异步删除
- HGETALL 大 Hash → 用 HSCAN 分批
- ZRANGE 大 ZSet（0 -1）→ 用 ZSCAN 分批
- 复杂 Lua 脚本 → 拆分优化
- 大量 key 同时过期 → 定期删除循环触发（最多 25ms/次）
- fork 子进程 → 大内存（>32GB）fork 阻塞数十 ms

### 线上 Redis 变慢排查八步

```
1. SLOWLOG GET 10 → 定位慢命令
2. INFO stats → instantaneous_ops_per_sec（QPS 是否异常升高）
3. INFO memory → mem_fragmentation_ratio（碎片率 >1.5 或 <1.0）
4. INFO persistence → rdb_last_bgsave_status（BGSAVE 是否失败或耗时异常）
5. INFO stats → latest_fork_usec（fork 耗时，>1 秒需关注）
6. 检查 swap：used_memory_rss < used_memory → 发生了 swap！
7. redis-cli --latency → 检查网络延迟
8. 检查 CPU：单核跑满？NUMA 跨节点访问？
```

### 内存碎片

```
碎片率 = used_memory_rss (OS 实际分配) / used_memory (Redis 认为使用)

碎片率 ≈ 1.0 → 正常
碎片率 > 1.5 → 碎片严重 → 开启 activedefrag（4.0+）
碎片率 < 1.0 → 发生了 swap！OS 把部分内存换到磁盘 → 性能急剧下降

自动整理（4.0+）：
  activedefrag yes
  active-defrag-threshold-lower 10   # 碎片率 > 1.1 开始
  原理：serverCron 中逐步移动内存块，归并碎片
```

### 连接池配置

```
PoolSize ≈ QPS × avg_latency(秒) × 2（预留峰值）

示例：1000 QPS，平均延迟 2ms（0.002s）
  PoolSize ≈ 1000 × 0.002 × 2 = 4 → 实际设 10-30（考虑 GC 和网络抖动）

关键参数：
  PoolSize：并发连接上限
  PoolTimeout：池满后等多久（建议 3-5s），太短高峰期直接报错
  MinIdleConns：预热连接（PoolSize 的 20-30%），避免冷连接 TCP 握手
  ConnMaxLifetime：定时刷新连接（建议 30m）
```

### 危险命令禁用

```conf
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG ""
rename-command SHUTDOWN ""
```

## 四、关键设计

### 为什么 Redis 事务不支持回滚？

Redis 认为运行时错误（如对 String 执行 LPUSH）是代码 bug——这种错误不应该在生产中发生。加上回滚机制会大幅增加复杂度和性能开销，而收益很小。这是典型的"简单胜过复杂"的设计哲学。

### Pipeline 不是原子的

Pipeline 本质是批量发送 + 批量接收，不是 MULTI/EXEC。Pipeline 中的命令在到达服务端后按正常顺序执行，但中间可能被其他客户端的命令插入。如果需要原子性 + 条件判断，用 Lua。

## 五、面试高频问题

**Q1：KEYS 命令为什么被禁用？**

KEYS * 是 O(N) 遍历所有 key，在 Redis 单线程模型下会阻塞所有其他客户端请求。1000 万 key 的 KEYS * 可能阻塞数百 ms。替代方案：SCAN（游标分批，不阻塞）。

**Q2：Pipeline 和事务有什么区别？**

Pipeline 是网络优化（N 命令 = 1 次 RTT），不保证原子性，中间可能被其他客户端命令插入。MULTI/EXEC 保证原子性（命令之间不插入），但不支持条件判断。Lua 既保证原子性又支持条件判断。

**Q3：Redis 变慢怎么排查？**

八步法：慢查询日志 → QPS → 碎片率 → 持久化状态 → fork 耗时 → swap 检查 → 网络延迟 → CPU。最常见的慢因：KEYS/DEL 大 key/HGETALL 大 Hash/复杂 Lua/同时过期/fork。

**Q4：连接池满了会怎样？**

等待 PoolTimeout 超时 → 返回错误。应用层需要：记录日志 + 监控告警 + 触发降级（返回缓存旧值/默认值）+ 考虑扩容。

## 六、总结（速记版）

- **Pipeline**：N 命令 = 1 次 RTT，非原子
- **事务**：不支持回滚 + 不支持条件判断 → 复杂场景用 Lua
- **限流**：ZSet 滑动窗口 + Lua 原子执行
- **SCAN**：替代 KEYS，游标分批，可能重复需去重
- **慢查询**：SLOWLOG GET 定位慢命令
- **内存碎片**：碎片率 > 1.5 → activedefrag，< 1.0 → swap 危险
- **连接池**：PoolSize ≈ QPS × latency × 2，MinIdleConns 预热

---

# 十、高级数据结构专题

## 一、背景（为什么需要）

除了 5 种基础类型，Redis 还提供 Bitmap、HyperLogLog、Stream、Pub/Sub、Geo、WATCH 等高级特性。这些是面试加分项，考察广度。

## 二、核心概念

### Bitmap —— 海量布尔值存储

本质上是一个 String，但 Redis 提供了按位操作命令。把 String 看成连续的 bit 数组。

```
SETBIT key offset 1       → 把第 offset 位设为 1
GETBIT key offset         → 获取第 offset 位
BITCOUNT key [start end]  → 统计 1 的个数
BITOP AND/OR/XOR r k1 k2  → 位运算（多 Bitmap 间 AND/OR/XOR）
BITPOS key 1              → 返回第一个 1 的位置
```

**签到场景**：10 亿用户签到状态 = 10 亿 bit ≈ 125MB。如果用 Set 存已签到用户 ID（int64）：10 亿 × 8B = 8GB。Bitmap 省 64 倍。

**注意**：Bitmap 适合"ID 连续且密集"的场景。如果只有 offset=1 和 offset=10 亿两个位是 1，中间 10 亿个 0 仍占 ~125MB——此时用 Set 更省内存。

### HyperLogLog —— UV 统计神器

概率性基数统计算法，估算集合中不重复元素数量。

```
PFADD uv:page1 user1 user2 user3
PFCOUNT uv:page1           → 约等于 3
PFMERGE uv:total uv:p1 uv:p2 → 合并多个 HLL
```

核心特性：
- 每个 HLL **固定 12KB**（无论数据量多大！）
- 可统计 2^64 个不同元素
- 标准误差 **0.81%**（< 1%）

**原理简述**：基于伯努利试验——抛硬币连续 K 次正面的概率 = 1/2^K。HLL 将元素 hash 为 64 位：低 14 位选择桶（16384 个桶），高 50 位统计"从低位起连续 0 的个数"。16384 × 6bit/桶 = 12KB。PFCOUNT = 调和平均数 × 16384 × 偏差修正。

**限制**：只能统计基数，不能查某元素是否存在。基数 < 1000 时误差可能较大。适合 10 万+ UV 场景。

### Stream —— 原生消息队列（5.0+）

持久化消息队列，支持消费组和 ACK 确认。

```
XADD mystream * field1 val1       → 生产消息（* = 自动生成 ID）
XREAD COUNT 2 BLOCK 5000 STREAMS mystream 0  → 独立消费
XGROUP CREATE mystream mygroup $ MKSTREAM    → 创建消费组
XREADGROUP GROUP mygroup c1 STREAMS mystream >  → 消费组消费（> = 未投递）
XACK mystream mygroup "msg_id"    → ACK 确认
XPENDING mystream mygroup         → 查看待确认消息（PEL）
XCLAIM mystream mygroup c2 60000 "msg_id"  → 转移超时消息给其他消费者
```

**Stream vs List 做消息队列**：

| 特性 | List (LPUSH+BRPOP) | Stream |
|------|-------------------|--------|
| 消费组 | 不支持 | 支持 XREADGROUP |
| ACK 确认 | 不支持 | 支持 XACK |
| 消息回溯 | 消费即删除 | **消息持久保留** |
| PEL 待确认列表 | 无 | 有（XPENDING） |

> Stream 适合轻量级 + 已有 Redis 的场景。对比 Kafka：没有分区并行和零拷贝优化。消息 ID 格式：`<milliseconds>-<sequence>`。

### Pub/Sub —— 实时推送（消息不持久化！）

```
SUBSCRIBE ch1 ch2          → 订阅
PUBLISH ch1 "msg"          → 发布
PSUBSCRIBE news:*          → 模式订阅（通配符）
```

**致命缺陷**：消息不持久化（订阅者不在线 → 永久丢失）、无 ACK、积压断开连接、无法回溯。

**适用**：实时通知（WebSocket 跨进程分发）、配置热更新、缓存失效广播。
**不适用**：重要业务消息、需要 ACK/重试/回溯。需要可靠性用 Stream 或 Kafka。

**Shard Pub/Sub（Redis 7.0+）**：`SSUBSCRIBE` → 消息只在同一 shard 内传播，Cluster 模式下减少跨节点网络流量。

### Geo —— 地理位置

基于 ZSet + Geohash 编码实现。经纬度编码为 52 位整数作为 ZSet score。

```
GEOADD cities 116.40 39.90 "Beijing"
GEORADIUS cities 116.40 39.90 100 km    → 100km 内城市
GEODIST cities "Beijing" "Shanghai" km  → 距离
```

**原理**：二维坐标 → Geohash → 一维 52 位整数 → ZSet score。GEORADIUS = ZRANGEBYSCORE + Haversine 精确过滤。空间相邻 ≈ 数值相邻，ZSet 的 ZRANGEBYSCORE 本身就是高效范围查询。

### WATCH —— Redis 的 CAS 乐观锁

```
WATCH stock
stock = GET stock
if stock > 0:
    MULTI
    DECR stock
    EXEC      → 如果 stock 被其他客户端修改过，返回 nil → 重试
```

原理：WATCH 标记 key 为"被监听"，其他客户端修改 → 标记 dirty，EXEC 时检查 dirty → 为 true 则拒绝执行。

**WATCH vs SET NX**：WATCH（乐观锁）适合冲突少的场景，不阻塞但需重试循环；SET NX（悲观锁）适合冲突频繁的场景，阻塞等待但需要处理死锁和续期。

### Redis Functions（Redis 7.0+）

Redis 7.0 引入的 Functions 替代 Lua 脚本的"临时性"：
- 函数存储在服务端（`FUNCTION LOAD`），持久化到 RDB/AOF，重启后仍存在
- 支持 Function Library 管理和版本控制
- 避免每次执行 Lua 都重新传输脚本（减少网络开销）

## 三、底层原理

### 布隆过滤器的底层

布隆过滤器 = Bitmap + k 个 Hash 函数。添加元素时 k 个位置 SETBIT 为 1，查询时检查 k 个位置是否都为 1。本质：用极小的误判率换取极大的内存节省。

### HyperLogLog 的调和平均数

HLL 不用算术平均（会被极端值拉偏），而用调和平均数——受大值影响小，更稳定地估计基数。这是 HLL 误差 < 1% 的关键数学工具。

## 四、关键设计

### 为什么 Stream 不如 Kafka 适合大规模消息？

Stream 没有 Partition 的概念——消息在单 key 内是有序的，但无法像 Kafka 那样把数据分布到多个 Broker 实现并行读写和水平扩展。Stream 适合"已有 Redis，不想引入新组件"的轻量级消息场景。

### 为什么 Pub/Sub 不持久化？

设计取舍。持久化需要存储消息 → 需要内存/磁盘空间 → 订阅者离线期间消息积压 → 积压太多导致 OOM。Pub/Sub 选择了"简单实时"的定位，如果需要可靠消息，用 Stream 或 Kafka。

## 五、面试高频问题

**Q1：Bitmap 的 offset 很大但中间很多 0，会浪费内存吗？**

会。Bitmap 底层是 String，offset 决定了 String 的长度。如果只有 offset=1 和 offset=10 亿两个位是 1，中间 10 亿个 0 仍占 ~125MB。Bitmap 适合"ID 连续且密集"的场景。

**Q2：HLL 能判断某个元素是否存在吗？**

不能。HLL 只能统计基数，不支持 PFEXISTS。要判断存在用 Set 或布隆过滤器。HLL 也支持 PFMERGE（合并多个 HLL）。

**Q3：Stream 和 List 做消息队列有什么区别？**

Stream 支持消费组 + ACK + 消息持久保留 + PEL 待确认列表，List 消费即删除且无 ACK。Stream 适合需要可靠性的消息场景。

**Q4：WATCH 和 SET NX 怎么选？**

WATCH（乐观锁）适合冲突少的场景，不阻塞但需重试循环。SET NX（悲观锁）适合冲突频繁的场景，阻塞等待但需处理死锁和续期。

## 六、总结（速记版）

- **Bitmap**：String 位视图，签到 1 亿用户 ≈ 12.5MB，适合 ID 密集场景
- **HLL**：固定 12KB、误差 < 1%，只统计基数不查成员，调和平均数
- **Stream**：5.0 消息队列，消费组 + XACK + PEL，轻量级
- **Pub/Sub**：实时推送不持久，适合通知/配置刷新，不适合重要消息
- **Geo**：ZSet + Geohash，GEORADIUS = ZRANGEBYSCORE + Haversine
- **WATCH**：CAS 乐观锁，EXEC 检测冲突 → 失败重试
- **Functions（7.0+）**：服务端存储函数，持久化，替代临时 Lua 脚本

---

# 十一、面试速查附录

## Go 语言使用 Redis

**主流客户端**：

| 客户端 | 特点 | 推荐度 |
|--------|------|--------|
| **go-redis** | 功能最全，支持单/哨兵/Cluster/Stream/Lua/Pipeline | 最流行 |
| rueidis | 高性能，服务端缓存辅助 | 高性能场景 |

**生产级配置**：
```go
rdb := redis.NewClient(&redis.Options{
    Addr:            "localhost:6379",
    PoolSize:        10,
    MinIdleConns:    3,
    DialTimeout:     5 * time.Second,
    ReadTimeout:     3 * time.Second,
    WriteTimeout:    3 * time.Second,
    PoolTimeout:     4 * time.Second,
})
```

**Go 常见坑**：

| 坑 | 原因 | 解决 |
|----|------|------|
| 连接泄漏 | 每次 NewClient 不复用 | 全局单例 |
| 连接池耗尽 | PoolSize 太小 | 合理设置 + 监控 PoolTimeout |
| 大 key OOM | 一次读大 value | SCAN 分页 + 限制单次大小 |
| Pipeline 误用 | 命令依赖前结果 | 仅无依赖场景用 |
| context 未传 | 用 Background 而非请求 ctx | 始终传请求 ctx |

## 面试前 10 分钟速过

```
═══════════════════════════════════════════
Redis 本质：
  内存全局 dict + redisObject(type+encoding+ptr) + 多编码按规模切换
  单线程命令执行 + epoll + 持久化 + 高可用

═══════════════════════════════════════════
数据结构核心：
  SDS: O(1)长度/二进制安全/预分配/5种header → 3编码(int/embstr/raw)
  dict: 双表渐进rehash/写进新表/读先旧后新 → Hash/Set/ZSet底层
  skiplist: 随机层高P=0.25/64层/span算rank → +dict双结构服务ZSet
  quicklist: 双向链表+listpack节点 → List底层
  listpack: 紧凑连续/存自身长度(无连锁更新) → 7.0替代ziplist

═══════════════════════════════════════════
设计哲学：
  小数据→紧凑结构(listpack/intset/embstr)省内存
  大数据→查询结构(hashtable/skiplist)保性能

═══════════════════════════════════════════
过期淘汰：
  过期：惰性(访问删)+定期(每100ms随机20个,>25%继续,≤25ms限制)
  淘汰：8种, LRU近似(采样淘汰), LFU对数计数+衰减

═══════════════════════════════════════════
持久化：
  RDB: fork+COW快照,文件紧凑恢复快,丢最后快照后
  AOF: 追加命令,everysec丢1秒,rewrite压缩
  混合: RDB头+AOF增量,恢复快+数据完整=生产首选

═══════════════════════════════════════════
高可用：
  主从: 全量(RDB+buffer)+增量(PSYNC+backlog) 异步复制
  哨兵: SDOWN→ODOWN→Raft选Leader→选新Master(优先级>offset>runid)
  Cluster: 16384slot/CRC16/Gossip/MOVED+ASK/hash tag/自带故障转移

═══════════════════════════════════════════
分布式锁：
  SET NX PX + Lua验证解锁 + Watchdog续期 + 可重入(Hash+计数器) + Redlock多节点
  适合"效率"场景，不适合"正确性"→强一致用ZK

═══════════════════════════════════════════
缓存设计：
  穿透(不存在)→布隆/空值  击穿(热key过期)→互斥锁
  雪崩(大量过期)→随机TTL/高可用  一致性→Cache Aside+双删+Canal
  本地缓存vs Redis→纳秒不共享 vs 微秒共享  Redis单机10w+ MySQL 3k-5k

═══════════════════════════════════════════
工程：
  BigKey→拆分+UNLINK  HotKey→本地缓存复制  KEYS→SCAN
  Pipeline→N命令=1RTT(非原子)  事务→不支持回滚
  限流→ZSet滑动窗口+Lua  慢查询→SLOWLOG GET
  碎片率>1.5→activedefrag  <1.0→swap危险
  连接池→PoolSize≈QPS×latency×2, MinIdleConns预热
```

## 面试最终 1 分钟总结

```
Redis 是一个基于内存的数据结构服务器。它通过 redisObject 统一封装 value，
在小数据时用 listpack/intset/embstr 紧凑存储节省内存，大数据时转 hashtable/skiplist 保证性能。

高性能源于：内存操作 + 高效结构 + 单线程无锁 + epoll + RESP简单协议 +
Pipeline + 6.0 IO线程 + 渐进式rehash + 紧凑编码。

可靠性方面：RDB快照(fork+COW) + AOF日志(everysec) + 混合持久化(生产首选)；
主从复制(全量RDB+增量backlog) + Sentinel自动故障转移(Raft选Leader) +
Cluster自动分片(16384 slot/CRC16/Gossip)。

工程方面：分布式锁 SET NX PX + Lua防误删 + Watchdog续期；
缓存穿透用布隆过滤器，击穿用互斥锁，雪崩用随机TTL+高可用；
一致性用 Cache Aside + 延迟双删 + Canal兜底。

Redis的精髓在于"按场景选择最合适的方案"——从底层编码到高可用架构，每个决策都在做trade-off。
```

## 易错说法清单

| 错误 | 正确 |
|------|------|
| Redis 是单线程的 | 命令执行单线程，但 fork 子进程/bio 线程/6.0 IO 线程 = 多线程进程 |
| Redis 就是一个缓存 | NoSQL 数据库，可做队列/排行榜/分布式锁/实时计算 |
| KEYS 可以用在生产 | KEYS 阻塞全局，生产禁用！用 SCAN |
| LRU 是精确的 | Redis 是近似 LRU（随机采样淘汰），精度约 70% |
| RDB 不丢数据 | 丢最后快照后到宕机之间的数据 |
| 事务支持回滚 | Redis 事务不支持回滚，运行时错误不撤销 |
| Pipeline 是原子的 | Pipeline 只是网络优化，不保证原子性 |
| 分布式锁 SET NX 后 EXPIRE 就行 | 两个命令不原子，中间崩溃 → 死锁，用 SET NX PX |
| Redis 和 Memcached 差不多 | 数据结构/持久化/高可用等维度 Redis 全面超越 |

---

> **版本说明**：本笔记以 Redis 7.x 为参考版本。6.0 引入 IO 线程，7.0 listpack 替代 ziplist。具体配置以当前版本官方文档为准。
> **参考资源**：Redis 官方文档、Redis 设计与实现（黄健宏）、Redis 源码、Redisson 官方文档、go-redis 文档

---

# 四、过期删除与内存淘汰

## 一、背景（为什么需要）

Redis 是内存数据库，内存有限。需要两套机制管理内存：过期删除处理"设了 TTL 的 key 何时删除"，内存淘汰处理"内存满了淘汰哪些 key"。

## 二、核心概念（是什么）

**过期删除**：针对设置了 EXPIRE/TTL 的 key，删除策略是惰性+定期组合。
**内存淘汰**：当 used_memory > maxmemory 时，按配置策略淘汰 key 腾空间。

两者触发时机不同：过期删除由 key 的 TTL 触发；内存淘汰由内存不足触发。可以同时生效。

## 三、底层原理

### 过期 key 存储

```
redisDb 中有两个 dict：
  dict       ← 存所有 key-value
  expires    ← 存过期 key → 到期时间戳（毫秒）
```

### 为什么不用"定时删除"？

定时删除 = 每个设了 TTL 的 key 都有一个定时器，到期立刻删除。问题：
1. 百万 key = 百万定时器，CPU 开销巨大
2. 大量 key 同时过期 → CPU 全用于处理定时器回调 → 正常命令阻塞
3. Redis 单线程模型下完全不可接受。**用少量内存浪费换 CPU 稳定性。**

### 惰性删除

每次访问 key 时检查：在 expires 字典中查到期时间 → 过期则删除并返回 nil。CPU 友好但冷 key 长期不访问则永久占内存。

### 定期删除（核心）

serverCron 每 100ms 执行一次：
1. 从 expires 字典**随机抽取 20 个** key
2. 删除已过期的 key
3. 过期比例 > 25%（20 个中 >5 个过期）→ 继续循环（说明过期 key 浓度高）
4. 每轮最多执行 **25ms** → 超时强制停止，让出 CPU

> 25% 阈值是经验值：比例低说明大部分没过期，继续扫收益小。25ms 限制保证不阻塞主线程。

### 内存淘汰 8 种策略

```
noeviction（默认）     → 不淘汰，写命令返回错误

volatile-lru           → 过期 key 中淘汰最近最少使用
volatile-lfu           → 过期 key 中淘汰频率最低（4.0+）
volatile-random        → 过期 key 中随机淘汰
volatile-ttl           → 过期 key 中淘汰剩余 TTL 最短

allkeys-lru（推荐）    → 所有 key 中淘汰 LRU
allkeys-lfu（推荐）    → 所有 key 中淘汰 LFU
allkeys-random         → 所有 key 中随机淘汰
```

### 近似 LRU——不是精确 LRU！

精确 LRU 需要双向链表维护访问顺序（每次 GET/SET 移动节点），百万 key 的维护开销不可接受。Redis 做法：每次淘汰时**随机采样 maxmemory-samples 个 key**（默认 5），从中选最久未使用的删除。采样越大越精确（10 个≈90% 精度）。

### LFU——对数计数器 + 衰减

利用 redisObject.lru 字段的 24 位：
- 高 16 位：最后衰减时间（分钟级）
- 低 8 位：对数计数器（0-255），不是每次访问 +1，而是**概率递增**（counter 越大递增概率越低）

随时间流逝 counter 自动衰减 → "曾经热但最近不热"的 key 能被淘汰。

## 四、关键设计

**为什么 LRU/LFU 是近似的？** Redis 的设计权衡：精确 LRU 的内存开销（双向链表）与其收益不成正比。随机采样 5 个 key 已有 ~70% 精度，10 个 ~90%。对缓存淘汰场景来说，70% 精度已经足够好。

## 五、面试高频问题

**Q1：过期删除和内存淘汰有什么区别？**

过期删除针对设了 TTL 的 key 到期删除（惰性+定期策略）；内存淘汰是内存达 maxmemory 上限时按策略淘汰 key（可能淘汰未过期的）。两者触发条件不同。

**Q2：为什么 Redis 不立即删除过期 key？**

立即删除需要为每个 key 维护定时器，大量 key 同时过期时 CPU 全部处理过期回调，阻塞正常命令。Redis 选择惰性+定期组合——用少量内存换取 CPU 稳定和低延迟。

**Q3：LRU 是精确的吗？**

不是。Redis 使用近似 LRU：随机采样 maxmemory-samples 个 key（默认 5），从中选最久未使用的淘汰。不需要维护全局链表，牺牲约 30% 精度换取巨大的内存和执行效率。

**Q4：allkeys-lru 和 volatile-lru 怎么选？**

纯缓存场景（所有 key 都有 TTL 或都可淘汰）用 allkeys-lru；部分 key 需要永久保留用 volatile-lru（只淘汰有过期时间的）。

## 六、总结（速记版）

- **过期删除** = 惰性（访问删）+ 定期（每 100ms 随机 20 个，>25% 继续，≤25ms 限制）
- **淘汰策略** = 8 种，生产常用 allkeys-lru/lfu
- **LRU** = 近似，随机采样淘汰，不维护全局链表。samples=5 精度 ~70%
- **LFU** = 8 位对数计数器 + 概率递增 + 时间衰减

---

# 五、持久化

## 一、背景（为什么需要）

内存数据断电即失。Redis 需要持久化保证数据不丢失，同时不能影响内存读写性能。

## 二、核心概念

三种持久化方式：RDB（快照）、AOF（命令日志）、混合持久化（4.0+，RDB+AOF 结合）。

## 三、底层原理

### RDB：fork + COW 快照

```
BGSAVE 流程：
1. 主线程 fork() 创建子进程
2. fork 后父子进程共享同一份物理内存（只读）
3. 子进程遍历内存数据 → 写入 RDB 文件
4. 父进程继续处理客户端请求
   ├── 读操作：直接读共享内存 → 不触发 COW
   └── 写操作：OS 复制被修改的内存页（COW）→ 父进程修改新页
5. 子进程完成 → 原子 rename 替换旧 RDB 文件

COW（Copy-On-Write）写时复制：
  fork 后不立即复制内存，父进程写某页（4KB）时才复制该页。
  子进程看到的是 fork 时刻的数据快照。
```

**触发方式**：手动（SAVE 阻塞/BGSAVE 非阻塞）、自动（save 900 1 等配置）、主从全量同步。

**优点**：文件紧凑、恢复极快、适合全量备份。
**缺点**：两次快照之间数据丢失；fork 大内存（>32GB）时页表复制阻塞数十 ms；写密集时 COW 导致内存翻倍。

### AOF：追加命令日志

```
AOF 将每条写命令追加到日志文件（文本格式）。
重启时重放命令恢复数据。

三种 fsync 策略：
  always    → 每条命令都 fsync → 最安全、最慢
  everysec  → 每秒 fsync 一次 → 最多丢 1 秒数据（推荐）
  no        → OS 决定何时 fsync → 性能最好、最不安全
```

**AOF Rewrite（重写）**：AOF 文件随时间无限增长 → fork 子进程根据当前内存数据生成最简命令集 → 原子 rename 替换。

```
原始 AOF（多次 INCR）:           重写后:
  SET counter 0                  SET counter 5
  INCR counter                   （5 条命令压缩为 1 条）
  INCR counter
  INCR counter
  INCR counter
```

### 混合持久化（Redis 4.0+，生产推荐）

```
AOF Rewrite 时的文件格式：
┌──────────────────────┐
│ RDB 二进制快照数据    │ ← 加载极快（二进制）
├──────────────────────┤
│ 重写期间的 AOF 增量命令 │ ← 数据完整（补全快照后数据）
└──────────────────────┘

配置：aof-use-rdb-preamble yes
```

## 四、关键设计

**RDB vs AOF 重启优先级**：同时开启时优先加载 AOF（数据更完整）。混合持久化下 AOF 文件头部是 RDB 格式。

**fork 的风险**：fork 操作本身需要复制父进程页表，大内存下（如 32GB）页表复制可能阻塞主线程数十 ms。通过 `latest_fork_usec` 指标监控。

## 五、RDB vs AOF vs 混合持久化对比

| 维度 | RDB | AOF everysec | 混合持久化 |
|------|-----|-------------|-----------|
| 数据安全 | 可能丢几分钟 | 最多丢 1 秒 | 最多丢 1 秒 |
| 恢复速度 | **最快** | 慢（逐条重放） | **快** |
| 文件大小 | 紧凑 | 较大 | 中等 |
| 生产推荐 | 备份保留 | 可配合 | **首选** |

## 六、面试高频问题

**Q：RDB 和 AOF 各有什么优缺点？怎么选？**

RDB = fork+COW 生成快照，文件紧凑恢复快，但两次快照间数据丢失。AOF = 追加每条写命令，最多丢 1 秒（everysec），但文件大恢复慢。生产用混合持久化：RDB 快照作前缀 + AOF 增量，兼顾速度和安全。

## 七、总结（速记版）

- **RDB**：fork + COW 快照，文件小恢复快，丢最后快照后数据
- **AOF**：追加命令日志，everysec 丢 1 秒，rewrite 压缩
- **混合**：RDB 头 + AOF 增量，恢复快 + 数据完整 = 生产首选
- **fork 风险**：大内存 fork 阻塞数十 ms，关注 latest_fork_usec

---

# 六、主从复制与高可用

## 一、背景（为什么需要）

单机 Redis 三个问题：容量有限（一台机器内存上限）、并发有限（单机 ~10w QPS）、单点故障（宕机→服务不可用）。

解决路径：主从复制（数据冗余+读写分离）→ Sentinel 哨兵（自动故障转移）→ Cluster（数据分片+水平扩展）。

## 二、主从复制

### 全量同步（FULL RESYNC）——首次连接

```
1. Slave → PSYNC ? -1（首次连接，不知道 Master runid/offset）
2. Master 返回 FULLRESYNC <runid> <offset>
3. Master 执行 BGSAVE 生成 RDB + 同时将新命令暂存 replication buffer
4. Master 发送 RDB 给 Slave
5. Slave 清空旧数据 → 加载 RDB
6. Master 发送 buffer 中增量命令
7. Slave 执行增量命令 → 同步完成
```

### 增量同步（Partial Resync）——断线重连

```
1. Slave → PSYNC <runid> <offset>
2. Master 检查：runid 匹配 + offset 在 repl_backlog_buffer 范围内？
3. 是 → 只发 offset 之后命令（增量同步）
4. 否 → 退化为全量同步
```

**repl_backlog_buffer**：Master 端环形缓冲区（默认 1MB），记录最近写命令。大小决定 Slave 断线多久还能增量同步。生产建议适当调大（如 64MB）。

### 复制是异步的——可能丢数据

Master 返回客户端后才异步同步 Slave。缓解：`min-replicas-to-write 1` + `min-replicas-max-lag 10`（Slave 不足或延迟大则拒绝写入）。

## 三、哨兵 Sentinel

### 职责：监控 + 通知 + 故障转移 + 配置提供

### SDOWN vs ODOWN

主观下线（SDOWN）：单个哨兵 PING 超时认为 Master 不可用。
客观下线（ODOWN）：≥ quorum 个哨兵都认为不可用 → 触发故障转移。

### 故障转移流程

```
1. ODOWN → 哨兵之间 Raft 选 Leader
2. Leader 从 Slave 中选新 Master：
   ① slave-priority 越小越优先
   ② 复制 offset 越大越优先（数据最新）
   ③ runid 字典序最小（tie-breaker）
3. Leader 向新 Master 发 SLAVEOF NO ONE
4. 通知其他 Slave 复制新 Master
5. Pub/Sub 通知客户端新 Master 地址
```

**哨兵部署**：至少 3 个哨兵（奇数），quorum = N/2+1。

## 四、Redis Cluster

### 16384 个 Hash Slot

```
slot = CRC16(key) % 16384

3 个 Master 节点示例：
  Node-1: slot 0~5460    (5461)
  Node-2: slot 5461~10922 (5462)
  Node-3: slot 10923~16383 (5461)
```

**为什么 16384？** 心跳包用位图表示 slot 分布，16384 bit = 2KB；65536 bit = 8KB 心跳包太大。

### MOVED vs ASK 重定向

MOVED：key 的 slot 永久在其他节点 → 更新本地 slot 表，重定向。ASK：slot 迁移中 → 临时重定向，下次仍问原节点。

### Gossip 协议

节点间通过 PING/PONG 交换信息（节点状态、slot 分布、故障检测）。去中心化，无单点。

### Hash Tag

`{user:1001}:name` 和 `{user:1001}:age` → 只对 `{}` 内容计算 CRC16 → 同一 slot → 支持 MGET、事务。

### 脑裂与缓解

网络分区 → 旧 Master 被隔离但未宕机 → 新 Master 选出 → 两个 Master 同时写 → 数据不一致。缓解：`min-replicas-to-write 1`（Master 感知不到足够 Slave 则拒绝写入）。

## 五、面试高频问题

**Q1：主从全量同步和增量同步区别？**

全量 = BGSAVE 生成 RDB + 传 RDB + 传 buffer 增量。增量 = PSYNC + 检查 offset 在 backlog 范围内 → 只传差异。首次/backlog 过小 → 全量；断线重连且 backlog 覆盖 → 增量。

**Q2：哨兵选 Master 流程？**

SDOWN → ODOWN(≥quorum) → Raft 选 Leader 哨兵 → 选新 Master（优先级 > offset > runid）→ 执行 failover → 通知。

**Q3：Cluster 的 16384 怎么来的？**

16384 = 2^14，slot 分布位图 2KB。如果用 65536 需 8KB 心跳包。节点数不超 1000，16384 足够分配。

**Q4：脑裂怎么解决？**

无法完全避免（CAP 取舍）。通过 min-replicas-to-write 缓解：Master 感知不到足够 Slave 时拒绝写入，牺牲可用性换一致性。

## 六、总结（速记版）

- **主从**：全量(RDB+buffer) + 增量(PSYNC+backlog)，异步复制可能丢数据
- **哨兵**：SDOWN→ODOWN→Raft选Leader→选新Master（优先级>offset>runid）
- **Cluster**：16384 slot/CRC16/Gossip/MOVED+ASK/Hash Tag/脑裂缓解

---

# 七、分布式锁

## 一、背景（为什么需要）

分布式系统中多个服务实例共享资源，需要互斥机制保证同时只有一个实例操作。

## 二、基础实现

```redis
SET lock_key unique_value NX PX 30000
```

- NX：key 不存在才设置（互斥）
- PX 30000：30 秒过期（防死锁——客户端崩溃后锁自动释放）
- unique_value：每个客户端生成 UUID，释放时验证身份

## 三、释放锁——必须用 Lua

错误做法是直接 DEL——可能误删别人的锁（你的锁过期→别人拿到锁→你 DEL→删了别人的锁）。

```lua
-- 正确做法：GET 验证 value 匹配后再 DEL
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

## 四、锁续期——Redisson Watchdog

业务执行时间 > 锁过期时间 → 锁提前释放 → 并发安全问题。

**Watchdog 方案**：加锁时默认 30s，启动后台线程每 10s 检查→业务还在执行→重置为 30s（续期）。进程崩溃→Watchdog 停止→锁自然过期。锁的生命周期跟随业务执行时间。

## 五、Redlock（多节点）

单 Redis 节点故障→所有锁丢失。Redlock：向 N（5）个独立 Redis 节点依次加锁，过半（N/2+1）成功且总耗时<过期时间→加锁成功。解决单点故障。

**争议**：Martin Kleppmann 认为时钟漂移和 GC STW 场景下不安全。业界共识：Redis 锁适合"效率"场景（防重复计算），不适合"正确性"场景（金融扣款→用 ZooKeeper）。

## 六、面试高频问题

**Q：分布式锁怎么实现？释放为什么用 Lua？**

SET key uuid NX PX 30000（互斥+防死锁+身份标识）。Lua 释放：GET 验证 value 匹配后再 DEL，保证原子性（防误删）。长时间任务用 Watchdog 自动续期。

## 七、总结（速记版）

- **加锁**：`SET key uuid NX PX 30000`
- **释放**：Lua 脚本 GET 验证后 DEL（防误删）
- **续期**：Watchdog 每 10s 续期到 30s
- **多节点**：Redlock 过半机制。争议：时钟漂移/GC → 安全场景用 ZK
