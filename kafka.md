# Kafka 面试复习笔记

> 适用：Go 后端面试准备 · 系统复习 · 面试速查
> 版本参考：Kafka 3.x/4.x（4.0 起仅支持 KRaft 模式）

---

# 一、Kafka 概述

## 一、背景（为什么需要）

分布式系统中服务间直接 RPC 调用带来三个核心问题：

1. **紧耦合**：上游 A 需感知下游 B/C/D，新增下游要改上游代码
2. **雪崩风险**：下游慢/挂 → 上游阻塞 → 线程耗尽 → 全链路崩溃
3. **峰值压力**：瞬时流量不可控，后端按峰值资源部署浪费巨大

Kafka 作为中间缓冲层，用"日志"的抽象同时解决这三个问题。

## 二、核心概念（是什么）

Kafka 是一个 **分布式、高吞吐、可持久化、基于发布/订阅模型的事件流平台**。本质是**分布式提交日志（Distributed Commit Log）**，消息队列只是它的一种使用方式。

| 概念 | 一句话 |
|------|--------|
| **Topic** | 消息的逻辑分类，类似数据库的表 |
| **Partition** | 物理存储 + 并行读写的基本单位，是一个有序追加日志 |
| **Replica** | Partition 的副本（Leader + Follower），保证高可用 |
| **Consumer Group** | 组内负载均衡（1 Partition → 1 Consumer），组间广播 |
| **Offset** | 消费进度指针，分区内单调递增 |

**面试标准回答（约 30 秒）**：

> Kafka 可以理解为一个高吞吐的分布式消息系统。通过 Topic 分类消息，Partition 实现水平扩展和并行读写，Replica/Leader/Follower/ISR 保证高可用，Consumer Group 实现并行消费。高性能依赖顺序写磁盘、PageCache、零拷贝、批量发送、压缩和分区并行六项机制。常用于日志采集、削峰填谷、系统解耦和实时数据管道。

**Kafka 与消息队列的本质区别**：

| 维度 | 传统 MQ（RabbitMQ） | Kafka |
|------|---------------------|-------|
| 数据模型 | 队列模型：消息消费后删除 | 日志模型：消息按时间保留，不随消费删除 |
| 消费方式 | Push 为主 | Pull 为主 |
| 消息回溯 | 不支持 | 支持，可重复消费历史数据 |
| 存储设计 | 内存为主，持久化为辅 | 磁盘为主，顺序追加写 |
| 扩展方式 | 垂直扩展为主 | 水平扩展（Partition 分布到多 Broker） |
| 吞吐量 | 万级/秒 | 百万级/秒 |

## 三、底层原理

### 为什么 Kafka 是"日志"而不是"队列"

Kafka 设计之初的目标是解决 LinkedIn 的**用户活动流**问题——海量数据、需要回溯、需要多消费者独立消费。传统消息队列的"消费即删除"模型无法满足。

Kafka 选择了**不可变、只追加（append-only）的日志模型**：

```
Partition 日志：
[msg0] [msg1] [msg2] [msg3] ... [msgN]  ← 只追加，永不修改
  ↑                           ↑
 最早                        最新
```

这个设计的三个关键推论：
1. 消息不随消费删除 → 天然支持回溯和多组独立消费
2. 只追加不修改 → 可以做顺序写磁盘（性能核心）
3. Offset 由 Consumer 管理 → 消费者按自身节奏消费

### 发展历史与关键里程碑

| 版本 | 关键变化 | 影响 |
|------|---------|------|
| Kafka 0.8 (2013) | 引入 Replica 机制 | 高可用 |
| Kafka 0.9 (2015) | Offset 迁移到 `__consumer_offsets`，Consumer Group | 去 ZK 依赖 |
| Kafka 0.11 (2017) | 幂等性 + 事务，Exactly Once | 可靠性里程碑 |
| Kafka 2.4 (2019) | Cooperative Rebalancing | 消费端重大改进 |
| Kafka 2.8 (2021) | KRaft 早期预览 | 去 ZK 起点 |
| Kafka 3.5+ | ZK 标记为 deprecated | 迁移过渡期 |
| Kafka 3.6+ (2023) | Tiered Storage 早期预览 | 分层存储，冷热分离 |
| Kafka 4.0 (2024) | 仅支持 KRaft | 彻底去 ZK |

## 四、关键设计

### 为什么设计成 Pull 而不是 Push？

Broker 不知道 Consumer 的处理能力。Push 模式下 Broker 推太快会压垮 Consumer。Pull 让 Consumer 按自身处理速度拉取，且方便批量拉取和 Offset 回溯。

代价：Consumer 没数据时可能空拉（通过 `fetch.min.bytes` + `fetch.max.wait.ms` 减少）。

### 为什么设计成日志型而不是队列型？

日志型的核心价值：**多消费者组独立消费 + 消息回溯**。同一份数据，实时计算组、数据归档组、监控告警组各自独立消费，互不影响。这在队列模型里需要复制多份数据。

## 五、面试高频问题

**Q1：Kafka 和 RabbitMQ 的核心区别？**

核心区别在数据模型。Kafka 是日志模型——消息写入后按时间保留，消费完不删除，可回溯。RabbitMQ 是队列模型——消费后删除，不可回溯。这导致吞吐量、适用场景本质不同：Kafka 适合高吞吐日志/流处理，RabbitMQ 适合业务消息和复杂路由。

> **追问：为什么日志模型天然比队列模型吞吐高？** 因为日志模型只需顺序追加写（O(1)），不需要管理消息的删除、索引维护、确认重入队等操作。队列模型每消费一条消息就要从存储中删除，涉及随机 I/O 和元数据更新。

**Q2：既然 Kafka 也是发消息，为什么不直接叫消息队列？**

消息队列暗示消息消费后消失。Kafka 可以做到：同一消息被多组独立消费、回溯三天前的消息重新处理、新消费者从最早位置追数据。Kafka 更像不可变的追加日志文件，消息队列只是它的一种使用方式。所以它叫"event streaming platform"而不是"message queue"。

> **追问：Kafka 能当传统消息队列用吗？** 技术上可以，但可能需要额外设计。例如实现延迟消息、死信队列、消息优先级，这些传统 MQ 原生的功能在 Kafka 中需要自行实现。所以选型时要看场景是否需要这些特性。

**Q3：为什么选择 Kafka 而不是其他 MQ？**

看场景。高吞吐日志/埋点/流处理 → Kafka；订单/交易/延迟消息 → RocketMQ；传统企业应用/复杂路由 → RabbitMQ。

## 六、总结（速记版）

- **定位**：分布式事件流平台 = 分布式提交日志
- **核心四件套**：Topic → Partition → Replica → Consumer Group
- **快的原因**：顺序写 + PageCache + 零拷贝 + 批量 + 压缩 + 分区并行
- **三大场景**：削峰填谷、系统解耦、异步通信
- **本质理解**：Kafka 是日志，不是队列

---

# 二、Kafka 核心架构

## 一、背景（为什么需要）

单机 Broker 有物理上限：磁盘 2TB、顺序写 600MB/s、网卡 10Gbps。现实中的业务需要 PB 级存储、百万 QPS 吞吐。架构设计的目标是：**用多台廉价机器组成一个看起来像"一台超级机器"的集群**。

## 二、核心概念

### 组件关系图

```
Broker Cluster
├── Broker-1
│   ├── Partition-0 [Leader]     ← 生产者和消费者读写此副本
│   └── Partition-1 [Follower]   ← 从 Leader-2 同步数据
├── Broker-2
│   ├── Partition-1 [Leader]
│   └── Partition-2 [Follower]
├── Broker-3
│   ├── Partition-2 [Leader]
│   └── Partition-0 [Follower]
└── Controller（某个 Broker 担任）
    └── 管理 Broker 上下线、Leader 选举、元数据同步
```

| 组件 | 作用 | 关键认知 |
|------|------|---------|
| **Producer** | 发送消息 | 分区策略、acks、幂等、批量 |
| **Broker** | Kafka 服务器 | 存储消息、管理副本 |
| **Controller** | 集群协调者 | 特殊 Broker，管 Leader 选举和元数据分发 |
| **Topic** | 逻辑主题 | 不存数据，只是分类标签 |
| **Partition** | 物理存储单元 | 有序追加日志，并行读写的基本单位 |
| **Leader** | 主副本 | 生产者和消费者读写此副本 |
| **Follower** | 从副本 | 从 Leader 拉取数据同步 |
| **Consumer** | 消费者 | poll() 拉取消息 |
| **Consumer Group** | 消费者组 | 组内负载均衡，组间广播 |

### 组件关系的三个核心规则

1. **Topic 是逻辑概念**：不存储数据，数据存在 Partition 里
2. **Partition 是物理单位**：每个 Partition 是一个目录，内含 .log + .index + .timeindex 文件
3. **Consumer Group 分配规则**：同组内 1 个 Partition 同时只能被 1 个 Consumer 消费；不同组之间独立

## 三、底层原理

### Partition 是最核心的设计决策

Partition 实现了"分而治之"：

**原因 1：突破单机存储上限**
```
Topic "user-behavior" 每天 500GB
单 Broker 磁盘 2TB → 只够存 4 天
分 10 个 Partition 分布到 10 台 Broker → 可存 40 天
```

**原因 2：突破单机 I/O 瓶颈**
```
单分区吞吐 ≈ 50MB/s
1 个 Partition → 最多 50MB/s
10 个 Partition → 理论 500MB/s（线性扩展）
```

**原因 3：保证局部有序**
Kafka 只保证单分区内有序。如果需要全局有序，只能 1 个 Partition（牺牲扩展性）。实际上大多数场景只需要"同一业务实体的消息有序"——用 key 路由即可。

**原因 4：支撑 Consumer Group 负载均衡**
Consumer Group 内以 Partition 为粒度分配。Partition 数 = 同组消费者的最大并行度。

### 为什么 Partition 数只能增加不能减少？

1. **数据迁移成本**：减少分区意味着历史数据需要搬迁，涉及海量数据复制
2. **Offset 语义破坏**：Partition 内 Offset 是连续递增的，删除后出现空洞，消费者 Offset 管理混乱
3. **Key 映射变化**：`hash(key) % partitionNum`，partitionNum 变了，同类消息散到不同分区，历史数据的顺序关系断裂
4. **设计哲学**：Kafka 选择"简单可靠"，不支持复杂的在线数据重分布

> 如果必须减少：新建 Topic（少分区）→ 双写 → 迁移历史数据 → 切换消费者 → 删旧 Topic。

### Consumer Group 与 Partition 的关系原理

```
Topic (3 Partitions) + Consumer Group (5 Consumers)：
  C0 → P0
  C1 → P1
  C2 → P2
  C3 空闲 ← Consumer 数 > Partition 数，多出来的浪费
  C4 空闲
```

核心公式：**有效并行度 = min(Partition 数, Consumer 数)**

### Broker 网络线程模型（Reactor 模式）

Kafka Broker 使用 **Reactor（多路复用 + 线程池）** 模式处理网络请求，是支撑高并发连接的基础：

```
Acceptor Thread（每个监听端口 1 个）
  │  accept() 接受新 TCP 连接
  │  轮询分发给 Processor Threads
  ↓
Processor Threads（默认 3 个，可配）
  │  从 SocketChannel 读取请求 → 放入 RequestQueue
  │  从 ResponseQueue 取出响应 → 写回 SocketChannel
  ↓
RequestQueue（有界队列，默认 500）
  ↓
Handler Threads（IO 线程池，默认 8 个）
  │  处理业务逻辑：追加日志、读 Partition、处理 Offset 请求等
  ↓
Kafka Request Handler 将结果放入 ResponseQueue
```

**面试关键认知**：

| 层 | 作用 | 如果成为瓶颈 |
|----|------|-------------|
| Acceptor | 接受连接 | 连接数 > 几千/秒 时可能成为瓶颈 |
| Processor | 网络读写 | `num.network.threads` 调大 |
| RequestQueue | 缓冲请求 | 满了 → 新请求被阻塞（`queued.max.requests`） |
| Handler | 业务处理 | `num.io.threads` 调大 |

> **面试追问**：为什么 Handler 线程数默认只有 8，而 Processor 默认只有 3？因为大部分请求是磁盘追加写（极快），真正的瓶颈在网络 I/O 和磁盘 I/O，不在 CPU 计算。Handler 只做"把消息追加到 PageCache"这一件事，O(1) 操作，8 个线程足够。

### Controller 选举与故障转移深入

**Controller 选举流程（KRaft 模式）**：

```
1. 启动时，Quorum 节点间通过 Raft 协议选举 Active Controller
2. 选举成功的节点成为 Active Controller，其余为 Standby
3. Active Controller 维护集群所有元数据：
   - Broker 注册/注销
   - Topic/Partition 的创建、删除
   - Leader 选举触发
   - 分区副本分配
4. Controller 将元数据变更包装为 Raft 日志条目
5. Quorum 中多数派确认后，元数据变更生效
6. Active Controller 将最新元数据通过 RPC 推送给所有 Broker
```

**Controller 故障转移关键时间线**：

```
Active Controller 宕机
  → Raft 选举超时触发（通常 100-300ms）
  → 新 Controller 选举（秒级）
  → 新 Controller 从 Raft Log 重放元数据
  → 恢复完整集群状态视图
  → 开始处理集群变更请求

ZK 模式下需要 10-30 秒，KRaft 模式下只需 1-3 秒
```

> **追问：KRaft 如何防止脑裂？** 使用 Raft 协议的 Term + Quorum 机制。Active Controller 必须持续获得多数派节点的投票才能维持领导地位。如果网络分区导致旧 Controller 孤立，它无法获得多数票，自动降级为 Follower。新的多数派分区会选出新 Controller。

## 四、关键设计

### Partition 的 trade-off

| 设计 | 优点 | 代价 |
|------|------|------|
| 多 Partition | 高吞吐、高并行 | 文件句柄多、元数据多、Rebalance 慢 |
| 少 Partition | 管理简单、单分区吞吐高 | 并行度不足、无法水平扩展 |

**估算公式**：`分区数 ≈ max(峰值吞吐/单分区吞吐, 期望并行度) × 预留系数(1.5~2)`

### 为什么 Follower 用 Pull 而不是 Leader Push？

Follower 主动从 Leader 拉取（类似 Consumer 的 Pull 模型）：
- Follower 按自身同步进度决定拉取量和频率
- 不会因为 Leader 推送太快导致 Follower 跟不上
- 落后时 Follower 可以加大拉取速度追赶
- Leader 不需要追踪每个 Follower 的进度（简化 Leader 端设计）

这与 Consumer 的 Pull 模型遵循相同的设计哲学：**让接收端控制速率**。

## 五、面试高频问题

**Q1：Topic 和 Partition 什么关系？**

Topic 是逻辑概念，不存数据；Partition 是物理存储单元。一个 Topic 可以拆成多个 Partition，分布在不同 Broker 上。数据实际存在 Partition 的 .log 文件中。

**Q2：Consumer 数量超过 Partition 数会怎样？**

多出来的 Consumer 空闲，不会消费任何消息。有效并行度 = min(Partition 数, Consumer 数)。所以加 Consumer 前先看 Partition 数够不够。

**Q3：为什么 Partition 数不能减少？**

数据迁移成本太高、Offset 序列会断裂、Key 映射会变化。Kafka 选择简单可靠的设计：只增不减。必须减的话只能重建 Topic 再迁移。

**Q4：Partition 越多越好吗？**

不是。每个 Partition 对应磁盘上至少 3 个文件（.log + .index + .timeindex），对应 Broker 内存中的索引结构，对应 Controller 的元数据条目。过多导致：文件句柄耗尽、元数据压力、Rebalance 变慢、Leader 选举成本增加。

**Q5：Broker 的网络线程模型是什么？为什么这样设计？**

Reactor 模式：1 个 Acceptor 接收连接 → 3 个 Processor 做网络 I/O → RequestQueue 缓冲 → 8 个 Handler 处理。这样设计的原因：Kafka 的核心操作（追加写、顺序读）都是 O(1) 的，CPU 不是瓶颈，网络 I/O 和磁盘 I/O 才是。少而精的线程模型避免了线程切换开销和锁竞争。

## 六、总结（速记版）

- **Topic**：逻辑分类，不存数据
- **Partition**：物理存储，并行读写，有序追加日志
- **Replica**：Leader + Follower 副本 = 高可用
- **Consumer Group**：同组 1 Partition → 1 Consumer，组间广播
- **并行度上限** = min(Partition 数, Consumer 数)
- **网络模型**：Reactor 模式 = Acceptor + Processor + Handler 三层

---

# 三、Producer 生产者

## 一、背景（为什么需要）

Producer 是消息的入口。消息不丢、不重、有序这三个目标有一半靠 Producer 配置达成。理解 Producer 的缓冲、分区、重试、幂等机制，才能在生产中正确配置。

## 二、核心概念

### 发送流程七步

```
业务线程 send()
  → Interceptor（拦截器）
  → Serializer（序列化 key/value）
  → Partitioner（选择目标分区）
  → RecordAccumulator（写入缓冲区，按 TopicPartition 聚合成批次）
  → Sender 线程（从缓冲区取出就绪批次）
  → Broker（网络发送 + 等待 ACK）
```

### 核心参数速查

| 参数 | 作用 | 默认值 | 调优方向 |
|------|------|--------|---------|
| `buffer.memory` | 总缓冲区大小 | 32MB | 增大 → 可容纳更多未确认消息 |
| `batch.size` | 单批次大小上限 | 16KB | 增大 → 吞吐↑，延迟↑ |
| `linger.ms` | 批次等待时间 | 0（立即发送） | 增大 → 吞吐↑，延迟↑ |
| `acks` | 应答级别 | 1 | all → 可靠性最高 |
| `retries` | 重试次数 | Integer.MAX | 配合幂等性使用 |
| `enable.idempotence` | 幂等性 | false | 开启 → 防重试重复 |
| `max.in.flight.requests.per.connection` | 单连接未确认请求数 | 5 | =1 保证顺序，>1 提高吞吐 |
| `compression.type` | 压缩算法 | none | snappy/lz4/zstd |
| `delivery.timeout.ms` | 消息投递总超时 | 120s | 包含重试的所有时间 |

### 分区策略（优先级从高到低）

1. 代码显式指定 partition → 直接写入
2. 指定了 key → `hash(key) % partitionNum` → 同一 key 进同一分区
3. 都没指定 → **黏性分区**（Sticky）：先选一个分区持续写入，批次满或超时再切换

> 黏性分区优于轮询：提高批次聚合效率，减少小批次。

**面试追问：如果 key 相同的消息必须严格有序，但 hash 冲突导致热点分区怎么办？**

两种思路：
1. 自定义 Partitioner——对热点 key 做子分区：`partition = hash(key + "_" + (seq % N)) % totalPartitions`，消费端按子分区聚合排序
2. 接受单分区有序的限制——热点 key 单独一个分区，做好容量规划

> 本质上这是"有序 vs 均匀分布"的 trade-off，Kafka 的设计倾向于简单：你要有序，就接受热点风险；你要均匀，就放弃严格有序。

## 三、底层原理

### RecordAccumulator 缓冲机制

```
RecordAccumulator（总缓冲区 = buffer.memory，默认 32MB）
  └── ConcurrentMap<TopicPartition, Deque<ProducerBatch>>
        ├── TopicA-P0 → [Batch1(未满)] [Batch2(未满)]
        ├── TopicA-P1 → [Batch3(已满，等待发送)]
        └── TopicB-P0 → [Batch4(未满)]

Sender 线程从 Deque 头部取出满的/超时的 Batch 发送
```

每个 `ProducerBatch` 对应一次网络请求。两个条件触发发送：
- `batch.size` 满了 → 立即发送
- `linger.ms` 到期 → 即使不满也发

**本质是吞吐量和延迟的权衡**：

```
linger.ms=0：    来一条发一条 → 延迟最低，吞吐最差
linger.ms=10ms： 等 10ms 聚合 → 吞吐提升 10x+，单条延迟 +10ms
linger.ms=100ms：批次更大 → 吞吐最高，单条延迟可能 +100ms
```

### buffer.memory 满了怎么办？

业务线程调用 `send()` 时，如果缓冲区满（所有消息都在等待发送），`send()` 阻塞，最多阻塞 `max.block.ms` 毫秒（默认 60 秒）。超时抛 `TimeoutException`。

> **追问：buffer.memory 满了说明什么？** 说明生产速度 > Sender 线程发送速度。可能原因：Broker 端处理慢、网络带宽不足、`batch.size`/`linger.ms` 配置不当导致大量小批次堆积。需要从 Producer 端监控 `bufferpool-wait-time` 指标。

### 超时参数关系

```
delivery.timeout.ms >= request.timeout.ms + linger.ms

send() → 写缓冲区 → Sender 发送 → 等 ack → 失败了重试 → 超时/异常
         |______________ delivery.timeout.ms _______________|
                         |__ request.timeout.ms __|
```

Producer 不是无限重试。`delivery.timeout.ms` 是总超时，超过后通过回调或 Future 返回异常。

### Sticky Partitioner（黏性分区）深层原理

Kafka 2.4+ 引入的 Sticky Partitioner 替代了传统的 RoundRobin：

```
RoundRobin（旧）:
  消息1 → P0, 消息2 → P1, 消息3 → P2, 消息4 → P0, ...
  缺点：每个分区的批次都很小，频繁发送小批次 → 网络开销大

Sticky（新）:
  消息1-100 → P0（批次满了，发送）
  消息101-200 → P1（批次满了，发送）
  ...
  优点：批次聚合效率最大化，减少小批次
```

> 本质：Sticky 用"微小的不均匀"换取了"显著的批次效率提升"。黏在一个分区直到批次满了/超时，再换下一个。

### Producer Interceptor（拦截器）

Interceptor 是 Producer 发送链路中的 Hook 点，可以在消息发送前后插入自定义逻辑：

```
ProducerInterceptor 接口：
  onSend(ProducerRecord) → ProducerRecord   // 发送前：可修改/过滤消息
  onAcknowledgement(RecordMetadata, Exception)  // 发送后：可记录日志/监控
```

**典型用途**：
```java
// 示例 1：自动注入时间戳 + TraceID
public class EnrichInterceptor implements ProducerInterceptor<String, String> {
    @Override
    public ProducerRecord<String, String> onSend(ProducerRecord<String, String> record) {
        // 注入 TraceID（从 ThreadLocal/RPC Context 获取）
        String traceId = TraceContext.getTraceId();
        record.headers().add("X-Trace-Id", traceId.getBytes());
        // 注入发送时间戳
        record.headers().add("X-Send-Timestamp", 
            String.valueOf(System.currentTimeMillis()).getBytes());
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception e) {
        if (e != null) {
            metrics.recordSendFailure(metadata.topic(), metadata.partition());
        } else {
            metrics.recordSendSuccess(metadata.topic(), metadata.partition());
        }
    }
}

// 配置
props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,
    "com.example.EnrichInterceptor,com.example.MonitorInterceptor");
```

**常见使用场景**：
| 场景 | 实现 |
|------|------|
| 全链路追踪 | onSend() 注入 TraceID/SpanID 到 Headers |
| 消息审计 | onSend() 记录消息摘要，onAcknowledgement() 记录结果 |
| 消息过滤 | onSend() 中根据规则返回 null 跳过该消息 |
| 统一监控 | onAcknowledgement() 统一打点 QPS/失败率/延迟 |
| 抢占分片 | onSend() 中按业务规则修改 partition 字段 |

> **注意**：Interceptor 在业务线程中同步执行，不要在 onSend() 中做耗时操作（如查数据库）。只做轻量级的 Headers 注入和监控打点。

## 四、关键设计

### 为什么用异步发送 + 回调而不是同步发送？

| 方式 | 吞吐 | 延迟 | 适用场景 |
|------|------|------|---------|
| 异步（默认） | 最高 | 极低（send 立即返回） | 生产环境常用 |
| 异步 + 回调 | 高 | 低 | **推荐**，可感知结果 |
| 同步 `send().get()` | 低 | 高（阻塞等响应） | 强依赖发送结果的场景 |

> 不要在回调里手动重试——Producer 本身已有 retries 机制，双重重试会放大重复消息问题。

### 同步发送和异步发送的选型逻辑

- 99% 场景用异步 + 回调：吞吐高，失败通过监控发现
- 同步只用于"必须知道发送结果才能继续"的场景（如简单的 demo 或测试）

## 五、面试高频问题

**Q1：Producer 怎么知道发给哪个 Broker？**

Producer 首次连接 `bootstrap.servers` 中任意 Broker → 发送 Metadata 请求获取集群元数据（Topic 有哪些分区、每个分区的 Leader 在哪个 Broker）→ 缓存元数据 → 后续直接向 Leader Broker 发送。元数据会定期刷新（`metadata.max.age.ms`，默认 5 分钟）。

> **追问：元数据刷新期间 Leader 切换了怎么办？** 发送失败后 Producer 会收到 `NOT_LEADER_FOR_PARTITION` 错误 → 立即触发元数据强制刷新 → 获取新 Leader 信息 → 重试发送。这个流程是自动的。

**Q2：batch.size 和 linger.ms 什么关系？**

batch.size 是空间维度——消息攒够这么多字节就发；linger.ms 是时间维度——消息等这么久就算不够也发。两者取"先到者"。增大 batch.size 和 linger.ms 都能提高吞吐，但增加单条消息延迟。

**Q3：如何保证 Producer 发送顺序？**

单分区内顺序由两个机制保证：
- 未开幂等性：`max.in.flight.requests.per.connection=1`（同一时刻只有一个请求在飞）
- 开启幂等性：Kafka 通过 Sequence Number 机制自动检测并修正乱序，可设置 ≤5

