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
| Kafka 2.8 (2021) | KRaft 早期预览 | 去 ZK 起点 |
| Kafka 3.5+ | ZK 标记为 deprecated | 迁移过渡期 |
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

**Q2：既然 Kafka 也是发消息，为什么不直接叫消息队列？**

消息队列暗示消息消费后消失。Kafka 可以做到：同一消息被多组独立消费、回溯三天前的消息重新处理、新消费者从最早位置追数据。Kafka 更像不可变的追加日志文件，消息队列只是它的一种使用方式。所以它叫"event streaming platform"而不是"message queue"。

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

## 四、关键设计

### Partition 的 trade-off

| 设计 | 优点 | 代价 |
|------|------|------|
| 多 Partition | 高吞吐、高并行 | 文件句柄多、元数据多、Rebalance 慢 |
| 少 Partition | 管理简单、单分区吞吐高 | 并行度不足、无法水平扩展 |

**估算公式**：`分区数 ≈ max(峰值吞吐/单分区吞吐, 期望并行度) × 预留系数(1.5~2)`

## 五、面试高频问题

**Q1：Topic 和 Partition 什么关系？**

Topic 是逻辑概念，不存数据；Partition 是物理存储单元。一个 Topic 可以拆成多个 Partition，分布在不同 Broker 上。数据实际存在 Partition 的 .log 文件中。

**Q2：Consumer 数量超过 Partition 数会怎样？**

多出来的 Consumer 空闲，不会消费任何消息。有效并行度 = min(Partition 数, Consumer 数)。所以加 Consumer 前先看 Partition 数够不够。

**Q3：为什么 Partition 数不能减少？**

数据迁移成本太高、Offset 序列会断裂、Key 映射会变化。Kafka 选择简单可靠的设计：只增不减。必须减的话只能重建 Topic 再迁移。

**Q4：Partition 越多越好吗？**

不是。每个 Partition 对应磁盘上至少 3 个文件（.log + .index + .timeindex），对应 Broker 内存中的索引结构，对应 Controller 的元数据条目。过多导致：文件句柄耗尽、元数据压力、Rebalance 变慢、Leader 选举成本增加。

## 六、总结（速记版）

- **Topic**：逻辑分类，不存数据
- **Partition**：物理存储，并行读写，有序追加日志
- **Replica**：Leader + Follower 副本 = 高可用
- **Consumer Group**：同组 1 Partition → 1 Consumer，组间广播
- **并行度上限** = min(Partition 数, Consumer 数)

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

### 超时参数关系

```
delivery.timeout.ms >= request.timeout.ms + linger.ms

send() → 写缓冲区 → Sender 发送 → 等 ack → 失败了重试 → 超时/异常
         |______________ delivery.timeout.ms _______________|
                         |__ request.timeout.ms __|
```

Producer 不是无限重试。`delivery.timeout.ms` 是总超时，超过后通过回调或 Future 返回异常。

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

### unclean.leader.election.enable

| 配置 | 含义 | 优点 | 风险 |
|------|------|------|------|
| `false`（推荐） | 只从 ISR 选 Leader | 数据可靠性高 | ISR 全挂时分区不可用 |
| `true` | 允许 OSR 当 Leader | 可用性高 | 可能丢失未同步数据 |

## 四、关键设计

### 为什么 Leader 只从 ISR 选举？

ISR 中的副本与 Leader 数据基本一致（差距在 `replica.lag.time.max.ms` 内），从 ISR 选举最小化数据丢失。硬要等所有副本全同步再确认，性能和可用性都受影响——ISR 是可靠性和性能之间的动态平衡。

### 为什么用稀疏索引而不是 B+ 树？

Kafka 是 append-only 日志，不需要随机更新索引。稀疏索引：维护成本极低、内存占用极小、二分 + 顺序扫描够快。B+ 树适合需要随机读写和范围查询的场景（如数据库），Kafka 不需要。

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