**Q4：Producer 一直发不出去会怎样？**

`delivery.timeout.ms` 超时后，消息通过回调返回异常。缓冲区满时 `send()` 阻塞最多 `max.block.ms`。这两个超时都不会导致 Producer 崩溃，但会阻塞业务线程。

## 六、总结（速记版）

- **发送流程**：拦截 → 序列化 → 分区 → 缓冲区聚合 → Sender 线程发送
- **缓冲核心**：batch.size（空间维度）+ linger.ms（时间维度）
- **分区策略**：指定 partition > key 哈希 > 黏性分区
- **吞吐公式**：增大 batch + 增大 linger + 压缩 → 吞吐提升
- **可靠性铁三角**：acks + retries + 幂等性

---

# 四、Broker 存储与副本机制

## 一、背景（为什么需要）

Broker 是 Kafka 的核心——它如何存储数据、管理副本、选举 Leader，直接决定数据的可靠性、可用性和持久性。这是 Kafka 最区别于其他 MQ 的地方。

## 二、核心概念

### 副本管理三大概念

| 概念 | 全称 | 含义 |
|------|------|------|
| **AR** | Assigned Replicas | 一个分区的所有副本集合 |
| **ISR** | In-Sync Replicas | 与 Leader 保持同步的副本集合 |
| **OSR** | Out-of-Sync Replicas | 落后太多被踢出 ISR 的副本 |

```
AR = ISR + OSR
ISR = Leader + 同步正常的 Follower
```

Follower 落后太多 → 踢出 ISR → 追上后重新加入 ISR。这个动态机制保证 ISR 中始终是"可用且数据基本一致"的副本。

### 消费者可见性相关概念

| 概念 | 全称 | 含义 |
|------|------|------|
| **LEO** | Log End Offset | 当前副本日志末端位置 = 最新 offset + 1 |
| **HW** | High Watermark | 消费者可见的最大位置，= ISR 中所有副本 LEO 的最小值 |
| **LSO** | Last Stable Offset | 事务场景下消费者可见的稳定位置 |

> 只有 HW 之前的数据才对消费者可见。这保证了：消费者读到的数据，一定已被 ISR 中所有副本确认，Leader 切换后不会"消失"。

### 磁盘物理结构

```
Topic: order
  └── Partition-0/
        ├── 00000000000000000000.log       ← 消息数据
        ├── 00000000000000000000.index     ← offset → 物理位置（稀疏索引）
        ├── 00000000000000000000.timeindex ← 时间 → offset
        ├── 00000000000000000100.log       ← 下一个 Segment
        ├── 00000000000000000100.index
        └── ...
```

## 三、底层原理

### Controller 工作机制

Controller 是集群的"大脑"，由某个 Broker 担任：
- 监听 Broker 上下线（通过 ZK 临时节点或 KRaft 成员变更）
- 触发 Partition Leader 选举
- 维护 Leader/ISR 信息并同步给所有 Broker
- 管理分区副本状态变更

### ISR 机制原理

**为什么需要 ISR？** 如果等所有副本都同步才确认，一个慢副本会拖慢整个分区。ISR 允许只等"跟得上"的副本，在可靠性和性能之间取得平衡。

Follower 落后判断标准：`replica.lag.time.max.ms`（默认 30 秒）内未追上 Leader 的 LEO。

**为什么 Leader 只从 ISR 中选举？** ISR 中的副本数据与旧 Leader 基本一致，从 ISR 选举可以最小化数据丢失。如果从 OSR 选，新 Leader 可能落后很多，已提交的数据丢失。

**ISR 收缩与扩张的详细流程**：

```
ISR 收缩（Shrink）：
  1. Leader 追踪每个 Follower 的最后同步时间
  2. 某 Follower 超过 replica.lag.time.max.ms（默认 30s）未同步
  3. Leader 向 Controller 发起 ISR 变更请求
  4. Controller 更新 ZooKeeper/KRaft 中的 ISR 记录
  5. ISR 缩小 → 如果 min.insync.replicas 不满足了 → Producer 写入收到异常

ISR 扩张（Expand）：
  1. Follower 追上了 Leader 的 LEO
  2. Leader 检测到该 Follower 已同步
  3. Controller 更新 ISR → 重新加入
  4. ISR 扩大 → 满足 min.insync.replicas → 恢复正常写入
```

> **追问：ISR 收缩对 Producer 有什么影响？** 如果 `acks=all` 且 `min.insync.replicas=2`，ISR 从 3 个缩小到 2 个 → 仍然满足 → 写入正常。如果 ISR 从 3 个缩小到 1 个（只剩 Leader）→ 不满足 min.isr=2 → Producer 写入抛 `NotEnoughReplicasException`。这是一个重要的保护机制。

### HW/LEO 的副本一致性原理

```
Leader LEO=10, F1 LEO=8, F2 LEO=9
HW = min(10, 8, 9) = 8

消费者可以读到 offset 0~7
offset 8~9 对消费者不可见

→ 即使 Leader 此时宕机，新 Leader（如 F2，LEO=9）也有 offset 0~9 的数据
→ offset 10 的数据只有 Leader 有，Leader 宕机后丢失，但因为 HW=8 所以消费者从未见过
```

**Follower 故障恢复**：踢出 ISR → Leader 继续工作 → Follower 恢复后截断不一致日志 → 从 Leader 重新同步 → 追上后重新加入 ISR

**Leader 故障恢复**：Controller 检测 → 从 ISR 选新 Leader → 其他副本与新 Leader 对齐 → 高于 HW 的不一致数据被截断 → 继续服务

### Segment + 稀疏索引原理

**为什么要切 Segment？** 单个大文件无法按时间/大小删除历史数据，索引维护成本也高。切 Segment 后可以按文件维度删除、压缩和索引。

**稀疏索引原理**：不是每条消息都建索引（全量索引内存占用太大），只给间隔 offset 建索引。

```
查找 offset=500 的过程：
  .index 中二分查找 ≤500 的最近索引条目 → 找到 offset=450 的位置
  → 从 .log 的该位置顺序扫描 → 找到 offset=500 的消息
```

> 核心 trade-off：用几 KB 内存 + 毫秒级顺序扫描，换全量索引需要的 MB 级内存。

### 日志清理策略

| 策略 | 原理 | 适合场景 |
|------|------|---------|
| **delete** | 按时间/大小删除旧 Segment | 日志、埋点、行为事件 |
| **compact** | 相同 key 只保留最新 value | 用户状态、配置、changelog |

```
compact 示例：
offset=0 key=user1 value=v1
offset=3 key=user1 value=v2    → compact 后只保留 → key=user1 value=v3
offset=9 key=user1 value=v3
```

### Log Compaction 算法深入

Log Compaction 由后台 **Log Cleaner** 线程池执行：

```
Log Cleaner 工作流程：
1. 选择脏率（dirty ratio）最高的日志段
   脏率 = 该段中可被清理的字节数 / 总字节数
2. 从该段的起始 offset 开始顺序读取
3. 维护一个 key → (offset, value) 的映射表（使用内存哈希表）
4. 遇到重复 key → 覆盖映射表中的 entry
5. 遍历结束后，映射表中保留的是每个 key 的最新值
6. 将映射表中的条目写入新的 clean Segment
7. 删除旧的 dirty Segment
8. 更新索引文件
```

> **为什么要有脏率阈值？** 如果每次对所有 Segment 做 compaction，开销太大。只选"脏率最高"的段，用有限资源获取最大收益。`min.cleanable.dirty.ratio` 默认 0.5，即只有脏率 > 50% 的段才被清理。

**Compact 不删除的是什么？**
- 每个 key 的**最新** value 不会被删除（即使 key 没有出现在最新 Segment 中）
- 带特殊 null value 的"墓碑消息"（tombstone）——用于标记 key 已删除
- 墓碑消息在保留 `delete.retention.ms`（默认 24 小时）后会被彻底删除

**为什么墓碑消息要保留 24 小时？** 给滞后 Consumer 足够的时间看到"这个 key 已被删除"，否则滞后 Consumer 在新数据中没看到这个 key 会以为它还存在。

### 元数据管理：ZK → KRaft 演进

**ZooKeeper 模式下 ZK 存储的元数据**：
```
/brokers/ids/[0,1,2]  ← Broker 注册信息
/brokers/topics/<topic>/partitions/  ← Leader、ISR、AR 信息
/controller  ← 当前 Controller 的 Broker ID
/controller_epoch  ← Controller 版本号（防脑裂）
```

> ZK 不参与实际数据传输路径，只存元数据 + 通知机制。但 ZK 挂掉，元数据变更（Leader 切换）会受影响。

**为什么要迁到 KRaft？**

1. **减少运维复杂度**：Kafka 4.0 起一个进程搞定，无需单独维护 ZK 集群
2. **元数据一致性更强**：ZK 基于 Zab 异步同步，KRaft 基于 Raft 日志复制，一致性和恢复速度更好
3. **Controller 切换更快**：KRaft 模式下 Quorum 已有完整元数据日志，切换秒级
4. **大规模集群性能**：10 万+ Partition 时 ZK 的元数据变更通知成为瓶颈
5. **安全性统一**：一套认证和权限管理

**KRaft 防脑裂**：使用 Raft 协议的 term + 选举 quorum 机制。任何时刻最多只有一个 Active Controller 能获多数票。旧 Controller 发现自己 term 落后自动降级。

**KRaft 模式下的元数据日志结构**：

```
KRaft Metadata Log（类似 Raft Log）：
┌────────────────────────────────────────────┐
│ Entry 1: RegisterBroker(id=0, host=...)    │
│ Entry 2: RegisterBroker(id=1, host=...)    │
│ Entry 3: CreateTopic(name="orders", ...)   │
│ Entry 4: CreatePartition(topic="orders", ..)│
│ Entry 5: LeaderElection(partition=0, ...)  │
│ Entry 6: UpdateISR(partition=0, isr=[0,1]) │
│ ...                                         │
└────────────────────────────────────────────┘

所有 Quorum 节点的 Metadata Log 完全一致（通过 Raft 复制）
任何一个节点都可以通过重放 Log 恢复完整集群状态
```

### Preferred Leader Election 与自动均衡

当 Broker 宕机再恢复后，它的分区 Leader 已经被迁到其他 Broker。恢复后这个 Broker 上只有 Follower，导致 Leader 分布不均——这是生产中最常见的负载倾斜原因。

**Preferred Leader（优先 Leader）**：
- 每个 Partition 创建时的第一个副本就是 Preferred Leader
- Preferred Leader 通常按 RoundRobin 分配 → 保证初始 Leader 分布均匀
- Broker 宕机恢复后，Preferred Leader 仍是 Follower → Leader 分布不再均匀

```
Topic "orders", 3分区, 3副本, Broker 0/1/2
创建时（均匀）：
  P0: L=0, F=1,2     ← Preferred Leader = 0
  P1: L=1, F=2,0     ← Preferred Leader = 1
  P2: L=2, F=0,1     ← Preferred Leader = 2
  每个 Broker 一个 Leader ✅

Broker-0 宕机后（不均）：
  P0: L=1, F=2,0(宕机)  ← Leader 迁到 Broker-1
  P1: L=1, F=2,0(宕机)
  P2: L=2, F=0(宕机),1
  Broker-1: 2 个 Leader, Broker-2: 1 个 ❌

Broker-0 恢复后（仍不均）：
  P0: L=1, F=2,0
  P1: L=1, F=2,0
  P2: L=2, F=0,1
  Broker-1: 2 个 Leader, Broker-2: 1 个 ❌（不会自动恢复！）
```

**修复方式**：

```bash
# 方式 1：手动触发单 Topic 的 Preferred Leader Election
bin/kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type preferred --topic orders --partition 0

# 方式 2：触发所有 Topic 的 Preferred Leader Election
bin/kafka-leader-election.sh --bootstrap-server localhost:9092 \
  --election-type preferred --all-topic-partitions

# 方式 3：自动均衡（Broker 配置）
auto.leader.rebalance.enable=true          # 默认开启
leader.imbalance.check.interval.seconds=300  # 每 5 分钟检查
leader.imbalance.per.broker.percentage=10    # 不均衡超过 10% 触发
```

**自动均衡的工作机制**：
```
Controller 后台线程定期检查：
  1. 计算每个 Broker 的 Leader 数与平均数的偏差百分比
  2. 如果偏差 > leader.imbalance.per.broker.percentage（默认 10%）
  3. 自动对偏差最大的 Broker 触发 Preferred Leader Election
  4. 只迁移 Leader 角色（不搬迁数据！）→ 秒级完成
```

> **关键认知**：Preferred Leader Election 不涉及数据复制！它只是把 Leader 角色还给 Preferred Replica——因为 Preferred Replica 本来就同步了数据（在 ISR 中）。对 Producer/Consumer 的影响极小（短暂不可用 < 几秒）。

**面试追问：为什么宕机恢复后不自动还回去？**

为了防止 **Leader 反复切换（Flip-Flop）**。如果 Broker 网络不稳定——上来就当 Leader、又断、又上来——Leader 反复切换会导致 Producer/Consumer 频繁收到 `NOT_LEADER_FOR_PARTITION` 错误。Kafka 选择"让 Leader 留在稳定运行的 Broker 上，等管理员手动或后台慢慢均衡"。

### Follower Fetching（KIP-392，Kafka 2.4+）

传统上 Consumer 只能从 **Leader 副本** 读取数据。KIP-392 允许 Consumer 从**最近的 Follower 副本** 读取：

```
传统模式（读 Leader）：
  Consumer(DC-1) ───── 跨 DC 拉取 ─────→ Leader(DC-2)
  问题：跨 AZ 流量费用高、延迟高

Follower Fetching 模式（KIP-392）：
  Consumer(DC-1) ── 本地 AZ 拉取 ──→ Follower(DC-1)
  Follower 的数据和 Leader 同步（差距 < replica.lag.time.max.ms）
  优势：零跨 AZ 流量、延迟更低
```

**开启方式（Consumer 端配置）**：
```properties
# Consumer 告诉 Broker 自己所在的 rack
client.rack=dc1

# Broker 端需开启
replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
```

**Follower Fetching 的限制和设计考量**：
| 限制 | 原因 |
|------|------|
| 只能读 HW 之前的数据 | Follower 的 HW ≤ Leader 的 HW，读到的是已提交数据 |
| Follower 不在 ISR 中不可读 | OSR 中的数据可能与 Leader 不一致 |
| 默认关闭 | 需显式配置 `client.rack` + 部署 rack-aware 拓扑 |

> **面试追问：Follower 落后 Leader 了，Consumer 读到的数据会"延迟"吗？** 会。Consumer 读 Follower 时能读到的只有 Follower 的 HW 之前的数据。如果 Follower 落后 Leader 100ms，Consumer 看到的数据就有 100ms 延迟。这就是"读己之所写"（Read Your Writes）语义的代价——如果要读最新数据，仍需从 Leader 读。

### 网络分区与脑裂场景深度分析

面试中问到"两个机房断网"或"网络分区"时，需要按 ZK 模式和 KRaft 模式分别回答：

**场景 1：单集群跨 AZ 部署，AZ 间网络中断**

```
初始状态：3 Broker 分别位于 AZ-1/2/3，ZK/KRaft Controller 在 AZ-1

网络中断：AZ-1 与 AZ-2/3 隔离

┌──────────────┐        ┌──────────────┐
│   AZ-1       │   ✗    │  AZ-2 + AZ-3 │
│  Broker-1    │        │ Broker-2     │
│  Controller  │        │ Broker-3     │
│  多数分区 Leader │     │ 少数分区 Leader│
└──────────────┘        └──────────────┘
```

**ZK 模式下的行为**：
```
AZ-1（隔离侧,含 ZK 多数派）：
  - ZK 多数派在 AZ-1 → ZK 集群正常
  - Controller 维持领导地位
  - Broker-2/3 心跳超时 → Controller 认为它们宕机
  - Broker-2/3 上的 Leader 分区 → Controller 在 Broker-1 上选新 Leader
  - Producer 连接 Broker-1 → 正常写入 ✅
  - 但 Broker-1 只有 1 个副本 → min.isr=2 不满足 → 写入被拒绝 ❌

AZ-2/3（隔离侧,ZK 少数派）：
  - ZK 少数派 → ZK 无法选举、只读
  - 无法选举新 Controller → 无 Leader 可用
  - Producer 连接 Broker-2/3 → 写入失败 ❌
  - Consumer 也全部失败 ❌

结论：两个隔离区全部不可写！
```

**KRaft 模式下的行为**：
```
假设 Controller Quorum 在 AZ-1(2节点) + AZ-2(1节点)

AZ-1 隔离（Quorum: 2/3 节点）：
  - Raft 多数派存活 → Active Controller 继续工作
  - 能选举新 Leader（但 Broker-1 是唯一可用副本）
  - 同 ZK 模式：min.isr 不满足 → 写入拒绝 ❌

AZ-2/3 隔离（Quorum: 1/3 节点）：
  - Raft 少数派 → 无法选举 Controller
  - 写入/读取全部失败 ❌

KRaft 的改进只在 Controller 切换速度，不能绕过 min.isr 的可用性约束。
```

> **核心结论**：跨 AZ 部署时，**数据路径**（min.isr）才是可用性的瓶颈，不是 **元数据路径**（ZK vs KRaft）。如果要保证网络分区下的可用性，必须每个 AZ 都有 >= min.isr 个副本——这通常需要 >= 3 AZ 各至少 1 个副本，min.isr = 副本数 - 1。

**防脑裂的三种机制**：

| 机制 | 层面 | 如何防止 |
|------|------|---------|
| **Controller Epoch** | Controller 级别 | 每个 Controller 任期有唯一递增 epoch，旧 epoch 的请求被忽略 |
| **Leader Epoch** | Partition 级别 | 每次 Leader 变更 epoch +1，旧 Leader 发现自己 epoch 落后自动降级 |
| **KRaft Raft Term** | 元数据共识 | Raft 协议的 Term + Quorum 确保只有一个 Active Controller |

### Topic 删除的异步流程

`kafka-topics.sh --delete --topic xxx` 执行后，Topic 不是立刻从磁盘消失：

```
Topic 删除流程（异步，分阶段）：

阶段 1：逻辑删除（秒级）
  Controller 收到删除请求
  → ZK/KRaft 中将 Topic 标记为 deleted
  → 通知所有 Broker：该 Topic 的所有分区停止服务
  → Producer/Consumer 立即收到 UnknownTopicOrPartitionException

阶段 2：物理删除（异步，分钟~小时级）
  每个 Broker 的后台线程扫描本地磁盘
  → 发现标记为 deleted 的分区目录
  → 逐个删除 Segment 文件
  → 删除分区目录
  → 删除完成

阶段 3：元数据清理
  Controller 清理 Topic 相关的 ZK 节点 / KRaft 元数据
  → Topic 彻底从集群消失
```

> **为什么是异步的？** 一个 Topic 可能有数百个分区、数 TB 数据。同步删除会阻塞 Controller 事件循环线程，影响集群其他操作。异步删除时 Controller 可以继续处理 Leader 选举、Broker 上下线等关键事件。

**配置控制**：
```properties
# 每个 Broker 同时删除的分区数上限
delete.topic.enable=true                  # 是否允许删除
num.recovery.threads.per.data.dir=1       # 每个数据目录 1 个线程处理删除
```

**面试追问：删除 Topic 后磁盘没立即释放怎么办？** 这是正常行为，因为异步删除。通过 `kafka-topics.sh --list` 确认 Topic 已不在列表中即可。如果急需释放磁盘，可以用 `kafka-delete-records.sh` 提前清空数据或直接 SSH 到 Broker 手动 `rm -rf` 分区目录（高危操作，需停 Broker）。

### 多磁盘 JBOD 支持

Kafka 支持为一个 Broker 配置多个数据目录（JBOD: Just a Bunch Of Disks），不同于 RAID：

```properties
# server.properties - 多个目录逗号分隔
log.dirs=/data/kafka-logs-1,/data/kafka-logs-2,/data/kafka-logs-3
```

**Kafka 的磁盘分配策略（RoundRobin）**：
```
创建新分区时，Kafka 选择"分区数最少"的目录，而不是"剩余空间最多"的目录。

/data/kafka-logs-1：20 个分区，剩余 500GB
/data/kafka-logs-2：10 个分区，剩余 50GB  ← 剩余少但分区少
/data/kafka-logs-3：15 个分区，剩余 300GB

新分区 → 分配到 /data/kafka-logs-2（分区最少）
```

> **这是 Kafka 的一个已知局限**：不考虑剩余空间，只看分区数量。生产环境中如果磁盘容量不同，可能出现"分区分布均匀但空间分布严重不均"的情况。解决方案：使用相同容量的磁盘，或用 Cruise Control / kafka-reassign-partitions.sh 定期均衡。

**一块盘满了会怎样**：
```
/data/kafka-logs-2 磁盘写满
  → 该目录上的所有分区停止接受写入
  → Producer 写入这些分区时收到异常
  → 其他目录 (/data/kafka-logs-1,3) 正常
  → Broker 不会自动迁移分区到其他目录（这是 Kafka 的设计限制）

预防措施：
  - 使用磁盘容量告警（>80% 紧急处理）
  - 配置 log.retention.bytes 限制每个 Topic 的总存储量
  - 定期监控各目录的分区和磁盘使用情况
```

| | RAID | JBOD |
|------|------|------|
| 数据冗余 | RAID 5/6 提供冗余 | 不提供冗余，靠 Kafka 自身副本机制 |
| 容量利用 | 有损耗（RAID 5 损耗 1 块盘） | 100% 利用 |
| 单盘故障 | RAID 可容忍 1-2 块盘故障 | 故障盘上分区不可用，需重建 |
| 性能 | 受限于 RAID 控制器 | 多盘并行读写，吞吐更高 |
| Kafka 推荐 | 不推荐（副本已提供冗余，RAID 浪费） | **推荐**（更多并行 I/O + 副本保障） |

> **面试一句话**：Kafka 用 JBOD 而不是 RAID，因为 Kafka 的副本机制已经提供了数据冗余，RAID 是多余的。JBOD 提供多盘并行 I/O，代价是单盘故障时 Broker 上部分分区不可用（但副本在其他 Broker 上可用）。

### Tiered Storage（分层存储，KIP-405）

Kafka 3.6+ 引入的分层存储，实现**冷热数据分离**：

```
┌─────────────────────────────────────┐
│   Hot Tier（本地 SSD）               │
│   - 最近写入的数据                    │
│   - 高频读取的数据                    │
│   - 低延迟访问                       │
└─────────────────────────────────────┘
           │ 自动搬迁（按时间/大小策略）
           ↓
┌─────────────────────────────────────┐
│   Cold Tier（远程对象存储）           │
│   - S3 / GCS / Azure Blob Storage    │
│   - 历史数据                         │
│   - 高延迟但低成本                    │
│   - 对 Producer/Consumer 透明        │
└─────────────────────────────────────┘
```

**核心价值**：
1. **存储成本降低 80-90%**：对象存储比本地 SSD 便宜得多
2. **保留时间极大延长**：可以保留数月甚至数年的数据用于合规/回溯分析
3. **运维简化**：不再因为磁盘满而频繁扩容，计算和存储独立扩展
4. **容灾增强**：对象存储天然多 AZ 冗余

> **面试追问：Consumer 读冷数据时性能如何？** 延迟会增加（从 ms 级 → 100ms+ 级），但对历史数据回溯场景完全可接受。Kafka 会预取（prefetch）数据到本地缓存以降低后续请求的延迟。

## 四、关键设计

### 为什么 Leader 只从 ISR 选举？

ISR 中的副本与 Leader 数据基本一致（差距在 `replica.lag.time.max.ms` 内），从 ISR 选举最小化数据丢失。硬要等所有副本全同步再确认，性能和可用性都受影响——ISR 是可靠性和性能之间的动态平衡。

### 为什么用稀疏索引而不是 B+ 树？

Kafka 是 append-only 日志，不需要随机更新索引。稀疏索引：维护成本极低、内存占用极小、二分 + 顺序扫描够快。B+ 树适合需要随机读写和范围查询的场景（如数据库），Kafka 不需要。

### Rack Awareness（机架感知）

生产环境中 Kafka 集群通常跨多个可用区（AZ）或物理机架部署。如果同一 Partition 的副本全在同一个机架上，该机架断电 → 分区完全不可用。**Rack Awareness** 保证副本分布到不同机架：

```
broker.rack 配置：
  Broker-0: rack=az1
  Broker-1: rack=az2
  Broker-2: rack=az3

Topic "orders", 3 分区, 3 副本：
  Partition-0: Leader(Broker-0,az1), Follower(Broker-1,az2), Follower(Broker-2,az3)  ✅
  Partition-1: Leader(Broker-1,az2), Follower(Broker-2,az3), Follower(Broker-0,az1)  ✅
  Partition-2: Leader(Broker-2,az3), Follower(Broker-0,az1), Follower(Broker-1,az2)  ✅

每个分区的 3 个副本分属 3 个不同 AZ，任一 AZ 故障不影响整体可用性
```

**机架感知的两层保障**：

| 保障 | 机制 | 效果 |
|------|------|------|
| **副本分布** | 同一 Partition 的副本不放在同一 rack | AZ 故障时仍有副本可用 |
| **消费者就近读取** | Consumer 优先从同 rack 的副本拉取 | 减少跨 AZ 流量费用 |

> Kafka 3.4+ 支持 `client.rack` 配置，Consumer 带上自己的 rack 信息后，Broker 优先返回同 rack 的 Follower 副本（不一定是 Leader），利用 Follower Fetch 能力降低跨 AZ 延迟和成本。

**面试追问：Rack Awareness 和 min.insync.replicas 有什么关系？**
```
假设：3 AZ 各 1 个 Broker，min.isr=2，AZ-1 故障
  → AZ-1 的 Broker 宕机 → ISR 从 3 缩小到 2
  → 仍然满足 min.isr=2 → Producer 正常写入
  → AZ-1 恢复后自动追数据 → ISR 扩张回 3

如果 min.isr=3：
  → AZ-1 故障 → ISR 缩小到 2 → 不满足 min.isr=3
  → Producer 写入被拒绝 → 可用性受损

建议：min.isr < Broker 数(跨 AZ 时)，通常是 Broker 数 - 1
```

### 为什么不用 `O_DIRECT` 或 `O_SYNC`？

Kafka 信任 OS 的 PageCache 管理策略。`O_DIRECT` 绕过 PageCache 反而丢失了 OS 的预读和缓存能力。`O_SYNC` 每次写都同步刷盘，性能极差。Kafka 用异步刷盘 + 多副本保证可靠性。

## 五、面试高频问题

**Q1：ISR 和 AR 有什么区别？**

AR（Assigned Replicas）= 一个分区的所有副本。ISR（In-Sync Replicas）= AR 的子集，是与 Leader 保持同步的副本。AR = ISR + OSR。ISR 是动态变化的——Follower 落后被踢出，追上后重新加入。

**Q2：HW 是干什么的？**

HW（High Watermark）= ISR 中所有副本 LEO 的最小值。消费者只能读到 HW 之前的数据。这个机制保证：消费者读到的数据已经被 ISR 中所有副本确认，Leader 切换后这部分数据不会丢失。

**Q3：Kafka 为什么不支持减少分区？**

减少分区需要：重新分布海量数据、破坏 Offset 序列连续性、破坏 key 到分区的映射关系。Kafka 选择了简单可靠的设计——只增不减。

**Q4：Kafka 使用哪种索引结构？为什么？**

稀疏索引（sparse index）。不记录每条消息的位置，只记录间隔 offset 的位置。查找时二分定位最近索引点 → 顺序扫描少量数据。几 KB 内存换毫秒级查找，非常适合 append-only 的日志场景。

**Q5：Follower 拉取 vs Leader 推送？**

Follower 主动从 Leader 拉取（类似 Consumer 的 Pull 模型）。好处是 Follower 可以根据自身同步进度决定拉取量，不会因为 Leader 推送太快导致 Follower 跟不上。

**Q6：Log Compaction 的原理是什么？什么场景需要？**

后台 Log Cleaner 线程扫描日志段，对相同 key 的消息只保留最新 value。适合以 key-value 为模型的场景（如 CDC changelog、KTable 状态存储）。不适合日志/事件类数据——这些数据的每条消息都是独立的事件。

## 六、总结（速记版）

- **Controller**：集群协调者，管 Leader 选举和元数据同步
- **AR = ISR + OSR**：ISR 是"跟得上"的动态副本集
- **HW = min(所有 ISR 副本的 LEO)**：消费者只能读到 HW 之前
- **存储结构**：Partition → Segment(.log + .index + .timeindex)
- **稀疏索引**：少量内存换毫秒级查找
- **清理策略**：delete（按时间删）vs compact（按 key 保留最新）
- **KRaft 替代 ZK**：运维简化 + 一致性增强 + Controller 切换更快
- **Tiered Storage**：冷热分离，降低存储成本，延长保留时间

---

# 五、Kafka 性能原理

## 一、背景（为什么需要）

"Kafka 为什么快"是大厂面试必问题，考察的是对操作系统和 I/O 模型的理解深度。不能只背结论，要理解 PageCache、零拷贝、顺序写的底层原理。

## 二、核心概念

Kafka 高吞吐的**六大支柱**：

1. **分区并行** → 水平扩展
2. **顺序写磁盘** → 避免随机 I/O
3. **PageCache** → 利用 OS 内核缓存
4. **零拷贝（sendfile）** → 减少 CPU 拷贝和上下文切换
5. **批量发送** → 减少网络请求
6. **压缩** → 减少传输量和磁盘占用

## 三、底层原理

### 1. 顺序写磁盘

**为什么顺序写比随机写快？**

机械硬盘（HDD）：
```
随机写：寻道(3-15ms) + 旋转等待(2-4ms) + 写入(μs级)
顺序写：写入(μs级) ← 磁头已在正确位置
```

SSD：
- 顺序写 ≈ 500MB/s - 3GB/s
- 随机写 ≈ 50MB/s - 500MB/s
- 差距缩小但依然存在（原因是写放大——SSD 以 Page 读写，以 Block 擦除，随机写导致更多 Partial Page Write 和 GC，写放大可达 2-5 倍）

**Kafka 如何保证顺序写？**
1. Partition 是只追加日志文件，不修改已写入的数据
2. 消息按到达顺序追加到文件末尾
3. 每个 Partition 在磁盘上是连续的写入流

> 不修改已写入的数据——这是顺序写的根本前提。对比 MySQL B+Tree：更新一行需要找到 Page → 读入内存 → 修改 → 写回 → 随机 I/O。

### 2. PageCache

**为什么依赖 PageCache 而不是自己管理缓存？**

自己管理（JVM 堆缓存）的三个致命问题：
1. JVM 堆内存受 GC 管理，大堆 = 长 GC 停顿
2. 数据从内核态拷到 JVM 堆有一次拷贝开销
3. OS 已有成熟的页缓存机制，无需在用户态再造轮子

**PageCache 工作流程**：
```
写入：
  Kafka 调用 write() → 数据拷到 PageCache（内核态）→ write() 立即返回
  → OS 后台线程异步将脏页刷盘

读取：
  Kafka 调用 read() → 检查 PageCache
    → 命中（PageCache hot）→ 零拷贝返回（纯内存操作，极快）
    → 未命中 → 从磁盘读到 PageCache → 再返回
```

**为什么 PageCache 特别适合 Kafka？**
- Producer 写入有时间局部性：最新写入的数据很快被 Consumer 读取
- Consumer 顺序读取：OS 的预读（read-ahead）机制自动把后续数据加载到 PageCache
- Kafka 不修改数据：PageCache 中的页永远不会因写而变脏

**实践中的表现**：
```
正常消费（Consumer 跟上）：最新消息在 PageCache 中 → 纯内存速度
滞后消费（Consumer 落后很多）：旧消息被 OS 换出 PageCache → 磁盘 I/O 速度
```

> Kafka 不使用 `O_DIRECT`（绕过 PageCache），也不使用 `O_SYNC`（同步刷盘）。信任 OS 的 PageCache 策略。Broker 需要足够系统内存容纳活跃 Partition 的工作集。

### 3. 零拷贝（sendfile）

**传统方式（4 次拷贝 + 4 次上下文切换）**：
```
磁盘 →(DMA)→ 内核 PageCache →(CPU)→ 用户态 Buffer →(CPU)→ 内核 Socket Buffer →(DMA)→ 网卡
```

**零拷贝方式（2 次 DMA 拷贝 + 2 次上下文切换）**：
```
磁盘 →(DMA)→ 内核 PageCache →(DMA)→ 网卡
```

核心收益：消除 CPU 拷贝、减少上下文切换、提高网络发送效率。

**sendfile vs mmap**：
- mmap：将文件映射到用户态内存，省了一次读拷贝，但写网络时仍需 CPU 将数据从 mmap 区拷到 Socket Buffer
- sendfile：直接从 PageCache DMA 到网卡，彻底消除 CPU 拷贝
- Kafka 是"读磁盘 → 发网络"模式，sendfile 最合适

**零拷贝的适用条件**：
```
sendfile() 只在以下场景有效：
- Consumer 从磁盘读取数据并直接通过网络发送
- 不需要在用户态修改数据
- 使用普通的 TCP 传输（非 SSL）

❌ 不适用：
- Consumer 落后太多，数据不在 PageCache 中（需先读盘到 PageCache）
- 使用 SSL/TLS（需要在用户态加密）
- 需要对消息做格式转换/过滤
```

> **追问：SSL/TLS 模式下为什么不能用 sendfile？** SSL 加密需要在用户态操作数据（对称加密 + MAC 计算），sendfile 在内核态完成传输无法做加密。所以开启 SSL 会损失零拷贝的部分性能收益。这也就是为什么 Kafka 生产环境 SSL 终端通常在反向代理层做。

### 4. 批量发送

RecordAccumulator 按 TopicPartition 将消息聚合成 ProducerBatch → 一次网络请求发一批消息。batch.size + linger.ms 控制批次大小和时间。

### 5. 压缩

Producer 端压缩 → Broker 原样存储（不解压）→ Consumer 端解压。减少网络 I/O 和磁盘 I/O，代价是 CPU。

压缩算法选择：
| 算法 | 压缩率 | 速度 | 适用 |
|------|--------|------|------|
| snappy | 中等 | 快 | 平衡之选 |
| lz4 | 中等 | 最快 | 延迟敏感 |
| zstd | 最高 | 中等 | 带宽/磁盘敏感 |
| gzip | 高 | 慢 | 归档场景 |

### 6. 分区并行

Topic 拆成多个 Partition，分布在不同 Broker → 多 Broker 并行写 + 多 Consumer 并行读 → 吞吐水平扩展。

### 端到端延迟分解

理解 Kafka 的延迟构成有助于性能调优：

```
端到端延迟 ≈ Producer 攒批时间(linger.ms)
            + 网络 RTT(Producer → Leader)
            + Broker 追加写入时间(PageCache，< 1ms)
            + Follower 同步时间(acks=all 时才需要等)
            + 网络 RTT(Leader → Consumer)
            + Consumer poll 间隔

典型值（linger.ms=5ms, acks=1, 正常网络）：
  P50: 5-10ms
  P99: 10-50ms
  P999: 50-200ms（受 GC、网络抖动影响）
```

**延迟峰值常见原因**：
1. `linger.ms` 设置过大（攒批时间成为基础延迟）
2. `acks=all` 且 ISR 中有慢副本（等待最慢的 Follower 确认）
3. Broker GC 停顿（尤其是 ZK 模式下的老年代 GC）
4. 网络拥塞或丢包重传
5. Consumer poll 间隔过大

### JVM GC 对 Kafka 性能的影响

Kafka Broker 是 JVM 应用，GC 停顿会直接影响整个集群的延迟和可用性：

**GC 导致的级联故障链**：
```
Broker GC Full GC 停顿（2-5 秒）
  → 该 Broker 上的所有 Leader 分区暂停响应
  → Producer 写入超时 → 触发重试 → 放大流量
  → Follower 无法拉取数据 → 超过 replica.lag.time.max.ms
  → ISR 收缩 → 恢复后 ISR 扩张 → 元数据震荡
  → Controller 如果恰好 GC → ZK 会话超时 → 误判 Controller 宕机
  → 触发不必要的 Controller 重选举
  → 集群范围影响
```

**GC 对 Controller 的影响尤为严重**：
- ZK 模式下：Controller 与 ZK 有会话超时（默认 18s），Full GC 超过 18s → 会话断开 → Controller 被误判宕机 → 新 Controller 选举 → 原 Controller 恢复后发现不再是 Controller → 自杀重启（称为 "Controller 脑裂自杀"）
- KRaft 模式下：类似，通过 Raft 选举超时机制处理，但切换更快

**GC 调优建议**：

| 配置 | 建议值 | 原因 |
|------|--------|------|
| GC 算法 | **G1GC** | 可预期停顿时间，CMS 已废弃 |
| `MaxGCPauseMillis` | 20-50ms | 平衡吞吐和延迟 |
| 堆大小 | 6-8GB（非越大越好） | 太大 → Full GC 时间长；太小 → Young GC 频繁 |
| `InitiatingHeapOccupancyPercent` | 45-55 | 提前触发并发标记，避免 Full GC |
| `-XX:+DisableExplicitGC` | 开启 | 防止 `System.gc()` 触发 Full GC |
| PageCache 内存 | 堆外的 OS 内存 ≥ 堆大小 | 给 PageCache 留足够空间 |

> **核心认知**：Kafka 的性能不只取决于 JVM 堆大小，更取决于给 OS 留了多少系统内存给 PageCache。堆 6GB + PageCache 20GB >> 堆 26GB + PageCache 0。要始终给 OS 留足够内存——这是 Kafka 比传统 Java 应用更特殊的地方。

## 四、关键设计

### 六大机制的叠加效应

单独拿出任何一个机制都不足以解释 Kafka 的高性能。这六项是叠加和相互增强的关系：

- 分区并行 + 顺序写：多台机器的多块磁盘同时顺序写
- PageCache + 零拷贝：数据从磁盘到网卡几乎不经过 CPU
- 批量 + 压缩：一条 TCP 连接发一捆压缩后的消息

### 为什么不自己管理缓存？

这是区分"会用 Kafka"和"理解 Kafka"的关键问题。自己管理缓存（JVM 堆）有三个致命问题：GC 停顿、额外内存拷贝、重复造轮子。Kafka 选择了信任 OS，把精力集中在"怎样让 OS 的 PageCache 更有效"上——比如顺序写有利于预读、分区设计让工作集可控。

## 五、面试高频问题

**Q1：Kafka 为什么比 RabbitMQ 快？**

架构差异决定了性能上限不同：
1. Kafka 是日志型存储，顺序追加写；RabbitMQ 是队列型，涉及索引维护、消息删除
2. Kafka 用 Partition 水平扩展；RabbitMQ 以单节点为主
3. Kafka 消费者批量 Pull；RabbitMQ 逐条 Push
4. Kafka 用零拷贝（sendfile）直接 DMA 传输；RabbitMQ 需要经过 Erlang 虚拟机

**Q2：零拷贝具体怎么实现的？**

Linux `sendfile()` 系统调用。数据从磁盘 DMA 到内核 PageCache，再 DMA 到网卡，不需要经过用户态。一次 `sendfile()` 调用完成两次 DMA 拷贝，省去了把数据从内核态拷到用户态再拷回内核态的两步 CPU 拷贝。

**Q3：PageCache 满了怎么办？**

OS 会自动淘汰冷页（LRU 策略），Kafka 无法直接控制。如果工作集远超内存 → 频繁 PageCache miss → 性能下降 → 需要加内存或减少活跃 Partition 数。

> **追问：怎么判断 PageCache 是否够用？** 查看 Broker 的 `pagecache-hit-rate` 指标，或间接看 consumer fetch 的延迟——如果最新消费的延迟也高（> 100ms），说明数据不在 PageCache 中。

**Q4：SSD 时代顺序写还重要吗？**

重要，但原因变了。SSD 没有寻道时间，但随机写导致写放大——SSD 以 Page(4KB) 读写、以 Block(几 MB) 擦除，随机写触发更多 Partial Page Write 和 GC。顺序写可以整 Block 写入，写放大接近 1，延长 SSD 寿命。

## 六、总结（速记版）

- **六大支柱**：分区并行 + 顺序写 + PageCache + 零拷贝 + 批量 + 压缩
- **顺序写**：append-only 不修改，避免随机 I/O
- **PageCache**：OS 内核缓存，绕过 JVM GC，利用预读
- **零拷贝**：sendfile，DMA 直接搬运，省 CPU
- **批量 + 压缩**：摊薄网络开销，snappy/lz4/zstd 选其一

---

# 六、Consumer 消费者

## 一、背景（为什么需要）

Consumer 是数据的出口端，涉及消费模型、Rebalance、Offset 管理三个核心问题。线上 90% 的 Kafka 问题出在消费端。理解 Rebalance 机制是面试区分度所在。

## 二、核心概念

### 消费模型：Pull

Consumer 通过 `poll()` 主动拉取消息：

```
while (true) {
    records = consumer.poll(Duration.ofMillis(1000));
    process(records);
    consumer.commitSync(); // 或 commitAsync()
}
```

### 核心参数

| 参数 | 作用 | 关键认知 |
|------|------|---------|
| `group.id` | 消费者组 ID | 决定组归属，相同的组共享分区 |
| `enable.auto.commit` | 自动提交 Offset | **生产建议关闭** |
| `auto.offset.reset` | 无 Offset 时从哪开始 | earliest / latest / none |
| `max.poll.records` | 单次 poll 最大条数 | 影响处理压力和 Rebalance 风险 |
| `max.poll.interval.ms` | 两次 poll 最大间隔 | 超时 → Rebalance |
| `session.timeout.ms` | 会话超时 | 心跳超时 → 踢出组 |
| `heartbeat.interval.ms` | 心跳间隔 | 通常 < session.timeout.ms / 3 |
| `partition.assignment.strategy` | 分区分配策略 | Range / RoundRobin / Sticky / CooperativeSticky |
| `fetch.min.bytes` | 最少拉取字节数 | Broker 攒够这么多数据才返回 |
| `fetch.max.wait.ms` | 拉取等待超时 | 攒不够 min.bytes 时，最多等这么久 |
| `fetch.max.bytes` | 单次拉取最大字节数 | 必须 ≥ 单条最大消息 |
| `isolation.level` | 事务隔离级别 | read_uncommitted / read_committed |

### Consumer Fetch 协议细节

Consumer 拉取消息的完整流程：

```
Consumer 端                             Broker 端（Leader）
    │                                        │
    │──── FetchRequest ───────────────────→  │
    │     partition, offset,                 │
    │     min_bytes, max_bytes,              │
    │     max_wait_ms                        │
    │                                        │
    │                                  检查 PageCache/磁盘
    │                                  如果数据量 < min_bytes：
    │                                    等待新数据到来...
    │                                    最多等 max_wait_ms
    │                                  如果数据量 ≥ min_bytes：
    │                                    立即返回
    │                                        │
    │←─── FetchResponse ─────────────────── │
    │      records (batch), throttle_time    │
    │                                        │
    Consumer 处理 → 提交 Offset → 再次 Fetch     │
```

> **关键设计理念**：`fetch.min.bytes` + `fetch.max.wait.ms` 共同解决 Pull 模型的"空拉"问题——让 Broker 替 Consumer 攒数据，减少了 Consumer 空拉的开销，同时保证了低延迟（超时机制）。

### 三个易混淆的超时参数

| 参数 | 判断什么 | 触发后果 |
|------|---------|---------|
| `heartbeat.interval.ms` | 多久发一次心跳 | 不直接踢人 |
| `session.timeout.ms` | Consumer 是否存活 | 心跳超时 → 踢出组 |
| `max.poll.interval.ms` | Consumer 是否正常 poll | 两次 poll 间隔太长 → Rebalance |

> **关键结论**：心跳正常 ≠ 消费正常。心跳由后台线程发送，但业务线程卡住了不 poll，超过 `max.poll.interval.ms` 仍然触发 Rebalance。

## 三、底层原理

### Consumer Group 初始化流程

```
1. Consumer-1 和 Consumer-2 向 Coordinator 发送 JoinGroup 请求
2. Coordinator 选出 Consumer Leader（第一个加入的 Consumer）
3. Consumer Leader 制定分区分配方案
4. Consumer Leader 通过 SyncGroup 提交方案
5. Coordinator 向所有 Consumer 下发分配结果
6. Consumer 开始消费 + 维持心跳
```

**Coordinator 定位规则**：`hash(group.id) % __consumer_offsets 分区数` → 对应分区 Leader 所在 Broker 即 Coordinator。

> Consumer Leader ≠ Broker Leader。Consumer Leader 是消费者组内负责制定分配方案的消费者。

### Consumer Group 状态机（五种状态）

理解 Consumer Group 的状态转换是深入理解 Rebalance 的关键：

```
                    ┌──────────────────────────┐
                    │         Empty             │  ← 组不存在或所有成员已离开
                    │  (无成员,无分配)           │
                    └──────────┬───────────────┘
                               │ Consumer 加入
                               ↓
                    ┌──────────────────────────┐
                    │    PreparingRebalance     │  ← Coordinator 收到 JoinGroup 请求
                    │  (等待所有成员 JoinGroup)   │     暂停消费，生成新分配方案
                    └──────────┬───────────────┘
                               │ 所有成员 JoinGroup 完成
                               ↓
                    ┌──────────────────────────┐
                    │    CompletingRebalance    │  ← Consumer Leader 提交分配方案
                    │  (通过 SyncGroup 下发方案)  │     Coordinator 向所有成员下发结果
                    └──────────┬───────────────┘
                               │ 分配方案下发完成
                               ↓
                    ┌──────────────────────────┐
                    │          Stable           │  ← 正常消费状态
                    │  (正在消费,维持心跳)        │
                    └──────────┬───────────────┘
                               │ Consumer 离开/超时
                               ↓
                    ┌──────────────────────────┐
                    │    PreparingRebalance     │  ← 重新进入 Rebalance
                    │          ...              │
                    └──────────────────────────┘

                    所有 Consumer 离开
                               │
                               ↓
                    ┌──────────────────────────┐
                    │          Dead             │  ← 组彻底消亡（元数据被清理）
                    └──────────────────────────┘
```

**状态转换详解**：

| 状态 | 消费是否进行 | 触发下一状态的条件 |
|------|------------|------------------|
| **Empty** | 否 | Consumer 发送 JoinGroup |
| **PreparingRebalance** | **暂停所有消费** | 所有成员完成 JoinGroup |
| **CompletingRebalance** | **暂停所有消费** | SyncGroup 完成 |
| **Stable** | ✅ 正常消费 | Consumer 离开/超时/分区变化 |
| **Dead** | 否 | 长时间无活动，元数据被 GC |

> **面试关键结论**：消费只在 **Stable** 状态下进行。PreparingRebalance 和 CompletingRebalance 都是 STW 状态——这就是 Rebalance 导致消费暂停的根本原因。

**Eager vs Cooperative 的状态机差异**：

```
Eager 协议：
  Stable → PreparingRebalance（全停）→ CompletingRebalance（全停）→ Stable

Cooperative 协议：
  Stable → PreparingRebalance（只暂停需迁移的分区）→ CompletingRebalance → Stable
           ↑ 未迁移分区继续消费！                           ↑ 迁移完成
```

> **追问：为什么 Eager 协议要全部暂停？** 因为 Eager 协议依赖"所有 Consumer 重新加入 → Consumer Leader 拿到完整成员列表 → 全局重新分配"这个流程。如果部分 Consumer 还在消费，分配方案可能冲突。Cooperative 协议通过多轮协商，每次只迁一部分分区来解决这个问题。

### Rebalance 机制

**触发条件**：

| 触发条件 | 说明 | 严重程度 |
|---------|------|---------|
| Consumer 正常加入/退出 | 启动或正常关闭 | 低 |
| Consumer 异常宕机 | 进程崩溃、OOM、kill -9 | 高 |
| 心跳超时 | `session.timeout.ms` 内未收到心跳 | 高 |
| Poll 超时 | 两次 `poll()` 间隔 > `max.poll.interval.ms` | 高（需告警） |
| Topic 分区数变化 | Admin 增加了分区 | 中 |
| 订阅 Topic 变化 | `subscribe()` 改变订阅 | 低 |

**Rebalance 协议演进**：

| 版本 | 特点 | 问题 |
|------|------|------|
| **Eager**（老） | Rebalance 期间**所有 Consumer 停止消费** | Stop-The-World，期间大量积压 |
| **Cooperative**（2.4+） | **增量式再平衡**，只迁移必要分区 | 更平滑，未迁移分区继续消费 |

**Eager Rebalance 流程**：
```
所有 Consumer 暂停消费 → 全部重新 JoinGroup → 重新分配 → 恢复消费
                   ↑ STW 开始                          ↑ STW 结束
```

**CooperativeSticky 改进**：
```
Consumer 不暂停 → 计算最少迁移分区数 → 只对需迁移的分区：暂停→重新分配→恢复
```

### CooperativeSticky 算法核心思想

```
假设：
  当前分配：C0→[P0,P1], C1→[P2,P3], C2→[P4,P5]
  新 Consumer C3 加入，目标分配：C0→[P0,P2], C1→[P1,P3], C2→[P4], C3→[P5]

Eager 模式：
  全部暂停 → 全部分区重新分配 → 全部恢复
  影响：12 个分区全部暂停

CooperativeSticky 模式：
  只迁移分区：P0,P1,P2,P3（受影响的 C0 和 C1），P4,P5 不变
  C2 继续消费 P4 → 不暂停
  C3 直接开始消费 P5 → 不暂停
  影响：仅 4 个分区暂停

核心公式：只迁移"分配发生变化"的分区
```

### 频繁 Rebalance 为什么是"生产杀手"

```
场景：Consumer 处理耗时不稳定（有时 100ms，有时 5s）

→ 一次 poll 拉 500 条，赶上大消息，处理了 6 分钟
→ 超过 max.poll.interval.ms=5min → 触发 Rebalance
→ Rebalance 期间所有 Consumer 暂停 30 秒
→ 积压增加 → 恢复后拉更多消息 → 处理更慢
→ 再次触发 Rebalance → 恶性循环，集群"僵住"
```

**优化方向**：
1. `max.poll.records` 设合理值（如 500），不贪多
2. `max.poll.interval.ms` 设足够大（如 10 分钟），给慢处理留 buffer
3. 单条慢处理设置超时 + 跳过 + 写入死信队列
4. 使用 CooperativeSticky 减少 Rebalance 影响范围
5. 静态成员（`group.instance.id`）：Consumer 重启后 Coordinator 会等它回来而不是立刻触发 Rebalance
6. 监控 Rebalance 频率：每分钟都有 = 配置或业务有严重问题

### 分区分配策略