## 六、总结（速记版）

- **Controller**：集群协调者，管 Leader 选举和元数据同步
- **AR = ISR + OSR**：ISR 是"跟得上"的动态副本集
- **HW = min(所有 ISR 副本的 LEO)**：消费者只能读到 HW 之前
- **存储结构**：Partition → Segment(.log + .index + .timeindex)
- **稀疏索引**：少量内存换毫秒级查找
- **清理策略**：delete（按时间删）vs compact（按 key 保留最新）
- **KRaft 替代 ZK**：运维简化 + 一致性增强 + Controller 切换更快

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

## 六、总结（速记版）

- **Pull 模型**：消费者主动拉，可控制速度
- **Coordinator**：管理消费者组的 Broker，`hash(groupId) % __consumer_offsets 分区数` 定位
- **Rebalance 触发**：成员变化、分区变化、超时（心跳/poll）
- **Rebalance 危害**：消费暂停、积压、重复消费
- **优化**：静态成员 + CooperativeSticky + 合理 max.poll.records + 解耦 poll 和处理

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

- **存储**：`__consumer_offsets`（Kafka 内部 Topic）
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

### Rebalance 风暴

**场景**：大量 Consumer 同时重启（如滚动发布）→ 短时间内密集触发 Rebalance → 集群反复 STW，消费完全停滞。

**解决方案**：
1. **静态成员**（`group.instance.id`）：Consumer 重启后 Coordinator 在 `session.timeout.ms` 内给它"留位"，视为临时离开而非退出
2. **分批重启**：滚动发布时分批重启，每批之间有足够间隔
3. **延长 `session.timeout.ms`**：给重启留更多缓冲时间
4. **使用 CooperativeSticky**：即使 Rebalance 也只迁移必要分区

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

### Kafka Connect & Streams

| 组件 | 定位 | 典型用法 |
|------|------|---------|
| **Kafka Connect** | 数据集成框架 | MySQL Binlog → Source Connector → Kafka → Sink Connector → ES/HDFS/S3 |
| **Kafka Streams** | 嵌入式流处理库 | 轻量级实时计算，适合简单聚合/转换 |
| **Flink** | 独立流处理引擎 | 重量级有状态流批一体，适合复杂计算 |

### MirrorMaker 2.0（跨集群数据复制）

基于 Kafka Connect 框架，实现 Kafka 集群间的数据同步：

```
源集群 → MirrorSourceConnector → 目标集群（保留原始 Offset、分区信息）
源集群 ← MirrorCheckpointConnector ← 目标集群（同步 Consumer Group Offset）
源集群 ← MirrorHeartbeatConnector ← 目标集群（心跳监控延迟）
```

适用场景：多机房容灾、跨区域数据聚合、云迁移。核心优势：保留 Offset 映射关系，消费者可无缝切换到目标集群。

## 四、关键设计

### Kafka Connect 的设计哲学

"大多数数据集成需求都是通用的"——从 DB 读 Binlog、写入 HDFS、同步到 ES。与其让每个人写胶水代码，不如提供标准化的 Connector 框架。Source/Sink 模式让数据集成变成配置而不是开发。

### 安全机制

| 机制 | 作用 | 生产建议 |
|------|------|---------|
| SSL/TLS | 加密传输 | 开启 |
| SASL/SCRAM/Kerberos | 身份认证 | 根据公司安全策略 |
| ACL | 权限控制（Topic/Group/Cluster 级别） | 生产必须开启 |

## 五、面试高频问题

**Q1：Kafka vs RocketMQ vs RabbitMQ 怎么选？**

日志/埋点/大数据管道选 Kafka，订单/交易/延迟消息选 RocketMQ，传统企业应用/复杂路由选 RabbitMQ。如果只选一个，大厂一般 Kafka + RocketMQ 组合。

**Q2：Go 项目用哪个 Kafka 客户端？**