| 策略 | 方式 | 特点 |
|------|------|------|
| **Range** | 按 Topic 单独分配 | 多 Topic 场景前几个 Consumer 负载偏高 |
| **RoundRobin** | 所有分区轮询分配 | 更均匀 |
| **Sticky** | 尽量均匀 + 保持上次分配 | 减少分区迁移 |
| **CooperativeSticky** | **增量式**再平衡 | 只迁移必要分区，减少暂停 |

**Range 策略为什么不均匀（具体例子）**：

```
Topic-A: 3 个分区, Topic-B: 3 个分区, 2 个 Consumer

Range 分配（每个 Topic 单独算）：
  Topic-A: C0→[PA0,PA1], C1→[PA2]       ← C0 2个，C1 1个
  Topic-B: C0→[PB0,PB1], C1→[PB2]       ← C0 2个，C1 1个
  总计: C0=4 个分区, C1=2 个分区           ← 严重不均！

RoundRobin 分配（所有分区统一排）：
  [PA0,PA1,PA2,PB0,PB1,PB2] → C0→[PA0,PA2,PB1], C1→[PA1,PB0,PB2]
  总计: C0=3 个分区, C1=3 个分区           ← 均匀
```

> 多 Topic 订阅场景，尽量用 RoundRobin 或 Sticky，避免 Range。

### Consumer Interceptor（消费者拦截器）

Consumer 也有 Interceptor 链，在 `poll()` 返回结果前执行：

```
ConsumerInterceptor 接口：
  onConsume(ConsumerRecords) → ConsumerRecords   // poll() 返回前：可过滤/修改消息
  onCommit(Map<TopicPartition, OffsetAndMetadata>)  // commit 时：可修改提交的 Offset
```

**典型用途**：
```java
// 示例：消费端延迟监控
public class LagMonitorInterceptor implements ConsumerInterceptor<String, String> {
    @Override
    public ConsumerRecords<String, String> onConsume(ConsumerRecords<String, String> records) {
        for (ConsumerRecord<String, String> r : records) {
            long lag = System.currentTimeMillis() - r.timestamp();
            metrics.recordLag(r.topic(), r.partition(), lag);
            if (lag > 60_000) {  // 超过 1 分钟的陈旧数据告警
                log.warn("Stale message: topic={}, partition={}, lag={}ms", 
                    r.topic(), r.partition(), lag);
            }
        }
        return records;
    }
}
```

**常见使用场景**：
| 场景 | 实现 |
|------|------|
| 消费延迟监控 | onConsume() 中计算 `now - record.timestamp()` |
| 消息过滤 | onConsume() 中过滤掉不需要的消息（如脏数据） |
| 消息有效性检验 | onConsume() 中校验必备字段，不合格的 skip 或写死信 |
| 全局去重 | onConsume() 中查 Redis/BloomFilter，命中则跳过 |
| 批量限流 | onConsume() 中 sleep 控制消费速率 |

> **与 Interceptor 在业务处理外的区别**：Interceptor 在 poll() 返回前执行，跨所有 Consumer 实例生效，是全局的"横切关注点"。业务特定的处理逻辑仍应在 ConsumeClaim / 消息处理函数中实现。

## 四、关键设计

### 为什么是 Pull 而不是 Push？

| 模型 | 优点 | 缺点 |
|------|------|------|
| Push | 实时性强 | 容易推太快压垮 Consumer |
| Pull | 按能力消费，可批量拉取，灵活 Offset 管理 | 无数据时可能空拉 |

Kafka 选择 Pull 的核心原因：Consumer 最清楚自己的处理能力，让 Consumer 控制消费速度比让 Broker 猜测更合理。空拉问题通过 `fetch.min.bytes`（至少攒够多少才返回）+ `fetch.max.wait.ms`（最多等多久）缓解。

### 为什么 Rebalance 是"必要的恶"

Rebalance 是 Consumer Group 动态负载均衡的机制——没有它，Consumer 加入/退出后负载无法自动调整。但它的代价是消费暂停和重复消费风险。实践中不是"消灭" Rebalance，而是"减少"它——通过合理配置让 Rebalance 只在真正需要时触发。

## 五、面试高频问题

**Q1：Rebalance 什么时候触发？怎么减少？**

触发：Consumer 上下线、心跳超时、poll 超时、分区数变化、订阅变化。减少：使用静态成员、CooperativeSticky 策略、合理设置 `max.poll.records` 和 `max.poll.interval.ms`、分离 poll 和处理线程。

**Q2：Kafka 为什么用 Pull 不用 Push？**

Consumer 按自身处理能力控制消费速度，避免 Broker 推太快压垮 Consumer，且便于批量拉取和灵活管理 Offset。空拉问题通过 `fetch.min.bytes` 配合 `fetch.max.wait.ms` 缓解。

**Q3：session.timeout.ms 和 max.poll.interval.ms 的区别？**

`session.timeout.ms` 判断 Consumer 是否存活（心跳层面），超时 → 踢出组；`max.poll.interval.ms` 判断 Consumer 是否正常消费（poll 层面），超时 → Rebalance。心跳正常不代表消费正常——后台心跳线程独立于 poll 线程。

**Q4：怎么彻底避免 Rebalance？**

无法"彻底避免"，只能减少。Rebalance 是 Consumer Group 动态负载均衡的基础机制。如果消费者稳定且处理耗时受控，可调大超时参数减少误触发。

**Q5：fetch.min.bytes 和 fetch.max.wait.ms 如何配合？**

Consumer 发 FetchRequest 时指定这两个参数。如果 Broker 端数据量 < `fetch.min.bytes`，Broker 就等着新数据进来，但最多等 `fetch.max.wait.ms`。两者配合解决"空拉"又保证低延迟——比如 `min.bytes=1MB, max.wait=500ms`，没有 1MB 数据时最多等 500ms 就返回现有数据。

## 六、总结（速记版）

- **Pull 模型**：消费者主动拉，可控制速度
- **Coordinator**：管理消费者组的 Broker，`hash(groupId) % __consumer_offsets 分区数` 定位
- **Rebalance 触发**：成员变化、分区变化、超时（心跳/poll）
- **Rebalance 危害**：消费暂停、积压、重复消费
- **优化**：静态成员 + CooperativeSticky + 合理 max.poll.records + 解耦 poll 和处理
- **Fetch 协议**：min.bytes + max.wait.ms 配合解决空拉

---

# 七、Offset 管理

## 一、背景（为什么需要）

Offset 是 Kafka 消费进度的唯一标识。提交策略直接决定消息是重复还是丢失。这是生产中最常见问题区。

## 二、核心概念

Offset 是 Consumer 在某个 Partition 上消费到的位置，分区内单调递增，不同分区独立。

```
Partition-0: [offset=0] [offset=1] [offset=2] [offset=3] [offset=4] [offset=5]
                                                              ↑
                                                        Consumer 当前位置
```

注意：Consumer 提交的是**下一次要消费的位置**（不是已消费的最后一条）。

**存储位置**：
- Kafka 0.9 以前：ZooKeeper
- Kafka 0.9 以后：内部 Topic `__consumer_offsets`（默认 50 分区，3 副本）

### `__consumer_offsets` 内部结构

```
__consumer_offsets（内部 Topic，默认 compact 策略）
Key:   [GroupID, Topic, Partition]  的字节编码
Value: [Offset, Metadata, Timestamp, ...]

为什么用 compact 而不是 delete？
  → 一个 Consumer Group 对每个 TopicPartition 只需要最新的 Offset
  → 旧的 Offset 值自动被 compact 掉
  → 减少内部 Topic 的存储开销
```

> **追问：为什么是 50 个分区？** `__consumer_offsets` 的分区数 = `offsets.topic.num.partitions`（默认 50）。这决定了 Coordinator 的并发度——50 个分区意味着最多 50 个 Broker 可以同时担任 Coordinator。对于大多数集群，50 足够；超大集群（数千 Consumer Group）可适当增加。

## 三、底层原理

### 自动提交 vs 手动提交

| 方式 | 行为 | 风险 |
|------|------|------|
| 自动提交 `enable.auto.commit=true` | 每隔 `auto.commit.interval.ms` 自动提交 | 消息没处理完就提交 → 漏消费；处理完了没提交就宕机 → 重复消费 |
| `commitSync()` | 同步提交，阻塞等待 Broker 确认 | 阻塞消费线程，影响吞吐；但失败自动重试 |
| `commitAsync()` | 异步提交，不阻塞 | 失败不会自动重试（重试可能导致乱序覆盖） |

**生产建议**：关闭自动提交，在业务处理成功后手动提交。推荐 `commitAsync` + 异常时 `commitSync` 兜底。

### 重复消费和漏消费的成因

```
重复消费：
  拉取消息 → 业务处理成功 → Offset 还没提交 → 宕机 → 重启后从旧 Offset 重新消费
  → 解决：业务侧幂等（唯一键、去重表、Redis 去重、状态机）

漏消费：
  拉取消息 → 先提交 Offset → 业务还没处理 → 宕机 → 重启后从新 Offset 开始
  → 避免：永远先处理业务，成功后再提交 Offset
```

### commitAsync 为什么不自动重试？

假设先提交 offset=100（失败），再提交 offset=200（成功），如果此时重试提交 offset=100 成功了 → 覆盖掉了 offset=200 → 后续消费从 100 开始 → 重复消费。Kafka 选择不在 commitAsync 内自动重试，由调用方决定如何处理。

## 四、关键设计

### 为什么 Offset 从 ZK 迁移到内部 Topic？

1. ZK 不是设计用来做高频写操作的（每次消费都要写 Offset）
2. 内部 Topic 和普通 Topic 享受一样的持久化和副本保障
3. 减少对 ZK 的依赖（为最终去 ZK 做铺垫）

### auto.offset.reset 的三个值

| 值 | 含义 | 适用场景 |
|------|------|---------|
| `latest` | 从最新位置开始（默认） | 新消费者只关心新数据 |
| `earliest` | 从最早位置开始 | 新消费者需要全量历史数据 |
| `none` | 没 Offset 就抛异常 | 必须从已有 Offset 消费，否则报错 |

## 五、面试高频问题

**Q1：你们项目用自动提交还是手动提交？为什么？**

手动提交。自动提交的时机不可控——可能消息还没处理完就提交了，也可能处理完了但还没提交。手动提交让我们精确控制"处理成功后提交"，配合业务幂等实现至少一次语义。

**Q2：重复消费怎么解决？**

根本原因：业务处理成功 + Offset 未提交 = 下次会重新消费。解决：业务侧幂等——数据库唯一键（INSERT IGNORE / ON DUPLICATE KEY）、Redis 去重（SET NX）、业务状态机检查当前状态再决定是否处理。

**Q3：消息丢失（漏消费）怎么解决？**

永远先处理业务，成功后再提交 Offset。反过来（先提交再处理）如果业务失败，Offset 已经前进，消息就丢了。

**Q4：commitAsync 失败了怎么办？**

记录失败、下次 poll 时重试、或最终用 `commitSync` 兜底。关键是不能让 async 重试覆盖了更新的提交。

**Q5：怎么从指定时间点回溯消费？**

```java
Map<TopicPartition, OffsetAndTimestamp> offsets = consumer.offsetsForTimes(timestampMap);
consumer.seek(topicPartition, offset);
```

## 六、总结（速记版）

- **存储**：`__consumer_offsets`（Kafka 内部 Topic，compact 策略）
- **手动提交**：先处理业务，成功后提交 → 至少一次语义
- **重复消费**：处理成功但 Offset 未提交 → 业务幂等解决
- **漏消费**：先提交后处理 → 永远先处理后提交
- **回溯**：`offsetsForTimes()` + `seek()`

---

# 八、可靠性保障

## 一、背景（为什么需要）

消息不丢、不重、有序是消息系统的三大目标。Kafka 通过多层机制逼近这些目标，但各有边界条件。能说清楚三层保障的边界和局限，是大厂面试通过的标志。

## 二、核心概念

### 三种消息语义

| 语义 | 含义 | 实现方式 |
|------|------|---------|
| **At Most Once** | 最多一次，可能丢 | acks=0，自动提交 Offset |
| **At Least Once** | 至少一次，可能重复 | acks=1/all，先处理后提交 |
| **Exactly Once** | 精准一次 | 幂等性 + 事务 + 业务幂等 |

### acks 三种级别

| acks | 行为 | 可靠性 | 丢数据场景 |
|------|------|--------|-----------|
| **0** | 发出即成功 | 最低 | 网络问题、Broker 宕机 |
| **1** | Leader 写入即返回 | 中等 | Leader 写入后同步前宕机 |
| **all/-1** | ISR 全部确认 | 最高 | 所有副本同时宕机（概率极低） |

## 三、底层原理

### 消息不丢的完整配置（三端配合）

```properties
# Producer 端
acks=all
retries>0
enable.idempotence=true

# Broker 端
replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false

# Consumer 端
enable.auto.commit=false
# 业务处理成功后再手动提交 Offset
```

> `acks=all` 不是绝对不丢。如果 `replication.factor=1` 或 `min.insync.replicas=1`，Leader 单点仍可能丢数据。

### 幂等性原理（解决 Producer 重试导致的单分区重复）

**步骤 1：Producer 获取 PID（Producer ID）**
首次连接时 Broker 分配全局唯一 PID，整个 Producer 会话期间不变。存储在 `__transaction_state` 内部 Topic。

**步骤 2：每条消息分配递增 Sequence Number**
Producer 为每个 `<PID, TopicPartition>` 维护递增计数器。每次向同一分区发消息，SeqNumber +1。

**步骤 3：Broker 端去重判断**
```
if incoming.SeqNumber <= committed.SeqNumber:
    → 重复消息，丢弃（返回 success 但不写入）
elif incoming.SeqNumber == committed.SeqNumber + 1:
    → 期待的下一条，正常写入
else:
    → SeqNumber 跳跃，抛出 OutOfOrderSequenceException
```

**PID 的生命周期**：

```
Producer 启动
  → 首次 send() 时向 Broker 发起 InitProducerIdRequest
  → Broker 从 __transaction_state 分配全局唯一 PID
  → Producer 本地缓存 PID + 重置所有分区 SeqNumber=0
  → 正常发送消息（携带 PID + SeqNumber）
  → Producer 崩溃/重启
  → PID 失效，新会话获得新 PID
  → 旧 PID 的 SeqNumber 序列无法延续
```

> **关键点**：PID 不持久化在 Broker 端。Broker 重启后所有 PID 状态丢失，但 Producer 会重新申请 PID。这是设计权衡——如果持久化所有 Producer 的状态，Broker 的写入路径会复杂很多。

**幂等性失效场景**：
| 场景 | 原因 | 解决方案 |
|------|------|---------|
| Producer 重启 | PID 变化，旧 SeqNumber 无效 | 业务幂等兜底 |
| 跨分区 | 不同分区 SeqNumber 独立 | 事务机制 |
| 跨会话 | 新 PID 无法去重旧 PID 的消息 | 业务幂等兜底 |

> **核心结论**：幂等性解决的是"网络抖动导致的重试重复"，不解决"Producer 崩溃重启产生的跨会话重复"。业务幂等永远是最后防线。

### 事务机制（解决跨分区/跨 Topic 原子性）

**为什么需要事务？幂等性只解决单分区重复。当"消费 A → 处理 → 写入 B → 提交 A 的 Offset"涉及跨分区操作时，需要事务保证原子性。**

**事务参与者**：
```
Transaction Coordinator (TC)     ← 管理事务状态，类似 2PC 的协调者
Producer                          ← 事务发起者
Consumer (isolation.level=read_committed) ← 只读已提交数据
```

**事务执行流程**：
```
1. initTransactions() → Producer 获得 PID 和 Transactional ID 的绑定
2. beginTransaction() → TC 记录事务状态为 ONGOING
3. send() + sendOffsetsToTransaction() → 消息追加到 Partition，标记为"事务性"
4. commitTransaction() → TC 向每个涉及的 Partition 写入 Commit Marker
   → read_committed 消费者可以看到这些消息
5. 或 abortTransaction() → TC 写入 Abort Marker → 消息被跳过
```

**Transaction Marker**：Kafka 没有集中式事务日志。每个事务在涉及的 Partition 上写入 Control Batch 标记提交或中止。Consumer 读取到 Marker 后才知道前面的消息是否有效。

```
Partition 日志：
[msgA(Txn1)] [msgB(Txn1)] [Commit Marker(Txn1)] [msgC(非事务)] [msgD(Txn2)] [Abort Marker(Txn2)]
                                                    ↑ 可见                          ↑ 被跳过
```

**事务隔离级别**：
| 级别 | 配置 | 行为 |
|------|------|------|
| `read_uncommitted` | 默认 | 可以看到未提交的事务消息 |
| `read_committed` | 生产推荐 | 只看到已提交的事务消息 |

**事务的性能代价**：每个事务需要与 TC 交互（额外 RTT）、Commit Marker 写多个 Partition、维护 TC 状态、事务超时受 `transaction.timeout.ms` 限制（默认 15 分钟）。只在需要跨分区原子性的场景使用。

### Zombie Fencing（僵尸 Producer 隔离）

事务场景中有一个关键的边界条件：**Producer 宕机重启后，旧会话的未完成事务怎么处理？**

**Zombie Producer 问题**：
```
1. Producer-A（Transactional ID="txn-1"）开启事务 T1，发送了部分消息
2. Producer-A 宕机（网络断开，TC 不知道它宕机）
3. 运维启动新 Producer-B（相同 Transactional ID="txn-1"）
4. Producer-B 调用 initTransactions()
5. 此时... 如果 Producer-A 恢复，它会继续尝试提交 T1
   → 可能覆盖 Producer-B 的写入 → 数据错乱！
```

**Kafka 的 Zombie Fencing 机制**：

```
initTransactions() 的 Fencing 流程：
  1. Producer-B 向 TC 发起 InitProducerIdRequest，携带 Transactional ID
  2. TC 检查该 Transactional ID 是否已有活跃的 Producer
  3. TC 发现 Producer-A 的会话存在
  4. TC 递增该 Transactional ID 的 Producer Epoch（如 1 → 2）
  5. TC 向 Producer-B 返回新 PID + Epoch=2
  6. Producer-A 如果后续尝试提交 T1 → TC 检查 Epoch
     → Epoch 已变为 2，而 Producer-A 持有的是 Epoch=1
     → TC 拒绝 → 抛出 ProducerFencedException
  7. Producer-A 收到 Fenced 异常 → 不能再做任何操作 → 它成为"Zombie"
```

> **面试追问：Zombie Fencing 和 幂等性中的 PID 机制有什么关系？** 幂等性中的 PID 是 Producer 会话级（重启失效），事务中的 Producer Epoch 是 Transactional ID 级（跨会话持久化）。Epoch 是"进阶版 PID"——它跨会话持久化在 TC 中，重启后可以被递增，从而 Fencing 掉旧会话。

**Fencing 的三层防护总结**：
```
1. PID + SeqNumber    → 防单会话内重试重复（幂等性）
2. Producer Epoch     → 防旧会话 Zombie 提交（事务 Fencing）
3. 业务幂等           → 防跨系统、跨集群的重复（最终兜底）
```

### 消息顺序性

**核心结论：Kafka 只保证单分区内有序。**

乱序来源：重试。
```
Request1 失败 → Request2 成功 → Request3 成功 → Request1 重试成功
→ Broker 端写入顺序：2, 3, 1（乱了）
```

配置建议：
```properties
# 未开启幂等性：严格有序
max.in.flight.requests.per.connection=1

# 开启幂等性：Kafka 通过 SeqNumber 机制保证单分区有序
enable.idempotence=true
max.in.flight.requests.per.connection <= 5
```

### Kafka 到外部系统的 Exactly Once

**Kafka 不能天然保证到外部系统的 Exactly Once！**

```
Kafka → Consumer → MySQL/Redis/ES
```

Kafka 无法管理外部系统的事务，Offset 提交和外部写入不是原子操作。

**四种解决方案**：

| 方案 | 原理 | 复杂度 | 适用场景 |
|------|------|--------|---------|
| **业务幂等** | 唯一键/去重表/Redis SET NX | 低 | 有天然唯一键的场景 |
| **本地消息表 + 定时补偿** | Consumer 写 MySQL 时同时写一张"去重表"，消费前检查 | 中 | 一般业务场景 |
| **Outbox 模式** | 业务写入 Outbox 表 + CDC → Kafka，保证 DB 和消息原子 | 中 | 需要保证发送可靠的场景 |
| **Flink Checkpoint + 两阶段提交** | Flink 管理 Kafka Offset 和外部 Sink 的事务边界 | 高 | 流计算场景 |

**Flink Checkpoint + 两阶段提交原理**：
```
1. Flink 启动 Checkpoint → 触发所有 Operator 做快照
2. Kafka Consumer 记录当前 Offset → 暂不提交
3. 外部 Sink（如 MySQL）开启事务 → 写入数据 → 预提交
4. Checkpoint 完成 → Flink 提交 Kafka Offset + 提交 Sink 事务
5. 如果失败 → 回滚到上一个 Checkpoint → Kafka Offset 回退 + Sink 事务回滚
```
> 核心思想：Checkpoint 是分布式事务的协调点。Checkpoint 成功 → 全提交；Checkpoint 失败 → 全回滚。

## 四、关键设计

### 为什么幂等性只保证单会话？

PID 是 Producer 连接时分配的，重启后 PID 变化。Kafka 选择不在 Broker 端持久化 PID 的 SeqNumber 状态（重启后状态丢失），因为持久化所有 Producer 的 SeqNumber 代价太大——这是一个权衡：简单高效 vs 绝对去重。

### 事务为什么用 Marker 而不是集中式日志？

集中式事务日志会成为单点瓶颈。每个 Partition 独立存储 Marker，读取时 Consumer 自然就看到了——不需要额外的查询步骤。这是典型的"去中心化"设计：让数据在它应该在的地方。

### 三层保障的边界

```
幂等性 → 单分区、单会话内去重
事务   → 多分区、跨 Topic 原子性
业务幂等 → 跨会话、跨系统、最终兜底
```

每一层解决不同范围的问题，不能指望一个机制解决所有问题。

## 五、面试高频问题

**Q1：acks=all 就绝对不丢吗？**

不绝对。还必须满足：副本数 ≥ 3、`min.insync.replicas` ≥ 2、`unclean.leader.election.enable=false`。如果 ISR 中所有副本同时物理损毁且不可恢复（如机房火灾），数据仍会丢失。

> **追问：min.insync.replicas=2，但 ISR 只有 1 个（Leader），Producer 写入会怎样？** 抛 `NotEnoughReplicasException`，写入被拒绝。这是一种**宁可不可用也不丢数据**的设计选择。

**Q2：幂等性和事务有什么区别？**

幂等性解决单分区内 Producer 重试导致的重复，限制：单会话 + 单分区。事务解决跨分区/跨 Topic 的原子写入 + Offset 提交，典型场景 consume-process-produce。事务依赖幂等性。

**Q3：Kafka 能保证 Exactly Once 吗？**

分场景。Kafka 内部 consume-process-produce 可以通过事务保证原子性。但 Kafka 到外部系统（MySQL/Redis）不能天然保证，需要外部系统的幂等或事务配合。

**Q4：如何保证消息顺序？**

Kafka 只保证单分区有序。需要有序的消息用相同 key 写入同一分区。Producer 侧：未开幂等性时 `max.in.flight.requests=1`，开幂等性后可 ≤5（Kafka 自动修正乱序）。

**Q5：幂等性在什么情况下失效？**

Producer 重启后 PID 变化 → 新 PID 的 SeqNumber 从 0 开始 → 无法去重旧 PID 已发送的消息。所以幂等性只保证单会话内去重，跨会话重复需要业务幂等兜底。

## 六、总结（速记版）

- **acks 三级**：0（不确认）、1（Leader 确认）、all（ISR 确认）
- **不丢五件套**：acks=all + retries + rf=3 + min.isr=2 + 手动提交
- **幂等性**：PID + SeqNumber，解决单分区单会话重试重复
- **事务**：TC + Control Batch Marker，多分区原子写入
- **顺序**：key 路由 → 单分区有序 + inflight 控制
- **兜底**：业务幂等永远是最后防线

---

# 九、消息积压与运维调优

## 一、背景（为什么需要）

消息积压是线上最频繁的 Kafka 问题。考察的是排查思路和应急能力，不是背配置。面试官的提问通常是："Consumer Lag 一直在涨，你怎么排查？"

## 二、核心概念

积压本质：**生产速度 > 消费速度 → Consumer Lag 持续增长**。

Consumer Lag = 最新消息 Offset - 当前消费 Offset。

## 三、底层原理

### 排查五步法

```
第一步：看生产端 → 流量是否突增？
  是 → 活动/日志暴增/上游异常重试
  否 → 第二步

第二步：看消费端 → 消费者是否变慢？
  是 → DB 慢/RPC 慢/下游限流/业务逻辑变复杂
  否 → 第三步

第三步：看分区数 → 分区数是否不足？
  是 → 增加分区或 Consumer
  否 → 第四步

第四步：看拉取参数 → max.poll.records/fetch.max.bytes 太小？
  是 → 调大
  否 → 第五步

第五步：看 Rebalance → 是否频繁 Rebalance？
  是 → 调大超时参数/优化处理耗时
```

### 分级解决方案

**短期应急（5 分钟内生效）**：

| 方案 | 操作 | 风险 |
|------|------|------|
| 增加 Consumer 实例 | 水平扩容 | 前提：Partition 数 ≥ 新 Consumer 数 |
| 调大拉取参数 | 增大 `max.poll.records`、`fetch.max.bytes` | 单次处理更慢，可能触发 poll 超时 |
| 批量处理消息 | 攒一批再批量写 DB | 增加单条消息延迟 |
| 临时扩容下游 | 增加 DB 连接池 | 成本 |
| 非核心业务降级 | 暂停非核心消费 | 部分业务暂时不可用 |

**中期优化（1-3 天）**：

| 方案 | 操作 | 注意 |
|------|------|------|
| 增加分区数 | `kafka-topics.sh --alter --partitions N` | 只能增不能减 |
| 拆分慢 Topic | 按业务维度拆分 | Consumer 需适配 |
| 优化 DB 写入 | 批量 INSERT、异步写入 | 显著降低消费耗时 |
| 消费逻辑异步化 | poll 线程 → 线程池处理 → 有序提交 Offset | 注意顺序和幂等 |
| 更换分配策略 | Range → CooperativeSticky | 减少 Rebalance 影响 |

**长期建设**：
- Lag 分级告警：>1 万通知、>10 万告警、>100 万紧急
- 上游限流：从源头控制 QPS
- 消息生命周期管理：过期消息自动清理
- 容量规划：根据 DAU 增长提前扩容
- Chaos Engineering：定期演练宕机和大促峰值

### 部分分区积压

如果只有某几个分区积压，说明 **key 分布不均**——比如用 userId 做 key，某个大 V 的消息量是普通用户的 1000 倍。

解决方案：
- 自定义 Partitioner 给热点 key 单独分区
- 生产端对热点 key 拆分（userId_0, userId_1 ... userId_N）
- 临时将热点分区转发到专门的消费者组

### 死信队列与消费重试设计

消费失败不应无限重试——会阻塞主消费链路。标准做法：

```
原 Topic → 处理失败 → 重试 Topic（有限重试，如 3 次）→ 仍然失败 → 死信 Topic（DLT）
```

死信消息需保留：原始消息体 + 失败原因 + 异常堆栈 + 失败时间 + 重试次数，便于后续排查和补偿。

Go 实现要点：
```go
const maxRetries = 3
if retryCount < maxRetries {
    producer.SendToRetryTopic(msg)  // 发到重试 Topic，延迟后重新消费
} else {
    producer.SendToDLT(msg)          // 发到死信 Topic，触发告警，人工介入
}
```

> 关键设计：重试 Topic 和死信 Topic 的消费者是独立的，不阻塞主消费链路。

### Consumer pause/resume

当下游（如 DB、RPC）限流或线程池满时，不应继续拉新消息。`pause()` 暂停拉取但不释放分区所有权（区别于 Rebalance），`resume()` 恢复。

```go
consumer.Pause(partitions)   // 暂停拉取，不释放分区
consumer.Resume(partitions)  // 恢复拉取
```

适用场景：下游限流、局部降级、线程池满时暂缓消费。

### 监控告警体系

| 监控对象 | 核心指标 | 告警阈值参考 |
|---------|---------|------------|
| **Producer** | 发送 QPS、失败率、发送延迟 | 失败率 > 1% 告警 |
| **Broker** | 磁盘使用率、网络吞吐、URP（未同步副本）、ISR 变化 | 磁盘 > 80%、URP > 0 |
| **Consumer** | Consumer Lag、消费速率、Rebalance 频率 | Lag > 10 万告警，> 100 万紧急 |
| **Topic** | 生产速率、消费速率、分区大小不均 | 速率突变告警 |

关键：Lag 分级告警（通知 → 告警 → 紧急），Producer 端监控 `bufferpool-wait-time` 判断缓冲区是否不足。

### JMX 监控 + Prometheus 集成

Kafka 暴露丰富的 **JMX（Java Management Extensions）** 指标，生产环境通常通过 Prometheus + Grafana 采集和展示：

**采集架构**：
```
Kafka Broker (JMX Exporter Agent)
  │  HTTP /metrics
  ↓
Prometheus Server（定期拉取）
  ↓
Grafana Dashboard（可视化 + 告警规则）
```

**JMX Exporter 配置示例**：
```yaml
# kafka-jmx-exporter.yml
lowercaseOutputName: true
rules:
  # Broker 核心指标
  - pattern: "kafka.server<type=BrokerTopicMetrics, name=(.*)><>(Count|MeanRate)"
    name: kafka_server_brokertopicmetrics_$1_$2
  # 未同步副本数
  - pattern: "kafka.server<type=ReplicaManager, name=UnderReplicatedPartitions><>Value"
    name: kafka_server_replicamanager_underreplicatedpartitions
  # ISR 收缩/扩张
  - pattern: "kafka.server<type=ReplicaManager, name=IsrShrinksPerSec><>Count"
    name: kafka_server_replicamanager_isrshrinks_total
  # 活跃 Controller 计数（>1 表示脑裂！）
  - pattern: "kafka.controller<type=KafkaController, name=ActiveControllerCount><>Value"
    name: kafka_controller_activecontroller

# 启动时通过 JAVA_TOOL_OPTIONS 注入：
# export KAFKA_OPTS="-javaagent:/opt/jmx_exporter/jmx_prometheus_javaagent.jar=7071:/opt/jmx_exporter/config.yml"
```

**面试必知的关键 JMX Bean**：

| JMX Bean | 指标 | 告警含义 |
|----------|------|---------|
| `ReplicaManager/UnderReplicatedPartitions` | URP 数量 | > 0 且持续 → 副本同步异常 |
| `KafkaController/ActiveControllerCount` | 活跃 Controller 数 | ≠ 1 → 脑裂或选举中 |
| `KafkaController/OfflinePartitionsCount` | 离线分区数 | > 0 → 分区不可用 |
| `BrokerTopicMetrics/BytesInPerSec` | 写入速率 | 异常暴增/暴跌 |
| `BrokerTopicMetrics/BytesOutPerSec` | 读取速率 | 同上 |
| `RequestHandlerAvgIdlePercent` | Handler 空闲率 | < 0.3 → Handler 线程不足 |
| `NetworkProcessorAvgIdlePercent` | Processor 空闲率 | < 0.3 → Processor 线程不足 |
| `SessionExpireListener/ZkAuthFailuresPerSec` | ZK 认证失败 | > 0 → 安全配置问题 |

**Grafana 面板建议**：
- 第一行（黄金指标）：写入/读取吞吐 + 请求延迟 P99 + URP + 活跃 Controller
- 第二行：磁盘使用率 + 网络吞吐 + ISR 变化 + 分区数
- 第三行：按 Topic 的 QPS + Lag + 分区大小分布
- 告警线标注：磁盘 80%、Lag 10 万、URP > 0 持续 1 分钟

> **关键实践**：URP 和 ActiveControllerCount 是最重要的 Broker 级告警指标——URP > 0 意味着 ISR 不完整、写入可能受影响；ActiveController ≠ 1 意味着集群可能脑裂。

### 磁盘不均衡与在线迁移

**问题**：分区大小不均（热点 Topic 的某些分区远大于其他分区）或 Leader 分布不均（某些 Broker 上 Leader 过多），导致磁盘使用不均衡和负载倾斜。

**解决**：
```bash
# 生成分区重分配计划
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json --broker-list "0,1,2" --generate

# 执行迁移（注意限流避免影响在线业务）
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassign.json --execute

# 查看迁移进度
bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassign.json --verify
```

> 高危操作：数据迁移期间限流（`throttle` 参数），避免打满磁盘 I/O 影响正常读写。

### 动态配置热加载

生产环境频率最高的问题之一是"改了个配置要不要重启 Broker"。Kafka 支持部分配置**动态生效**，无需重启：

| 配置类型 | 是否动态 | 修改方式 | 示例 |
|---------|---------|---------|------|
| **Topic 级配置** | ✅ 动态 | `kafka-configs.sh --alter` | `retention.ms`, `min.insync.replicas` |
| **Broker 级动态配置** | ✅ 动态 | `kafka-configs.sh --alter --entity-type brokers` | `unclean.leader.election.enable` |
| **Client 级配额** | ✅ 动态 | `kafka-configs.sh --alter --entity-type users` | `producer_byte_rate` |
| **Broker 核心参数** | ❌ 需重启 | 修改 `server.properties` | `log.dirs`, `num.network.threads`, `num.partitions` |
| **JVM 参数** | ❌ 需重启 | 修改 `KAFKA_HEAP_OPTS` | 堆大小、GC 参数 |

**Topic 级动态配置示例**：
```bash
# 修改 Topic 的消息保留时间（从 7 天改为 3 天）
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders --alter \
  --add-config retention.ms=259200000

# 修改 Topic 的 min.insync.replicas
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders --alter \
  --add-config min.insync.replicas=2

# 查看 Topic 的当前配置（含默认值和覆盖值）
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders --describe

# 删除自定义配置，恢复默认
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders --alter \
  --delete-config retention.ms
```

**Broker 级动态配置示例**：
```bash
# 全局修改 unclean.leader.election.enable（无需重启所有 Broker）
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type brokers --entity-default --alter \
  --add-config unclean.leader.election.enable=false
```

**动态配置 vs server.properties 的优先级**：
```
动态配置 > server.properties（静态配置） > 默认值

修改后：
  → Broker 收到 AlterConfig 请求
  → 更新内存中的配置缓存
  → 立即对新的请求生效（无需重启）
  → 持久化到 ZK / KRaft 元数据
```

**必须重启才能改的核心参数**：
| 参数 | 为什么必须重启 |
|------|--------------|
| `log.dirs` | 数据目录路径，磁盘挂载在启动时初始化 |
| `num.network.threads` | 线程池大小在启动时创建 |
| `num.io.threads` | 同上 |
| `num.partitions` | 默认分区数，影响新 Topic 创建行为 |
| `listeners` / `advertised.listeners` | 网络绑定在启动时完成 |
| `zookeeper.connect` / `controller.quorum.voters` | 集群连接信息 |
| 所有 JVM 参数 | JVM 启动时确定 |

> **面试一句话**：改 Byte/Time 类的配置（retention、大小、超时）通常能动态生效；改线程数、磁盘路径、网络监听的必须重启。实时生效的是 `kafka-configs.sh` 操作的动态配置，覆盖 `server.properties` 中的静态值。

### Rebalance 风暴

**场景**：大量 Consumer 同时重启（如滚动发布）→ 短时间内密集触发 Rebalance → 集群反复 STW，消费完全停滞。

**解决方案**：
1. **静态成员**（`group.instance.id`）：Consumer 重启后 Coordinator 在 `session.timeout.ms` 内给它"留位"，视为临时离开而非退出
2. **分批重启**：滚动发布时分批重启，每批之间有足够间隔
3. **延长 `session.timeout.ms`**：给重启留更多缓冲时间
4. **使用 CooperativeSticky**：即使 Rebalance 也只迁移必要分区

### Kafka 配额（Quota）机制

Kafka 支持对客户端请求进行限流，防止单个 Client 打爆集群：

```bash
# 设置 Producer 配额（限制 User-A 的 Producer 流量）
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type users --entity-name UserA --alter \
  --add-config 'producer_byte_rate=1048576'  # 1MB/s

# 设置 Consumer 配额
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type users --entity-name UserB --alter \
  --add-config 'consumer_byte_rate=2097152'  # 2MB/s

# 设置请求速率配额
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type users --entity-name UserA --alter \
  --add-config 'request_percentage=50'  # 最多占 50% 的请求处理时间
```

> 配额在 Broker 端以延迟响应的方式实现——超配额时不是拒绝请求，而是延迟处理（throttle time），由 Client 端读取 throttle_time 后自行降速。

## 四、关键设计

### 分区数设计原则

| 分区过少 | 分区过多 |
|---------|---------|
| 并行度不足 | 文件句柄多（每个 Partition ≈ 3 个文件描述符） |
| 消费能力上不去 | 元数据压力大 |
| 容易积压 | Rebalance 变慢 |
| | Leader 选举成本增加 |

**估算公式**：
```
分区数 ≈ max(峰值吞吐 / 单分区吞吐, 期望消费者并行度) × 预留系数(1.5~2)
```

**容量估算**：
```
日活 100 万 × 每人 100 条 × 每条 1KB = 100GB/天
平均写入：100GB / 86400s ≈ 1.2MB/s
峰值（20 倍）：≈ 24MB/s
磁盘容量 = 100GB × 3 副本 × 7 天 / 0.7 ≈ 3TB
```

**生产铁律**：
- `auto.create.topics.enable=false`：防止 Topic 名写错自动创建
- 分区数提前规划，预留未来增长空间
- 重要 Topic 规划好副本数、保留时间、压缩策略和权限

## 五、面试高频问题

**Q1：Consumer Lag 一直在涨，怎么排查？**

五步法：生产流量 → 消费速度 → 分区数限制 → 拉取参数 → Rebalance 是否频繁。先定位是"生产太快"还是"消费太慢"，再对症下药。

**Q2：加 Consumer 为什么速度没提升？**

大概率是 Partition 数限制。同组有效并行度 = min(Partition 数, Consumer 数)。先检查 Partition 数是否 ≥ Consumer 数。

**Q3：能不能无限加 Consumer？**

不能。受 Partition 数限制。多出来的 Consumer 空闲浪费。

**Q4：为什么生产环境要关闭自动创建 Topic？**

防止 Topic 名写错后消息被写入错误的 Topic、以及使用不合理的默认配置（如分区数=1、副本数=1）。生产环境应显式创建 Topic。

**Q5：死信队列怎么设计？**

原 Topic → 消费失败 → 重试 Topic（有限次数，如 3 次，间隔递增）→ 仍失败 → 死信 Topic。死信保留原始消息 + 失败原因 + 异常堆栈 + 时间戳。死信消费者独立运行，不阻塞主消费链路，触发告警后人工介入补偿。

**Q6：大量 Consumer 同时重启会导致什么？**

Rebalance 风暴。短时间内密集触发 Rebalance → 集群反复 Stop-The-World → 消费完全停滞。解决：静态成员（`group.instance.id`）+ 分批重启 + 延长 `session.timeout.ms` + CooperativeSticky 策略。

**Q7：磁盘使用不均衡怎么办？**

kafka-reassign-partitions.sh 在线迁移分区。注意限流避免影响正常读写。日常监控分区大小分布和 Leader 分布，提前发现热点。

## 六、总结（速记版）

- **积压本质**：生产 > 消费
- **排查顺序**：生产流量 → 消费速度 → 分区数 → 拉取参数 → Rebalance
- **应急方案**：加 Consumer、调大参数、批量处理、扩容下游、降级非核心
- **死信队列**：有限重试 → 死信 Topic → 告警 + 人工补偿
- **Rebalance 风暴**：静态成员 + 分批重启 + CooperativeSticky
- **长期方案**：加分区、拆 Topic、优化业务、监控告警
- **分区铁律**：只能增不能减，提前规划 + 预留空间

---

# 十、生态选型与 Go 实践

## 一、背景（为什么需要）

面试中必问"Kafka vs RocketMQ vs RabbitMQ"，考察选型能力。Go 项目中如何正确使用 Kafka 也是高频实操题。

## 二、核心概念

### 三大 MQ 对比

| 对比项 | Kafka | RocketMQ | RabbitMQ |
|--------|-------|----------|----------|
| 核心定位 | 分布式事件流/高吞吐日志 | 分布式业务消息中间件 | 传统消息队列 |
| 吞吐量 | **极高（百万级）** | 高（十万级） | 中等（万级） |
| 消息模型 | Topic + Partition + CG | Topic + Queue + CG | Exchange + Queue + Binding |
| 顺序消息 | 单分区内有序 | 原生支持顺序消息 | 可实现 |
| 延迟消息 | 需自行设计 | **原生 18 级延迟** | 插件/TTL+DLX |
| 事务消息 | Kafka 内部事务 | **原生支持** | 支持 |
| 消息回溯 | **天然支持** | 支持 | 不支持 |
| 典型场景 | 日志、埋点、数据管道、流处理 | 订单、交易、业务异步 | 企业应用、复杂路由 |

### 选型决策树

```
日志/埋点/大数据管道 → Kafka
订单/交易/延迟消息/业务解耦 → RocketMQ
传统企业应用/复杂路由/AMQP 标准 → RabbitMQ
```

### 为什么订单系统更推荐 RocketMQ 而不是 Kafka？

1. **延迟消息**：订单 30 分钟未支付自动取消 → RocketMQ 原生 18 个延迟级别，Kafka 需自建（延迟 Topic + 时间轮 + 定时扫描）
2. **消费重试**：订单处理失败 → RocketMQ 内置重试队列（失败自动重试，间隔递增），Kafka 需自己实现
3. **消息过滤**：RocketMQ 支持 Tag/SQL 过滤，Consumer 只拉感兴趣的消息；Kafka 需拉全部再过滤

## 三、底层原理

### Go 主流客户端