Sarama 最流行，功能全面；confluent-kafka-go 性能最高但需要 CGO；kafka-go 简单易用。推荐 Sarama（生产验证最充分）。

**Q3：Consumer 处理消息的最佳实践？**

关闭自动提交 → poll 拉取消息 → 业务处理成功 → `MarkMessage` 标记 → 定期手动提交。处理失败的不要提交 Offset，让下次重新消费。避免在 poll 线程里做耗时操作。

**Q4：多机房 Kafka 如何同步数据？**

MirrorMaker 2.0（基于 Kafka Connect）。Source Connector 复制消息到目标集群并保留 Offset 映射；Checkpoint Connector 同步 Consumer Group Offset；Heartbeat Connector 监控同步延迟。消费者可无缝切换到目标集群。

## 六、总结（速记版）

- **选型口诀**：日志 Kafka、交易 RocketMQ、企业 RabbitMQ
- **Go 客户端**：Sarama（流行）、confluent-kafka-go（高性能）、kafka-go（简洁）
- **Go 铁律**：手动提交 Offset、复用实例、避免并发提交、处理失败不提交
- **Connect**：数据集成标准化，少写胶水代码
- **Streams**：轻量流处理，比 Flink 简单但功能有限
- **MirrorMaker 2.0**：跨集群复制，保留 Offset 映射，多机房容灾

---

# 十一、附录：面试速查

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
```

## 高频面试题 TOP 20

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
| 9 | 幂等性怎么实现？ | PID + Partition + SeqNumber 三元组去重 |
| 10 | 事务解决什么？ | 跨分区/跨 Topic 写入 + Offset 提交的原子性 |
| 11 | Rebalance 触发条件？ | Consumer 上下线、心跳超时、poll 超时、分区变化、订阅变化 |
| 12 | 怎么减少 Rebalance？ | 静态成员、CooperativeSticky、合理 max.poll.records、解耦 poll 和处理 |
| 13 | Offset 存哪？ | `__consumer_offsets` 内部 Topic |
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
| 24 | 磁盘不均衡？ | kafka-reassign-partitions.sh 在线迁移，注意限流 |
| 25 | 监控哪些指标？ | Producer: QPS/失败率/延迟 | Broker: 磁盘/URP/ISR | Consumer: Lag/速率/Rebalance |

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

## 知识脑图

```text
Kafka
├── 定位：分布式事件流平台 = 分布式提交日志
├── 为什么需要：削峰填谷、系统解耦、异步通信、发布订阅
├── 核心架构：Topic(逻辑) → Partition(物理+并行) → Replica(高可用) → CG(负载均衡)
├── Producer：拦截→序列化→分区→缓冲聚合→Sender发送 | 缓冲=空间(batch)+时间(linger)
├── Broker/存储：Controller协调 | ISR动态副本集 | HW消费者可见边界 | Segment+稀疏索引
├── 性能六大支柱：分区并行、顺序写、PageCache、零拷贝(sendfile)、批量、压缩
├── Consumer：Pull模型 | Coordinator管理组 | Rebalance动态分配 | CooperativeSticky优化
├── Offset：__consumer_offsets | 手动提交=至少一次 | 业务幂等=最终精准一次
├── 可靠性三层：acks+副本+ISR(不丢) | 幂等+事务+业务幂等(不重) | key路由+inflight控制(有序)
├── 线上问题：积压五步排查 | Rebalance优化 | 分区数设计 | 容量规划
└── 生态选型：Kafka(日志/流) ≈ RocketMQ(业务/延迟) ≠ RabbitMQ(路由/企业)
```

---

> **版本说明**：本笔记以 Kafka 3.x/4.x 为参考版本。Kafka 4.0 起仅支持 KRaft 模式。具体参数默认值请以当前集群版本官方文档为准。
> **参考资源**：Apache Kafka 官方文档（Producer/Consumer Configs、KRaft、Design Docs）