| 客户端 | 特点 | 推荐度 |
|--------|------|--------|
| **[sarama](https://github.com/IBM/sarama)** | Go 原生实现，功能全面，支持 Consumer Group、幂等、事务 | 最流行 |
| **[confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)** | librdkafka C 库的 Go 绑定，性能最高 | 性能优先 |
| **[kafka-go](https://github.com/segmentio/kafka-go)** | 纯 Go，API 简洁 | 简单场景 |

### Go Producer 生产级配置

```go
config := sarama.NewConfig()
config.Producer.RequiredAcks = sarama.WaitForAll          // acks=all
config.Producer.Retry.Max = 3                              // 重试 3 次
config.Producer.Idempotent = true                          // 开启幂等性
config.Producer.Compression = sarama.CompressionSnappy     // snappy 压缩
config.Producer.Flush.Frequency = 10 * time.Millisecond    // linger.ms=10ms
config.Producer.Flush.Messages = 1000                      // batch.size
config.Producer.Return.Successes = true                    // 需要确认成功的消息
config.Net.MaxOpenRequests = 5                             // max.in.flight

// 异步发送 + 回调处理结果
producer.Input() <- msg
select {
case success := <-producer.Successes():
    // 记录成功，可用于监控
case err := <-producer.Errors():
    log.Printf("发送失败: %v", err.Err)
    // 不要直接重试！Producer 已有 retries
}
```

### Go Consumer 生产级配置

```go
config := sarama.NewConfig()
config.Consumer.Group.Rebalance.Strategy = sarama.NewBalanceStrategySticky()
config.Consumer.Offsets.AutoCommit.Enable = false          // 关闭自动提交
config.Consumer.Offsets.Initial = sarama.OffsetNewest       // 无 Offset 从最新开始
config.Consumer.MaxProcessingTime = 10 * time.Minute        // max.poll.interval.ms

func (h *ConsumerHandler) ConsumeClaim(
    sess sarama.ConsumerGroupSession,
    claim sarama.ConsumerGroupClaim,
) error {
    for msg := range claim.Messages() {
        if err := processMessage(msg); err != nil {
            log.Printf("处理失败: %v, offset: %d", err, msg.Offset)
            continue  // 不提交 Offset，下次重新消费
        }
        sess.MarkMessage(msg, "")  // 标记已消费
    }
    return nil
}
```

### Go 中常见 Kafka 坑

| 坑 | 原因 | 解决 |
|----|------|------|
| **Goroutine 泄漏** | Consumer 通道未正确关闭 | `defer consumer.Close()`，监听 context 取消 |
| **Offset 提交乱序** | 多 Goroutine 并发处理 + 并发提交 | 单 Goroutine 顺序提交，或用 `MarkOffset` 精确控制 |
| **内存暴涨** | Producer 缓冲区设置过大 | 合理设置 `ChannelBufferSize`，监控 `buffer.memory` |
| **Consumer 卡住** | 处理耗时 > MaxProcessingTime | 拆分 poll 线程 + worker 线程池 |
| **连接泄漏** | 每次发消息都 NewClient | 复用 Producer/Consumer 实例，单例模式 |

### Go Consumer 优雅关闭模板

```go
func runConsumer(ctx context.Context, group sarama.ConsumerGroup, topics []string) {
    handler := &ConsumerHandler{}
    for {
        select {
        case <-ctx.Done():
            log.Println("收到退出信号，正在优雅关闭...")
            return
        default:
            // Consume 在 Rebalance 时会返回，外层用 for 循环保证继续消费
            if err := group.Consume(ctx, topics, handler); err != nil {
                log.Printf("Consume error: %v", err)
            }
        }
    }
}

// main 中
ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
defer cancel()
go runConsumer(ctx, group, topics)
<-ctx.Done()
// 此时 Consumer Group 的 Consume 会在下一次 Rebalance 时退出
group.Close()  // 触发 LeaveGroup，主动通知 Coordinator
```

> **关键点**：不要依赖 `group.Close()` 来保证优雅退出。应该在收到信号后、业务处理完最后一批消息后、手动提交 Offset，再关闭。`group.Close()` 只是通知 Coordinator 离开组。

### Kafka Connect & Streams

| 组件 | 定位 | 典型用法 |
|------|------|---------|
| **Kafka Connect** | 数据集成框架 | MySQL Binlog → Source Connector → Kafka → Sink Connector → ES/HDFS/S3 |
| **Kafka Streams** | 嵌入式流处理库 | 轻量级实时计算，适合简单聚合/转换 |
| **Flink** | 独立流处理引擎 | 重量级有状态流批一体，适合复杂计算 |

### Kafka Streams 核心抽象（面试重点）

Kafka Streams 是嵌入在 Java 应用中的流处理库，不需要独立集群。面试常考三个核心抽象的区别：

| 抽象 | 语义 | 数据模型 | 典型场景 |
|------|------|---------|---------|
| **KStream** | 无界**事件流**（每条消息独立） | INSERT only | 点击流、日志、传感器读数 |
| **KTable** | 可更新的**快照表**（key 唯一，新值覆盖旧值） | INSERT + UPDATE + DELETE | 用户配置、账户余额、产品信息 |
| **GlobalKTable** | 全分区广播的**全局表**（每个实例持有全量数据） | 全量副本 | 汇率表、国家代码、小量静态参考数据 |

**KStream vs KTable 的本质区别**：

```
KStream（事件流）：
  key=user1, value=v1  →  独立事件
  key=user1, value=v2  →  独立事件（不会覆盖 v1）
  key=user1, value=v3  →  独立事件（不会覆盖 v2）
  计数：= 3 条记录（每条都算）

KTable（更新流 / Changelog）：
  key=user1, value=v1  →  upsert user1=v1
  key=user1, value=v2  →  upsert user1=v2（覆盖 v1）
  key=user1, value=v3  →  upsert user1=v3（覆盖 v2）
  计数：= 1 条记录（只保留最新）
```

> **面试一句话**：KStream 是 INSERT only 的事件流；KTable 是支持 upsert 的更新流，底层是一个 compacted Topic 的物化视图。

**KStream + KTable Join 的面试场景**：
```
场景：用户行为流（KStream）join 用户画像表（KTable），实时补全用户信息

KStream: {userId: 1, event: "click", productId: 100}
  → join ←
KTable:  {userId: 1, name: "张三", level: "VIP", city: "北京"}
  → 结果： {userId: 1, event: "click", productId: 100, name: "张三", level: "VIP", city: "北京"}

关键：Join 时要求 KStream 和 KTable 的 key 匹配（通过 repartition 保证）
```

**窗口（Windowing）机制**：

Kafka Streams 支持四种窗口，是流计算的核心：

| 窗口类型 | 行为 | 适用场景 |
|---------|------|---------|
| **Tumbling** | 固定大小、不重叠 | 每小时订单统计、每分钟 PV |
| **Hopping** | 固定大小、有重叠 | 最近 5 分钟滑动窗口、每 1 分钟更新一次 |
| **Session** | 动态大小、按活动间隔合并 | 用户会话分析、登录→操作→退出 |
| **Sliding** | 固定大小、按事件触发 | 每次交易后计算过去 10 分钟的关联事件 |

```
Tumbling(窗口=5分钟)：
  [0-5) [5-10) [10-15) ... ← 边界不重叠

Hopping(窗口=5分钟, 步长=1分钟)：
  [0-5) [1-6) [2-7) ... ← 窗口重叠 4 分钟

Session(间隔超时=30分钟)：
  [活动1...活动2]  间隔20分  [活动3...活动4]  ← 间隔>30分则分窗
```

**State Store 与 Changelog 的关系**：

```
Kafka Streams 实例
  ├── State Store（本地 RocksDB 或内存）← 快速查询
  │     │ 每个 key 的变更
  │     ↓
  └── Changelog Topic（Kafka Topic）       ← 容错恢复
        └── 本实例宕机后，新实例从 Changelog Topic 重放恢复 State Store
```

> **核心设计**：State Store 是本地缓存，Changelog Topic 是持久化的变更日志。宕机恢复时从 Changelog 重放重建 State Store。这就是"事件溯源（Event Sourcing）"模式。

**Kafka Streams 的局限性（选型参考）**：
| 场景 | Kafka Streams | Flink |
|------|--------------|-------|
| 简单聚合/过滤/Join | ✅ 推荐，无需独立集群 | 可以但过重 |
| 有状态复杂计算 | 中级 | ✅ 强状态管理和 Checkpoint |
| Exactly Once 到外部系统 | ❌ 需要自己实现 | ✅ 两阶段提交 |
| SQL 接口 | KSQL | ✅ 完善 Table/SQL API |
| 运维复杂度 | 低（嵌入式 JAR） | 高（独立集群） |
| 多语言支持 | Java/Scala 为主 | Java/Python/SQL |

### MirrorMaker 2.0（跨集群数据复制）

基于 Kafka Connect 框架，实现 Kafka 集群间的数据同步：

```
源集群 → MirrorSourceConnector → 目标集群（保留原始 Offset、分区信息）
源集群 ← MirrorCheckpointConnector ← 目标集群（同步 Consumer Group Offset）
源集群 ← MirrorHeartbeatConnector ← 目标集群（心跳监控延迟）
```

适用场景：多机房容灾、跨区域数据聚合、云迁移。核心优势：保留 Offset 映射关系，消费者可无缝切换到目标集群。

### 多活/双活部署模式（架构题高频）

大厂面试中常问"两个机房各一套 Kafka，如何容灾切换"。以下是三种主流模式：

**模式 1：Active-Standby（主备）**
```
                    ┌─────────────────┐
                    │   全局流量入口    │
                    └────────┬────────┘
                             │ 正常情况：全部流量 → 主集群
                    ┌────────┴────────┐
                    ↓                  │ 主集群故障时切换
            ┌──────────────┐   ┌──────┴───────┐
            │  主集群(DC-1) │   │  备集群(DC-2)  │
            │  (正常读写)    │   │  (只同步数据)   │
            └──────┬───────┘   └──────────────┘
                   │ MirrorMaker 2.0 异步复制
                   └────────────────→
```
- 优点：架构简单，无数据冲突
- 缺点：切换有数据延迟（异步复制），备集群资源浪费
- 适用：灾备场景，RTO 分钟级可接受

**模式 2：Active-Active（双活）**
```
        ┌─────────────────────────────────┐
        │         全局路由层（GSLB）        │
        │    userId hash → 就近/指定机房    │
        └────────┬────────────────┬───────┘
                 ↓                ↓
        ┌──────────────┐  ┌──────────────┐
        │  DC-1 集群    │  │  DC-2 集群    │
        │  (用户 A-M)  │  │  (用户 N-Z)  │
        └──────┬───────┘  └──────┬───────┘
               │ 双向 MirrorMaker     │
               └────────────────────┘
```
- 优点：两个集群同时服务，资源利用率高
- 挑战：双向复制需防消息循环、数据一致性保障、热点用户跨机房
- 适用：全球化部署，用户就近接入

**模式 3：Stretch Cluster（延伸集群）**
```
        ┌─────────────────────────────────┐
        │     KRaft Controller (3 节点)    │
        │    DC-1(2节点) + DC-2(1节点)     │
        └─────────────────────────────────┘
                       │
        ┌──────────────┴──────────────┐
        ↓                             ↓
  ┌──────────────┐            ┌──────────────┐
  │ Broker (DC-1)│            │ Broker (DC-2)│
  │ rack=dc1     │            │ rack=dc2     │
  └──────────────┘            └──────────────┘
        │                             │
        └────────── 同一 Kafka 集群 ───┘
           Rack Awareness 保证副本跨 DC 分布
```
- 核心：跨 AZ/DC 组成一个集群，通过 rack.id 控制副本分布
- 要求：DC 间网络延迟 < 15ms、高带宽、稳定
- 优点：零切换时间（Leader 自动 Failover）
- 挑战：网络分区时只有 Majority Controller 侧可用

**三种模式对比**：

| 维度 | Active-Standby | Active-Active | Stretch Cluster |
|------|---------------|---------------|-----------------|
| RTO | 分钟级（需 DNS 切换） | 秒级（自动路由） | 秒级（自动 Failover） |
| RPO | 有少量数据丢失（异步） | 有少量数据丢失 | 零丢失（同步复制） |
| 资源利用率 | 50%（备集群闲置） | 100% | 100% |
| 网络要求 | 低（异步即可） | 中 | 高（< 15ms 延迟） |
| 运维复杂度 | 低 | 高（双向同步、防循环） | 高（对网络质量敏感） |
| 适用场景 | 同城灾备 | 全球化部署 | 同城多 AZ（低延迟） |

> **面试一句话**：大多数公司选 Active-Standby + MirrorMaker 2.0，架构简单、满足 RTO < 5 分钟的要求。跨机房延迟低的用 Stretch Cluster 追求零 RPO。全球化部署才考虑 Active-Active。

### Schema Registry（模式注册中心）

生产环境中 Kafka 消息格式需要严格管理，**Schema Registry** 是最佳实践：

```
Producer → Serializer（查 Schema Registry 获取 Schema ID）
        → 发送 [Schema ID(4字节) + Avro 二进制数据]   ← 不含完整 Schema！
        → Broker 存储（不解压，原样保存）
        → Consumer → Deserializer（凭 Schema ID 从 Registry 获取 Schema）
                   → 解码为业务对象
```

**为什么需要 Schema Registry？**
1. **前后兼容**：Schema 变更时保证老消费者能读新消息（FORWARD 兼容）或新消费者能读老消息（BACKWARD 兼容）
2. **减少消息体积**：消息中只带 4 字节 Schema ID，不带 Schema 本身（相比 JSON 省 50-90% 空间）
3. **强制约束**：注册时校验兼容性，阻止破坏性变更上线
4. **数据治理**：统一管理所有 Topic 的数据格式，方便下游消费

**兼容性模式**：
| 模式 | 含义 | 适用场景 |
|------|------|---------|
| BACKWARD | 新 Schema 可读旧数据 | Consumer 先升级 |
| FORWARD | 旧 Schema 可读新数据 | Producer 先升级 |
| FULL | 两者都满足 | 最严格 |
| NONE | 不检查 | 仅开发环境 |

> **面试追问：为什么推荐 Avro 而不是 JSON？** Avro 是二进制格式 + Schema 分离存储，消息体远比 JSON 小（省 50-90% 空间），解析更快（无需逐字节解析文本），且 Schema Registry 提供了兼容性保障。JSON 的优势是可读性好，适合调试。

**Confluent Schema Registry 的核心 API**：
```
POST /subjects/{topic}-value/versions   # 注册新 Schema
GET  /subjects/{topic}-value/versions/latest  # 获取最新 Schema
GET  /schemas/ids/{id}                  # 按 ID 获取 Schema
POST /compatibility/subjects/{topic}-value/versions/latest  # 兼容性检查
```

### Kafka on Kubernetes 关键考量

在 K8s 上部署 Kafka 需要特别注意：

| 关注点 | 原因 | 最佳实践 |
|--------|------|---------|
| **持久化存储** | Broker 宕机后数据不能丢 | 使用 StatefulSet + PVC + 本地 SSD StorageClass |
| **网络** | Kafka 依赖稳定的网络标识 | StatefulSet 提供稳定的 Pod 名和网络标识 |
| **资源隔离** | Broker 对磁盘 I/O 和网络敏感 | 设置 CPU/Memory Request=Limit（避免超卖），磁盘用本地卷 |
| **反亲和性** | 同一 Partition 的副本不应在同一节点 | `podAntiAffinity` 确保副本分散 |
| **KRaft Quorum** | Controller 节点需要固定标识 | KRaft 节点使用 StatefulSet 的稳定 DNS 名 |
| **优雅关闭** | Broker 关闭前需迁移 Leader | `preStop` 钩子中调用 `controlled-shutdown` |

> 生产环境建议使用 Strimzi Operator 或 Confluent for Kubernetes (CFK) 管理 Kafka on K8s 的生命周期，避免手写 StatefulSet。

## 四、关键设计

### Kafka Connect 的设计哲学

"大多数数据集成需求都是通用的"——从 DB 读 Binlog、写入 HDFS、同步到 ES。与其让每个人写胶水代码，不如提供标准化的 Connector 框架。Source/Sink 模式让数据集成变成配置而不是开发。

### 安全机制

| 机制 | 作用 | 生产建议 |
|------|------|---------|
| SSL/TLS | 加密传输 | 开启 |
| SASL/SCRAM/Kerberos | 身份认证 | 根据公司安全策略 |
| ACL | 权限控制（Topic/Group/Cluster 级别） | 生产必须开启 |

**ACL 示例**：
```bash
# Producer 需要有 Topic 的 WRITE 权限
bin/kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:producer --operation Write --topic orders

# Consumer 需要有 Topic 的 READ 权限 + Group 的 READ 权限
bin/kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:consumer --operation Read --topic orders
bin/kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:consumer --operation Read --group order-group
```

## 五、面试高频问题

**Q1：Kafka vs RocketMQ vs RabbitMQ 怎么选？**

日志/埋点/大数据管道选 Kafka，订单/交易/延迟消息选 RocketMQ，传统企业应用/复杂路由选 RabbitMQ。如果只选一个，大厂一般 Kafka + RocketMQ 组合。

**Q2：Go 项目用哪个 Kafka 客户端？**

Sarama 最流行，功能全面；confluent-kafka-go 性能最高但需要 CGO；kafka-go 简单易用。推荐 Sarama（生产验证最充分）。

**Q3：Consumer 处理消息的最佳实践？**

关闭自动提交 → poll 拉取消息 → 业务处理成功 → `MarkMessage` 标记 → 定期手动提交。处理失败的不要提交 Offset，让下次重新消费。避免在 poll 线程里做耗时操作。

**Q4：多机房 Kafka 如何同步数据？**

MirrorMaker 2.0（基于 Kafka Connect）。Source Connector 复制消息到目标集群并保留 Offset 映射；Checkpoint Connector 同步 Consumer Group Offset；Heartbeat Connector 监控同步延迟。消费者可无缝切换到目标集群。

**Q5：Schema Registry 有什么用？生产必须吗？**

不必须但强烈建议。核心价值：消息格式兼容性管理（避免破坏性变更）、减少消息体积（Schema ID 替代完整 Schema）、数据治理（统一格式）。一旦消息格式频繁变更或多团队协作，Schema Registry 的收益就非常明显。

**Q6：为什么 Kafka 消息推荐用 Avro 而不是 JSON？**

Avro 是二进制格式 + Schema 外置，消息体比 JSON 小 50-90%（因为不需要存储字段名），序列化/反序列化速度快 3-5 倍（二进制解析 vs 文本解析），且 Schema Registry 提供了自动化兼容性检查。

**Q7：Kafka on K8s 的最大挑战是什么？**

存储和网络的稳定性。Kafka 是为物理机/VM 设计的，依赖稳定的磁盘 I/O 和网络。K8s 的 Pod 漂移和共享网络可能影响性能。解决方案：本地 SSD StorageClass + StatefulSet + 反亲和性 + 专用节点池。

## 六、总结（速记版）

- **选型口诀**：日志 Kafka、交易 RocketMQ、企业 RabbitMQ
- **Go 客户端**：Sarama（流行）、confluent-kafka-go（高性能）、kafka-go（简洁）
- **Go 铁律**：手动提交 Offset、复用实例、避免并发提交、处理失败不提交
- **Connect**：数据集成标准化，少写胶水代码
- **Streams**：轻量流处理，比 Flink 简单但功能有限
- **MirrorMaker 2.0**：跨集群复制，保留 Offset 映射，多机房容灾
- **Schema Registry**：格式兼容性管理 + 消息体积优化 + 数据治理

---

# 十一、Go 实战：Kafka 开发场景代码大全

> 适用：Go 后端开发 · 可直接复制使用的生产级代码模板
> 依赖：`github.com/IBM/sarama` v1.43+

## 一、项目初始化与配置封装

### 1.1 依赖安装

```bash
go get github.com/IBM/sarama
```

### 1.2 统一配置结构体

```go
package kafka

import (
    "time"
    "github.com/IBM/sarama"
)

type Config struct {
    Brokers []string `yaml:"brokers"`   // ["localhost:9092","localhost:9093"]
    Version string   `yaml:"version"`   // "3.6.0"
    // Producer
    Producer ProducerConfig `yaml:"producer"`
    // Consumer
    Consumer ConsumerConfig `yaml:"consumer"`
}

type ProducerConfig struct {
    Acks           string        `yaml:"acks"`             // "all" / "1" / "0"
    Compression    string        `yaml:"compression"`      // "snappy" / "lz4" / "zstd"
    LingerMs       time.Duration `yaml:"linger_ms"`
    BatchSize      int           `yaml:"batch_size"`
    MaxInFlight    int           `yaml:"max_in_flight"`
    Idempotent     bool          `yaml:"idempotent"`
    RetryMax       int           `yaml:"retry_max"`
    ReturnSuccess  bool          `yaml:"return_success"`
}

type ConsumerConfig struct {
    GroupID              string        `yaml:"group_id"`
    AutoCommit           bool          `yaml:"auto_commit"`
    OffsetInitial        string        `yaml:"offset_initial"` // "newest" / "oldest"
    MaxPollRecords       int           `yaml:"max_poll_records"`
    MaxProcessingTime    time.Duration `yaml:"max_processing_time"`
    SessionTimeout       time.Duration `yaml:"session_timeout"`
    HeartbeatInterval    time.Duration `yaml:"heartbeat_interval"`
    RebalanceStrategy    string        `yaml:"rebalance_strategy"` // "sticky" / "roundrobin"
}
```

### 1.3 Sarama 配置工厂函数

```go
func NewSaramaConfig(cfg Config) (*sarama.Config, error) {
    saramaCfg := sarama.NewConfig()

    // 版本
    version, err := sarama.ParseKafkaVersion(cfg.Version)
    if err != nil {
        return nil, fmt.Errorf("parse kafka version: %w", err)
    }
    saramaCfg.Version = version

    // ============ Producer 默认 ============
    // acks
    switch cfg.Producer.Acks {
    case "all", "-1":
        saramaCfg.Producer.RequiredAcks = sarama.WaitForAll
    case "1":
        saramaCfg.Producer.RequiredAcks = sarama.WaitForLocal
    default:
        saramaCfg.Producer.RequiredAcks = sarama.NoResponse
    }
    // 压缩
    saramaCfg.Producer.Compression = parseCompression(cfg.Producer.Compression)
    // 幂等性
    saramaCfg.Producer.Idempotent = cfg.Producer.Idempotent
    if cfg.Producer.Idempotent {
        saramaCfg.Producer.RequiredAcks = sarama.WaitForAll
        saramaCfg.Producer.Retry.Max = cfg.Producer.RetryMax
        saramaCfg.Net.MaxOpenRequests = 5 // 幂等模式下允许 ≤5
    }
    // 批量参数
    saramaCfg.Producer.Flush.Frequency = cfg.Producer.LingerMs
    saramaCfg.Producer.Flush.Messages = cfg.Producer.BatchSize
    saramaCfg.Producer.Retry.Max = cfg.Producer.RetryMax
    saramaCfg.Producer.Return.Successes = cfg.Producer.ReturnSuccess
    saramaCfg.Producer.Return.Errors = true

    // ============ Consumer 默认 ============
    saramaCfg.Consumer.Offsets.AutoCommit.Enable = cfg.Consumer.AutoCommit
    saramaCfg.Consumer.Offsets.Initial = parseOffsetInitial(cfg.Consumer.OffsetInitial)
    saramaCfg.Consumer.MaxProcessingTime = cfg.Consumer.MaxProcessingTime
    saramaCfg.Consumer.Group.Session.Timeout = cfg.Consumer.SessionTimeout
    saramaCfg.Consumer.Group.Heartbeat.Interval = cfg.Consumer.HeartbeatInterval
    saramaCfg.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{
        parseRebalanceStrategy(cfg.Consumer.RebalanceStrategy),
    }
    saramaCfg.Consumer.MaxWaitTime = 500 * time.Millisecond

    // 全局网络
    saramaCfg.Net.MaxOpenRequests = cfg.Producer.MaxInFlight
    saramaCfg.ChannelBufferSize = 256

    return saramaCfg, nil
}

func parseCompression(c string) sarama.CompressionCodec {
    switch c {
    case "snappy": return sarama.CompressionSnappy
    case "lz4":    return sarama.CompressionLZ4
    case "zstd":   return sarama.CompressionZSTD
    case "gzip":   return sarama.CompressionGZIP
    default:       return sarama.CompressionNone
    }
}

func parseOffsetInitial(o string) int64 {
    switch o {
    case "oldest", "earliest": return sarama.OffsetOldest
    default:                    return sarama.OffsetNewest
    }
}

func parseRebalanceStrategy(s string) sarama.BalanceStrategy {
    switch s {
    case "roundrobin": return sarama.NewBalanceStrategyRoundRobin()
    case "range":      return sarama.NewBalanceStrategyRange()
    default:           return sarama.NewBalanceStrategySticky()
    }
}
```

---

## 二、Producer 实战场景

### 2.1 基础异步发送（99% 场景）

```go
type AsyncProducer struct {
    producer sarama.AsyncProducer
    logger   *slog.Logger
}

func NewAsyncProducer(cfg Config, logger *slog.Logger) (*AsyncProducer, error) {
    saramaCfg, err := NewSaramaConfig(cfg)
    if err != nil {
        return nil, err
    }
    p, err := sarama.NewAsyncProducer(cfg.Brokers, saramaCfg)
    if err != nil {
        return nil, fmt.Errorf("create async producer: %w", err)
    }
    ap := &AsyncProducer{producer: p, logger: logger}

    // 后台 Goroutine 处理结果：不要在回调中重试！Producer 已有 retries
    go ap.handleResults()

    return ap, nil
}

func (ap *AsyncProducer) handleResults() {
    for {
        select {
        case succ := <-ap.producer.Successes():
            // 仅记录监控指标，不做业务逻辑
            metrics.RecordSendSuccess(succ.Topic, succ.Partition, succ.Offset)

        case err := <-ap.producer.Errors():
            ap.logger.Error("send failed",
                "topic", err.Msg.Topic,
                "key", string(err.Msg.Key),
                "err", err.Err,
            )
            metrics.RecordSendFailure(err.Msg.Topic)
        }
    }
}

// Send 发送单条消息（非阻塞）
func (ap *AsyncProducer) Send(ctx context.Context, topic string, key, value []byte, headers map[string]string) {
    msg := &sarama.ProducerMessage{
        Topic:   topic,
        Key:     sarama.ByteEncoder(key),
        Value:   sarama.ByteEncoder(value),
        Headers: mapToHeaders(headers),
    }
    // 支持 context 取消
    select {
    case ap.producer.Input() <- msg:
    case <-ctx.Done():
        ap.logger.Warn("send cancelled by context", "topic", topic)
    }
}

func (ap *AsyncProducer) Close() error {
    return ap.producer.Close()
}

func mapToHeaders(m map[string]string) []sarama.RecordHeader {
    headers := make([]sarama.RecordHeader, 0, len(m))
    for k, v := range m {
        headers = append(headers, sarama.RecordHeader{Key: []byte(k), Value: []byte(v)})
    }
    return headers
}
```

### 2.2 同步发送（强依赖结果的场景）

```go
type SyncProducer struct {
    producer sarama.SyncProducer
}

func NewSyncProducer(cfg Config) (*SyncProducer, error) {
    saramaCfg, err := NewSaramaConfig(cfg)
    if err != nil {
        return nil, err
    }
    // 同步 Producer：MaxOpenRequests 自动为 1
    p, err := sarama.NewSyncProducer(cfg.Brokers, saramaCfg)
    if err != nil {
        return nil, err
    }
    return &SyncProducer{producer: p}, nil
}

// SendAndWait 发送并等待确认（阻塞）
func (sp *SyncProducer) SendAndWait(
    ctx context.Context, topic string, key, value []byte,
) (partition int32, offset int64, err error) {
    msg := &sarama.ProducerMessage{
        Topic:   topic,
        Key:     sarama.ByteEncoder(key),
        Value:   sarama.ByteEncoder(value),
        // 支持 context 超时（Sarama 不支持原生 context，用 Metadata 传）
        Metadata: ctx,
    }
    part, off, err := sp.producer.SendMessage(msg)
    return part, off, err
}
```

### 2.3 幂等 Producer（防网络重试重复）

```go
func NewIdempotentProducer(cfg Config) (*AsyncProducer, error) {
    cfg.Producer.Acks = "all"
    cfg.Producer.Idempotent = true
    cfg.Producer.MaxInFlight = 5        // 幂等模式下可 >1
    cfg.Producer.RetryMax = 3
    cfg.Producer.ReturnSuccess = true

    return NewAsyncProducer(cfg, slog.Default())
}
```

### 2.4 事务 Producer（consume-process-produce）

```go
// 典型场景：消费 Topic-A → 处理 → 写入 Topic-B → 提交消费 Offset（原子）
func TransactionalConsumeProcessProduce(
    cfg Config,
    transactionalID string,
    consumeTopic, produceTopic string,
) error {
    saramaCfg, err := NewSaramaConfig(cfg)
    if err != nil {
        return err
    }
    saramaCfg.Producer.Idempotent = true
    saramaCfg.Producer.RequiredAcks = sarama.WaitForAll
    saramaCfg.Producer.Transaction.ID = transactionalID
    // 事务超时：Broker 端 transaction.timeout.ms 默认 15 分钟，Client 端需 ≤ Broker 端
    saramaCfg.Producer.Transaction.Timeout = 10 * time.Minute

    // 1. 创建带事务能力的 Producer
    producer, err := sarama.NewAsyncProducer(cfg.Brokers, saramaCfg)
    if err != nil {
        return err
    }
    defer producer.Close()

    // 2. 初始化事务（向 TC 注册 Transactional ID → 获取 PID + Epoch）
    if err := producer.BeginTxn(); err != nil {
        return fmt.Errorf("begin txn: %w", err)
    }

    // 3. 消费消息（独立 Consumer，不使用 Consumer Group 简化示例）
    consumer, _ := sarama.NewConsumer(cfg.Brokers, saramaCfg)
    defer consumer.Close()
    pc, _ := consumer.ConsumePartition(consumeTopic, 0, sarama.OffsetOldest)

    txnSession := producer.NewTransactionSession()
    ctx := context.Background()

    for msg := range pc.Messages() {
        // 3a. 发送到产出 Topic（事务性写入）
        producer.Input() <- &sarama.ProducerMessage{
            Topic: produceTopic,
            Key:   msg.Key,
            Value: msg.Value,
        }

        // 3b. 每处理一批提交一次事务（原子提交：产出消息 + 消费 Offset）
        if err := producer.AddOffsetsToTxn(
            map[string][]*sarama.PartitionOffsetMetadata{
                consumeTopic: {
                    {Partition: msg.Partition, Offset: msg.Offset + 1},
                },
            },
            txnSession.ConsumerGroupID(),
        ); err != nil {
            producer.AbortTxn()
            return err
        }

        if err := producer.CommitTxn(); err != nil {
            return err
        }

        // 3c. 开启下一个事务
        producer.BeginTxn()
        _ = ctx // 实际使用 context 控制生命周期
    }

    return nil
}
```

### 2.5 自定义 Partitioner（热点 Key 拆分）

```go
// HotKeyPartitioner 对热点 key 做子分区，避免单分区过热
type HotKeyPartitioner struct {
    fallback     sarama.Partitioner
    hotKeyPrefix string // 如 "hot_user_"
    subPartCount int32  // 如 10
}

func NewHotKeyPartitioner(topic string) sarama.Partitioner {
    return &HotKeyPartitioner{
        fallback:     sarama.NewHashPartitioner(topic),
        hotKeyPrefix: "hot_",
        subPartCount: 10,
    }
}

func (p *HotKeyPartitioner) Partition(
    msg *sarama.ProducerMessage, numPartitions int32,
) (int32, error) {
    key, _ := msg.Key.Encode()

    if strings.HasPrefix(string(key), p.hotKeyPrefix) {
        // 热点 key → 随机子分区
        sub := int32(rand.Intn(int(p.subPartCount)))
        return sub % numPartitions, nil
    }
    // 普通 key → hash 分区
    return p.fallback.Partition(msg, numPartitions)
}

func (p *HotKeyPartitioner) RequiresConsistency() bool { return false }

// 使用
// saramaCfg.Producer.Partitioner = NewHotKeyPartitioner
```

### 2.6 批量发送优化示例

```go
// 批量发送 N 条消息，全部发送完才返回
func BatchSend(ctx context.Context, ap *AsyncProducer, topic string,
    msgs []Message, timeout time.Duration) (success, fail int) {

    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    var wg sync.WaitGroup
    var mu sync.Mutex

    for _, m := range msgs {
        wg.Add(1)
        go func(m Message) {
            defer wg.Done()
            select {
            case ap.producer.Input() <- &sarama.ProducerMessage{
                Topic: topic,
                Key:   sarama.StringEncoder(m.Key),
                Value: sarama.ByteEncoder(m.Value),
                Metadata: m,
            }:
                mu.Lock()
                success++
                mu.Unlock()
            case <-ctx.Done():
                mu.Lock()
                fail++
                mu.Unlock()
            }
        }(m)
    }

    wg.Wait()
    return success, fail
}
```

---

## 三、Consumer 实战场景

### 3.1 Consumer Group 标准模式

```go
type ConsumerGroupHandler struct {
    logger   *slog.Logger
    processor MessageProcessor // 业务处理接口
    ready    chan struct{}      // 标记就绪
}

type MessageProcessor interface {
    Process(ctx context.Context, msg *sarama.ConsumerMessage) error
}

// Setup 在新 session 开始前调用（Rebalance 完成后）
func (h *ConsumerGroupHandler) Setup(session sarama.ConsumerGroupSession) error {
    close(h.ready) // 通知外部：准备好消费了
    return nil
}

// Cleanup 在 session 结束时调用（Rebalance 开始前）
func (h *ConsumerGroupHandler) Cleanup(session sarama.ConsumerGroupSession) error {
    h.logger.Info("consumer session ending",
        "member_id", session.MemberID(),
        "generation_id", session.GenerationID(),
    )
    return nil
}

// ConsumeClaim 消费分区（每个分区一个 Goroutine）
func (h *ConsumerGroupHandler) ConsumeClaim(
    session sarama.ConsumerGroupSession,
    claim sarama.ConsumerGroupClaim,
) error {
    for {
        select {
        case msg, ok := <-claim.Messages():
            if !ok {
                return nil
            }

            // 处理业务
            if err := h.processor.Process(session.Context(), msg); err != nil {
                h.logger.Error("process message failed",
                    "topic", msg.Topic,
                    "partition", msg.Partition,
                    "offset", msg.Offset,
                    "err", err,
                )
                // 处理失败：不 Mark，下次 Rebalance 后重新消费
                continue
            }

            // 成功：标记消息（异步提交）
            session.MarkMessage(msg, "")

        case <-session.Context().Done():
            return nil
        }
    }
}

// ============ 启动消费者组 ============
func RunConsumerGroup(ctx context.Context, cfg Config, topics []string,
    processor MessageProcessor, logger *slog.Logger) error {

    saramaCfg, err := NewSaramaConfig(cfg)
    if err != nil {
        return err
    }
    saramaCfg.Consumer.Offsets.AutoCommit.Enable = false // 手动提交

    group, err := sarama.NewConsumerGroup(cfg.Brokers, cfg.Consumer.GroupID, saramaCfg)
    if err != nil {
        return fmt.Errorf("create consumer group: %w", err)
    }
    defer group.Close()

    handler := &ConsumerGroupHandler{
        logger:    logger,
        processor: processor,
        ready:     make(chan struct{}),
    }

    // 错误通道
    go func() {
        for err := range group.Errors() {
            logger.Error("consumer group error", "err", err)
        }
    }()

    // 消费循环
    for {
        handler.ready = make(chan struct{})

        // Consume 在每次 Rebalance 后会返回，外层 for 循环重新加入
        err := group.Consume(ctx, topics, handler)
        if err != nil {
            if errors.Is(err, sarama.ErrClosedConsumerGroup) {
                return nil // 正常关闭
            }
            logger.Error("consume error", "err", err)
            time.Sleep(time.Second) // 退避重试
        }

        // 检查是否需要退出
        select {
        case <-ctx.Done():
            return nil
        default:
        }
    }
}
```

### 3.2 手动 Offset 提交（分段提交）

```go
// 每秒提交一次，每次提交上一批成功消息的 Offset
type ManualCommitter struct {
    mu         sync.Mutex
    offsets    map[topicPartition]int64 // 待提交 Offset
    lastCommit map[topicPartition]int64 // 已提交 Offset
    session    sarama.ConsumerGroupSession
}

type topicPartition struct {
    Topic     string
    Partition int32
}

func (mc *ManualCommitter) Mark(topic string, partition int32, offset int64) {
    mc.mu.Lock()
    defer mc.mu.Unlock()
    mc.offsets[topicPartition{topic, partition}] = offset + 1
}

func (mc *ManualCommitter) StartCommitLoop(ctx context.Context) {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            mc.commit()
        case <-ctx.Done():
            mc.commit() // 退出前必须最后一次提交
            return
        }
    }
}

func (mc *ManualCommitter) commit() {
    mc.mu.Lock()
    toCommit := mc.offsets
    mc.offsets = make(map[topicPartition]int64)
    mc.mu.Unlock()

    for tp, off := range toCommit {
        mc.session.MarkOffset(tp.Topic, tp.Partition, off, "")
    }
    mc.session.Commit() // 触发一次提交请求
}
```

### 3.3 消费指定 Partition + 指定 Offset（回溯重放）

```go
func ConsumeFromOffset(
    ctx context.Context, brokers []string, topic string,
    partition int32, startOffset int64,
    handler func(*sarama.ConsumerMessage) error,
) error {
    cfg := sarama.NewConfig()
    cfg.Consumer.Return.Errors = true

    consumer, err := sarama.NewConsumer(brokers, cfg)
    if err != nil {
        return err
    }
    defer consumer.Close()

    // 获取分区的最早/最新 Offset（用于边界检查）
    oldest, _ := consumer.GetOffset(topic, partition, sarama.OffsetOldest)
    newest, _ := consumer.GetOffset(topic, partition, sarama.OffsetNewest)

    if startOffset < oldest {
        startOffset = oldest
    }

    pc, err := consumer.ConsumePartition(topic, partition, startOffset)
    if err != nil {
        return err
    }
    defer pc.Close()

    slog.Info("start consuming",
        "topic", topic,
        "partition", partition,
        "start_offset", startOffset,
        "newest_offset", newest,
    )

    for {
        select {
        case msg := <-pc.Messages():
            if err := handler(msg); err != nil {
                return err
            }
        case err := <-pc.Errors():
            slog.Error("partition consumer error", "err", err)
        case <-ctx.Done():
            slog.Info("consume cancelled", "topic", topic, "partition", partition)
            return nil
        }
    }
}
```

### 3.4 Rebalance 事件监听

```go
// Sarama 通过 ConsumerGroupHandler 的 Setup/Cleanup 暴露 Rebalance 事件
// 下面展示如何在业务层感知 Rebalance

type RebalanceAwareHandler struct {
    ConsumerGroupHandler
    onRebalanceStart  func()
    onRebalanceFinish func(partitions []string)
}

func (h *RebalanceAwareHandler) Setup(session sarama.ConsumerGroupSession) error {
    partitions := make([]string, 0)
    for _, p := range session.Claims() {
        for _, pid := range p {
            partitions = append(partitions, fmt.Sprintf("%d", pid))
        }
    }
    h.onRebalanceFinish(partitions)
    // 重置 ready channel（父类逻辑）
    return h.ConsumerGroupHandler.Setup(session)
}

func (h *RebalanceAwareHandler) Cleanup(session sarama.ConsumerGroupSession) error {
    h.onRebalanceStart()
    return h.ConsumerGroupHandler.Cleanup(session)
}

// 使用示例
// handler := &RebalanceAwareHandler{
//     onRebalanceStart:  func() { metrics.IncRebalance() },
//     onRebalanceFinish: func(ps []string) { log.Printf("assigned partitions: %v", ps) },
// }
```

### 3.5 多 Topic 消费

```go
// 同一个 Consumer Group 可以订阅多个 Topic
// Consumer Leader 制定分配方案时会把所有 Topic 的分区统一分配

func RunMultiTopicConsumer(ctx context.Context, cfg Config,
    topics []string, // ["orders", "payments", "notifications"]
    processor MessageProcessor,
) error {
    // ... 同 RunConsumerGroup，只是 subscriptions 是多个 Topic
    return RunConsumerGroup(ctx, cfg, topics, processor, slog.Default())
}

// 在 ConsumeClaim 中根据 Topic 做路由
func (h *ConsumerGroupHandler) ConsumeClaim(
    session sarama.ConsumerGroupSession,
    claim sarama.ConsumerGroupClaim,
) error {
    for msg := range claim.Messages() {
        switch msg.Topic {
        case "orders":
            h.processOrder(session.Context(), msg)
        case "payments":
            h.processPayment(session.Context(), msg)
        default:
            h.processDefault(session.Context(), msg)
        }
        session.MarkMessage(msg, "")
    }
    return nil
}
```

---

## 四、典型业务场景

### 4.1 异步任务队列（任务分发）

```go
// 场景：用户提交视频转码任务 → Kafka → Worker 消费执行

// ===== Producer 端：提交任务 =====
type TaskMessage struct {
    TaskID   string    `json:"task_id"`
    TaskType string    `json:"task_type"` // "transcode" / "thumbnail" / "merge"
    Payload  string    `json:"payload"`   // JSON blob
    CreateAt time.Time `json:"create_at"`
}

func SubmitTask(ctx context.Context, ap *AsyncProducer, task TaskMessage) error {
    data, _ := json.Marshal(task)
    ap.Send(ctx, "video-tasks",
        []byte(task.TaskID), // key: 相同 taskID 重试有序
        data,
        map[string]string{
            "task_type": task.TaskType,
            "version":   "v1",
        },
    )
    return nil
}

// ===== Consumer 端：Worker 执行 =====
type TaskProcessor struct {
    workers map[string]TaskHandler
}

type TaskHandler func(ctx context.Context, msg TaskMessage) error

func (tp *TaskProcessor) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    var task TaskMessage
    if err := json.Unmarshal(msg.Value, &task); err != nil {
        return fmt.Errorf("unmarshal task: %w", err) // 不可恢复，需丢弃并告警
    }

    handler, ok := tp.workers[task.TaskType]
    if !ok {
        return fmt.Errorf("unknown task type: %s", task.TaskType)
    }

    // 业务去重（用 Redis SET NX）
    dedupKey := fmt.Sprintf("task:dedup:%s", task.TaskID)
    if !redisSetNX(ctx, dedupKey, "1", 24*time.Hour) {
        slog.Warn("duplicate task, skip", "task_id", task.TaskID)
        return nil // 幂等：重复任务视为成功
    }

    return handler(ctx, task)
}
```

### 4.2 事件驱动架构（Event-Driven）

```go
// 场景：订单创建 → 发布 OrderCreated 事件 → 多个下游服务独立消费
//        （物流服务、通知服务、数据分析各自消费同一 Topic、不同 Group）

// ===== 事件定义 =====
type OrderCreatedEvent struct {
    OrderID   string    `json:"order_id"`
    UserID    string    `json:"user_id"`
    Amount    int64     `json:"amount"`
    Items     []string  `json:"items"`
    Timestamp time.Time `json:"timestamp"`
}

// ===== 发布事件（带 TraceID） =====
func PublishOrderCreated(ctx context.Context, ap *AsyncProducer, event OrderCreatedEvent) {
    data, _ := json.Marshal(event)
    ap.Send(ctx, "order-events",
        []byte(event.OrderID), // key = orderID → 同订单事件有序
        data,
        map[string]string{
            "event_type": "OrderCreated",
            "trace_id":   trace.SpanContextFromContext(ctx).TraceID().String(),
        },
    )
}

// ===== 消费者 A：物流服务（独立 Group） =====
// group.id = "logistics-service"
// 消费 order-events → 创建运单
func (p *LogisticsConsumer) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    eventType := getHeader(msg, "event_type")
    if eventType != "OrderCreated" {
        return nil // 只处理关心的事件
    }
    var event OrderCreatedEvent
    json.Unmarshal(msg.Value, &event)

    // 从 Headers 恢复 TraceID
    ctx = trace.ContextWithSpanContext(ctx, trace.SpanContextFromContext(ctx).
        WithTraceID(parseTraceID(getHeader(msg, "trace_id"))))

    return p.createShipment(ctx, event)
}

// ===== 消费者 B：通知服务 =====
// group.id = "notification-service"  ← 不同 Group，独立消费同一 Topic
// 消费 order-events → 发送邮件/短信
```

### 4.3 死信队列完整实现

```go
type DLQConfig struct {
    MainTopic    string // "orders"
    RetryTopic   string // "orders-retry"
    DLQTopic     string // "orders-dlq"
    MaxRetries   int    // 3
    RetryBackoff []time.Duration // [1s, 10s, 60s]
}

type DLQProducer struct {
    ap *AsyncProducer
}

// SendToRetry 发送到重试 Topic（携带重试计数）
func (dp *DLQProducer) SendToRetry(
    ctx context.Context,
    originalMsg *sarama.ConsumerMessage,
    retryCount int,
    err error,
) {
    // 递增重试次数（存在 Header 中）
    headers := headersToMap(originalMsg.Headers)
    headers["x-retry-count"] = strconv.Itoa(retryCount + 1)
    headers["x-original-topic"] = originalMsg.Topic
    headers["x-error"] = err.Error()
    headers["x-error-time"] = time.Now().Format(time.RFC3339)

    dp.ap.Send(ctx, "orders-retry",
        originalMsg.Key,    // 保留原始 key
        originalMsg.Value,  // 保留原始 value
        headers,
    )
}

// SendToDLQ 发送到死信 Topic（终态：人工处理）
func (dp *DLQProducer) SendToDLQ(
    ctx context.Context,
    originalMsg *sarama.ConsumerMessage,
    retryCount int,
    lastError error,
) {
    headers := headersToMap(originalMsg.Headers)
    headers["x-retry-count"] = strconv.Itoa(retryCount)
    headers["x-original-topic"] = originalMsg.Topic
    headers["x-final-error"] = lastError.Error()
    headers["x-dlq-time"] = time.Now().Format(time.RFC3339Nano)

    dp.ap.Send(ctx, "orders-dlq",
        originalMsg.Key,
        originalMsg.Value,
        headers,
    )
    metrics.DLQIncrement("orders")
}

// ===== Consumer 端集成死信 =====
type DLQProcessor struct {
    dlq      *DLQProducer
    config   DLQConfig
    delegate MessageProcessor // 真实业务处理器
}

func (p *DLQProcessor) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    retryCount, _ := strconv.Atoi(getHeader(msg, "x-retry-count"))

    if err := p.delegate.Process(ctx, msg); err != nil {
        if retryCount >= p.config.MaxRetries {
            // 进入死信
            p.dlq.SendToDLQ(ctx, msg, retryCount, err)
            slog.Error("message sent to DLQ",
                "topic", msg.Topic,
                "partition", msg.Partition,
                "offset", msg.Offset,
                "retry_count", retryCount,
                "err", err,
            )
            return nil // 已进死信，视为消费成功（不阻塞后续消息）
        }

        // 进入重试 Topic
        p.dlq.SendToRetry(ctx, msg, retryCount, err)
        return nil // 同上
    }

    return nil
}

// ===== 重试 Topic 的消费者（延迟消费） =====
func RunRetryConsumer(ctx context.Context, cfg Config, delegate MessageProcessor) {
    // 重试 Topic 的 Consumer Group 独立于主 Topic
    cfg.Consumer.GroupID = "orders-retry-consumer"
    // 消费 orders-retry → 处理 → 成功则提交，失败再进 DLQ
    // （使用相同的 DLQProcessor 包装）
}
```

### 4.4 延时消费（基于多 Topic + 时间轮）

```go
// Kafka 没有原生延迟消息，用"多级延迟 Topic + 定时扫描"模拟

// 延迟级别：5s, 30s, 1min, 5min, 30min
var DelayLevels = []struct {
    Topic    string
    Delay    time.Duration
}{
    {"delay-5s", 5 * time.Second},
    {"delay-30s", 30 * time.Second},
    {"delay-1m", 1 * time.Minute},
    {"delay-5m", 5 * time.Minute},
    {"delay-30m", 30 * time.Minute},
}

// 发送延迟消息
func SendDelayed(ctx context.Context, ap *AsyncProducer,
    targetTopic string, key, value []byte, delay time.Duration,
) {
    // 选择最近的延迟级别
    level := selectDelayLevel(delay)
    headers := map[string]string{
        "x-target-topic": targetTopic,
        "x-deliver-after": time.Now().Add(delay).Format(time.RFC3339),
    }
    ap.Send(ctx, level.Topic, key, value, headers)
}

// 延迟消费者：到期后投递到目标 Topic
func DelayConsumer(ctx context.Context, delayLevel DelayLevel,
    ap *AsyncProducer, targetBrokers []string,
) error {
    // 消费 delay-5s / delay-30s 等 Topic
    // 检查 x-deliver-after Header
    // 如果时间到了 → 投递到 x-target-topic
    // 如果时间未到 → 不提交 Offset，等下次 poll 重新消费

    return RunConsumerGroup(ctx, Config{
        Brokers:  targetBrokers,
        Consumer: ConsumerConfig{
            GroupID:       fmt.Sprintf("delay-worker-%s", delayLevel.Topic),
            AutoCommit:    false,
            OffsetInitial: "newest",
        },
    }, []string{delayLevel.Topic}, &delayProcessor{
        ap:          ap,
        delayUntil:  delayLevel.Delay,
        targetTopic: "", // 从 Header 获取
    }, slog.Default())
}

type delayProcessor struct {
    ap         *AsyncProducer
    delayUntil time.Duration
    targetTopic string
}

func (p *delayProcessor) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    deliverAfter, _ := time.Parse(time.RFC3339, getHeader(msg, "x-deliver-after"))
    targetTopic := getHeader(msg, "x-target-topic")

    if time.Now().After(deliverAfter) {
        // 到期 → 投递到目标 Topic
        cleanHeaders := make(map[string]string)
        for _, h := range msg.Headers {
            if !strings.HasPrefix(string(h.Key), "x-") {
                cleanHeaders[string(h.Key)] = string(h.Value)
            }
        }
        p.ap.Send(ctx, targetTopic, msg.Key, msg.Value, cleanHeaders)
        return nil // 投递成功，Mark
    }
    // 未到期 → 不提交 Offset（下次重新消费）
    return fmt.Errorf("not yet, deliver after %s", deliverAfter)
}
```

### 4.5 批量聚合处理（攒批写 DB）

```go
// 攒够 N 条或 T 秒 → 批量写 MySQL，减少 DB 连接开销
type BatchProcessor struct {
    buffer   []MessageItem
    mu       sync.Mutex
    batchSize int           // 100
    flushInterval time.Duration // 5 秒
    dao      *OrderDAO
}

type MessageItem struct {
    Offset    int64
    Partition int32
    Message   *sarama.ConsumerMessage
}

func (bp *BatchProcessor) Start(ctx context.Context) {
    ticker := time.NewTicker(bp.flushInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            bp.flush(ctx)
        case <-ctx.Done():
            bp.flush(ctx) // 退出前最后 flush
            return
        }
    }
}

func (bp *BatchProcessor) Add(item MessageItem) {
    bp.mu.Lock()
    bp.buffer = append(bp.buffer, item)
    shouldFlush := len(bp.buffer) >= bp.batchSize
    bp.mu.Unlock()

    if shouldFlush {
        bp.flush(context.Background()) // 同步 flush 或通过 channel 通知
    }
}

func (bp *BatchProcessor) flush(ctx context.Context) {
    bp.mu.Lock()
    if len(bp.buffer) == 0 {
        bp.mu.Unlock()
        return
    }
    batch := bp.buffer
    bp.buffer = nil
    bp.mu.Unlock()

    // 批量 INSERT
    if err := bp.dao.BatchInsert(ctx, batch); err != nil {
        slog.Error("batch insert failed", "count", len(batch), "err", err)
        // 失败：消息不放回 buffer（由主消费循环处理重试逻辑）
        return
    }

    // 成功后由主消费循环统一 MarkOffset
    for _, item := range batch {
        metrics.RecordBatchSuccess(item.Message.Topic, item.Partition)
    }
}
```

### 4.6 顺序消费保证

```go
// 场景：同一个 Order 的创建、支付、发货必须有序

// 关键：相同 key → 同一分区 → 分区内有序 → 消费端串行处理

// Producer：用 OrderID 做 key
func PublishOrderEvent(ctx context.Context, ap *AsyncProducer, orderID string, event any) {
    data, _ := json.Marshal(event)
    ap.Send(ctx, "order-lifecycle", []byte(orderID), data, nil)
    // orderID 相同 → hash 后进同一分区 → 顺序保证
}

// Consumer：单分区内 Goroutine 必须串行处理
// Sarama 的 ConsumeClaim 每个分区一个 Goroutine，天然串行 ← 不要开新 Goroutine！
func (p *OrderLifecycleProcessor) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    var event OrderEvent
    json.Unmarshal(msg.Value, &event)

    switch event.Type {
    case "created":
        return p.handleCreated(ctx, event)
    case "paid":
        return p.handlePaid(ctx, event)
    case "shipped":
        return p.handleShipped(ctx, event)
    }
    return nil
}

// ⚠️ 错误做法（破坏顺序）：
// go func() { p.handleCreated(ctx, event) }()  ← 开 Goroutine 乱序！

// ✅ 正确做法：单分区内串行处理，只在需要并行时开 Worker 池 + 保证相同 key 进同一 Worker
```

---

## 五、可靠性 & 容错

### 5.1 优雅关闭（Graceful Shutdown）

```go
func RunWithGracefulShutdown(
    ctx context.Context,
    cfg Config,
    topics []string,
    processor MessageProcessor,
) error {
    // 捕获系统信号
    ctx, cancel := signal.NotifyContext(ctx, syscall.SIGINT, syscall.SIGTERM)
    defer cancel()

    group, err := sarama.NewConsumerGroup(cfg.Brokers, cfg.Consumer.GroupID, mustConfig(cfg))
    if err != nil {
        return err
    }

    handler := &ConsumerGroupHandler{
        logger:    slog.Default(),
        processor: processor,
        ready:     make(chan struct{}),
    }

    // 消费循环
    consumerErr := make(chan error, 1)
    go func() {
        defer close(consumerErr)
        for {
            if err := group.Consume(ctx, topics, handler); err != nil {
                if errors.Is(err, sarama.ErrClosedConsumerGroup) {
                    return
                }
                consumerErr <- err
                return
            }
            if ctx.Err() != nil {
                return
            }
            handler.ready = make(chan struct{})
        }
    }()

    // 等待信号或错误
    select {
    case <-ctx.Done():
        slog.Info("received shutdown signal")
    case err := <-consumerErr:
        slog.Error("consumer error, shutting down", "err", err)
    }

    // 1. 先取消 context（停止 Consumer 内部的新 poll）
    cancel()

    // 2. 等待当前批次处理完成（最多 30 秒）
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer shutdownCancel()

    done := make(chan struct{})
    go func() {
        group.Close() // 触发 LeaveGroup + 最后一次 Offset 提交
        close(done)
    }()

    select {
    case <-done:
        slog.Info("graceful shutdown complete")
    case <-shutdownCtx.Done():
        slog.Warn("graceful shutdown timed out, forcing close")
    }

    return nil
}
```

### 5.2 消费重试与指数退避

```go
// 带指数退避的重试处理器
type RetryProcessor struct {
    delegate    MessageProcessor
    maxRetries  int
    baseBackoff time.Duration // 100ms
    maxBackoff  time.Duration // 10s
}

func (p *RetryProcessor) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    var lastErr error
    for attempt := 0; attempt <= p.maxRetries; attempt++ {
        if attempt > 0 {
            backoff := time.Duration(math.Min(
                float64(p.baseBackoff)*math.Pow(2, float64(attempt-1)),
                float64(p.maxBackoff),
            ))
            slog.Info("retrying", "attempt", attempt, "backoff", backoff)
            select {
            case <-time.After(backoff):
            case <-ctx.Done():
                return ctx.Err()
            }
        }

        lastErr = p.delegate.Process(ctx, msg)
        if lastErr == nil {
            return nil
        }

        // 判断是否可重试
        if !isRetryable(lastErr) {
            return lastErr // 不可重试的错误直接返回
        }
    }
    return fmt.Errorf("exhausted %d retries: %w", p.maxRetries, lastErr)
}

func isRetryable(err error) bool {
    // 超时、网络错误、DB 死锁 → 可重试
    // 参数校验错误、数据格式错误 → 不可重试
    if errors.Is(err, context.DeadlineExceeded) { return true }
    if errors.Is(err, syscall.ECONNRESET) { return true }
    return false
}
```

### 5.3 消费熔断器

```go
// 当下游（DB/RPC）故障率超过阈值时自动暂停消费，避免雪崩
type CircuitBreaker struct {
    failureCount    atomic.Int64
    successCount    atomic.Int64
    threshold       float64 // 0.5 = 50% 失败率触发熔断
    windowSize      int     // 滑动窗口大小（请求数）
    cooldownPeriod  time.Duration // 熔断冷却期 30s
    state           atomic.Int32   // 0=Closed, 1=Open, 2=HalfOpen
    lastOpenTime    atomic.Value   // time.Time
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    switch cb.state.Load() {
    case 1: // Open
        if time.Since(cb.lastOpenTime.Load().(time.Time)) < cb.cooldownPeriod {
            return fmt.Errorf("circuit breaker open")
        }
        cb.state.Store(2) // → HalfOpen
    }

    err := fn()

    total := float64(cb.failureCount.Load() + cb.successCount.Load())
    if total >= float64(cb.windowSize) {
        failureRate := float64(cb.failureCount.Load()) / total
        if failureRate >= cb.threshold {
            cb.state.Store(1) // Open
            cb.lastOpenTime.Store(time.Now())
            cb.failureCount.Store(0)
            cb.successCount.Store(0)
        }
    }

    if err != nil {
        cb.failureCount.Add(1)
    } else {
        cb.successCount.Add(1)
    }
    return err
}

// 在 Consumer 中使用
func (p *CircuitBreakerProcessor) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    return p.breaker.Call(func() error {
        return p.delegate.Process(ctx, msg)
    })
    // 熔断时返回 error → Consumer 不提交 Offset → 消费暂停（不丢消息）
}
```

### 5.4 消费端去重（Redis + 本地缓存）

```go
type Deduplicator struct {
    redis  *redis.Client
    local  *cache.Cache // 本地 LRU 缓存，TTL 5 分钟
    ttl    time.Duration
}

func NewDeduplicator(redis *redis.Client) *Deduplicator {
    return &Deduplicator{
        redis: redis,
        local: cache.New(5*time.Minute, 10*time.Minute),
        ttl:   24 * time.Hour,
    }
}

func (d *Deduplicator) IsDuplicate(ctx context.Context, msgKey string) (bool, error) {
    // L1: 本地缓存（减少 Redis 调用）
    if _, found := d.local.Get(msgKey); found {
        return true, nil
    }

    // L2: Redis
    ok, err := d.redis.SetNX(ctx, "dedup:"+msgKey, "1", d.ttl).Result()
    if err != nil {
        return false, err
    }
    if !ok {
        d.local.Set(msgKey, true, cache.DefaultExpiration)
        return true, nil
    }
    return false, nil
}
```

---

## 六、可观测性

### 6.1 Prometheus Metrics

```go
package metrics

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    // Producer
    MessagesSent = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "kafka_producer_messages_sent_total",
        Help: "Total messages sent successfully",
    }, []string{"topic"})

    MessagesFailed = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "kafka_producer_messages_failed_total",
        Help: "Total messages failed to send",
    }, []string{"topic"})

    SendLatency = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "kafka_producer_send_latency_seconds",
        Help:    "Producer send latency",
        Buckets: []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1},
    }, []string{"topic"})

    // Consumer
    MessagesConsumed = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "kafka_consumer_messages_consumed_total",
        Help: "Total messages consumed",
    }, []string{"topic", "partition"})

    ConsumerLag = promauto.NewGaugeVec(prometheus.GaugeOpts{
        Name: "kafka_consumer_lag",
        Help: "Consumer lag per partition",
    }, []string{"topic", "partition", "group"})

    ProcessingLatency = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "kafka_consumer_processing_latency_seconds",
        Help:    "Consumer processing latency",
        Buckets: prometheus.DefBuckets,
    }, []string{"topic"})

    // Rebalance
    RebalancesTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "kafka_consumer_rebalances_total",
        Help: "Total consumer group rebalances",
    }, []string{"group"})

    // DLQ
    DLQMetrics = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "kafka_dlq_messages_total",
        Help: "Total messages sent to DLQ",
    }, []string{"topic"})
)

func RecordSendSuccess(topic string, partition int32, offset int64) {
    MessagesSent.WithLabelValues(topic).Inc()
}

func RecordSendFailure(topic string) {
    MessagesFailed.WithLabelValues(topic).Inc()
}

func DLQIncrement(topic string) {
    DLQMetrics.WithLabelValues(topic).Inc()
}

func IncRebalance() { RebalancesTotal.WithLabelValues("my-group").Inc() }

// ===== 在 Consumer 中上报 Lag =====
func (h *ConsumerGroupHandler) Setup(session sarama.ConsumerGroupSession) error {
    // 上报当前消费 Lag
    for topic, partitions := range session.Claims() {
        for _, p := range partitions {
            ConsumerLag.WithLabelValues(topic, fmt.Sprintf("%d", p), "my-group").Set(0)
        }
    }
    close(h.ready)
    return nil
}

func RecordBatchSuccess(topic string, partition int32) {
    MessagesConsumed.WithLabelValues(topic, fmt.Sprintf("%d", partition)).Inc()
}

// ===== 导出 HTTP Metrics =====
func StartMetricsServer(addr string) {
    http.Handle("/metrics", promhttp.Handler())
    go http.ListenAndServe(addr, nil)
}
```

### 6.2 OpenTelemetry 全链路追踪

```go
// 在 Producer 端注入 Trace Context（通过 Kafka Headers）
func InjectTraceContext(ctx context.Context, msg *sarama.ProducerMessage) {
    span := trace.SpanFromContext(ctx)
    if !span.SpanContext().IsValid() {
        return
    }
    propagator := propagation.TraceContext{}
    carrier := propagation.MapCarrier{}
    propagator.Inject(ctx, carrier)

    for k, v := range carrier {
        msg.Headers = append(msg.Headers, sarama.RecordHeader{Key: []byte(k), Value: []byte(v)})
    }
}

// 在 Consumer 端恢复 Trace Context
func ExtractTraceContext(msg *sarama.ConsumerMessage) context.Context {
    carrier := propagation.MapCarrier{}
    for _, h := range msg.Headers {
        carrier[string(h.Key)] = string(h.Value)
    }
    propagator := propagation.TraceContext{}
    return propagator.Extract(context.Background(), carrier)
}

// 使用示例
func (p *TracedProcessor) Process(ctx context.Context, msg *sarama.ConsumerMessage) error {
    // 从 Kafka Headers 恢复 Trace Context
    ctx = ExtractTraceContext(msg)
    ctx, span := otel.Tracer("kafka-consumer").Start(ctx, "process-order")
    defer span.End()

    span.SetAttributes(
        attribute.String("kafka.topic", msg.Topic),
        attribute.Int("kafka.partition", int(msg.Partition)),
        attribute.Int64("kafka.offset", msg.Offset),
    )

    return p.delegate.Process(ctx, msg)
}
```

### 6.3 健康检查

```go
// Kubernetes liveness/readiness probes
type HealthChecker struct {
    producer    sarama.AsyncProducer
    consumerGrp sarama.ConsumerGroup
}

// Liveness：进程是否存活（简单返回 OK）
func (hc *HealthChecker) LivenessHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("ok"))
}

// Readiness：能否正常读写 Kafka
func (hc *HealthChecker) ReadinessHandler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
    defer cancel()

    // Producer 健康：发送一条测试消息到内部健康检查 Topic
    hc.producer.Input() <- &sarama.ProducerMessage{
        Topic: "__health_check",
        Value: sarama.StringEncoder("ping"),
    }

    // Consumer 健康：Consumer Group 是否在 Stable 状态
    // Sarama 没有直接暴露 Group 状态，通过检查最近 poll 时间间接判断

    select {
    case <-ctx.Done():
        w.WriteHeader(http.StatusServiceUnavailable)
        w.Write([]byte("timeout"))
    default:
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("ready"))
    }
}
```

---

## 七、测试

### 7.1 单元测试（Mock Sarama）

```go
// 使用接口抽象 Kafka 依赖，测试时 Mock
type MessagePublisher interface {
    Publish(ctx context.Context, topic string, key, value []byte, headers map[string]string) error
}

// 真实实现
type KafkaPublisher struct {
    ap *AsyncProducer
}

func (p *KafkaPublisher) Publish(ctx context.Context, topic string, key, value []byte, headers map[string]string) error {
    p.ap.Send(ctx, topic, key, value, headers)
    return nil
}

// Mock 实现（测试用）
type MockPublisher struct {
    Messages []ProducerMessage
    mu       sync.Mutex
}

func (m *MockPublisher) Publish(ctx context.Context, topic string, key, value []byte, headers map[string]string) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.Messages = append(m.Messages, ProducerMessage{Topic: topic, Key: key, Value: value, Headers: headers})
    return nil
}

// 测试业务逻辑时注入 MockPublisher
func TestOrderService_CreateOrder(t *testing.T) {
    mockPub := &MockPublisher{}
    svc := NewOrderService(mockPub)

    err := svc.CreateOrder(context.Background(), OrderRequest{UserID: "123"})
    assert.NoError(t, err)
    assert.Len(t, mockPub.Messages, 1)
    assert.Equal(t, "order-events", mockPub.Messages[0].Topic)
}
```

### 7.2 集成测试（Testcontainers + Kafka）

```go
// 使用 testcontainers-go 启动真实 Kafka 做集成测试
// go get github.com/testcontainers/testcontainers-go/modules/kafka

import (
    "context"
    "testing"
    "time"
    "github.com/IBM/sarama"
    "github.com/testcontainers/testcontainers-go/modules/kafka"
)

func TestKafkaIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }

    ctx := context.Background()

    // 1. 启动 Kafka 容器
    kafkaContainer, err := kafka.Run(
        ctx,
        "confluentinc/confluent-local:7.6.0",
        kafka.WithClusterID("test-cluster"),
    )
    if err != nil {
        t.Fatal(err)
    }
    defer kafkaContainer.Terminate(ctx)

    brokers, err := kafkaContainer.Brokers(ctx)
    if err != nil {
        t.Fatal(err)
    }

    // 2. 创建测试 Topic
    admin, _ := sarama.NewClusterAdmin(brokers, sarama.NewConfig())
    admin.CreateTopic("test-topic", &sarama.TopicDetail{
        NumPartitions:     1,
        ReplicationFactor: 1,
    }, false)
    defer admin.Close()

    // 3. 发送消息
    cfg := sarama.NewConfig()
    cfg.Producer.Return.Successes = true
    producer, _ := sarama.NewSyncProducer(brokers, cfg)
    defer producer.Close()

    msg := &sarama.ProducerMessage{
        Topic: "test-topic",
        Key:   sarama.StringEncoder("key-1"),
        Value: sarama.StringEncoder("hello world"),
    }
    part, off, err := producer.SendMessage(msg)
    assert.NoError(t, err)
    t.Logf("sent to partition %d offset %d", part, off)

    // 4. 消费消息
    consumer, _ := sarama.NewConsumer(brokers, sarama.NewConfig())
    defer consumer.Close()
    pc, _ := consumer.ConsumePartition("test-topic", 0, sarama.OffsetOldest)

    select {
    case received := <-pc.Messages():
        assert.Equal(t, "key-1", string(received.Key))
        assert.Equal(t, "hello world", string(received.Value))
    case <-time.After(10 * time.Second):
        t.Fatal("timeout waiting for message")
    }
}
```

---

## 八、速记总结

- **Iron Law**：手动提交 Offset、复用 Producer/Consumer 实例、处理失败不提交、同 key 有序不并发
- **面试常用**：
  - 异步发送 + Success/Error channel 处理结果（不手动重试）
  - `sarama.NewBalanceStrategySticky()` + 静态成员减少 Rebalance
  - `sarama.Producer.Idempotent = true` 防网络重试重复
  - 死信队列：用 Header 传 `x-retry-count` + `x-original-topic`
- **生产必备**：
  - Prometheus 监控 QPS/失败率/Lag/Rebalance 次数
  - OpenTelemetry TraceID 通过 Headers 跨服务传递
  - 优雅关闭：signals → cancel ctx → drain batch → commit offset → Close

---

# 十二、附录：面试速查

## 常见命令速查

```bash
# Topic 操作
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic first --partitions 3 --replication-factor 3
bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic first
bin/kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic first --partitions 6  # 只能增不能减

# 消费者组
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group test-group  # 看 LAG

# 查看 Offset
bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic topic_name --time -1  # 最新
bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic topic_name --time -2  # 最早

# 测试
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic first
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic first

# 配额管理
bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type users --entity-name UserA --describe
bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type users --entity-name UserA --alter --add-config 'producer_byte_rate=1048576'

# ACL 管理
bin/kafka-acls.sh --bootstrap-server localhost:9092 --list
```

## 高频面试题 TOP 25

| # | 问题 | 核心答案要点 |
|---|------|-------------|
| 1 | Kafka 是什么？ | 分布式事件流平台，本质是分布式提交日志 |
| 2 | 为什么快？ | 六大支柱：分区并行、顺序写、PageCache、零拷贝、批量、压缩 |
| 3 | 如何保证不丢？ | Producer acks=all + Broker rf≥3 + min.isr≥2 + Consumer 手动提交 |
| 4 | 重复消费怎么办？ | 至少一次语义是 Kafka 默认，业务幂等解决（唯一键/去重表/Redis） |
| 5 | 如何保证顺序？ | 单分区有序。key 路由同一分区 + inflight=1 或开启幂等性 |
| 6 | acks 三种级别？ | 0=不等确认，1=Leader 确认，all=ISR 全确认 |
| 7 | ISR 是什么？ | 与 Leader 保持同步的副本集合，动态调整 |
| 8 | HW/LEO 概念？ | LEO=日志末尾，HW=ISR 最小 LEO，消费者只能读到 HW 之前 |
| 9 | 幂等性怎么实现？ | PID + Partition + SeqNumber 三元组去重，单会话单分区有效 |
| 10 | 事务解决什么？ | 跨分区/跨 Topic 写入 + Offset 提交的原子性；Zombie Fencing 防僵尸 Producer |
| 11 | Rebalance 触发条件？ | Consumer 上下线、心跳超时、poll 超时、分区变化、订阅变化 |
| 12 | 怎么减少 Rebalance？ | 静态成员、CooperativeSticky、合理 max.poll.records、解耦 poll 和处理 |
| 13 | Offset 存哪？ | `__consumer_offsets` 内部 Topic，compact 策略 |
| 14 | 消息积压排查？ | 生产流量 → 消费速度 → 分区数 → 参数 → Rebalance 五步法 |
| 15 | Pull vs Push？ | Pull：按能力消费，批量拉取，灵活 Offset 管理 |
| 16 | ZK vs KRaft？ | ZK 外部依赖，KRaft 内聚管理；Kafka 4.0 起仅支持 KRaft |
| 17 | Partition 为何不能减少？ | 数据迁移成本、Offset 断裂、Key 映射破坏 |
| 18 | 分区分配策略对比？ | Range(不均)、RR(均匀)、Sticky(减少迁移)、CooperativeSticky(增量) |
| 19 | MQ 三者对比？ | Kafka=日志高吞吐、RocketMQ=业务消息延迟事务、RabbitMQ=传统路由 |
| 20 | Kafka 到 MySQL 的 Exactly Once？ | Kafka 不能天然保证，需业务幂等或两阶段提交 |
| 21 | 死信队列怎么设计？ | 有限重试 → DLT → 告警 + 人工补偿，独立消费者不阻塞主链路 |
| 22 | Rebalance 风暴？ | 大量 Consumer 同时重启 → 密集 Rebalance → STW。静态成员 + 分批 + CooperativeSticky |
| 23 | 多机房数据同步？ | MirrorMaker 2.0，保留 Offset 映射，Consumer 可无缝切换 |
| 24 | 磁盘不均衡？ | kafka-reassign-partitions.sh 在线迁移 + Preferred Leader Election 修复 |
| 25 | 监控哪些指标？ | Producer: QPS/失败率/延迟 \| Broker: 磁盘/URP/ISR/ActiveController \| Consumer: Lag/速率/Rebalance |
| 26 | Rack Awareness？ | broker.rack 保证副本跨 AZ，配合 min.isr 和 Follower Fetching 实现就近读取 |
| 27 | 网络分区/脑裂怎么防？ | Controller Epoch + Leader Epoch + KRaft Raft Term 三层防护 |
| 28 | GC 对 Kafka 的影响？ | Full GC → ISR 收缩 → 延迟毛刺 → Controller 可能被误判宕机 → 堆别太大、留给 OS |
| 29 | JBOD vs RAID？ | Kafka 推荐 JBOD：副本已提供冗余，RAID 多余；JBOD 多盘并行 I/O 更高 |
| 30 | Preferred Leader Election？ | 宕机恢复后 Leader 不自动迁回；用 kafka-leader-election.sh 或 auto.leader.rebalance 修复 |

## 场景题强化

**场景 1：设计一个秒杀系统的异步消息架构**

> 需求：100 万并发下单 → 异步削峰 → 库存扣减 → 订单生成，要求不丢消息、不超卖。

**解答思路**：
1. **前端/网关层限流**：令牌桶/漏桶 → 只放行 N 个请求进入后端
2. **下单请求 → Kafka（Topic: seckill-orders）**：acks=all, 分区按 userId 路由（同一用户的请求有序）
3. **库存预扣减**：Redis 扣减库存（原子操作 DECR），扣完直接返回失败
4. **订单生成 Consumer**：消费 seckill-orders → 校验 Redis 库存（二次确认）→ 写 MySQL 订单 → 提交 Offset
5. **兜底**：定时任务对比 Redis 和 MySQL，补偿不一致
6. **关键配置**：不丢五件套 + 手动提交 + 数据库唯一键（订单号）防重

**场景 2：Kafka 集群升级，如何保证零停机、零数据丢失？**

> 需求：Kafka 2.x → 3.x 大版本升级，服务不能中断。

**解答思路**：
1. **逐台滚动重启**：一次只停一台 Broker，等它完全恢复再加入集群后再停下一台
2. **升级前**：确保所有 Partition 的 ISR 完整（≥ min.isr），关闭 `unclean.leader.election`
3. **重启 Broker 前**：`controlled.shutdown` → Leader 自动迁移到其他 ISR 副本 → 等待迁移完成 → 再停机
4. **重启后**：等待 Broker 重新追上 ISR → 检查 URP 为 0 → 确认正常 → 下一台
5. **Producer/Consumer 端**：配置 `bootstrap.servers` 包含多台 Broker，Broker 短暂不可用时自动切换
6. **监控**：全程监控 Lag、ISR 变化、URP，一旦异常立即回滚

**场景 3：100 个 Topic，每个 10 个分区，总数 1000 个 Partition。发现某几个 Broker 负载远高于其他，怎么排查和解决？**

**解答思路**：
1. **排查 Leader 分布**：`kafka-topics.sh --describe` 统计每个 Broker 上的 Leader 数 → 可能 Leader 不均衡
2. **排查分区大小**：检查各分区磁盘占用 → 可能热点 Topic 集中在某几个 Broker
3. **解决 Leader 不均衡**：使用 `kafka-leader-election.sh` 触发 Preferred Leader Election 或使用自动均衡（`auto.leader.rebalance.enable=true`）
4. **解决磁盘不均衡**：`kafka-reassign-partitions.sh` 迁移分区（注意限流）
5. **长期方案**：新 Topic 创建时规划好分区分布；使用 Cruise Control 自动化负载均衡

## 一句话回答模板（面试急救）

| 问题 | 一句话 |
|------|--------|
| Kafka 是什么？ | 分布式提交日志系统，支持高吞吐、持久化、多消费者独立消费的发布订阅平台。 |
| 为什么快？ | 顺序写磁盘 + PageCache 缓存 + 零拷贝 sendfile 传输 + 批量压缩 + 分区并行。 |
| 怎么不丢？ | Producer acks=all + Broker 多副本 ISR ≥ 2 + Consumer 处理后再提交 Offset。 |
| 怎么不重？ | 幂等性防 Producer 重试重复 + 事务保证跨分区原子性 + 业务幂等兜底跨会话重复。 |
| 怎么有序？ | 相同 key 进同一分区，单分区内有序；开启幂等性可同时保证有序和高吞吐。 |
| Rebalance？ | Consumer Group 动态负载均衡机制，期间可能暂停消费；用 CooperativeSticky + 静态成员减少影响。 |
| Rebalance 为什么暂停消费？ | CG 状态机中只有 Stable 状态能消费，PreparingRebalance 和 CompletingRebalance 都是停止的。 |
| 积压怎么办？ | 先定位生产端还是消费端，再生产端限流/消费端扩容/加分区/优化处理逻辑。 |
| K8s 上能用吗？ | 可以，用 StatefulSet + 本地 SSD + 反亲和性 + Strimzi Operator 管理。 |
| Exactly Once？ | Kafka 内部 consume-process-produce 可以，到外部系统需业务幂等或两阶段提交配合。 |
| KRaft 比 ZK 好在哪里？ | 运维更简单（无外部依赖）、元数据一致性更强（Raft vs Zab）、Controller 切换更快（秒级）。 |
| Rack Awareness 是什么？ | 将同一 Partition 的副本分布到不同机架/AZ，防止单 AZ 故障导致数据不可用。 |
| GC 对 Kafka 有什么影响？ | Full GC 导致 Leader 分区暂停响应 → 延迟毛刺 → ISR 收缩 → Controller 可能被误判宕机。 |
| KStream 和 KTable 的区别？ | KStream 是事件流（每条独立），KTable 是更新流（按 key 覆盖），底层是 compacted Topic 的物化视图。 |
| 多机房怎么部署？ | 大多数选 Active-Standby + MirrorMaker 2.0；低延迟跨机房可选 Stretch Cluster；全球化选 Active-Active。 |
| 怎么监控 Kafka？ | JMX Exporter → Prometheus → Grafana；核心指标：URP、ActiveController、Lag、Handler 空闲率。 |
| Leader 分布不均怎么办？ | Preferred Leader Election（数据不搬迁，只切角色）或开启 auto.leader.rebalance。 |
| Follower Fetching 是什么？ | Consumer 配置 client.rack 后可从最近的 Follower 读取，省跨 AZ 流量、降延迟（有数据延迟代价）。 |
| Zombie Fencing 怎么实现？ | Transactional ID 绑定 Producer Epoch，新会话递增 Epoch，旧会话的提交/发送被 TC 拒绝。 |
| 哪些配置改了要重启？ | 线程数、磁盘路径、网络监听、JVM 参数必须重启；retention/min.isr/quota 类可动态生效。 |
| Kafka 脑裂怎么防？ | Controller Epoch（防旧 Controller）+ Leader Epoch（防旧 Leader）+ KRaft Raft Term（防元数据分裂）。 |
| JBOD 一块盘满了会怎样？ | 该盘上分区停止写入，但不影响其他盘的分区；Kafka 不会自动迁移 → 需日常监控磁盘使用率。 |

## 易错说法清单

| 错误 | 正确 |
|------|------|
| Kafka 保证全局有序 | Kafka 只保证**单分区内**有序 |
| 消费完消息就删除 | Kafka 按**保留策略**（时间/大小）删除 |
| acks=all 就绝对不丢 | 还要配合副本数、min.isr、unclean election |
| Exactly Once 适用于所有外部系统 | 事务主要保证 Kafka 内部 |
| Consumer 越多越好 | 有效并行度受 Partition 数限制 |
| 分区越多越好 | 过多增加文件句柄、元数据、Rebalance 成本 |
| 自动提交最安全 | 时机不可控，可能重复/漏消费 |
| Rebalance 只是重新分配没影响 | 导致消费停顿和重复消费风险 |
| 幂等性能解决所有重复 | 只解决单会话单分区重试重复 |
| 零拷贝 = 完全没有拷贝 | 只是消除了 CPU 拷贝，DMA 拷贝仍然存在 |
| Kafka 依赖 ZK 传输数据 | ZK 只存元数据，数据路径不经过 ZK |
| Kafka 只能在物理机上跑 | K8s + StatefulSet + 本地 SSD 也可以，但需注意存储和网络 |
| JVM 堆越大 Kafka 越快 | Kafka 需要给 OS 留够内存做 PageCache，堆 6GB + PageCache 20GB > 堆 26GB |
| Rebalance 时所有 Consumer 必定全停 | Cooperative 协议只暂停需迁移的分区，其他分区继续消费 |
| Kafka 副本越多越可靠 | 副本越多同步开销越大，3 副本 + rack aware 已足够大多数场景 |
| KStream 和 KTable 可以互换 | KStream 是事件流（每条都是 INSERT），KTable 是更新流（按 key upsert），语义完全不同 |

## 知识脑图

```text
Kafka
├── 定位：分布式事件流平台 = 分布式提交日志
├── 为什么需要：削峰填谷、系统解耦、异步通信、发布订阅
├── 核心架构：Topic(逻辑) → Partition(物理+并行) → Replica(高可用) → CG(负载均衡)
│   ├── Broker 网络模型：Reactor = Acceptor(accept) + Processor(I/O) + Handler(业务)
│   ├── 容灾与分布：Rack Awareness(副本跨AZ) | Preferred Leader(自动均衡) | JBOD(多盘并行)
│   └── 网络分区：脑裂防三种机制(Epoch/Term) | ZK vs KRaft 行为差异
├── Producer：拦截→序列化→分区→缓冲聚合→Sender发送 | 缓冲=空间(batch)+时间(linger)
│   └── 扩展点：ProducerInterceptor(消息增强/监控打点/全链路追踪)
├── Broker/存储：Controller协调 | ISR动态副本集 | HW消费者可见边界 | Segment+稀疏索引
│   ├── 存储增强：Tiered Storage(冷热分离) | Log Compaction(按key保留最新)
│   ├── 副本管理：Follower Fetching(读本地副本省跨AZ流量) | 动态配置热加载
│   └── 元数据：KRaft(Raft共识,去ZK) | Controller选举(多数派,防脑裂) | Topic删除异步流程
├── 性能六大支柱：分区并行、顺序写、PageCache、零拷贝(sendfile)、批量、压缩
│   ├── JVM GC 调优：G1GC + 留足PageCache内存 | GC停顿→ISR收缩→延迟毛刺
│   └── 端到端延迟：P50 5-10ms | P99 10-50ms | P999 50-200ms
├── Consumer：Pull模型 | Coordinator管理组 | 状态机(5状态) | Rebalance动态分配
│   ├── Fetch协议：min.bytes + max.wait.ms 解决空拉 | Follower Fetching 就近读取
│   ├── CooperativeSticky增量再平衡 | 静态成员减少Rebalance
│   └── 扩展点：ConsumerInterceptor(延迟监控/过滤/去重)
├── Offset：__consumer_offsets(compact) | 手动提交=至少一次 | 业务幂等=最终精准一次
├── 可靠性三层：acks+副本+ISR(不丢) | 幂等+事务+业务幂等(不重) | key路由+inflight控制(有序)
│   ├── 事务进阶：TC协调 | Control Batch Marker | Zombie Fencing(Producer Epoch防僵尸)
│   └── 三层 Fencing：PID+SeqNumber(会话内) | Producer Epoch(跨会话) | 业务幂等(跨系统)
├── 线上运维：积压五步排查 | Rebalance优化 | 分区数设计 | 容量规划 | 配额 | JMX监控 | 动态配置
│   └── Prometheus + Grafana：URP/ActiveController/Handler空闲率/Lag
├── 生态选型：Kafka(日志/流) ≈ RocketMQ(业务/延迟) ≠ RabbitMQ(路由/企业)
└── 周边工具：Schema Registry(格式) | MirrorMaker(跨集群/多活) | Connect(集成) | Streams(流处理)
    └── Streams 核心：KStream(事件流) | KTable(更新流) | GlobalKTable(全量广播) | Windowing(4种窗口)

---

> **版本说明**：本笔记以 Kafka 3.x/4.x 为参考版本。Kafka 4.0 起仅支持 KRaft 模式。Tiered Storage 在 3.6+ 为早期预览，4.0+ 逐步稳定。具体参数默认值请以当前集群版本官方文档为准。
> **参考资源**：Apache Kafka 官方文档（Producer/Consumer Configs、KRaft、Design Docs）、Kafka KIP（KIP-405 Tiered Storage、KIP-429 KRaft、KIP-54 Sticky Partition）、Confluent 官方博客
