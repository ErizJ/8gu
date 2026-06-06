# MySQL 面试复习笔记

> 适用：后端开发面试准备 · 系统复习 · 面试速查
> 版本参考：MySQL 8.0（InnoDB 引擎）

---

<a id="toc"></a>

# 目录

- [一、MySQL 概述与架构](#ch1)
  - [DB/DBMS/SQL 三者关系 / SQL 五大分类](#sql-五大分类)
  - [一条 SELECT 的执行全流程](#一条-select-的执行全流程)
  - [SQL 逻辑执行顺序（高频）](#sql-逻辑执行顺序高频)
- [二、存储引擎](#ch2)
  - [InnoDB vs MyISAM](#innodb-vs-myisam)
  - [Buffer Pool（缓冲池）/ Change Buffer](#buffer-pool缓冲池)
  - [Buffer Pool 脏页淘汰 ★★★★](#buffer-pool-脏页淘汰-flush-策略)
  - [Doublewrite Buffer / AHI / 行格式](#doublewrite-buffer)
- [三、索引](#ch3)
  - [索引分类 / 聚簇索引 vs 二级索引](#索引分类)
  - [B+ 树 vs Hash/红黑树/B 树/跳表](#b-树为什么-mysql-选它)
  - [联合索引与最左前缀 / 覆盖索引 / ICP](#联合索引与最左前缀原则)
  - [Index Dive ★★★★ / Skip Scan ★★★](#index-dive索引跳水----优化器怎么估算行数-)
  - [降序索引 / 函数索引 / 生成列](#降序索引mysql-80)
  - [自增主键 vs UUID / 索引代价 / 低区分度字段](#为什么推荐自增主键)
- [四、事务与 MVCC](#ch4)
  - [ACID / 并发三大问题 / 四种隔离级别](#acid-四大特性)
  - [MVCC 实现机制（ReadView + undo 版本链）](#mvcc-实现机制)
  - [RC vs RR / 快照读 vs 当前读](#快照读-vs-当前读)
  - [Purge 机制 / 长事务危害](#为什么-undo-log-事务提交后不立即删除)
- [五、锁机制](#ch5)
  - [锁分类 / InnoDB 三种行锁](#锁分类)
  - [Next-Key Lock 防幻读 / 行锁在索引上 / 意向锁 / 死锁](#next-key-lock-如何防幻读)
  - [AUTO-INC 锁 ★★★★](#auto-inc-锁自增锁)
  - [SKIP LOCKED / NOWAIT（8.0+）★★★★](#skip-locked--nowaitmysql-80)
  - [不同条件下加了什么锁？（高频场景题）★★★★★](#五面试高频问题)
- [六、日志系统](#ch6)
  - [redo log / undo log / binlog 三大日志对比](#三大日志对比)
  - [两阶段提交（2PC）/ Group Commit ★★★★](#两阶段提交2pc)
  - [一条 UPDATE 的完整执行过程](#一条-update-的完整执行过程)
- [七、SQL 优化与 EXPLAIN](#ch7)
  - [EXPLAIN 关键字段 / type / Extra / key_len ★★★★★](#explain-关键字段)
  - [EXPLAIN ANALYZE（8.0.18+）★★★★★](#explain-analyzemysql-8018)
  - [Hash Join ★★★★ / Semi-join 优化](#hash-joinmysql-8018)
  - [慢查询排查六步法 / 深分页优化 / 大表操作](#慢查询排查六步法)
  - [SQL 改写实战模式 ★★★★★](#sql-改写实战模式-)
  - [Online DDL（大表变更）★★★★★](#online-ddl大表变更不停机-)
  - [Optimizer Trace / MRR / 索引合并](#optimizer-trace)
- [八、高可用架构](#ch8)
  - [主从复制 / GTID 复制 ★★★★★](#gtid-复制mysql-56-核心特性)
  - [多源复制 / 读写分离 / 延迟复制](#多源复制multi-source-replication)
  - [MGR / InnoDB Cluster ★★★★](#mgrmysql-group-replication与-innodb-cluster-)
  - [分库分表 + 数据迁移方案 ★★★★★](#分库分表扩容数据迁移方案高频场景题)
- [九、SQL 基础与表设计速查](#ch9)
  - [三大范式 / 数据类型 / NULL 处理](#数据库三大范式)
  - [DELETE/TRUNCATE/DROP / 重复插入避免 / JOIN / UNION](#delete--truncate--drop)
  - [子查询 / IN vs EXISTS / SQL 函数 / COUNT](#子查询分类)
  - [建表规范 / 主键 vs 唯一索引 / 外键 / 视图/存储过程/触发器](#建表规范)
- [十、MySQL 8 新特性与高级特性](#ch10)
  - [窗口函数 / TempTable / 隐藏索引 / CTE / 降序索引](#窗口函数-vs-group-by)
- [十一、MySQL 性能参数调优 ★★★★★](#ch11)
  - [关键参数分类 / 调优优先级](#关键参数分类)
  - [监控与诊断工具](#监控与诊断工具)
  - [典型配置（32G SSD）](#典型配置32g-内存-ssd-服务器)
- [十二、Golang + MySQL 实战指南](#ch12)
  - [驱动选型（database/sql / sqlx / GORM / sqlc）](#驱动选型)
  - [初始化连接池 / CRUD 基础操作](#场景一初始化连接池)
  - [事务处理 / SELECT FOR UPDATE](#场景三事务处理)
  - [批量插入 / 分页查询 / NULL 处理](#场景五批量插入高效)
  - [sqlx 结构化映射 / 读写分离](#场景九sqlx-简化映射)
  - [MySQL 分布式锁 / 连接池配置 / 防 SQL 注入](#场景十一mysql-分布式锁)
  - [面试高频问题（6 问）](#五面试高频问题-1)
- [十三、面试速查附录](#ch13)
  - [项目场景题（秒杀 / 一致性 / 大事务 / Online DDL / SKIP LOCKED 队列）](#项目场景题)
  - [面试前 10 分钟速过](#面试前-10-分钟速过)
  - [面试最终 1 分钟总结](#面试最终-1-分钟总结)
  - [易错说法清单](#易错说法清单)

---

<a id="ch1"></a>

# 一、MySQL 概述与架构

## 一、背景（为什么需要）

程序运行中数据存在内存里，断电即丢。数据库解决了三个根本问题：持久化保存、结构化组织、并发安全访问。

## 二、核心概念

### DB / DBMS / SQL 三者关系

- **DB（Database）**：存储数据的仓库（如 `order_db`）
- **DBMS（Database Management System）**：管理数据库的软件（如 MySQL、PostgreSQL）
- **SQL（Structured Query Language）**：操作数据库的语言

```
DBMS 管理多个 DB → DB 中有多个 Table → 用户通过 SQL 操作数据
```

### SQL 五大分类

| 分类 | 全称 | 作用 | 典型语句 |
|------|------|------|---------|
| DDL | Data Definition Language | 定义数据库对象 | CREATE / ALTER / DROP / TRUNCATE |
| DML | Data Manipulation Language | 操作数据 | INSERT / UPDATE / DELETE |
| DQL | Data Query Language | 查询数据 | SELECT |
| DCL | Data Control Language | 权限控制 | GRANT / REVOKE |
| TCL | Transaction Control Language | 事务控制 | COMMIT / ROLLBACK / SAVEPOINT |

### MySQL 逻辑架构

```
客户端 → 连接层（认证 + 线程分配）
       → Server 层（解析器 → 优化器 → 执行器）
       → 存储引擎层（InnoDB / MyISAM / Memory）
```

### 与 Redis 的配合

MySQL 存核心业务数据（强一致性），Redis 做缓存加速层（热点数据）。架构：MySQL + Redis + ES（搜索）。

## 三、底层原理

### 一条 SELECT 的执行全流程

```
1. 连接层：客户端 TCP 连接 → 身份认证 → 从线程池分配连接线程
2. 解析器：词法分析（识别关键字、表名、列名）+ 语法分析 → 生成 AST
3. 优化器：基于代价模型选择最优执行计划（用哪个索引、JOIN 顺序等）
4. 执行器：检查用户权限 → 调用存储引擎 API 获取数据
5. 存储引擎（InnoDB）：先查 Buffer Pool → 命中返回 → 未命中从磁盘读数据页
6. 结果集返回给客户端
```

> MySQL 8.0 已移除查询缓存（Query Cache），因为命中率低且维护开销大。

### SQL 逻辑执行顺序（高频）

```
书写顺序：SELECT → FROM → JOIN → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT

执行顺序：
  FROM → ON → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

- WHERE 在 GROUP BY 前 → WHERE 不能用聚合函数
- HAVING 在 GROUP BY 后 → HAVING 可以用聚合函数
- ORDER BY 在 SELECT 后 → ORDER BY 可以用 SELECT 中定义的别名

## 四、关键设计

### 为什么 MySQL 8.0 移除了查询缓存？

查询缓存在高并发下命中率极低（写操作会失效相关缓存），维护开销大（需要管理缓存失效），且加全局锁导致并发瓶颈。MySQL 8.0 彻底移除，将缓存职责交给应用层（Redis）。

## 五、面试高频问题

**Q1：一条 SELECT 语句的执行过程？**

连接认证 → 解析器生成 AST → 优化器选执行计划 → 执行器调存储引擎 API → InnoDB 查 Buffer Pool → 未命中从磁盘读 → 返回结果。

**Q2：SQL 执行顺序和书写顺序有什么不同？**

书写是 SELECT→FROM→WHERE→GROUP BY→HAVING→ORDER BY→LIMIT。执行是 FROM→WHERE→GROUP BY→HAVING→SELECT→ORDER BY→LIMIT。所以 WHERE 不能引用 SELECT 别名，ORDER BY 可以。

**Q3：MySQL 和 Redis 的分工？**

MySQL 负责核心业务数据的强一致性存储和事务保障；Redis 负责热点数据缓存、Session、计数器、分布式锁等高性能访问场景。两者配合，各有专攻。

## 六、总结（速记版）

- MySQL = 连接层 + Server 层（解析/优化/执行）+ 存储引擎层
- SQL 执行顺序：FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
- QPS 查询缓存已移除（MySQL 8.0），缓存交给 Redis
- DDL 操作结构，DML 操作数据，DQL 查询，TCL 管事务

---

<a id="ch2"></a>

# 二、存储引擎

## 一、背景（为什么需要）

不同业务场景对数据存储有不同需求：有些需要事务，有些追求读性能，有些只需临时存储。可插拔存储引擎让 MySQL 灵活适配。

## 二、核心概念

### InnoDB vs MyISAM

| 对比项 | InnoDB | MyISAM |
|--------|--------|--------|
| 事务 | 支持（ACID） | 不支持 |
| 锁粒度 | 行锁 | 表锁 |
| 外键 | 支持 | 不支持 |
| 崩溃恢复 | 支持（redo log） | 不支持 |
| COUNT(*) | 需扫描 | O(1)（单独存储行数） |
| 主键 | 必须有（聚簇索引） | 可以没有 |
| 数据文件 | `.ibd`（独立表空间） | `.MYD` + `.MYI` |
| 适用 | OLTP 高并发读写 | 大量读极少写（已基本淘汰） |

### InnoDB 数据文件

| 文件 | 作用 |
|------|------|
| `.ibd` | 每表独立表空间（数据 + 索引） |
| `ibdata1` | 系统表空间（数据字典、undo log、doublewrite buffer） |
| `ib_logfile0/1` | redo log 文件（循环写） |
| `mysql-bin.000001` | binlog（Server 层，主从复制） |
| `relay log` | 从库中继日志 |

### InnoDB 数据页结构

每个页 16KB，结构：
```
页头（38B）| 记录区（User Records）| 空闲区 | 页目录（Page Directory）| 页尾（8B）
```

页目录用于页内二分查找，加速定位到具体记录组。

## 三、底层原理

### Buffer Pool（缓冲池）

InnoDB 最核心的内存结构。缓存数据页和索引页，减少磁盘 I/O。

```
Buffer Pool 内部结构：
  ├── LRU List（按访问热度管理缓存页）
  │     ├── Young 区（热数据，占 5/8）
  │     └── Old 区（冷数据，占 3/8）
  ├── Free List（空闲页）
  └── Flush List（脏页链表，按最早修改时间排序）

读取流程：
  查 Buffer Pool → 命中 → 直接返回
              → 未命中 → 从磁盘加载到 Free List → 返回

写入流程：
  修改 Buffer Pool 中的页 → 标记为脏页 → 加入 Flush List
  → 写 redo log → 异步刷盘（checkpoint 时批量写）
```

**为什么用 LRU 变种而不是标准 LRU？** 全表扫描可能一次性淘汰所有热数据。MySQL 增加 Old 区：新读入的页先放 Old 区头部，只有被再次访问才移到 Young 区。全表扫描的页在 Old 区被快速淘汰，不影响 Young 区的热数据。

**`innodb_old_blocks_time`** 控制页在 Old 区的最短停留时间（ms，默认 1000）。页进入 Old 区后，必须等待该时间后被再次访问才允许移到 Young 区。防止全表扫描时短时间内大量页涌入 Young 区。

> 全表扫描场景优化：增大 `innodb_old_blocks_time`（如 2000）→ 全表扫描页在 Old 区停留更久才可能晋升 → 更有效隔离扫描污染。

### Change Buffer

对于非唯一二级索引，如果目标索引页不在 Buffer Pool 中，InnoDB 先将修改缓存在 Change Buffer 中（内存），等该页被读入时再合并（merge）。避免随机读 I/O。

> Change Buffer 适用于写多读少的场景（如日志表、流水表）。对于读多写少的场景（如用户表），索引页大概率已在 Buffer Pool，Change Buffer 收益不大。

### Buffer Pool 脏页淘汰（Flush 策略）★★★★

Buffer Pool 空间有限，需要淘汰旧页腾出空间给新页。InnoDB 的淘汰策略是 **LRU 变种 + 脏页刷盘策略**：

**1. LRU 链表中的页面状态**：
- **干净页（Clean Page）**：内存和数据文件一致 → 可直接淘汰
- **脏页（Dirty Page）**：内存被修改但未刷盘 → 淘汰前必须先刷盘

**2. 脏页刷盘时机**（checkpoint 机制）：
```
触发条件（满足任一即刷脏页）：
1. Flush List 脏页数量 > innodb_max_dirty_pages_pct（默认 90%）
   → 脏页占比 90% 时强制刷盘
2. redo log 空间不足 → 对应脏页必须先刷盘才能覆盖 redo log
3. Buffer Pool 空闲页不够 → 淘汰 LRU 尾巴时遇到脏页
4. 定期刷盘 → 后台 Master Thread 每秒/每 10 秒检查
```

**3. LRU 淘汰具体流程**：
```
需要淘汰一页时：
  a. 从 LRU 尾部取一页
  b. 是干净页 → 直接淘汰（很快）
  c. 是脏页 → 先 fsync 刷到磁盘 → 变成干净页 → 淘汰（慢）
  d. 若尾部脏页过多 → 触发"邻居页刷新"（Flush Neighbor Page）
     → 清理连续脏页区域，提升刷盘效率但增加 I/O 峰值
```

**4. 关键参数**：
```bash
innodb_max_dirty_pages_pct = 90    # 脏页最大占比
innodb_flush_neighbors = 1         # SSD 设为 0（关闭相邻页刷新）
innodb_lru_scan_depth = 1024       # LRU 尾部扫描深度
innodb_io_capacity = 200           # 后台 I/O 能力（SSD 设为 2000+）
```

**面试要点**：`innodb_io_capacity` 是关键性能参数——设置不当会导致刷盘太慢（脏页堆积）或太快（I/O 过载）。SSD 建议 2000-5000。

### Doublewrite Buffer

解决**部分页写入（partial write）**问题：InnoDB 页 16KB，OS 页 4KB，写数据页时可能只写了部分就宕机，页损坏后 redo log 也无法修复。

```
流程：
  1. 脏页先写入内存中的 doublewrite buffer
  2. 顺序写到磁盘 doublewrite 区域（系统表空间，顺序写快）
  3. 再随机写到真正数据文件位置
  4. 宕机恢复时：数据文件页损坏 → 从 doublewrite 区域恢复完整页 → 应用 redo log
```

性能开销约 5-10%，默认开启。

### Adaptive Hash Index（AHI）

InnoDB 自动将高频访问的 B+ 树路径缓存为哈希索引，加速等值查询。无需人工干预，InnoDB 根据访问模式自动构建。

### InnoDB 行格式（Row Format）

| 格式 | 特点 | 适用场景 |
|------|------|---------|
| **COMPACT** | 基础格式，变长字段超过 768B 时超长部分存溢出页 | MySQL 5.0 引入 |
| **DYNAMIC**（默认） | 变长字段超过 40B 即完全存溢出页，只保留 20B 指针 | MySQL 5.7+ 默认，适合大字段 |
| **REDUNDANT** | 最旧格式，兼容性 | 已淘汰 |
| **COMPRESSED** | 在 DYNAMIC 基础上支持页压缩 | 存储敏感场景 |

> 溢出页（Overflow Page）：行数据超过页大小（约 8KB 限制）时，变长字段（VARCHAR/TEXT/BLOB）的前缀存在主数据页，完整内容存在溢出页。DYNAMIC 格式更激进地把大字段外存，主数据页保留更多行 → Buffer Pool 效率更高。

## 四、关键设计

### 为什么 InnoDB 成为默认引擎？

1. 事务 + 行锁 → 高并发写入
2. redo log + WAL → 崩溃恢复
3. MVCC → 读写互不阻塞
4. Buffer Pool → 高效缓存，减少磁盘 I/O

### 为什么互联网项目不使用外键？

1. 外键增加写入检查成本，降低写入性能
2. 增加锁竞争范围
3. 表结构耦合强，不利于分库分表和 schema 变更
4. 高并发场景在应用层保证一致性更灵活

## 五、面试高频问题

**Q1：InnoDB 和 MyISAM 的核心区别？**

InnoDB 支持事务和行锁，适合高并发写入；MyISAM 不支持事务，只有表锁，并发写性能差。InnoDB 通过 redo log 保证崩溃恢复。生产环境几乎都用 InnoDB。

**Q2：Buffer Pool 的 LRU 为什么有 Young/Old 区？**

防止全表扫描一次性淘汰所有热数据。新页先进入 Old 区，只有再次被访问才进入 Young 区。全表扫描的页在 Old 区被快速淘汰。

**Q3：Doublewrite Buffer 解决什么问题？**

部分页写入（partial write）。16KB 数据页写一半宕机 → 页损坏 → redo log 无法修复（需要完整页作为基础）。doublewrite 先顺序写备份再随机写真实位置，宕机后可从备份恢复。

## 六、总结（速记版）

- **InnoDB 核心四件套**：Buffer Pool + Change Buffer + Doublewrite + AHI
- **Buffer Pool**：LRU 变种（Young/Old 区），防全表扫描污染
- **Change Buffer**：非唯一二级索引的写缓存，减少随机读
- **Doublewrite**：防 partial write，性能开销 ~5-10%

---

<a id="ch3"></a>

# 三、索引

## 一、背景（为什么需要）

全表扫描 O(N) 在千万级数据量下不可接受。索引将查询从 O(N) 降到 O(log N)，是数据库性能的第一要素。

## 二、核心概念

### 索引分类

| 维度 | 分类 |
|------|------|
| 数据结构 | B+ 树（默认）、Hash（Memory 引擎）、全文索引（FULLTEXT） |
| 物理存储 | **聚簇索引**（数据即索引）、**二级索引**（存主键回表） |
| 字段数量 | 单列索引、联合索引（复合索引） |
| 约束 | 主键索引、唯一索引、普通索引、前缀索引 |
| MySQL 8 新特性 | 降序索引、隐藏索引、函数索引 |

### 聚簇索引 vs 二级索引

**聚簇索引**：叶子节点存**完整行数据**。主键就是聚簇索引。找到索引即找到数据。每表只有一个。

**二级索引**：叶子节点存**索引列值 + 主键值**。查到后需用主键回聚簇索引查完整行（**回表**）。每表可有多个。

**如果没有显式主键？** InnoDB 按以下顺序选择聚簇索引：
1. 第一个非空唯一索引
2. 都没有 → 自动生成 6 字节隐藏 `row_id`

## 三、底层原理

### B+ 树：为什么 MySQL 选它？

**B+ 树是一种多路平衡搜索树**：

```
内节点：只存 key（索引列值），不存数据 → 单个节点可存大量 key → 树极矮
叶子节点：存完整数据（聚簇）或主键值（二级），通过双向链表相连
```

**千万级数据树高计算**：每页 16KB，假设 key 8B + 指针 6B = 14B/条目，每页约 1170 个 key。叶子节点每条约 1KB，每页约 16 条。3 层 B+ 树可存 `1170×1170×16 ≈ 2190 万` 条记录。

**B+ 树 vs 其他数据结构**：

| 对比 | 关键差异 | 为什么不适合 MySQL |
|------|---------|-------------------|
| vs Hash | Hash 等值 O(1) 但**不支持范围查询**和排序 | 数据库大量范围查询 |
| vs 红黑树 | 二叉树高度 O(log₂N)，百万数据 ~20 层 | 每层一次 I/O，20 次 I/O 不可接受 |
| vs B 树 | B 树内节点存数据 → 容量小 → 树更高；范围查询需中序遍历 | B+ 树叶子链表天然支持范围查询 |
| vs 跳表 | 跳表节点随机分布，磁盘局部性差 | B+ 树节点 = 磁盘页，充分利用预读 |

### 联合索引与最左前缀原则

联合索引 `(a, b, c)` 的 B+ 树按 `(a, b, c)` 字典序排列：

```
先按 a 排序 → a 相同按 b 排序 → b 相同按 c 排序
```

这就是**最左前缀原则**的根源：
- `WHERE a = 1` ✅ 可用（a 有序）
- `WHERE a = 1 AND b = 2` ✅ 可用（a 定位后 b 有序）
- `WHERE b = 2` ❌ 不可用（没有 a，b 全局无序）
- `WHERE a = 1 AND c = 3` ⚠️ 只能用到 a（跳过 b，c 无序）

**范围查询注意**：`WHERE a = 1 AND b > 2 AND c = 3`，只能用到 `(a, b)`，c 失效。

### 覆盖索引

SELECT 的列全部在索引中 → 从二级索引叶子节点就能获取所有数据 → **不需要回表**。

```sql
-- 有联合索引 (name, age)
SELECT name, age FROM users WHERE name = 'Alice';  -- 覆盖索引，不回表
SELECT name, age, email FROM users WHERE name = 'Alice';  -- 需要回表（email 不在索引中）
```

EXPLAIN Extra 显示 `Using index` = 覆盖索引。

### 索引下推（ICP，MySQL 5.6+）

使用联合索引时，将部分 WHERE 过滤下推到存储引擎层（索引层）执行：

```sql
-- 联合索引 (age, name)
SELECT * FROM users WHERE age > 20 AND name LIKE '%li%';
```

无 ICP：引擎按 age > 20 找到所有行回表 → Server 层过滤 name
有 ICP：引擎在索引层就判断 name LIKE '%li%' → 只对通过的行回表

Extra 显示 `Using index condition` = ICP。

### Index Dive（索引跳水）—— 优化器怎么估算行数 ★★★★

**面试题**："`EXPLAIN` 中的 rows 是怎么算出来的？5000 万行数据，优化器怎么决定走索引还是全表扫描？"

MySQL 优化器基于**统计信息 + Index Dive** 估算扫描行数：

**1. 统计信息（基数 Cardinality）**：
- InnoDB 为每个索引维护基数（不同值的数量）
- `SHOW INDEX FROM t` → Cardinality 列
- 通过**采样**（随机选 N 个页，统计不同值数）估算
- `innodb_stats_persistent=ON`（持久化统计信息，默认），`innodb_stats_sample_pages=20`

**2. Index Dive（索引跳水）**：
- 对于等值查询：优化器"跳入"索引，沿 B+ 树查找第一个匹配记录和最后一个匹配记录 → 两者之间的记录数 = 估算行数
- 对于范围查询：每个等值条件跳一次（"dive"），个数累加

**3. 什么时候跳过 Index Dive？**
- `eq_range_index_dive_limit`（默认 200）：当 IN 列表中的值 > 200 个时，不再逐个 dive，直接使用统计信息估算（`rows = 总行数 / Cardinality`）
- **陷阱**：IN (2001个值) → 跳过 dive → 可能严重低估或高估 → 优化器选错索引！

**实例**：
```sql
-- 表 5000 万行，索引 idx_status，status 只有 3 个值
WHERE status = 1
-- Cardinality = 3, 总行 5000 万
-- Index Dive: 精确算出 status=1 的行数（假设 1000 万）
-- rows ≈ 1000 万 → 优化器发现占比 20%，认为回表成本太高 → 全表扫描

-- 如果用统计估算（跳过 dive）：5000万/3 ≈ 1667 万 → 更不准
```

**面试要点**：统计信息的准确度决定优化器选择。`ANALYZE TABLE` 更新统计信息。Index Dive 在少量等值条件下很准，但大量 IN 条件会跳过 dive 可能导致误判。

### 索引跳过扫描（Index Skip Scan，MySQL 8.0.13+）★★★

联合索引 `(a, b)`，当查询只用到 b 时，早期 MySQL 只能全表扫描。MySQL 8.0.13+ 的 Skip Scan 会**枚举 a 的每个不同值**，分别用索引搜索 b：

```sql
-- 联合索引 (gender, age)
SELECT * FROM users WHERE age = 25;
-- gender 只有 2 个值 → 优化器可能枚举 gender='M' 和 'F'
-- 分别查 WHERE gender='M' AND age=25 和 WHERE gender='F' AND age=25
-- 从"不能用索引"变为"用两次索引"
```

前提：联合索引第一列区分度低（值种类少）。如果 gender 有 10000 种，枚举代价太大 → 不会用 Skip Scan。
Extra 显示 `Using index for skip scan`。

### 前缀索引

对长字符串（URL、邮箱）只取前 N 个字符建索引：

```sql
CREATE INDEX idx_email ON users(email(10));
```

好处：节省空间。缺点：无法用覆盖索引（索引中不是完整值，仍需回表验证）。前缀长度选择：`SELECT COUNT(DISTINCT LEFT(email, N)) / COUNT(*)` 评估区分度。

## 四、关键设计

### 降序索引（MySQL 8.0+）

早期 MySQL 只支持升序索引，`ORDER BY a ASC, b DESC` 即使有索引 `(a, b)` 也会 filesort。8.0 真正支持降序索引：

```sql
-- 8.0+ 降序索引 → ORDER BY a ASC, b DESC 可直接利用索引有序性
CREATE INDEX idx_a_b ON t(a ASC, b DESC);
EXPLAIN SELECT * FROM t ORDER BY a ASC, b DESC;  -- 无 filesort！
```

**适用**：多列排序中排序方向不一致的场景（如排行榜按分数 DESC + 时间 ASC）。

### 函数索引（MySQL 8.0.13+）

对列的**表达式结果**建索引，解决"WHERE 中对列做函数导致索引失效"的经典问题：

```sql
-- 以前：WHERE LOWER(email) = 'a@b.com' → 索引失效
-- 现在：建函数索引
CREATE INDEX idx_email_lower ON users((LOWER(email)));

-- 查询可以走索引了
SELECT * FROM users WHERE LOWER(email) = 'a@b.com';
```

> 函数索引本质是基于**生成列（Generated Column）**——MySQL 自动创建隐藏的虚拟列来存储表达式结果。

### 生成列（Generated Column）

不存业务逻辑，由其他列计算得出的列：

```sql
-- VIRTUAL（默认，不占存储，计算开销）：常用于函数索引
ALTER TABLE orders ADD COLUMN total_price DECIMAL(10,2)
  GENERATED ALWAYS AS (unit_price * quantity) VIRTUAL;

-- STORED（占存储，无计算开销）：适合高频查询的预计算字段
ALTER TABLE orders ADD COLUMN total_price DECIMAL(10,2)
  GENERATED ALWAYS AS (unit_price * quantity) STORED;
```

| 类型 | 存储 | 索引 | 适用 |
|------|------|------|------|
| VIRTUAL | 不占磁盘 | 可以建索引（= 函数索引） | 计算简单、低频查询 |
| STORED | 占磁盘 | 可以建索引 | 计算复杂、高频查询 |

> 面试要点：生成列最常见的落地场景就是**函数索引**。JSON 类型配合生成列 + 函数索引 = 对 JSON 字段中某个 key 建索引。

### 为什么推荐自增主键？

1. **插入性能**：自增 ID 总是追加到 B+ 树末尾 → 减少页分裂 → 顺序写
2. **存储空间**：BIGINT 8B vs UUID 36B，主键被所有二级索引引用，空间差异被放大
3. **分布式场景**：用雪花算法（Snowflake）生成趋势递增 ID → 全局唯一 + 顺序写入

### 索引的代价

- 写入性能下降：每次 INSERT/UPDATE/DELETE 需维护所有索引
- 存储空间：每个索引 = 一棵 B+ 树
- 优化器负担：索引过多 → 优化器评估时间增加 → 可能选错索引

**建索引原则**：给频繁出现在 WHERE、JOIN ON、ORDER BY、GROUP BY 中的列建，区分度高者优先。一般一张表不超过 5-6 个索引。

### 性别/状态字段适合建索引吗？

通常不适合单独建索引。区分度极低（如只有 0/1），索引选择性 ≈ 50%，优化器大概率选全表扫描（回表代价 > 全表扫描）。
例外：数据严重倾斜（如 99.9% 是 0，0.1% 是 1，查 1 时索引有效），或放入联合索引开头。

## 五、面试高频问题

**Q1：B+ 树为什么比 B 树更适合数据库索引？**

B+ 树内节点只存 key（不存数据），每页可存更多 key → 树更矮 → I/O 更少。叶子节点双向链表 → 范围查询只需顺序遍历。所有查询都到叶子节点 → 查询时间稳定。

**Q2：联合索引 `(a,b,c)`，查询 `WHERE a=1 AND c=3` 能用到索引吗？**

只能用到 a。b 被跳过，c 在 B+ 树中相对于 a 的范围内不是有序的。`c=3` 只能在 a 的结果集上过滤。

**Q3：索引失效的常见场景？**

对索引列做函数/计算、隐式类型转换、LIKE 前导通配符 (`'%abc'`)、违反最左前缀、OR 两侧不全有索引、`!=`/`NOT IN`、数据量太少优化器放弃索引。

**Q4：回表是什么？如何避免？**

通过二级索引找到主键，再用主键去聚簇索引查完整行的过程。避免方法：建立**覆盖索引**——将 SELECT 需要的列都加入二级索引。

**Q5：为什么 MySQL 不用跳表？**

跳表节点内存随机分布，磁盘局部性差（相邻节点可能在不同磁盘页）；树高不固定（随机层高，最坏情况性能不稳定）。B+ 树节点 = 磁盘页，相邻数据物理连续，磁盘预读效果好。

**Q6：降序索引和函数索引分别解决什么问题？**

降序索引（8.0+）：解决多列排序方向不一致时的 filesort（如 `ORDER BY a ASC, b DESC`）。函数索引（8.0.13+）：解决对列做函数/表达式导致索引失效的问题（如 `WHERE LOWER(email) = 'a@b.com'`），本质是基于隐藏生成列。

## 六、总结（速记版）

- **B+ 树**：内节点只存 key、叶子存数据 + 双向链表、千万数据 3-4 层
- **聚簇索引**叶子存数据，**二级索引**叶子存主键 → 回表
- **最左前缀**：联合索引从最左列开始才能利用有序性
- **覆盖索引**：SELECT 的列全在索引中 → 不回表
- **降序索引**（8.0+）：真正支持 DESC 排序，避免多列混合排序时的 filesort
- **函数索引**（8.0.13+）：对表达式建索引，基于隐藏生成列
- **索引失效本质**：无法利用 B+ 树的有序性定位数据

---

<a id="ch4"></a>

# 四、事务与 MVCC

## 一、背景（为什么需要）

并发场景下，多个事务同时读写同一数据。需要一套机制保证：数据不错（一致性）、不互相干扰（隔离性）、不丢失（持久性）、不半途而废（原子性）。

## 二、核心概念

### ACID 四大特性

| 特性 | 含义 | 实现机制 |
|------|------|---------|
| **原子性 (Atomicity)** | 事务全部成功或全部回滚 | undo log |
| **一致性 (Consistency)** | 事务前后数据满足约束 | 其他三者共同保证 |
| **隔离性 (Isolation)** | 并发事务互不干扰 | 锁 + MVCC |
| **持久性 (Durability)** | 提交后永久保存 | redo log |

### 并发事务三大问题

| 问题 | 说明 | 举例 |
|------|------|------|
| **脏读** | 读到未提交数据 | A 改余额未提交，B 读到改后值 → A 回滚 → B 读到错误 |
| **不可重复读** | 同一事务两次读结果不同（行被修改） | A 读余额 100 → B 改成 200 提交 → A 再读为 200 |
| **幻读** | 同一事务两次范围查询行数不同（行被插入/删除） | A 查余额 > 100 返回 5 行 → B 插入 1 行 → A 再查返回 6 行 |

### 四种隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 | 实现 |
|------|------|-----------|------|------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 | 几乎无保护 |
| READ COMMITTED | 不会 | 可能 | 可能 | 每次 SELECT 新 ReadView |
| **REPEATABLE READ（默认）** | 不会 | 不会 | InnoDB 通过 Next-Key Lock 解决 | 首次 SELECT 生成 ReadView |
| SERIALIZABLE | 不会 | 不会 | 不会 | 所有 SELECT 自动加 S 锁 |

**为什么默认是 RR 而不是 RC？** 

两个原因：

1. **历史兼容原因（主因）**：早期 binlog 默认 STATEMENT 格式。RC 下 `UPDATE ... WHERE key>5` 在主库和从库重放时扫描到的行可能不同（从库的并发写入导致数据不同步），导致语句级复制不一致。RR 的快照读保证同一事务看到的视图一致 → STATEMENT binlog 安全。虽然现在推荐 ROW 格式（记录行变化而非 SQL 语句），但默认值保留了历史兼容性。

2. **传统“RR 解决幻读”认知**：Oracle 默认 RC，MySQL 默认 RR。MySQL 的 InnoDB 通过 Next-Key Lock 在 RR 下真正解决了幻读，提供了比标准 SQL 更强的 RR 隔离体验。

### RC 下也会产生幻读 —— 符合 SQL 标准，但容易踩坑

**RC 本来就允许幻读**（SQL 标准中 RC 不保证无幻读）。RC 每次 SELECT 生成新 ReadView → 能看到其他事务已提交的插入 → 符合 RC 定义。

但很多从 RR 迁移到 RC 的开发者会"意外"踩坑：RC 只使用 Record Lock（不用 Gap Lock），**当前读如 `SELECT FOR UPDATE` 在 RC 下也无法阻止幻读**——开发者往往误以为加锁就能防止插入：

```
T1: SELECT * FROM t WHERE id > 5 FOR UPDATE  -- 锁住 id>5 的现有行（但不能阻止插入 id=6）
T2: INSERT INTO t(id) VALUES(6); -- 成功！Gap 无锁 → 在 T1 的"范围"内插入
T1: SELECT * FROM t WHERE id > 5 FOR UPDATE  -- 多了一行 id=6 → 幻读
```

**结论**：RR + Next-Key Lock 提供了双重保障——MVCC 防快照读幻读，Gap Lock 防当前读幻读。如果业务切换到 RC，需要接受幻读的存在，或在应用层处理。

## 三、底层原理

### MVCC 实现机制

MVCC（多版本并发控制）让读不加锁，通过读取历史版本避免读写冲突。

**三大组件**：

```
1. 隐藏字段：
   trx_id：最近修改该行的事务 ID（6 字节）
   roll_pointer：指向 undo log 的指针（7 字节）

2. undo log 版本链：
   每次 UPDATE/DELETE 生成一条 undo log（存旧值）
   通过 roll_pointer 串成版本链 → 保存该行的所有历史版本

3. ReadView（读视图）：
   事务执行快照读时生成，包含：
   - m_ids：当前活跃（未提交）事务 ID 列表
   - min_trx_id：m_ids 中的最小值
   - max_trx_id：下一个事务 ID（= 已提交最大 + 1）
```

**版本可见性判断**：
```
trx_id < min_trx_id        → 已提交，可见 ✓
trx_id >= max_trx_id       → 未开始（在 ReadView 之后），不可见 ✗
trx_id 在 m_ids 中        → 活跃未提交，不可见 ✗
trx_id 不在 m_ids 中       → 已提交，可见 ✓
```

不可见时，沿着 `roll_pointer` 在 undo log 版本链中向前找，直到找到可见版本。

**RC vs RR 的 ReadView 区别**：
- RC：每次 SELECT 都生成**新 ReadView** → 能看到其他事务已提交的修改 → 不可重复读
- RR：只在**第一次 SELECT** 时生成 ReadView → 整个事务复用同一个 → 可重复读

### 快照读 vs 当前读

| 类型 | 语句 | 使用机制 |
|------|------|---------|
| **快照读** | 普通 `SELECT` | MVCC（ReadView），读历史版本，不加锁 |
| **当前读** | `SELECT FOR UPDATE` / `UPDATE` / `DELETE` | 读最新已提交版本 + 加锁 |

> RR 级别下：快照读不会出现幻读（MVCC 保证）；当前读可能出现幻读，但被 Next-Key Lock 阻止。

## 四、关键设计

### 为什么 InnoDB COUNT(*) 比 MyISAM 慢？

InnoDB 支持 MVCC 和事务，不同事务看到的行数可能不同，无法像 MyISAM 那样维护一个全局行数。COUNT(*) 需要扫描索引或表来统计。优化：用更小的二级索引扫描，或业务侧维护计数表。

### 为什么 undo log 事务提交后不立即删除？

MVCC 的其他事务可能还需要读取旧版本（undo log 版本链中的历史数据）。由 **purge 线程** 异步判断并清理不再被任何 ReadView 需要的旧版本。

**Purge 机制详解**：

```
undo log 分类：
  INSERT undo：INSERT 操作产生 → 事务提交后立即可删（外部事务不可见新插入的行）
  UPDATE undo：UPDATE/DELETE 操作产生 → 需等待所有引用该版本的 ReadView 消失

Purge 流程：
  1. 事务提交后，undo log 加入 history list
  2. Purge 线程（默认 4 个，innodb_purge_threads）从 history list 取 undo log
  3. 判断该版本对所有活跃 ReadView 是否不可见 → 不可见则物理删除
  4. 同时删除对应的索引记录（DELETE 标记的记录）
```

**长事务对 purge 的影响**：
```
长事务（未提交）→ 其 ReadView 中的 min_trx_id 是"低位水线"
→ purge 线程只能清理 trx_id < min_trx_id 的旧版本
→ 长事务期间产生的 UPDATE undo 全部无法清理
→ undo 表空间持续膨胀 → ibdata1 或 undo tablespace 暴涨
→ 版本链变长 → 快照读每次需要遍历更长版本链 → 性能下降
```

> 监控：`SHOW ENGINE INNODB STATUS` 中的 `History list length`（undo 历史链表长度）。正常 < 1000，持续增长说明有长事务或 purge 跟不上写入速度。

**长事务 = 性能杀手**：锁长时间持有 + undo 膨胀 + 主从延迟 + 回滚代价高。务必设置 `innodb_lock_wait_timeout` 和 `max_execution_time` 兜底。

## 五、面试高频问题

**Q1：MVCC 的实现原理？**

三个组件：trx_id + roll_pointer（隐藏字段）+ undo log 版本链 + ReadView。每次快照读时，通过 ReadView 判断版本可见性，不可见则沿版本链向前找历史版本。读不加锁，读写不阻塞。

**Q2：RC 和 RR 的实现区别？**

RC 每次 SELECT 生成新 ReadView → 能看到其他事务提交的修改 → 不可重复读。RR 只在第一次 SELECT 生成 ReadView → 整个事务读同一快照 → 可重复读。

**Q3：RR 级别下怎么防幻读？**

快照读：MVCC 保证（ReadView 不变，新插入行的 trx_id 不可见）。当前读：Next-Key Lock（行锁 + 间隙锁）阻止在查询范围内插入新行。

**Q4：事务隔离级别越高越好吗？**

不是。隔离级别越高 → 并发性能越低。SERIALIZABLE 将所有 SELECT 变当前读加 S 锁，几乎串行执行。RR 已满足绝大多数场景，不需要 SERIALIZABLE。

**Q5：举一个不适合脏读的具体场景？**

银行转账：事务 A 从 X 转 100 到 Y，先扣 X（未提交）→ 事务 B 查 X 余额读到扣后值 → 判断余额不足拒绝某操作 → 事务 A 回滚 → B 基于错误数据做了错误决策。库存扣减同理：A 扣库存未提交 → B 读到 0 拒绝下单 → A 回滚 → 误判。

**Q6：滥用事务或事务过大有什么弊端？**

1. 锁持有时间长 → 阻塞其他事务，降低并发，可能引发死锁
2. undo log 膨胀 → 大量旧版本 → MVCC 版本链查找变慢
3. 主从延迟增大 → 大事务 binlog 体积大，从库重放耗时长
4. 回滚代价高 → 大事务回滚需撤销大量操作
5. 连接占用长 → 连接池资源紧张

最佳实践：事务尽量短小，只包含必要的数据库操作；避免在事务中做网络请求或耗时计算；大批量操作拆分成小批次分批提交。

## 六、总结（速记版）

- **ACID**：undo log 保原子性，锁+MVCC 保隔离性，redo log 保持久性
- **MVCC 三件套**：trx_id + roll_pointer + ReadView → 读不加锁
- **RC vs RR**：ReadView 生成时机不同（每次 vs 首次）
- **RR 防幻读**：快照读靠 MVCC，当前读靠 Next-Key Lock
- **大事务** = 性能杀手：锁长 + undo 膨胀 + 主从延迟 + 回滚慢

---

<a id="ch5"></a>

# 五、锁机制

## 一、背景（为什么需要）

并发写同一行数据时，MVCC 无法解决（MVCC 只处理读写并发）。锁保证写写并发的正确性——同一时刻只有一个人能改。

## 二、核心概念

### 锁分类

```
按粒度：表锁（含 MDL 元数据锁）/ 行锁
按读写：共享锁（S 锁，读）/ 排他锁（X 锁，写）
按实现：Record Lock / Gap Lock / Next-Key Lock / 意向锁（IS/IX）
按态度：乐观锁（version CAS）/ 悲观锁（SELECT FOR UPDATE）
```

**表锁与行锁的作用**：
- **表锁**：保护整张表。DDL 操作（ALTER TABLE）会加 MDL 锁防并发读写；MyISAM 所有操作都加表锁。开销小但并发差
- **行锁**（InnoDB）：只锁定操作的行。并发高，但行锁加在索引上——如果 WHERE 条件没走索引，锁升级为全表行锁（退化为表锁效果）
- **MDL 锁（元数据锁）**：MySQL 5.5+ 自动加，保护表结构。DML 加 MDL 读锁（共享），DDL 加 MDL 写锁（排他）。DDL 被 DML 阻塞 → `Waiting for table metadata lock`

**并发写阻塞场景**：
- 两个 UPDATE 修改**同一行** → 第一个事务持有 X 锁 → 第二个阻塞等待
- 两个 UPDATE 修改**不同主键范围**（走索引）→ 锁不同索引记录 → **不阻塞**
- 两个 UPDATE 修改**不同范围但没走索引** → 全表所有行加锁 → **互相阻塞**（退化为表锁效果）

### InnoDB 三种行锁

| 锁类型 | 锁定范围 | 作用 |
|--------|---------|------|
| **Record Lock** | 锁住单条索引记录 | 防止该行被并发修改 |
| **Gap Lock** | 锁住索引记录之间的间隙 | 防止在该间隙插入新行 |
| **Next-Key Lock** | Record + Gap 组合 | **InnoDB 默认行锁方式**，左开右闭区间 |

## 三、底层原理

### Next-Key Lock 如何防幻读

假设表中有 id = 5, 10, 15，Next-Key Lock 锁定区间为：

```
(-∞, 5], (5, 10], (10, 15], (15, +∞)
```

`SELECT * FROM t WHERE id > 8 AND id < 12 FOR UPDATE` → 锁定 (5, 10] 和 (10, 15] → 无法在 8-12 范围内插入新行 → 防止幻读。

**退化规则**：
- 唯一索引等值查询且记录存在 → 退化为 Record Lock
- 唯一索引等值查询但记录不存在 → Gap Lock
- 非唯一索引等值查询 → Next-Key Lock

### 行锁是加在索引上的

行锁的粒度是**索引记录**，不是数据行。如果 WHERE 条件没走索引 → InnoDB 无法精确定位 → 对所有扫描到的行加锁 → 实际退化为表锁。

> 这就是为什么 UPDATE/DELETE 的 WHERE 条件必须走索引——否则严重影响并发性能。

### 意向锁（IS/IX）

表级锁，InnoDB 自动维护。事务加行锁前先在表上加意向锁，让后续表锁请求快速判断是否有行锁存在，无需逐行检查。

### 死锁

两个事务互相等待对方持有的锁。

```
T1: 锁 A → 等 B
T2: 锁 B → 等 A
→ 死锁
```

**InnoDB 处理**：自动检测死锁（wait-for graph 算法）→ 回滚代价较小的事务 → 释放锁。避免方法：按相同顺序加锁、减少锁范围、缩短事务时间。

> `innodb_deadlock_detect = ON`（默认）自动检测死锁。超高并发场景（如秒杀）死锁检测本身成为瓶颈时可设为 OFF，配合 `innodb_lock_wait_timeout` 处理。

### AUTO-INC 锁（自增锁）★★★★

插入含自增主键的行时，需要保证自增值的唯一性和连续性。InnoDB 通过 `innodb_autoinc_lock_mode` 控制：

| 模式 | 值 | 行为 | 适用场景 |
|------|-----|------|---------|
| 传统模式 | 0 | 所有 INSERT 加表级 AUTO-INC 锁，语句执行完释放 | 兼容老版本 |
| **连续模式（默认）** | 1 | 简单 INSERT（知道行数）用轻量级互斥量；批量 INSERT（INSERT...SELECT）加表锁 | 保证 STATEMENT binlog 安全 |
| 交叉模式 | 2 | 所有 INSERT 用轻量级互斥量，完全并发 | 性能最高，但自增值可能不连续 |

**模式 2 下自增值为什么不连续？** 多个事务并发 INSERT，自增值先分配给各事务，但某个事务回滚后该 ID 被废弃 → 出现空洞。

```
面试要点：
- 自增 ID 用完了怎么办？INT 上限 ~21 亿（达到后报错），BIGINT 几乎不会用完
- 分布式场景推荐雪花算法，避免依赖数据库自增
- ROW 格式 binlog + 交叉模式 = 最高并发插入性能
```

### SKIP LOCKED / NOWAIT（MySQL 8.0+）★★★★

高并发场景下，传统 `SELECT FOR UPDATE` 遇到锁会**一直等待**直到超时。8.0 新增两个选项：

```sql
-- SKIP LOCKED: 跳过已被锁定的行，返回未被锁的行（不等待）
-- 典型场景：任务队列消费
SELECT * FROM tasks WHERE status = 0
ORDER BY create_time LIMIT 10
FOR UPDATE SKIP LOCKED;

-- NOWAIT: 遇到锁立即报错，不等待
-- 典型场景：用户抢购（报错提示"抢的人太多"）
SELECT * FROM product WHERE id = 123
FOR UPDATE NOWAIT;
```

| 行为 | FOR UPDATE（默认） | SKIP LOCKED | NOWAIT |
|------|-------------------|-------------|--------|
| 行被锁时 | 等待直到超时 | 跳过该行 | 立即报错 |
| 适用场景 | 必须拿到那一行 | 队列消费、批量任务 | 用户交互、快速失败 |

## 四、关键设计

### 为什么 InnoDB 用 Next-Key Lock 而不是单纯的行锁？

单纯行锁只能防止并发修改同一行，无法防止在查询范围内插入新行（幻读）。Next-Key Lock = 行锁 + 间隙锁，同时阻止修改和插入，在 RR 级别下消除幻读。

### 为什么不所有查询都用 Gap Lock？

Gap Lock 会降低并发插入性能（间隙被锁住无法插入）。InnoDB 只在需要防止幻读时加（RR 级别下的当前读），RC 级别只用 Record Lock，牺牲隔离性换取性能。

## 五、面试高频问题

**Q1：InnoDB 有哪些锁？**

表锁、行锁（Record Lock）、间隙锁（Gap Lock）、Next-Key Lock（行+间隙，默认）、意向锁（IS/IX）。Next-Key Lock 是 RR 级别下防幻读的核心手段。

**Q2：两个 UPDATE 修改不同主键范围会阻塞吗？**

如果都走主键索引 → 不会阻塞（行锁加在不同索引记录上）。如果没走索引 → 可能全表所有行加锁 → 互相阻塞。

**Q3：死锁怎么发生的？怎么避免？**

两个事务相互等待对方持有的锁。避免：统一加锁顺序、减小锁范围、缩短事务、批量操作拆分。InnoDB 自动检测并回滚代价小的事务。

**Q4：SELECT FOR UPDATE 和普通 SELECT 的区别？**

普通 SELECT 是快照读，走 MVCC，不加锁。SELECT FOR UPDATE 是当前读，读最新已提交版本并加 X 锁，阻塞其他写操作。

**Q5：一个 UPDATE 语句在不同条件下加了什么锁？（高频场景题）★★★★★**

这是面试中最高频的锁场景题。不同隔离级别 + 不同索引条件 → 锁行为完全不同：

**场景 A：唯一索引 + 等值查询 + 记录存在（RC/RR 行为相同）**
```sql
-- id 是主键
UPDATE t SET c=1 WHERE id = 10;  -- 只锁 id=10 这一行的 X 锁（Record Lock）
```

**场景 B：唯一索引 + 等值查询 + 记录不存在（RC vs RR 行为不同）**
```sql
-- id=10 不存在
UPDATE t SET c=1 WHERE id = 10;
-- RC: 不加行锁（只加意向锁 IX）
-- RR: 在 id=10 所在的间隙加 Gap Lock，锁住 (5,10) 防止插入 id=10
```

**场景 C：普通索引（非唯一）+ 等值查询（RR 级别）**
```sql
-- age 是普通索引（非唯一），age=25 有 3 行记录
UPDATE t SET c=1 WHERE age = 25; -- RR 级别
-- 锁定 3 行 Record Lock + 这 3 行前后的 Gap Lock
-- = 多个 Next-Key Lock
```

**场景 D：无索引列 + 等值查询（致命场景）**
```sql
-- name 没有索引
UPDATE t SET c=1 WHERE name = 'Alice'; -- RR 级别
-- 全表扫描 → 对扫描到的每一行都加 Next-Key Lock
-- 实际效果 = 全表锁！任何插入都被阻塞
-- RC 级别：扫描到的行加 Record Lock（没有 Gap Lock），但加完后释放未匹配的行
```

**场景 E：主键范围查询（RR 级别）**
```sql
-- id 现有值：5, 10, 15, 20
UPDATE t SET c=1 WHERE id >= 10 AND id < 18;
-- 锁定：id=10(Record), (10,15)Gap, id=15(Record), (15,20)Gap
-- 等效区间：[10, 20) = 无法插入 id 在 10-20 的行
```

**Q6：RC 级别和 RR 级别的锁行为核心区别？**

| 对比 | RC | RR |
|------|-----|-----|
| Gap Lock | **不加** | 加（防幻读） |
| 扫描非匹配行 | 加锁后立即释放 | 一直持有到事务结束 |
| 并发插入 | 高（无间隙锁） | 受间隙锁影响 |
| 幻读 | 可能（当前读） | 不会（Gap Lock + MVCC） |
| 死锁概率 | 较低 | 较高（Gap Lock 范围大） |

**Q7：MySQL 8.0 的 SKIP LOCKED 和 NOWAIT 有什么区别？**

SKIP LOCKED 遇到被锁的行直接跳过，返回未被锁的行（适合任务队列消费）。NOWAIT 遇到锁立即报错（适合用户交互场景，快速失败）。默认的 FOR UPDATE 会一直等待直到超时。

**Q8：AUTO-INC 锁有几种模式？自增 ID 用完了怎么办？**

三种模式：传统（表锁，0）、连续（默认，1，简单 INSERT 轻量锁）、交叉（全部轻量，2，并发最高但不连续）。INT 上限约 21 亿→达到后报错 Duplicate entry，BIGINT 几乎不会用完。分布式推荐雪花算法避免依赖数据库自增。

- **Next-Key Lock** = Record Lock + Gap Lock，左开右闭区间
- **RR + Next-Key Lock** → 消除幻读
- **行锁在索引上**：不走索引 → 行锁升级为表锁
- **乐观锁**（version CAS）vs **悲观锁**（SELECT FOR UPDATE）：冲突少用乐观，冲突多用悲观
- **AUTO-INC 锁**：传统/连续（默认）/交叉 三种模式，分布式用雪花算法替代
- **SKIP LOCKED**（跳过被锁行，队列消费）/ **NOWAIT**（遇锁报错，快速失败）→ 8.0+

---

<a id="ch6"></a>

# 六、日志系统

## 一、背景（为什么需要）

内存写高性能但不持久，磁盘写持久但随机写慢。日志系统用**顺序写**架起两者桥梁，实现高性能 + 持久性。

## 二、核心概念

### 三大日志对比

| 日志 | 层级 | 类型 | 作用 | 写入方式 |
|------|------|------|------|---------|
| **redo log** | InnoDB 引擎层 | 物理日志（"第 X 页偏移 Y 改为 Z"） | 崩溃恢复（持久性） | 循环写 |
| **undo log** | InnoDB 引擎层 | 逻辑日志（"旧值是..."） | 事务回滚 + MVCC 版本链 | 随机写 |
| **binlog** | Server 层 | 逻辑日志（SQL/行变化） | 主从复制 + 时间点恢复 | 追加写 |

## 三、底层原理

### redo log：WAL（Write-Ahead Logging）

```
为什么需要 redo log？
  直接写 B+ 树数据页 = 随机 I/O（慢）
  → 引入 redo log：先顺序写日志（快）→ 异步批量刷数据页（慢但可以合并）
  → 事务提交时只保证 redo log 落盘，不保证数据页落盘

写入流程：
  1. 修改 Buffer Pool 中的页 → 标记脏页
  2. 将修改记录写入 redo log buffer（内存）
  3. 事务提交 → fsync redo log buffer 到磁盘（ib_logfile）
  4. 后台 checkpoint 批量将脏页刷到数据文件
  5. 刷盘后，对应 redo log 空间可被覆盖（循环写）

宕机恢复：
  redo log 中有记录 → 重做 → 保证已提交的数据不丢失
  redo log 中无记录 → 事务未提交 → 配合 undo log 回滚
```

**`innodb_flush_log_at_trx_commit`**：
- `1`（推荐）：每次提交都 fsync，最安全
- `2`：每次提交写 OS 缓存，每秒 fsync，OS 崩溃可能丢 1 秒
- `0`：每秒写 buffer 并 fsync，MySQL 崩溃可能丢 1 秒

### undo log：事务回滚 + MVCC

```
作用 1 - 事务回滚：
  执行 UPDATE 前 → 将旧值写入 undo log
  事务回滚时 → 用 undo log 中的旧值恢复数据

作用 2 - MVCC 版本链：
  每行数据的 roll_pointer → undo log 中的旧版本
  多个旧版本串成版本链 → 快照读沿版本链找到可见版本
```

存储：系统表空间或独立 undo 表空间。事务提交后不立即删除，由 purge 线程异步清理。

### binlog：主从复制 + 数据恢复

三种格式：
| 格式 | 内容 | 优点 | 缺点 |
|------|------|------|------|
| STATEMENT | 记录 SQL 语句 | 体积小 | 部分函数（NOW()）导致主从不一致 |
| **ROW（推荐）** | 记录行变化前后镜像 | 精确 | 体积大 |
| MIXED | 混合 | 折中 | 复杂性 |

### 两阶段提交（2PC）

保证 redo log 和 binlog 的一致性：

```
Prepare 阶段：InnoDB 写 redo log → 标记为 prepare 状态
写 binlog 阶段：Server 层写 binlog 到磁盘
Commit 阶段：InnoDB 将 redo log 标记为 commit 状态

崩溃恢复逻辑：
  redo log prepare + binlog 完整 → 提交（认为事务成功）
  redo log prepare + binlog 不完整 → 回滚
  redo log commit → 已完成，无需处理
```

> 为什么需要 2PC？如果没有 2PC：先写 binlog 再写 redo log → binlog 写了但 redo log 没写 → 主库宕机回滚，从库重放 binlog → 主从数据不一致。

### Group Commit（组提交）★★★★

高并发下多个事务排队等待 fsync，InnoDB 将**多个事务的 redo log / binlog 合并为一次磁盘 fsync**，大幅提升写入吞吐。

```
无组提交：
  T1 提交 → fsync redo → fsync binlog → T2 提交 → fsync redo → fsync binlog → ...
  → 10 个事务 = 20 次 fsync

有组提交：
  T1,T2,T3 同时提交 → 合并 redo fsync（1 次）→ 合并 binlog fsync（1 次）
  → 3 个事务 = 2 次 fsync → 吞吐提升 3x
```

**三个队列阶段**（MySQL 5.6+）：
```
flush 阶段 → sync 阶段 → commit 阶段
（redo log flush）→（binlog fsync）→（标记 commit）
每个阶段可容纳多个事务排队，各自合并 fsync
```

**面试要点**：这就是为什么 `sync_binlog=1` + `innodb_flush_log_at_trx_commit=1`（双1配置）在高并发下并非不可接受——组提交将多次 fsync 摊薄为一次，单次事务的感知延迟几乎不变，但系统整体吞吐可以维持。金融场景的双1配置 + 高性能 SSD 是可行的。

### 一条 UPDATE 的完整执行过程

以 `UPDATE users SET name='Bob' WHERE id=1` 为例：

```
1. 执行器通过存储引擎找到 id=1 的行（先查 Buffer Pool，未命中从磁盘读）
2. 将旧值写入 undo log（用于回滚和 MVCC）
3. 在 Buffer Pool 中修改数据页（将 name 改为 Bob），标记为脏页
4. 将修改写入 redo log buffer
5. 事务提交时：
   a. redo log 写盘，标记 prepare
   b. binlog 写盘
   c. redo log 标记 commit
6. 后台 checkpoint 异步将脏页刷到磁盘
```

## 四、关键设计

### 为什么不能只用 binlog 而不用 redo log？

binlog 是逻辑日志（SQL 语句或行变化），不知道数据页的物理位置。宕机恢复时无法确定哪些数据页已刷盘、哪些没有，无法精确重做。redo log 是物理日志（页号 + 偏移 + 值），可以精确重做。两者职责不同，缺一不可。

### 为什么要顺序写 redo log 而不是直接写 B+ 树？

磁盘顺序写（~100MB/s+）vs 随机写（~100 IOPS）。redo log 顺序追加写极快，事务提交只需等 redo log 刷盘（ms 级）。数据页的随机写可以异步批量完成，多次修改被合并为一次 I/O。

## 五、面试高频问题

**Q1：redo log 和 binlog 的区别？**

redo log 是 InnoDB 物理日志，循环写，用于崩溃恢复。binlog 是 Server 层逻辑日志，追加写，用于主从复制和数据恢复。两者通过两阶段提交保证一致性。

**Q2：两阶段提交的过程？**

Prepare（redo log prepare）→ 写 binlog → Commit（redo log commit）。崩溃恢复时：redo log prepare + binlog 完整 → 提交；否则 → 回滚。

**Q3：有了 undo log 为什么还需要 redo log？**

undo log 解决原子性（回滚），redo log 解决持久性（崩溃恢复）。职责不同。undo log 是"后悔药"（撤销），redo log 是"保险单"（重做）。

**Q4：redo log 在内存里还是在磁盘上？**

两处都有：redo log buffer（内存，事务执行过程中先写这里）和 redo log 文件（磁盘，`ib_logfile0/1`，事务提交时 fsync 到磁盘）。

**Q5：什么是 Group Commit？为什么双1配置下高并发还能有不错的性能？**

组提交将多个并发事务的 redo log / binlog fsync 合并为一次磁盘写入，大幅减少 fsync 次数，提升写入吞吐。双1配置 + SSD + 高并发 = 组提交摊薄每次 fsync 成本，性能仍然可接受。

## 六、总结（速记版）

- **redo log**：物理日志 + 循环写 + WAL → 崩溃恢复（持久性）
- **undo log**：逻辑日志 + 版本链 → 事务回滚 + MVCC
- **binlog**：逻辑日志 + 追加写 → 主从复制 + 时间点恢复
- **2PC**：prepare → binlog → commit，保证 redo log 和 binlog 一致
- **Group Commit**：多事务合并 fsync → 双1配置下仍有高吞吐
- **WAL 精髓**：先顺序写日志 → 后异步随机写数据页

---

<a id="ch7"></a>

# 七、SQL 优化与 EXPLAIN

## 一、背景（为什么需要）

慢 SQL 是数据库性能问题的主要来源。优化 SQL 的核心能力：会看执行计划、会建索引、会改写 SQL。

## 二、核心概念

### EXPLAIN 关键字段

| 字段 | 含义 | 关注点 |
|------|------|--------|
| **type** | 访问类型 | `ALL`（全表扫描）需优化 |
| **key** | 实际使用的索引 | `NULL` 表示未用索引 |
| **key_len** | 使用的索引长度 | 判断联合索引用了几列 |
| **rows** | 估算扫描行数 | 越小越好 |
| **Extra** | 附加信息 | `Using index`（好）/ `Using filesort`（需优化）/ `Using temporary`（需优化） |

### type 性能排序（从好到差）

```
const（主键等值）> eq_ref（唯一索引 JOIN）> ref（普通索引等值）
> range（索引范围）> index（全索引扫描）> ALL（全表扫描）
```

### Extra 重点值

| 值 | 含义 | 评价 |
|----|------|------|
| `Using index` | 覆盖索引，不回表 | ✅ 好 |
| `Using index condition` | 索引下推（ICP），引擎层过滤 | ✅ 好 |
| `Using where` | Server 层过滤 | 正常 |
| `Using filesort` | 需要额外排序 | ⚠️ 需优化 |
| `Using temporary` | 使用临时表 | ❌ 需优化 |

## 三、底层原理

### EXPLAIN key_len 计算（面试高频）★★★★★

**面试经典题**："联合索引 `(a INT, b VARCHAR(50), c INT)`，EXPLAIN 显示 key_len=214，用了几列？"

key_len 是 MySQL 在索引中**实际使用的字节数**，通过它可以精确判断联合索引用到第几列。

**计算规则**：

| 字段类型 | 计算公式 | 示例 |
|---------|---------|------|
| `CHAR(n)` | n × 字符集字节数 | `CHAR(10) utf8mb4` → 10×4=40，允许 NULL +1 → 41 |
| `VARCHAR(n)` | n × 字符集字节数 + 2（存长度） | `VARCHAR(50) utf8mb4` → 50×4+2=202，允许 NULL+1 → 203 |
| `INT` | 4 字节 | 4，允许 NULL +1 → 5 |
| `BIGINT` | 8 字节 | 8，允许 NULL +1 → 9 |
| `DATETIME` | 5 字节（5.6.4+），之前 8 字节 | 5，允许 NULL +1 → 6 |
| `TINYINT` | 1 字节 | 1，允许 NULL +1 → 2 |

**实战判断**：

```sql
-- 表：id INT NOT NULL, name VARCHAR(50), age INT
-- 联合索引：idx_name_age(name, age)
-- name: VARCHAR(50) utf8mb4 允许NULL → 50*4+2+1 = 203
-- age: INT 允许NULL → 4+1 = 5

-- 案例1：key_len = 203 → 只用到了 name 一列
EXPLAIN SELECT * FROM t WHERE name = 'Alice';

-- 案例2：key_len = 208 → 用到了 name + age 两列
EXPLAIN SELECT * FROM t WHERE name = 'Alice' AND age = 25;

-- 案例3：key_len = 203 → city 不在索引中，只用到了 name 一列
EXPLAIN SELECT * FROM t WHERE name = 'Alice' AND city = 'Beijing';  -- city 列未建索引
```

**面试回答模板**："key_len 表示索引中使用的字节数，通过计算各列的字节长度可以精确判断联合索引用到了第几列。VARCHAR 额外 +2（存长度），可 NULL 字段额外 +1。"

### EXPLAIN ANALYZE（MySQL 8.0.18+）★★★★★

**EXPLAIN vs EXPLAIN ANALYZE 的本质区别**：EXPLAIN 显示的是**优化器的估算**（cost estimate），不实际执行查询。EXPLAIN ANALYZE **真正执行查询**，输出每步的**实际耗时和实际行数**。

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 20;
-- 输出示例：
-- -> Filter: (users.age > 20)  (cost=5.2 rows=100) (actual time=0.05..0.12 rows=87 loops=1)
--     -> Table scan on users  (cost=5.2 rows=100) (actual time=0.03..0.09 rows=100 loops=1)
```

**关键信息解读**：
- `cost` = 优化器估算成本 / `rows` = 估算行数
- `actual time` = 第一步开始到结束的时间 / 第二步开始到结束的时间
- `actual rows` = 实际返回行数（与估算行数对比 → 发现统计信息是否准确）

**使用场景**：EXPLAIN 说 rows=100 但实际 rows=100000 → 统计信息过期或索引不匹配 → `ANALYZE TABLE` 更新统计信息或重建索引。

**与 Optimizer Trace 的关系**：
- EXPLAIN：看优化器选了哪个计划
- EXPLAIN ANALYZE：看实际执行情况和估算偏差
- Optimizer Trace：看优化器**为什么**这么选（候选计划 + 代价对比）
- 三者配合：Trace 找原因 → ANALYZE 验偏差 → 定优化方案

### Hash Join（MySQL 8.0.18+）★★★★

MySQL 8.0.18 之前，等值 JOIN 无索引时用 **BNL（Block Nested Loop）**——外层每行都扫描内层表，性能极差。8.0.18 引入 **Hash Join**：用 JOIN 条件建哈希表，O(N+M) 替代 O(N×M)。

```
BNL（老）：外表每行 → 扫描整个内表 → 速度 O(N × M)
Hash Join（新）：内表建哈希表 → 外表每行 O(1) 查找 → 速度 O(N + M)
```

```sql
-- 两个大表 JOIN 且无索引时，8.0.18+ 自动用 Hash Join
SELECT * FROM orders o JOIN customers c ON o.cust_id = c.id;
-- EXPLAIN ANALYZE 显示：-> Hash inner join (...)
```

**适用条件**：等值 JOIN + 无合适索引。不等值条件（`>`, `<`）仍用 Nested Loop。

> Hash Join 的内存受 `join_buffer_size` 限制。数据超过内存时会将内表数据分批写入临时文件（磁盘 Hash Join），性能下降——确保 `join_buffer_size` 足够，或给 JOIN 列建索引。

### Semi-join 优化（子查询自动改写）

MySQL 5.6+ 将 `IN (SELECT ...)` 子查询**自动转为 semi-join**（半连接），只检查外层某行是否在内层存在（匹配到一行即停止），避免全表扫描 + 去重。

```sql
-- 原始写法
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE amount > 100);

-- 优化器自动转为 semi-join → 等价于：
-- "users 中至少有一条 order.amount>100 的那些行"
```

**Semi-join 执行策略**（EXPLAIN Extra 可看到）：

| 策略 | Extra 显示 | 说明 |
|------|-----------|------|
| **FirstMatch** | `FirstMatch(users)` | 外表每行去内表找，找到第一条即返回 |
| **LooseScan** | `LooseScan` | 利用内表索引去重，仅扫描一次 |
| **MaterializeLookup** | `Materialize` | 内表结果先物化到临时表 → 建索引 → 精确查找 |
| **DuplicateWeedout** | `Start temporary / End temporary` | 临时表去重 |

**面试要点**：MySQL 5.6+ 的 `IN (SELECT ...)` 不再像 5.5 那样每条外层行都执行一遍子查询（DEPENDENT SUBQUERY），优化器会自动选择最佳 semi-join 策略。EXPLAIN 中看到 `FirstMatch`、`LooseScan` 等就是 semi-join 在起作用。

### 慢查询排查六步法

```
1. 开启慢查询日志：slow_query_log=ON, long_query_time=1（超1秒记录）
2. 找到具体慢 SQL
3. EXPLAIN 分析执行计划：重点看 type、key、rows、Extra
4. 根据分析定位问题：
   type=ALL → 加索引
   key=NULL → 检查索引失效（函数/类型转换/最左前缀等）
   rows 很大 → 索引区分度低，换索引或优化查询条件
   Using filesort → ORDER BY 列与索引不匹配
   Using temporary → GROUP BY/DISTINCT 无法利用索引
5. 优化方案：加索引 / 改 SQL / 覆盖索引 / 改写子查询为 JOIN
6. 验证优化效果：EXPLAIN 对比 rows 变化
```

### 深分页优化

```sql
-- 慢：需要跳过 100 万行
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;

-- 优化1：游标分页（推荐）
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 20;

-- 优化2：延迟关联
SELECT o.* FROM orders o
JOIN (SELECT id FROM orders ORDER BY id LIMIT 1000000, 20) t
ON o.id = t.id;
-- 子查询先只扫主键索引（不回表），再用主键回表取完整数据

-- 优化3：ES 接管分页（大规模搜索场景）
-- 列表/搜索类分页交给 Elasticsearch，MySQL 只负责单条详细数据查询
```

### 大表操作优化

```
DELETE / UPDATE 大表：
  不要一次性操作几千万行 → 长事务/锁竞争/undo膨胀/主从延迟
  正确做法：按主键或时间范围分批提交（每批 5000-10000 行）

分批删除示例：
  -- 删除 30 天前的数据，每次 5000 行，循环执行
  DELETE FROM logs WHERE create_time < '2024-01-01' LIMIT 5000;
  -- 循环直到 affected_rows = 0（可配合存储过程或脚本实现）

INSERT 大量数据：
  用批量 INSERT（VALUES (...), (...), (...)）→ 减少网络+解析+事务提交
  比单条循环 INSERT 快几十倍
```

### SQL 改写实战模式 ★★★★★

**面试经典题**："一条 SQL 很慢，没有合适索引可用，怎么改写？"

**1. OR → UNION ALL**

```sql
-- 慢（OR 可能使索引失效，合并两个结果集开销大）
SELECT * FROM orders WHERE buyer_id = 1 OR seller_id = 2;

-- 改写（各走各的索引，UNION ALL 不去重）
SELECT * FROM orders WHERE buyer_id = 1
UNION ALL
SELECT * FROM orders WHERE seller_id = 2;
```

**2. NOT IN → LEFT JOIN + IS NULL**

```sql
-- 慢（NOT IN 子查询每行判断，NULL 陷阱）
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM orders);

-- 改写（LEFT JOIN + IS NULL 走索引 JOIN）
SELECT u.* FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.user_id IS NULL;
```

**3. 大 IN → 临时表 JOIN**

```sql
-- IN (1000+个值) → 参数过多 + 索引选择偏离
SELECT * FROM users WHERE id IN (1,2,3,...,5000);

-- 改写：值列表先入临时表，再 JOIN
CREATE TEMPORARY TABLE tmp_ids (id INT PRIMARY KEY);
INSERT INTO tmp_ids VALUES (1),(2),(3),...,(5000);
SELECT u.* FROM users u INNER JOIN tmp_ids t ON u.id = t.id;
```

**4. LIKE 前缀模糊 → 反向索引 + 全文索引**

```sql
-- 慢（'%abc' 无法用索引）
SELECT * FROM users WHERE name LIKE '%abc%';

-- 方案A：全文索引（FULLTEXT），适合文本搜索场景
ALTER TABLE users ADD FULLTEXT idx_ft_name(name);
SELECT * FROM users WHERE MATCH(name) AGAINST('abc');

-- 方案B：反转字段 + 前缀索引（明确搜索后缀）
-- 新增列 reversed_name，反转存储，'%abc' → 变为 'cba%' → 可用索引
```

**5. COUNT 大表优化**

```sql
-- 慢（需扫描索引统计行数，MVCC 下无法缓存行数）
SELECT COUNT(*) FROM orders WHERE status = 1;

-- 方案A：业务侧维护计数表
UPDATE order_stats SET cnt = cnt + 1 WHERE status = 1; -- 事件驱动

-- 方案B：Redis 实时计数 + MySQL 定时校准

-- 方案C：用 EXPLAIN 估算（⚠️ 仅参考，偏差可能很大，不能用于精确计数）
EXPLAIN SELECT COUNT(*) FROM orders WHERE status = 1; -- 看 rows
```

## 四、关键设计

### 优化器选错索引怎么办？

1. `ANALYZE TABLE` 更新统计信息（最常见原因：统计信息不准导致代价估算错误）
2. **`FORCE INDEX(idx_name)`**：强制使用指定索引，优化器必须遵从
3. **`USE INDEX(idx_name)`**：建议使用指定索引，优化器可以忽略
4. **`IGNORE INDEX(idx_name)`**：忽略指定索引，从其余索引中选择

```sql
SELECT * FROM orders FORCE INDEX(idx_status) WHERE status = 1;
SELECT * FROM orders USE INDEX(idx_create_time) WHERE status = 1;
```

长期：找到根本原因（统计信息/索引设计问题），而不是依赖 FORCE INDEX。

### 索引优化的核心原则

- **查询模式驱动索引设计**：先找到慢 SQL → 根据 WHERE/ORDER BY/GROUP BY 建索引
- **联合索引列顺序**：等值条件放前面 > 范围条件放后面 > 区分度高优先
- **覆盖索引优先**：将 SELECT 常用列加入索引，避免回表
- **删除冗余索引**：`(a,b)` 已覆盖 `(a)` 的场景

### Using filesort 原理

filesort 发生在 ORDER BY 无法利用索引有序性时。MySQL 在 `sort_buffer_size`（默认 256KB）内存中排序：
- 排序数据 <= sort_buffer_size → **内存排序**（快排）
- 排序数据 > sort_buffer_size → **外部排序**（将数据分块排序 → 写入临时文件 → 归并合并）

优化方向：增大 sort_buffer_size、让 ORDER BY 列匹配索引有序性。

### Using temporary 临时表产生原因

MySQL 在以下场景会创建临时表（内存临时表超限则转磁盘）：
- GROUP BY 列与索引顺序不一致
- DISTINCT + ORDER BY 组合无法利用索引
- UNION（去重）操作
- 派生表（FROM 子查询）

Extra 出现 `Using temporary` 严重拉低性能 → 优化索引消除临时表、或改写 SQL。

### 索引合并（Index Merge）

MySQL 5.0+ 支持一个查询使用多个索引：

| 类型 | 行为 | Extra 显示 |
|------|------|-----------|
| **Intersection** | 多索引结果取交集（AND 条件） | `Using intersect(...)` |
| **Union** | 多索引结果取并集（OR 条件） | `Using union(...)` |
| **Sort-Union** | 先排序再取并集 | `Using sort_union(...)` |

```sql
-- 两个索引结果取交集
SELECT * FROM t WHERE a = 1 AND b = 2;  -- idx_a 和 idx_b 都有索引
```

### MRR（Multi-Range Read）

将二级索引查到的主键先排序，再**批量顺序回表**，而非逐条随机回表。减少随机 I/O，提升范围查询性能。`optimizer_switch='mrr=on,mrr_cost_based=on'`。

### Optimizer Trace

当 EXPLAIN 不够时，用 Optimizer Trace 查看优化器完整决策过程：
```sql
SET optimizer_trace='enabled=on';
SELECT * FROM t WHERE ...;
SELECT * FROM information_schema.OPTIMIZER_TRACE;
SET optimizer_trace='enabled=off';
```
输出包含：considered_execution_plans（候选计划）、rows_estimation（行数估算）、attached_conditions（条件附加）。帮助理解优化器为什么选了某个索引。

### Online DDL：大表变更不停机 ★★★★★

**面试题**："千万级数据的表，要加一个字段/索引，怎么做到不停机？"

**核心概念**：MySQL 5.6+ 支持 Online DDL，部分 DDL 操作允许并发 DML，减少锁表时间。

| ALGORITHM | 行为 | 是否阻塞读写 |
|-----------|------|------------|
| **INSTANT**（8.0+） | 仅改元数据，秒级完成 | 完全不阻塞 |
| **INPLACE** | 在原表上操作，避免数据拷贝 | 不阻塞读，开始和结束短暂锁（秒级） |
| **COPY** | 创建新表 + 拷贝数据 + 切换 | 全程阻塞写（可能需要数小时） |

**常见操作对锁的影响**：

| 操作 | 方式 | 说明 |
|------|------|------|
| 加普通索引 | `ALGORITHM=INPLACE, LOCK=NONE` | 不阻塞读写 |
| 加列（8.0） | `ALGORITHM=INSTANT` | 秒级完成，完全不阻塞 |
| 加列（5.7） | `ALGORITHM=INPLACE` | 短暂锁表 |
| 改列类型 | `ALGORITHM=COPY` | 全量拷贝，**长时间阻塞** |
| 加全文索引 | `ALGORITHM=INPLACE` | 需要 LOCK=SHARED |

**pt-online-schema-change（Percona Toolkit）**：对于不支持 Online DDL 的操作（如改列类型），使用触发器 + 影子表方案：

```
流程：
  1. 创建一张与原表结构相同的新表（含变更）
  2. 在原表上创建触发器（INSERT/UPDATE/DELETE）→ 自动同步到新表
  3. 按主键分批将原表数据拷贝到新表（chunk 机制，每次 ~1000 行）
  4. 数据同步完成后，原子 RENAME TABLE 切换
  5. 删除触发器 + 原表
```

**面试要点**：Online DDL 不是"完全不阻塞"——INPLACE 在开始准备和结束时各有一个短暂的排他锁窗口（通常毫秒~秒级）。真正完全不阻塞的是 8.0 的 INSTANT。主从环境下先在从库执行 DDL，再主从切换，最安全。

## 五、面试高频问题

**Q1：EXPLAIN 怎么看？重点关注什么？**

type（是否 ALL/全表扫描）、key（是否用索引）、rows（扫描行数）、Extra（Using filesort/Using temporary 需优化）。type=ALL 或 key=NULL 是严重问题。

**Q2：SQL 慢怎么排查？**

慢查询日志定位慢 SQL → EXPLAIN 分析执行计划 → 看扫描行数和是否走索引 → 加索引或改写 SQL → 对比优化前后 EXPLAIN 结果。

**Q3：深分页为什么慢？怎么优化？**

`LIMIT 1000000,20` 需要扫描并丢弃前 100 万行。优化：游标分页（`WHERE id > last_id LIMIT 20`）或延迟关联（子查询只扫主键索引）。

**Q4：COUNT(*) 在 InnoDB 中为什么慢？如何优化？**

InnoDB 不保存精确行数（MVCC 下不同事务看到行数不同），需扫描索引。优化：用更小的二级索引扫描、业务侧维护计数表。

**Q5：EXPLAIN 发现优化器选错索引怎么办？**

先 `ANALYZE TABLE` 更新统计信息。若仍选错：`FORCE INDEX(idx)` 强制指定、`USE INDEX(idx)` 建议、`IGNORE INDEX(idx)` 排除。长期应找出根本原因（统计信息不准、索引设计问题）。高级诊断用 `Optimizer Trace` 查看优化器完整决策过程。

**Q6：Using filesort 和 Using temporary 分别是什么？怎么优化？**

filesort：ORDER BY 无法利用索引有序性 → 需要额外排序（sort_buffer 内存排或外部归并排）。优化：ORDER BY 列匹配索引。temporary：GROUP BY/DISTINCT 无法利用索引 → 建临时表（内存优先，超限转磁盘）。优化：调整索引消除临时表。

**Q7：什么是索引合并（Index Merge）？**

一个查询使用多个索引，结果取交集（Intersection，AND 条件）或并集（Union，OR 条件）。是优化器的自动行为，Extra 显示 `Using intersect/union`。

**Q8：EXPLAIN 和 EXPLAIN ANALYZE 的区别？**

EXPLAIN 显示估算值和执行计划（不执行查询），EXPLAIN ANALYZE 真实执行查询并输出每步实际耗时和行数。当估算 rows 与实际偏差很大时，说明统计信息过期或索引设计有问题。

**Q9：MySQL 8.0 的 Hash Join 是什么？解决什么问题？**

8.0.18 前等值 JOIN 无索引用 BNL（O(N×M)），8.0.18+ 用 Hash Join（O(N+M)）。内表建哈希表，外表每行 O(1) 查找，性能显著提升。EXPLAIN 显示 `Hash inner join`。

**Q10：IN (SELECT ...) 子查询 MySQL 怎么优化？**

MySQL 5.6+ 自动将 IN 子查询转为 semi-join（半连接），策略有 FirstMatch、LooseScan、MaterializeLookup、DuplicateWeedout。不再像 5.5 那样逐行执行 DEPENDENT SUBQUERY。

**Q11：大表加字段/加索引怎么不停机？**

加索引用 `ALGORITHM=INPLACE, LOCK=NONE`（不阻塞读写）。加字段在 8.0 用 `ALGORITHM=INSTANT`（秒级完成）。改列类型等 COPY 操作用 pt-online-schema-change（触发器 + 影子表，分批拷贝）。最安全的方式：先在从库执行 DDL → 主从切换。

## 六、总结（速记版）

- **EXPLAIN**：看 type（ALL 要改）→ key（NULL 要加索引）→ rows → Extra
- **EXPLAIN ANALYZE**（8.0.18+）：实际执行 + 真实耗时 → 验证估算偏差
- **慢查询六步**：开日志 → 找 SQL → EXPLAIN → 定位 → 优化 → 验证
- **深分页**：游标分页（`WHERE id > last_id`）或延迟关联
- **filesort**：索引排序可避免 / **temporary**：索引 GROUP BY/DISTINCT 可避免
- **FORCE/USE/IGNORE INDEX**：临时干预索引选择，长期应治本
- **Hash Join**（8.0.18+）：无索引等值 JOIN 从 O(N×M) → O(N+M)
- **Semi-join**（5.6+）：IN 子查询自动改写为半连接
- **Online DDL**：INSTANT（8.0秒级）/ INPLACE（不阻塞读）/ COPY（阻塞）→ 大表用 pt-osc
- **大表操作**：分批提交，避免长事务

---

<a id="ch8"></a>

# 八、高可用架构

## 一、背景（为什么需要）

单机 MySQL 三个瓶颈：容量有限（磁盘上限）、并发有限（单机 QPS）、单点故障（宕机即不可用）。高可用体系：主从复制 → 读写分离 → 分库分表。

## 二、核心概念

### 主从复制原理

```
主库：写操作 → 记录 binlog
从库 IO Thread：连接主库 → 读取 binlog → 写入 relay log
从库 SQL Thread：读取 relay log → 重放操作 → 数据同步
```

**复制模式**：
- **异步复制**（默认）：主库写 binlog 即返回，不等从库确认，性能最好但主库宕机可能丢数据
- **半同步复制**：至少一个从库确认收到 binlog 后主库才返回，安全性更高
- **全同步**：所有从库确认后才返回，最安全但性能最差

### GTID 复制（MySQL 5.6+ 核心特性）★★★★★

传统复制用 `(binlog文件名, 位置偏移)` 定位复制位点，主库切换后从库需重新指定。GTID（Global Transaction Identifier）给每个事务分配**全局唯一 ID**，自动追踪复制位置。

**GTID 格式**：`server_uuid:transaction_id`，如 `3E11FA47-71CA-11E1-9E33-C80AA9429562:1-5`

**GTID vs 传统复制**：
| 对比 | 传统复制 | GTID 复制 |
|------|---------|----------|
| 定位方式 | `(binlog文件, position)` | 全局事务 ID |
| 主库切换 | 手动 CHANGE MASTER + 指定位点 | 自动找位点 |
| 故障恢复 | 复杂（可能重复/丢失数据） | 自动跳过已执行 GTID |
| 复制拓扑 | 简单主从 | 支持复杂的多主/链式复制 |
| 配置 | `CHANGE MASTER TO MASTER_LOG_FILE=...` | `CHANGE MASTER TO MASTER_AUTO_POSITION=1` |

**工作原理**：
```
主库：
  事务提交 → 分配 GTID → 写入 binlog（GTID Event）
  
从库：
  IO Thread 拉取 binlog → 取出 GTID → 写入 relay log
  SQL Thread 执行前检查：
    gtid_executed 中是否已有此 GTID？
      有 → 跳过（防止重复执行）
      无 → 执行事务 → 将 GTID 加入 gtid_executed
```

**为什么 GTID 不会重复？** 每个 GTID 在全局唯一（server_uuid 唯一），从库执行过的 GTID 记录在 `gtid_executed` 集合中，新事务的 GTID 若已存在则自动跳过。

**开启 GTID**：
```sql
gtid_mode = ON
enforce_gtid_consistency = ON
```

> 面试要点：GTID 简化了主从切换和故障恢复，是 MySQL 高可用的基础设施。MGR（MySQL Group Replication）依赖 GTID。

### 分库分表

**分表**：同一实例内拆多表 → 解决单表数据量过大（B+ 树层高增加）
**分库**：不同实例 → 解决单库 CPU/磁盘 I/O/连接数瓶颈

**分片策略**：
- **垂直拆分**：按业务模块（用户库、订单库）或冷热字段拆开
- **水平拆分**：同一表按规则（hash 取模、range、一致性哈希）分散到多个库/表

**分库分表后的问题**：跨库 JOIN 不支持（应用层组装）、分布式事务复杂、全局唯一 ID 需雪花算法、COUNT/ORDER BY 需汇总各分片。

**分片键（Sharding Key）选择原则**：
1. 选高频查询条件列（如 `user_id`）→ 让大部分查询落在单分片
2. 避免跨分片查询为主的分片键 → 否则分库分表失去意义
3. 数据尽量均匀分布 → 避免热点分片（如按时间分片，最新分片压力最大）
4. 最好与业务主键一致 → 减少全局 ID 映射的复杂度

## 三、底层原理

### 主从延迟的原因与处理

**原因**：主库并发写入多 → 从库 SQL Thread 单线程重放瓶颈（5.7+ 支持并行复制）、从库读压力大、网络延迟、大事务 binlog 体积大。

**处理**：
- 并行复制（MTS，`slave_parallel_workers`）
- 强制读主库（下单后立即查订单 → 走主库）
- 半同步复制减少延迟窗口
- 拆分大事务
- 监控 `Seconds_Behind_Master`

### 多源复制（Multi-Source Replication）

一个从库同时从**多个主库**拉取 binlog，汇总到同一个实例：

```
主库A（订单库） → 从库（汇总库） ← 主库B（用户库）
```

适用场景：多个分库汇总到报表库/数据仓库、跨库 JOIN 分析。从库需配置 `CHANNEL` 区分来自不同主库的数据流。

### 读写分离

主库写 + 从库读。问题：主从延迟导致刚写入的数据在从库查不到。解决：对一致性要求高的场景强制走主库（如用户刚修改个人资料后立即查看）。

**中间件方案**：
| 中间件 | 特点 |
|--------|------|
| **ProxySQL** | 高性能 MySQL 代理，支持 SQL 路由、读写分离、连接池、查询缓存。企业级首选 |
| **MySQL Router** | 官方轻量代理，配合 InnoDB Cluster 使用，自动读写分离 |
| **ShardingSphere-Proxy** | 分库分表 + 读写分离一体，Java 生态 |

### 延迟复制（Delayed Replication）

从库有意延迟 N 秒（如 1 小时）执行 relay log。核心价值：**误操作（DROP TABLE / DELETE 无 WHERE）恢复**——在延迟从库中数据尚未被删除，可快速找回。

```sql
CHANGE MASTER TO MASTER_DELAY = 3600;  -- 从库延迟 1 小时
```

> 延迟从库不是备份替代品，而是误操作的"后悔窗口"。通常配合备份策略一起使用。

### MGR（MySQL Group Replication）与 InnoDB Cluster ★★★★

**传统主从**的问题：手动故障转移、可能丢数据、主库是单点。MySQL 5.7+ 引入 MGR，基于 Paxos 协议的多主/单主复制：

```
MGR 单主模式（推荐生产）：
  一个 Primary（读写）+ 多个 Secondary（只读）
  主库宕机 → 剩余节点自动选主（秒级）→ 应用无感知

MGR 多主模式：
  所有节点可读写 → 冲突检测 → 冲突时回滚后提交的事务
```

**InnoDB Cluster = MGR + MySQL Router + MySQL Shell**：MySQL 官方完整高可用方案，一键部署、自动故障转移、读写分离。

| 对比 | 传统主从 | MGR |
|------|---------|-----|
| 故障转移 | 手动/第三方工具（MHA/Orch） | 自动 |
| 数据一致性 | 异步/半同步 | 基于 Paxos 的多数派确认 |
| 脑裂保护 | 无（依赖外部） | 内置（少数派自动驱逐） |
| 多主写入 | 不支持 | 支持（有冲突检测） |

> 面试要点：MGR 是 MySQL 官方高可用的未来方向。传统 MHA + 半同步的方案仍然是存量系统的常见选择。

## 四、关键设计

### 分库分表扩容：数据迁移方案（高频场景题）★★★★★

**面试题**："现有 1 亿数据在单表，要拆到 4 个库（16 张表），如何平滑迁移不停机？"

**核心难点**：
1. 历史数据如何迁移 → 量大（亿级），迁移时间长
2. 新写入数据如何处理 → 迁移期间仍在产生新数据
3. 如何切流不丢数据 → 切换瞬间不能有数据丢失或重复
4. 出问题如何回滚 → 必须有回退方案

**标准方案：双写 + 灰度切读 + 数据校验**：

```
阶段 1：存量迁移（离线）
  ├── 通过 Sqoop/DataX/自定义脚本 批量迁移历史数据
  ├── 按主键分段 + 并发导出 → 写入新库
  └── 不要求实时，可以慢，反复校验数据完整性

阶段 2：增量双写（核心阶段）
  ├── 应用层改造：写操作同时写老库 + 新库（MQ 异步写新库）
  ├── 用 Canal 监听老库 binlog → 实时同步变更到新库
  ├── 开启数据校验：定时对比老新库数据（全量校验 + 增量校验）
  └── 双写跑 1-2 天确保新库数据稳定

阶段 3：灰度切读
  ├── 小流量（1% → 10% → 50% → 100%）读切换到新库
  ├── 每阶段观察：新库 QPS / RT / 错误率是否正常
  └── 异常立即回退读流量到老库

阶段 4：下线老库
  ├── 确认新库稳定运行 N 天
  ├── 停止双写 → 只写新库
  └── 保留老库作为备份（不立即删除，1 个月后清理）
```

**数据一致性校验**：
- 全量对比：按分片逐一对比 crc32/行数
- 增量对比：对比最近 5 分钟的写入是否一致
- 补偿机制：发现不一致 → 自动重放 binlog 修复

**双写的幂等性保证**：
- INSERT：新库先查是否存在，存在则 UPDATE
- UPDATE：根据 id，不存在则 INSERT
- 雪花算法全局唯一 ID 防止主键冲突

**关键工具**：Canal（监听 binlog）、ShardingSphere（分库分表中间件）、DataX（离线数据迁移）。

### 为什么分库分表后不推荐物理外键？

分库场景下物理外键跨库不可用。分表场景下外键增加写入成本、锁竞争和表耦合。互联网高并发系统倾向在应用层维护逻辑外键。

### 主从复制为什么默认异步？

同步复制要求所有从库确认，性能最差（受最慢从库拖累）。异步复制只等主库写完 binlog，吞吐最高。大多数业务可接受短暂的主从延迟，用半同步复制做安全增强。

## 五、面试高频问题

**Q1：主从复制的流程？**

主库写 binlog → 从库 IO Thread 拉取 → 写入 relay log → 从库 SQL Thread 重放。异步复制默认不等从库确认；半同步至少等一个从库确认。

**Q2：主从延迟怎么处理？**

并行复制（MTS）、对一致性要求高的强制读主库、拆分大事务、半同步复制、优化从库硬件（SSD + 加大 Buffer Pool）。

**Q3：分库分表后有什么问题？**

跨库 JOIN 不支持（应用层组装）、分布式事务（复杂）、全局唯一 ID（雪花算法）、COUNT/ORDER BY 需汇总各分片（应用层合并）。分库分表 = 扩展能力 + 引入复杂性。

**Q4：什么时候该分库分表？**

单表数据量 > 2000 万行（B+ 树 4 层+）、单库 QPS 接近上限、磁盘 I/O 接近瓶颈、单库连接数打满。先优化 SQL/索引/缓存，再考虑分库分表（成本高）。

**Q5：MySQL 如何做备份恢复？**

- **逻辑备份**：`mysqldump`（导出 SQL 语句），灵活但恢复慢（逐条重放）
- **物理备份**：`XtraBackup`（拷贝数据文件），快但依赖文件系统和版本
- **恢复策略**：全量备份（每周）+ 增量 binlog（每天）→ 可恢复到任意时间点
- **Clone Plugin**（8.0.17+）：物理备份替代方案，在线克隆整个实例，无需 xtrabackup。适合构建从库、全量备份
- **PITR（Point-in-Time Recovery）**：全量备份 + binlog 按时间点重放

关键参数：`binlog_expire_logs_seconds`（8.0+，替代废弃的 `expire_logs_days`）控制 binlog 保留时间。

**Q6：MGR 和传统主从有什么区别？**

MGR 基于 Paxos 协议，自动故障转移（秒级）、内置脑裂保护、支持多主写入。传统主从需手动或第三方工具切换。InnoDB Cluster = MGR + MySQL Router + MySQL Shell，MySQL 官方完整高可用方案。

**Q7：延迟复制有什么用途？**

从库有意延迟 N 秒重放 relay log。核心场景：误操作恢复（DROP TABLE 后延迟从库数据还在）。不是备份替代品，是误操作的短期"后悔窗口"。

**Q8：读写分离中间件怎么选？**

ProxySQL（高性能、SQL 路由、连接池，企业首选）、MySQL Router（官方轻量，配合 InnoDB Cluster）、ShardingSphere-Proxy（分库分表+读写分离一体）。

## 六、总结（速记版）

- **主从复制**：binlog → relay log → 重放。异步（默认）/ 半同步（更安全）/ 全同步
- **主从延迟**：并行复制 + 强制读主库 + 拆分大事务
- **MGR**：Paxos 协议 + 自动故障转移 + 脑裂保护。InnoDB Cluster = 官方完整 HA 方案
- **延迟复制**：误操作恢复的"后悔窗口"
- **ProxySQL**：高性能读写分离中间件 + SQL 路由 + 连接池
- **分库分表**：hash 取模 / range / 一致性哈希。引入跨库 JOIN、分布式事务、全局 ID 问题
- **原则**：先优化再分库（SQL → 索引 → 缓存 → 读写分离 → 最后分库分表）

---

<a id="ch9"></a>

# 九、SQL 基础与表设计速查

## 一、背景（为什么需要）

SQL 基本功和表设计规范是面试的底层考察点。这些知识点零散但高频——面试官通过基础题快速判断你是否真正写过 SQL。

## 二、核心概念

### 数据库三大范式

| 范式 | 要求 | 反例 | 正确做法 |
|------|------|------|---------|
| **1NF** | 每列原子性，不可再分 | 地址字段存"北京-朝阳-望京" | 拆为省/市/区三列 |
| **2NF** | 非主键列完全依赖主键（消除部分依赖） | 联合主键(id, course)，但学生姓名只依赖 id | 拆表 |
| **3NF** | 非主键列不传递依赖主键 | 订单表有 user_id 和 user_name（user_name 依赖 user_id） | user_name 只存在用户表 |

> 实际开发不会严格遵守三范式——适当反范式（冗余字段）可减少 JOIN，提升查询性能。

**常见反范式做法**：
- 用户表冗余 `order_count`（订单数），避免每次 COUNT JOIN 订单表
- 订单表冗余 `user_name`、`product_title`，避免 JOIN 用户表和商品表
- 评论表冗余 `like_count`，避免实时 COUNT 点赞表
- 原则：读多写少的字段冗余是划算的，高频更新字段不宜冗余

### 数据类型要点

| 类型 | 关键点 |
|------|--------|
| **整型** | TINYINT(1B) / SMALLINT(2B) / INT(4B) / BIGINT(8B)。`INT(11)` 的 11 只是显示宽度，不影响存储和范围。MySQL 8 已废弃显示宽度 |
| **CHAR vs VARCHAR** | CHAR 定长（不足补空格到 n 字符），读写快但浪费空间。VARCHAR 变长（额外 1-2B 存长度），省空间但读写略慢。`VARCHAR(n)` 的 n 代表**字符数**，不是字节数，utf8mb4 下中文字符占 3-4 字节。⚠️ MySQL 5.0.3 之前 n 是**字节数**（历史陷阱，老面试题可能问到） |
| **DECIMAL vs FLOAT/DOUBLE** | 金额必须用 DECIMAL 定点数精确存储。FLOAT/DOUBLE 是浮点近似值，0.1+0.2≠0.3。`DECIMAL(10,2)` = 共 10 位，小数 2 位，最大 99999999.99 |
| **DATETIME vs TIMESTAMP** | DATETIME 不受时区影响（推荐业务时间），TIMESTAMP 受时区影响（存时间戳转换值），范围更小 |
| **TEXT** | 分 4 级：TINYTEXT(255B) / TEXT(64KB) / MEDIUMTEXT(16MB) / LONGTEXT(4GB)。不能有默认值，不能完整做索引（需前缀索引），数据存在溢出页 |
| **JSON** | MySQL 5.7+ 支持。灵活但不滥用——高频查询字段应独立成列并建索引 |
| **IP 地址** | 推荐 `INT UNSIGNED` 存储（4B），用 `INET_ATON()` / `INET_NTOA()` 转换。比 VARCHAR(15) 省空间、索引效率高、支持范围查询。IPv6 用 `VARBINARY(16)` |
| **布尔/状态** | TINYINT 即可，MySQL 中 BOOL/BOOLEAN 本质是 TINYINT(1) |

### NULL 处理要点

- `NULL = NULL` → `NULL`（不是 TRUE！），判断 NULL 必须用 `IS NULL` / `IS NOT NULL`
- `NULL` 参与任何算术运算结果都是 `NULL`（如 `100 + NULL = NULL`）
- 聚合函数（COUNT/SUM/AVG）自动忽略 NULL 值
- `COUNT(*)` 统计所有行，`COUNT(column)` 不统计 NULL
- `NOT IN` 子查询含 NULL → 全部返回空：`WHERE id NOT IN (1, 2, NULL)` → 等价于 `WHERE id <> 1 AND id <> 2 AND id <> NULL` → `id <> NULL` 永远是 NULL → WHERE 永远是 FALSE → 结果为空

> 建表建议：字段尽量 NOT NULL + 默认值。NULL 增加存储开销（1B 标记位）、索引复杂度和判断陷阱。

### DELETE / TRUNCATE / DROP

| 对比 | DELETE | TRUNCATE | DROP |
|------|--------|----------|------|
| 类型 | DML | DDL | DDL |
| 删除范围 | 可带 WHERE | 清空整表 | 删表结构+数据 |
| 自增值 | 通常保留 | 重置 | 表不存在 |
| 可回滚 | InnoDB 事务中可 | 通常不可 | 通常不可 |
| 触发器 | 可能触发 DELETE 触发器 | 不触发 | 不触发 |
| 性能 | 逐行删（慢） | 整表释放（快） | 直接删文件 |

### 避免重复插入数据

```sql
-- 方法1：冲突时忽略（不报错）
INSERT IGNORE INTO user(id, name) VALUES(1, 'Tom');

-- 方法2：冲突时更新（最推荐，语义清晰）
INSERT INTO user(id, name) VALUES(1, 'Tom')
ON DUPLICATE KEY UPDATE name = VALUES(name);

-- 方法3：冲突时先删后插（自增ID会变，不推荐）
REPLACE INTO user(id, name) VALUES(1, 'Tom');
```

> 前提：表上有主键或唯一索引约束。推荐 `ON DUPLICATE KEY UPDATE`。

### JOIN 类型与多表查询

| JOIN | 结果 |
|------|------|
| INNER JOIN | 两表交集 |
| LEFT JOIN | 左表全保留，右表无匹配补 NULL |
| RIGHT JOIN | 右表全保留，左表无匹配补 NULL（不如 LEFT JOIN 常用，调整表顺序可改写） |
| CROSS JOIN | 笛卡尔积（无连接条件） |

**笛卡尔积**：两张表所有行两两组合。100 行 × 10 行 = 1000 行。成因：没写连接条件或条件无效。避免：写正确的 JOIN ON。

**连接 N 张表至少需要 N-1 个有效连接条件**。

**自连接**：同一张表自己连自己，常用于层级关系。
```sql
SELECT e.name AS employee, m.name AS manager
FROM employee e LEFT JOIN employee m ON e.manager_id = m.id;
```

**非等值连接**：连接条件用范围而非等号。
```sql
SELECT e.name, e.salary, g.grade
FROM employee e JOIN salary_grade g
ON e.salary BETWEEN g.lowest_salary AND g.highest_salary;
```

**SQL92 vs SQL99 写法**：推荐 SQL99（`JOIN ... ON`），连接条件和过滤条件分离，可读性更好，不容易漏写条件导致笛卡尔积。

**ON vs WHERE（LEFT JOIN 中差异大）**：右表过滤条件放 ON → 不影响左表行保留；放 WHERE → 右表 NULL 行被过滤 → LEFT JOIN 退化为 INNER JOIN。内连接中 ON 和 WHERE 效果接近。

### UNION vs UNION ALL

| 对比 | UNION | UNION ALL |
|------|-------|-----------|
| 去重 | 去重（自动 DISTINCT） | 不去重 |
| 性能 | 需要额外排序去重 | 直接拼接，更快 |
| 使用场景 | 需要去重的场景 | 明确无重复时（推荐默认用 ALL） |

```sql
-- 如果明确两个结果集不重复，用 UNION ALL 避免不必要的排序
SELECT id, name FROM active_users
UNION ALL
SELECT id, name FROM archived_users;
```

> 面试重点：默认用 UNION ALL 而非 UNION，除非确实需要去重。UNION 的隐含 DISTINCT 会触发 filesort + temporary，性能差距可能几十倍。

### 子查询分类

| 类型 | 说明 | 操作符 |
|------|------|--------|
| **单行子查询** | 返回一行一列 | = / > / < / <> |
| **多行子查询** | 返回多行 | IN / ANY / ALL / SOME |
| **相关子查询** | 依赖外层查询的列 | EXISTS / 外层每行执行一次 |

```sql
-- ANY：大于子查询中任意一个（等价于大于最小值）
WHERE salary > ANY (SELECT salary FROM emp WHERE dept=10)

-- ALL：大于子查询中所有（等价于大于最大值）
WHERE salary > ALL (SELECT salary FROM emp WHERE dept=10)
```

### IN vs EXISTS

| 对比 | IN | EXISTS |
|------|-----|--------|
| 执行方式 | 先执行子查询得结果集，外层逐行匹配 | 外层每行执行子查询，遇匹配即返回 |
| 适合 | 子查询结果集小 | 外层结果小、子查询表大 |
| NULL 安全 | 列表含 NULL → `UNKNOWN`，NOT IN 危险 | 不受 NULL 影响 |

### SQL 常用函数

| 分类 | 常用函数 |
|------|---------|
| **字符串** | CONCAT、SUBSTRING、LENGTH（字节数）/ CHAR_LENGTH（字符数）、TRIM、UPPER/LOWER、REPLACE、LIKE |
| **数值** | ABS、CEIL/FLOOR、ROUND（四舍五入）/ TRUNCATE（直接截断）、MOD、RAND |
| **日期时间** | NOW()、DATE_FORMAT、DATEDIFF、DATE_ADD/DATE_SUB、YEAR/MONTH/DAY |
| **聚合** | COUNT、SUM、AVG、MAX/MIN、GROUP_CONCAT |
| **条件** | IF(expr, t, f)、IFNULL(val, default)、CASE WHEN THEN END |
| **类型转换** | CAST、CONVERT |

> 在 WHERE 中对索引列使用函数会导致索引失效。如 `WHERE YEAR(create_time)=2024` → 改为范围查询。

### COUNT(*) vs COUNT(1) vs COUNT(column)

- `COUNT(*)`：统计所有行，包括 NULL
- `COUNT(1)`：效果接近 COUNT(*)，MySQL 优化器同等对待
- `COUNT(column)`：只统计 column 不为 NULL 的行数

> InnoDB 的 COUNT(*) 需要扫描索引（MVCC 下不同事务看到行数不同），不像 MyISAM 用 O(1) 取全局行数。优化：用更小的二级索引扫描，或业务侧维护计数表。

### 建表规范

```
1. 显式主键（BIGINT AUTO_INCREMENT 或雪花 ID）
2. 显式 InnoDB 引擎 + utf8mb4 字符集
3. 字段尽量 NOT NULL + 默认值（避免 NULL 语义复杂、索引统计简单）
4. 金额用 DECIMAL
5. 时间字段 create_time / update_time（DEFAULT CURRENT_TIMESTAMP ON UPDATE）
6. 字段加 COMMENT
7. 大字段拆扩展表（如 article ↔ article_content）
8. 索引不超过 5-6 个
```

### 主键 vs 唯一索引

| 对比 | 主键 | 唯一索引 |
|------|------|---------|
| 是否可为 NULL | 不允许 | 允许（多个 NULL） |
| 表中数量 | 一个 | 多个 |
| InnoDB 存储 | 聚簇索引（叶子存完整行数据） | 二级索引（叶子存主键） |
| 本质 | 约束 + 索引 | 索引 |

### 外键约束

外键保证引用完整性：子表外键值必须在父表主键中存在。但互联网高并发系统**通常不使用物理外键**：增加写入检查成本、锁竞争扩大、表耦合强不利于分库分表。改为**应用层维护逻辑外键**。

## 三、底层原理

### 为什么自增 ID 比 UUID 快？

UUID 随机无序 → 插入 B+ 树中间位置 → 频繁页分裂 → 随机 I/O + 碎片严重。自增 ID 始终追加到 B+ 树末尾 → 顺序写 → 几乎不页分裂 → 高性能。

分布式场景用**雪花算法（Snowflake）**：64 位整数（趋势递增），兼顾全局唯一和顺序插入性能。

### 性别/低区分度字段建索引为什么不好？

索引选择性 = 不同值数 / 总行数。性别只有 2 个值 → 选择性 ≈ 0 → 走索引仍需大量回表 → 优化器判断全表扫描更快。例外：数据严重倾斜（如 status=0 仅 0.1%），查询该值时索引有效；或放入联合索引配合高区分度列使用。

### 视图、存储过程、函数、触发器

| 对象 | 作用 | 互联网使用建议 |
|------|------|-------------|
| **视图** | 虚拟表，保存 SELECT 逻辑不存数据 | 简化复杂查询 + 权限控制，但非性能优化工具 |
| **存储过程** | 预编译 SQL 逻辑封装 | 不推荐大量使用：移植性差、调试困难、不利于版本管理 |
| **存储函数** | 必须有返回值，SELECT 中调用 | 轻量计算可用，复杂业务放应用层 |
| **触发器** | INSERT/UPDATE/DELETE 自动触发 | 适合简单审计日志，不适合复杂业务（隐式执行难排查） |

**视图能否更新？** 简单视图（单表、无聚合）可以。含 GROUP BY、DISTINCT、JOIN、子查询、聚合函数的视图通常不可更新。

### MySQL 变量

- **系统变量**：`@@autocommit`、`SHOW VARIABLES`。GLOBAL（全局）/ SESSION（当前会话）
- **用户变量**：`SET @num = 10; SELECT @num;`，当前连接有效
- **局部变量**：存储过程/函数内 `DECLARE total INT DEFAULT 0;`

## 四、关键设计

### 为什么互联网项目不用物理外键？

1. 外键每次写入需检查父表 → 降低写入性能
2. 外键操作可能扩大锁范围 → 锁竞争
3. 表之间耦合增强 → 分库分表时不可用、schema 变更困难
4. 高并发下应用层保证一致性更灵活（补偿、重试、降级）

## 五、面试高频问题

**Q1：数据库三大范式是什么？实际开发中是否严格遵守？**

1NF：每列原子不可再分。2NF：非主键列完全依赖主键（消除部分依赖）。3NF：非主键列不能传递依赖主键。实际开发中为了性能常反范式——适当冗余减少 JOIN。

**Q2：CHAR 和 VARCHAR 区别？VARCHAR(n) 的 n 是字符还是字节？**

CHAR 定长（不足补空格），VARCHAR 变长（额外 1-2B 存长度）。`VARCHAR(n)` 的 n 代表**字符数**，utf8mb4 下中文字符占 3-4 字节。

**Q3：INT(1) 和 INT(10) 有什么区别？**

没有本质区别。存储范围完全相同（4 字节），括号数字只是显示宽度（配合 ZEROFILL 才有意义）。MySQL 8.0.17 后已废弃整数显示宽度。

**Q4：为什么金额用 DECIMAL 不用 FLOAT/DOUBLE？**

FLOAT/DOUBLE 用二进制近似小数 → 精度误差。DECIMAL 定点数精确存储。涉及金额/余额/价格必须用 DECIMAL。

**Q5：COUNT(*) 和 COUNT(column) 有什么区别？**

COUNT(*) 统计所有行（含 NULL），COUNT(column) 不统计 NULL。COUNT(1) 与 COUNT(*) 效果相同（MySQL 优化器同等处理）。

**Q6：主键和唯一索引有什么区别？**

主键非空 + 唯一，每表仅一个，InnoDB 中是聚簇索引。唯一索引允许 NULL（多个），每表多个，是二级索引。

**Q7：为什么不用物理外键？**

增加写入检查 + 锁竞争 + 表耦合强 + 不利于分库分表。互联网系统在应用层维护逻辑外键。

**Q8：自增 ID vs UUID 怎么选？**

自增 ID 顺序插入（无页分裂）、存储小（8B vs 36B）、查询快。UUID 全局唯一但随机插入导致页分裂。分布式推荐雪花算法：全局唯一 + 趋势递增。

**Q9：如何避免重复插入数据？**

`INSERT IGNORE`（忽略冲突）、`ON DUPLICATE KEY UPDATE`（冲突时更新，最推荐）、`REPLACE INTO`（先删后插，自增 ID 会变）。前提是有主键或唯一索引。

**Q10：IP 地址怎么存？**

用 `INT UNSIGNED`（4B）+ `INET_ATON()` / `INET_NTOA()` 转换，比 VARCHAR(15) 省 4 倍空间、索引效率更高、支持网段范围查询。

**Q11：什么是笛卡尔积？怎么避免？**

两张表所有行两两组合。成因：没写或写错连接条件。避免：写正确的 JOIN ON。多表连接至少需要 N-1 个有效连接条件。

**Q12：视图能不能更新？**

简单视图（单表、无聚合/去重/JOIN）可以。含 GROUP BY、DISTINCT、JOIN、子查询的视图不可更新。

**Q13：UNION 和 UNION ALL 的区别？**

UNION 自动去重（隐含 DISTINCT + filesort + temporary），UNION ALL 直接拼接不去重，性能差距可能几十倍。明确无重复时默认用 UNION ALL。

**Q14：NOT IN 有什么陷阱？**

子查询结果含 NULL → `WHERE id NOT IN (1, 2, NULL)` → 结果永远为空。原因：`id <> NULL` 永远返回 NULL（不是 TRUE/FALSE），WHERE 条件永远不满足。推荐用 `NOT EXISTS` 或 `LEFT JOIN ... IS NULL` 替代。

## 六、总结（速记版）

- **三大范式**：1NF 原子性 → 2NF 消除部分依赖 → 3NF 消除传递依赖。实际常反范式
- **DELETE**（DML 可回滚）/ **TRUNCATE**（DDL 清表快）/ **DROP**（删表）
- **CHAR** 定长 / **VARCHAR** 变长（n=字符数）/ 金额用 **DECIMAL**
- **IN** 适合子查询结果集小 / **EXISTS** 适合外层小子查询大 / NOT IN 怕 NULL
- **UNION ALL** 不去重（快）/ **UNION** 自动去重（慢，触发 filesort+temporary）
- **NULL 判断**用 IS NULL / NOT IN 含 NULL → 结果永远为空
- **物理外键**互联网基本不用 → 应用层维护逻辑外键
- **自增 ID** 顺序插入无页分裂 / **UUID** 随机导致页分裂 / 分布式用雪花算法
- **COUNT(*)** 统计所有行 / **COUNT(col)** 不统计 NULL / InnoDB 需扫描索引
- **视图**是虚拟表 / **存储过程/触发器**互联网不推荐滥用 / **CHECK** MySQL 8.0.16+ 生效
- **反范式**：读多写少的冗余字段提升性能，高频更新字段不宜冗余

---

<a id="ch10"></a>

# 十、MySQL 8 新特性与高级特性

## 一、背景（为什么需要）

MySQL 8 是重大版本升级，面试中了解新特性能体现广度和跟进能力。

## 二、核心特性

| 特性 | 说明 |
|------|------|
| **默认 utf8mb4** | 真正的 UTF-8 支持（emoji 等 4 字节字符） |
| **移除查询缓存** | 高并发下命中率低、维护成本高 |
| **窗口函数** | ROW_NUMBER / RANK / DENSE_RANK / LAG / LEAD |
| **CTE（公用表表达式）** | WITH 定义临时结果集，提高可读性，支持递归 |
| **降序索引** | 真正支持 `CREATE INDEX ... DESC` |
| **隐藏索引** | ALTER INDEX ... INVISIBLE，安全测试删除影响 |
| **原子 DDL** | CREATE/ALTER/DROP 原子化，不会出现中间状态 |
| **CHECK 约束生效** | MySQL 8.0.16+ CHECK 真正起作用 |
| **JSON 增强** | 更完善的 JSON 函数和索引支持 |
| **TempTable 引擎** | 内部临时表默认用 TempTable（内存），替代 MEMORY 引擎 |

### 窗口函数 vs GROUP BY

GROUP BY 将多行聚合为一行；窗口函数**不减少行数**，在每行上附加分组计算结果：

```sql
SELECT dept, name, salary,
       RANK() OVER(PARTITION BY dept ORDER BY salary DESC) AS rank
FROM employee;
```

### 内部临时表引擎变更（TempTable vs MEMORY）

MySQL 8.0 将内部临时表引擎从 MEMORY 改为 **TempTable**：

| 引擎 | 存储 | 限制 |
|------|------|------|
| MEMORY（旧） | 内存，不支持 TEXT/BLOB → 超过限制转磁盘 MyISAM | 固定行格式，浪费空间 |
| **TempTable（新）** | 内存（变长存储），不支持 TEXT/BLOB → 转磁盘 InnoDB | 节省内存、磁盘临时表仍支持事务 |

> 影响：`GROUP BY` / `DISTINCT` / `UNION` 产生的内部临时表更省内存，且磁盘溢出到 InnoDB（而非 MyISAM），性能更可预测。

### 隐藏索引的价值

不是直接 DROP 索引，而是先设为 INVISIBLE 观察性能影响，确认无问题后再真正删除。降低索引变更的风险。

## 三、面试高频问题

**Q1：MySQL 8 有哪些重要新特性？**

默认 utf8mb4、移除查询缓存、窗口函数 + CTE、降序索引、函数索引、隐藏索引、原子 DDL、CHECK 约束生效、TempTable 引擎、Clone 插件。

**Q2：窗口函数和 GROUP BY 的区别？**

GROUP BY 聚合多行为一行；窗口函数不减少行数，在每行上附加统计结果。适合排名、累计、移动平均等场景。

## 四、总结（速记版）

- **窗口函数** = 不聚合行，每行附加统计
- **CTE（WITH）** = 替代复杂子查询
- **隐藏索引** = 安全删除索引的方法
- **降序索引 + 函数索引** = 8.0 索引两大增强
- **TempTable** = 内部临时表新引擎，省内存 + 磁盘溢出到 InnoDB
- **原子 DDL** = DDL 要么全成功要么全失败

---

<a id="ch11"></a>

# 十一、MySQL 性能参数调优 ★★★★★

> 面试中被问"你们 MySQL 怎么调优的？"不只回答 EXPLAIN+索引，核心参数的调优是区分初级和高级的分水岭。

## 关键参数分类

### InnoDB Buffer Pool

```bash
innodb_buffer_pool_size = 物理内存的 60-80%  # 最最重要的参数！
# 示例：32G 内存服务器 → 设 24G
innodb_buffer_pool_instances = 8             # 多实例减少锁竞争（> 1G 时建议 4-8）
```

**为什么最重要**：Buffer Pool 缓存数据页 + 索引页。命中率直接决定磁盘 I/O 次数。`SHOW ENGINE INNODB STATUS` 查看 `Buffer pool hit rate`（目标 99%+）。

### Redo Log

```bash
# MySQL 8.0.30+ 推荐（动态调整，无需重启）
innodb_redo_log_capacity = 8G          # redo log 总容量（替代 innodb_log_file_size）

# MySQL 8.0.29 及更早版本
innodb_log_file_size = 2G              # redo log 文件大小（默认 48M 太小！）
innodb_log_files_in_group = 3          # redo log 文件数量（默认 2）
innodb_flush_log_at_trx_commit = 1     # 1=最安全, 2=高性能(丢1秒), 0=最快
```

**为什么 48M 太小**：redo log 空间小 → checkpoint 频繁 → 脏页频繁刷盘 → 性能下降。8.0.30+ 用 `innodb_redo_log_capacity`（动态调整），建议 8G+ 给更多缓冲空间。

### Binlog

```bash
sync_binlog = 1                  # 1=每次事务 fsync, N=每N次, 0=OS决定
binlog_format = ROW              # 必须 ROW（安全）
binlog_expire_logs_seconds = 604800  # binlog 保留 7 天（8.0+，替代 expire_logs_days）
max_binlog_size = 1G             # 单个 binlog 文件大小
```

**`sync_binlog=1` + `innodb_flush_log_at_trx_commit=1` = 双1配置，金融级安全但性能最低。高并发场景考虑 `sync_binlog=100` + 半同步复制。**

### 连接与线程

```bash
max_connections = 500-1000           # 最大连接数
thread_cache_size = 100              # 线程缓存（减少线程创建/销毁开销）
innodb_thread_concurrency = 0        # InnoDB 内部并发线程数（0=不限制）
```

### I/O 相关

```bash
innodb_io_capacity = 2000            # SSD → 2000-5000, HDD → 200
innodb_flush_neighbors = 0           # SSD → 0（关闭相邻页刷新）
innodb_read_io_threads = 8           # 读I/O线程数
innodb_write_io_threads = 8          # 写I/O线程数
innodb_adaptive_flushing = ON        # 自适应刷脏页（根据 redo log 生成速度）
```

### 排序与 JOIN 缓冲区

```bash
sort_buffer_size = 2M                # 排序缓冲区（每连接分配！设置过大会 OOM）
join_buffer_size = 2M                # JOIN 缓冲区（每连接分配，配合 Hash Join 提升性能）
tmp_table_size = 64M                 # 内存临时表上限
max_heap_table_size = 64M            # MEMORY 引擎表上限（与 tmp_table_size 配合）
```

> ⚠️ `sort_buffer_size` 和 `join_buffer_size` 是**每连接**分配的内存！连接数 × 缓冲区大小 = 实际内存使用。1000 连接 × 每条 2MB = 2GB，设太大容易 OOM。Hash Join 也使用 join_buffer_size。

### 表缓存与文件句柄

```bash
table_open_cache = 2000              # 打开表的缓存数
table_definition_cache = 2000        # 表定义缓存
open_files_limit = 65535             # 最大文件句柄数
```

### 事务与锁

```bash
innodb_lock_wait_timeout = 10        # 锁等待超时（秒，默认 50 太长了）
max_allowed_packet = 64M             # 最大包大小（批量插入/大数据字段）
transaction_isolation = REPEATABLE-READ  # 默认 RR，高并发可考虑 RC
innodb_deadlock_detect = ON          # 死锁检测（超高并发秒杀可设为 OFF）
max_execution_time = 0               # SELECT 最大执行时间（ms，0=不限，防止慢查询耗尽资源）
```

## 调优优先级（从高到低）

```
1. innodb_buffer_pool_size        # 影响最大，必须设
2. redo log 大小 + binlog 格式     # 数据安全底线
3. innodb_io_capacity             # SSD/HDD 场景差异大
4. innodb_flush_log_at_trx_commit / sync_binlog  # 安全 vs 性能权衡
5. max_connections + thread_cache # 连接管理
6. 其他参数按需调整
```

## 监控与诊断工具

```sql
-- Buffer Pool 命中率（目标 99%+）
SHOW ENGINE INNODB STATUS\G  -- 看 "Buffer pool hit rate"

-- History List 长度（undo 积压，正常 < 1000）
SHOW ENGINE INNODB STATUS\G  -- 看 "History list length"

-- 锁等待情况
SELECT * FROM performance_schema.data_locks;          -- 当前持有的锁
SELECT * FROM performance_schema.data_lock_waits;     -- 当前锁等待
SHOW ENGINE INNODB STATUS\G  -- 看 "LATEST DETECTED DEADLOCK"

-- 查看连接状态
SHOW PROCESSLIST;  -- 或 SELECT * FROM performance_schema.threads WHERE TYPE='FOREGROUND';

-- 慢查询统计
SELECT * FROM sys.statements_with_runtimes_in_95th_percentile;  -- 95分位慢查询
SELECT * FROM sys.schema_unused_indexes;                        -- 未使用的索引
```

**关键指标**：
| 指标 | 查询方式 | 健康值 |
|------|---------|--------|
| Buffer Pool 命中率 | `SHOW ENGINE INNODB STATUS` | > 99% |
| History list length | 同上 | < 1000 |
| 慢查询率 | 慢查询日志 | < 1% |
| 主从延迟 | `SHOW SLAVE STATUS` → `Seconds_Behind_Master` | 0 或 < 1s |
| 连接使用率 | `Threads_connected / max_connections` | < 70% |
| 脏页占比 | `SHOW GLOBAL STATUS LIKE '%dirty%'` | < 10% |

## 典型配置（32G 内存 SSD 服务器）

```bash
innodb_buffer_pool_size = 24G
innodb_buffer_pool_instances = 8
# 8.0.30+ 用 innodb_redo_log_capacity = 8G，旧版本用下面两行
innodb_log_file_size = 2G
innodb_log_files_in_group = 3
innodb_flush_log_at_trx_commit = 1
innodb_io_capacity = 3000
innodb_flush_neighbors = 0
sync_binlog = 1
binlog_format = ROW
binlog_expire_logs_seconds = 604800    # 7 天
max_connections = 800
thread_cache_size = 100
innodb_lock_wait_timeout = 10
```

---

<a id="ch12"></a>

# 十二、Golang + MySQL 实战指南

## 一、背景（为什么需要）

Go 是后端开发的主流语言之一，MySQL 是最常用的关系型数据库。掌握 Go 操作 MySQL 的最佳实践——从驱动选型到连接池配置、从事务处理到生产级坑点——是后端工程师的必备技能。

## 二、核心概念

### 驱动选型

| 库 | 定位 | 特点 |
|----|------|------|
| **[database/sql](https://pkg.go.dev/database/sql)** | Go 标准库接口 | 统一的数据库操作抽象，不依赖具体驱动 |
| **[go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)** | MySQL 驱动实现 | 最常用，实现了 `database/sql` 接口 |
| **[sqlx](https://github.com/jmoiron/sqlx)** | `database/sql` 增强 | 结构体映射、命名参数、`In()` 展开等 |
| **[GORM](https://gorm.io/)** | 全功能 ORM | 自动迁移、关联、Hook、链式查询 |
| **[sqlc](https://sqlc.dev/)** | 代码生成器 | 根据 SQL 生成类型安全的 Go 代码 |

```go
import (
    "database/sql"
    _ "github.com/go-sql-driver/mysql"  // 匿名导入，注册驱动
)
```

**推荐组合**：通用业务用 `sqlx`（兼顾性能和便利）、复杂多表关联用原生 SQL + `sqlx`、快速原型用 GORM、对性能极致要求用 `sqlc` 生成类型安全代码。

### database/sql 核心接口

```
sql.DB    → 连接池（goroutine 安全，全局单例）
sql.Tx    → 事务
sql.Rows  → 查询结果集（必须 Close）
sql.Row   → 单行结果
sql.Stmt  → 预处理语句
```

> 关键认知：`sql.DB` 是连接池，不是单个连接。创建一次、全局复用。不要频繁 `sql.Open`。

## 三、常用场景实战

### 场景一：初始化连接池

```go
package main

import (
    "database/sql"
    "log"
    "time"
    _ "github.com/go-sql-driver/mysql"
)

var db *sql.DB

func initDB() {
    dsn := "user:password@tcp(127.0.0.1:3306)/mydb?charset=utf8mb4&parseTime=true&loc=Local"
    // parseTime=true  → TIME/DATETIME 映射为 time.Time
    // loc=Local       → 使用本地时区
    // charset=utf8mb4 → 完整 UTF-8 支持（emoji）

    var err error
    db, err = sql.Open("mysql", dsn)
    if err != nil {
        log.Fatal(err)
    }

    // ★ 连接池配置（生产必设）
    db.SetMaxOpenConns(200)           // 最大连接数（根据 MySQL max_connections 决定）
    db.SetMaxIdleConns(50)            // 最大空闲连接数（保持一定量避免频繁建连）
    db.SetConnMaxLifetime(30 * time.Minute)  // 连接最大存活时间（小于 MySQL wait_timeout）
    db.SetConnMaxIdleTime(5 * time.Minute)   // 空闲连接最大存活时间

    // 验证连接
    if err = db.Ping(); err != nil {
        log.Fatal(err)
    }
    log.Println("MySQL connected")
}
```

**面试要点**：`SetConnMaxLifetime` 必须小于 MySQL 的 `wait_timeout`（默认 8h），否则 Go 拿到的连接可能已被 MySQL 关闭 → `driver: bad connection` 错误。

### 场景二：CRUD 基础操作

```go
// ─── CREATE ───
func createUser(name string, age int) (int64, error) {
    result, err := db.Exec(
        "INSERT INTO users(name, age) VALUES(?, ?)", name, age,
    )
    if err != nil {
        return 0, err
    }
    return result.LastInsertId()
}

// ─── READ（单行）───
func getUserByID(id int64) (*User, error) {
    var u User
    err := db.QueryRow(
        "SELECT id, name, age, created_at FROM users WHERE id = ?", id,
    ).Scan(&u.ID, &u.Name, &u.Age, &u.CreatedAt)
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, nil   // 没找到返回 nil，不返回 error
        }
        return nil, err
    }
    return &u, nil
}

// ─── READ（多行）───
func listUsers(minAge int) ([]User, error) {
    rows, err := db.Query(
        "SELECT id, name, age FROM users WHERE age >= ? ORDER BY id", minAge,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()  // ★ 必须 Close，否则连接不释放

    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Age); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    return users, rows.Err()  // ★ 检查遍历过程中的错误
}

// ─── UPDATE ───
func updateUserAge(id int64, age int) (int64, error) {
    result, err := db.Exec(
        "UPDATE users SET age = ? WHERE id = ?", age, id,
    )
    if err != nil {
        return 0, err
    }
    return result.RowsAffected()
}

// ─── DELETE ───
func deleteUser(id int64) (int64, error) {
    result, err := db.Exec("DELETE FROM users WHERE id = ?", id)
    if err != nil {
        return 0, err
    }
    return result.RowsAffected()
}
```

**三个必须记住的坑**：
1. `rows.Close()` 必须 `defer`——不 Close 连接泄漏，连接池打满
2. `rows.Err()` 必须检查——遍历中被中断（如网络断开）不会触发 `Scan` 报错
3. `QueryRow.Scan` 返回 `sql.ErrNoRows` 是正常的"没找到"，应返回 nil 而非 error

### 场景三：事务处理

```go
func transferMoney(fromID, toID int64, amount float64) error {
    tx, err := db.Begin()
    if err != nil {
        return fmt.Errorf("begin tx: %w", err)
    }
    defer func() {
        if err != nil {
            tx.Rollback()  // 出错回滚（忽略回滚本身的错误）
        }
    }()

    // 扣款
    result, err := tx.Exec(
        "UPDATE accounts SET balance = balance - ? WHERE id = ? AND balance >= ?",
        amount, fromID, amount,
    )
    if err != nil {
        return fmt.Errorf("debit: %w", err)
    }
    affected, _ := result.RowsAffected()
    if affected == 0 {
        return fmt.Errorf("insufficient balance")
    }

    // 加款
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?",
        amount, toID,
    )
    if err != nil {
        return fmt.Errorf("credit: %w", err)
    }

    return tx.Commit()
}
```

**事务最佳实践**：
- ✅ `defer tx.Rollback()` 确保异常时回滚（Commit 后 Rollback 是 no-op）
- ✅ 事务尽量短小——不在事务内做网络请求，不做复杂计算
- ✅ 乐观锁：`UPDATE ... SET version=version+1 WHERE version=?` → `RowsAffected()==0` 表示冲突
- ✅ 悲观锁：`SELECT ... FOR UPDATE` 放在事务第一条，尽早加锁减少死锁

### 场景四：SELECT FOR UPDATE（悲观锁）

```go
func reserveStock(productID int64, qty int) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()

    // ★ FOR UPDATE 必须在事务中，且尽早执行
    var currentStock int
    err = tx.QueryRow(
        "SELECT stock FROM products WHERE id = ? FOR UPDATE", productID,
    ).Scan(&currentStock)
    if err != nil {
        return fmt.Errorf("lock product: %w", err)
    }

    if currentStock < qty {
        return fmt.Errorf("stock insufficient: have %d, need %d", currentStock, qty)
    }

    _, err = tx.Exec(
        "UPDATE products SET stock = stock - ? WHERE id = ?", qty, productID,
    )
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

### 场景五：批量插入（高效）

```go
// 方案A：单条循环 INSERT（❌ 慢，每行一次网络往返）
for _, u := range users {
    db.Exec("INSERT INTO users(name) VALUES(?)", u.Name)
}

// 方案B：拼接多 VALUES（✅ 推荐，一次网络往返，批处理）
func batchInsert(users []User) error {
    if len(users) == 0 {
        return nil
    }

    // 构造：INSERT INTO users(name, age) VALUES (?,?),(?,?),(?,?)...
    placeholders := make([]string, len(users))
    args := make([]any, 0, len(users)*2)
    for i, u := range users {
        placeholders[i] = "(?,?)"
        args = append(args, u.Name, u.Age)
    }

    query := "INSERT INTO users(name, age) VALUES " + strings.Join(placeholders, ",")
    _, err := db.Exec(query, args...)
    return err
}

// 方案C：使用 sqlx.In()（✅ 更简洁）
// sqlx.In("INSERT INTO users(name) VALUES(?)", names) → 自动展开
```

> 每批建议 500-5000 行，太大增加 MySQL 解析成本和事务大小。

### 场景六：分页查询

```go
// 普通分页（OFFSET 方式，小数据量可用）
func listUsersPage(offset, limit int) ([]User, int, error) {
    var total int
    db.QueryRow("SELECT COUNT(*) FROM users").Scan(&total)

    rows, err := db.Query(
        "SELECT id, name, age FROM users ORDER BY id LIMIT ?, ?", offset, limit,
    )
    // ... 遍历 rows
    return users, total, nil
}

// 游标分页（推荐，避免深分页性能问题）
func listUsersCursor(lastID int64, limit int) ([]User, int64, error) {
    rows, err := db.Query(
        "SELECT id, name, age FROM users WHERE id > ? ORDER BY id LIMIT ?",
        lastID, limit,
    )
    if err != nil {
        return nil, 0, err
    }
    defer rows.Close()

    var users []User
    var maxID int64
    for rows.Next() {
        var u User
        rows.Scan(&u.ID, &u.Name, &u.Age)
        users = append(users, u)
        if u.ID > maxID {
            maxID = u.ID
        }
    }
    return users, maxID, rows.Err()
}
```

### 场景七：NULL 值处理

```go
// 数据库中 age 可为 NULL，Go 中没有对应类型

// 方案A：使用 sql.NullXXX（笨重但标准）
type User struct {
    ID   int64
    Name string
    Age  sql.NullInt64  // Valid + Int64
}
// 判断：if u.Age.Valid { fmt.Println(u.Age.Int64) }

// 方案B：使用指针（推荐，JSON 序列化友好）
type User struct {
    ID   int64
    Name string
    Age  *int  // nil = NULL
}

// 方案C：使用 COALESCE（数据库侧解决）
db.QueryRow("SELECT id, name, COALESCE(age, 0) FROM users WHERE id = ?", id)
```

### 场景八：预处理语句（Prepared Statement）

```go
// ★ 预编译 + 复用，防 SQL 注入 + 提升性能
stmt, err := db.Prepare("SELECT id, name FROM users WHERE age > ?")
if err != nil {
    return err
}
defer stmt.Close()

rows, err := stmt.Query(18)  // 第一次使用
rows, err = stmt.Query(25)   // 复用（MySQL 不需要重新解析）
```

> `database/sql` 内部也会缓存 `*sql.Stmt`，但显式 Prepare 更可控。注意 `db.Prepare` 占用一个连接，记得 Close。

### 场景九：sqlx 简化映射

```go
import "github.com/jmoiron/sqlx"

var db *sqlx.DB  // 包装 sql.DB

func listUsersSqlx(minAge int) ([]User, error) {
    var users []User
    err := db.Select(&users,       // ★ 自动映射结构体字段
        "SELECT id, name, age FROM users WHERE age >= ?", minAge)
    return users, err
}

func getUserSqlx(id int64) (*User, error) {
    var u User
    err := db.Get(&u,              // ★ 单行自动映射
        "SELECT id, name, age FROM users WHERE id = ?", id)
    return &u, err
}

// ★ Named 查询（推荐，SQL 和参数解耦）
func listByCondition(name string, age int) ([]User, error) {
    var users []User
    query := `SELECT id, name, age FROM users
               WHERE name = :name AND age >= :age`
    // 用结构体或 map 传参
    err := db.NamedQuery(&users, query, map[string]any{
        "name": name,
        "age":  age,
    })
    return users, err
}
```

### 场景十：读写分离

```go
// 简单方案：两个 sql.DB
var (
    masterDB *sql.DB  // 写库
    slaveDB  *sql.DB  // 读库
)

// 封装：业务层控制
func GetUserByID(id int64) (*User, error) {
    return queryUser(slaveDB, id)  // 读走从库
}

func UpdateUser(u *User) error {
    return updateUser(masterDB, u) // 写走主库
}

// 一致性要求高的场景：写后立即读走主库
func CreateAndReturn(user User) (*User, error) {
    id, err := insertUser(masterDB, user)  // 写主库
    if err != nil {
        return nil, err
    }
    return queryUser(masterDB, id)  // 立刻读也走主库，避免主从延迟
}
```

> 生产级方案：ProxySQL 中间件自动路由，应用层无需感知。

### 场景十一：MySQL 分布式锁

```go
// 基于 INSERT ... ON DUPLICATE KEY / GET_LOCK() 的轻量分布式锁

// 方案A：唯一键 + 过期时间（无锁争抢时用）
func acquireLock(db *sql.DB, lockKey string, ttl time.Duration) (bool, error) {
    // 先清理过期锁
    db.Exec("DELETE FROM distributed_locks WHERE lock_key = ? AND expire_at < NOW()", lockKey)

    _, err := db.Exec(
        `INSERT INTO distributed_locks(lock_key, expire_at)
         VALUES(?, NOW() + INTERVAL ? SECOND)`,
        lockKey, int(ttl.Seconds()),
    )
    if err != nil {
        // 唯一键冲突 → 锁已被持有
        if strings.Contains(err.Error(), "Duplicate entry") {
            return false, nil
        }
        return false, err
    }
    return true, nil
}

// 方案B：MySQL GET_LOCK()（连接级锁，连接断开自动释放）
func acquireLockNative(db *sql.DB, lockName string, timeout int) (bool, error) {
    var result int
    err := db.QueryRow("SELECT GET_LOCK(?, ?)", lockName, timeout).Scan(&result)
    if err != nil {
        return false, err
    }
    return result == 1, nil
}

func releaseLockNative(db *sql.DB, lockName string) error {
    _, err := db.Exec("SELECT RELEASE_LOCK(?)", lockName)
    return err
}
```

> Redis 分布式锁更常用（支持续期），MySQL 锁适用于已有 DB 不想引入 Redis 的简单场景。

## 四、关键设计

### 连接池配置原则

```go
// 最大连接数 = MySQL max_connections / 实例数 * 0.6（留余量给其他服务和运维连接）
db.SetMaxOpenConns(200)

// 空闲连接数 >= 高峰期并发数（避免频繁 TCP 三次握手开销）
db.SetMaxIdleConns(50)

// < MySQL wait_timeout（默认 8h），< 上层 LB/代理超时
db.SetConnMaxLifetime(30 * time.Minute)

// 空闲连接回收（清理长期不用的连接）
db.SetConnMaxIdleTime(5 * time.Minute)
```

### 错误处理模式

```go
// 判断"没找到"（QueryRow 返回 sql.ErrNoRows 不是异常）
if errors.Is(err, sql.ErrNoRows) {
    return nil, nil  // 返回 nil 表示"不存在"
}

// 判断"连接池耗尽"
if strings.Contains(err.Error(), "too many connections") {
    // 告警 + 降级
}

// 判断"MySQL 重启/断连"（driver: bad connection）
// driver 内部会自动重试，应用层一般无需处理
```

### 防 SQL 注入

```go
// ❌ 拼接 SQL → 注入风险
query := "SELECT * FROM users WHERE name = '" + name + "'"
db.Query(query)

// ✅ 占位符 → 驱动层转义
db.Query("SELECT * FROM users WHERE name = ?", name)

// ❌ 表名/列名不能用占位符（动态 ORDER BY 场景）
// ✅ 白名单校验
allowedColumns := map[string]bool{"id": true, "name": true, "age": true}
if !allowedColumns[orderBy] {
    return fmt.Errorf("invalid column: %s", orderBy)
}
query := fmt.Sprintf("SELECT * FROM users ORDER BY %s %s", orderBy, sortDir)
```

## 五、面试高频问题

**Q1：sql.DB 是连接还是连接池？可以并发使用吗？**

`sql.DB` 是连接池，不是单个连接。创建一次、全局复用。goroutine 安全，多个 goroutine 可以并发使用。底层自动管理连接的借用和归还。

**Q2：rows 为什么必须 Close？不 Close 会怎样？**

`rows.Close()` 释放数据库连接回连接池。不 Close → 连接泄漏 → 连接池耗光 → 后续请求全部阻塞等连接。`defer rows.Close()` 是强制习惯。

**Q3：Go 连接池怎么配？和 MySQL 参数有什么关系？**

`MaxOpenConns` < MySQL `max_connections` / 实例数；`ConnMaxLifetime` < MySQL `wait_timeout`（8h）；`MaxIdleConns` ≈ 高峰期并发数，避免频繁建连。空闲连接定期靠 `ConnMaxIdleTime` 回收。

**Q4：Go 里怎么处理 NULL 值？**

三种方式：`sql.NullInt64`/`sql.NullString`（标准库）、指针 `*int`/`*string`（nil = NULL，推荐）、`COALESCE`（数据库侧给默认值）。

**Q5：SQL 拼接和占位符的区别？什么场景必须拼接？**

占位符防 SQL 注入，值和 SQL 分离。表名/列名/ORDER BY 方向必须拼接——此时要用**白名单校验**防注入。永远不要把用户输入直接拼进 SQL。

**Q6：事务中做了网络请求会怎样？**

事务持有锁的时间被网络请求拉长 → 锁竞争加剧、可能引发死锁、undo log 膨胀、连接池占用。**事务内只做数据库操作，网络请求放到事务外。**

## 六、总结（速记版）

```go
// ─── 初始化三板斧 ───
sql.Open("mysql", dsn)                       // 1. 打开（全局唯一）
db.SetMaxOpenConns(200)                      // 2. 配连接池
db.Ping()                                    // 3. 验证连接

// ─── 查询铁三角 ───
rows, _ := db.Query(...); defer rows.Close() // 1. 查询
for rows.Next() { rows.Scan(...) }            // 2. 遍历
if err = rows.Err(); err != nil { ... }       // 3. 检查错误

// ─── 事务三原则 ───
defer tx.Rollback()   // 安全兜底
事务短小不跨网络       // 减少锁持有
FOR UPDATE 尽早执行   // 减少死锁概率

// ─── 常见坑 ───
rows 不 Close → 连接泄漏
QueryRow.Scan 没找到 → sql.ErrNoRows（不是异常）
ConnMaxLifetime > wait_timeout → bad connection
占位符不能用于表名/列名 → 必须拼接 + 白名单校验
```

---

<a id="ch13"></a>

# 十三、面试速查附录

## 项目场景题

### 场景一：秒杀系统

**问题**：高并发下超卖、数据库压力大。

**标准方案**：Redis 预扣库存（原子 DECR）→ MQ 异步下单削峰 → MySQL 最终落地。防超卖三层：Redis 扣减 + DB 唯一索引 + 业务校验。

### 场景二：MySQL + Redis 一致性

**方案**：Cache Aside（更新 DB → 删缓存）+ 延迟双删（删缓存 → 更新 DB → sleep → 再删缓存）兜底。大规模用 Canal 监听 binlog 异步删缓存（最终一致性，秒级延迟）。

### 场景三：大事务问题

**危害**：锁时间长、undo log 膨胀（阻止 purge 清理）、主从延迟增大、回滚代价高。解决：拆分事务、分批提交、事务内不做网络 I/O。监控 History list length 和锁等待时间。

### 场景四：大表加字段/索引不停机

**问题**：千万级表加字段，阻塞写操作数小时。

**方案**：加索引用 `ALGORITHM=INPLACE, LOCK=NONE`。8.0 加字段用 `ALGORITHM=INSTANT`（秒级）。改列类型等不支持 Online DDL 的操作 → pt-online-schema-change（触发器 + 影子表分批拷贝 → 原子切换）。最安全：先在从库执行 → 主从切换。

### 场景五：消息队列消费（MySQL 版）

**问题**：多个消费者从 MySQL 拉取待处理任务，不能重复消费、不能互相阻塞。

**方案**：`SELECT ... FOR UPDATE SKIP LOCKED` 跳过已被其他消费者锁定的行，高效处理不等待。

```sql
-- 消费者取 10 条待处理任务
START TRANSACTION;
SELECT * FROM tasks WHERE status = 0
ORDER BY create_time LIMIT 10
FOR UPDATE SKIP LOCKED;
-- 处理完成后标记 status = 1
COMMIT;
```

## 面试前 10 分钟速过

```
═══════════════════════════════════════════
架构与执行：
  连接层 → Server层(解析→优化→执行) → InnoDB引擎层
  SQL执行顺序：FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT

═══════════════════════════════════════════
存储引擎：
  InnoDB：事务+行锁+redo+MVCC → OLTP首选
  Buffer Pool：LRU变种(Young/Old) + innodb_old_blocks_time → 防全表扫描污染
  Change Buffer：非唯一二级索引写缓存
  Doublewrite：防partial write
  临时表引擎：8.0 TempTable 替代 MEMORY → 更省内存

═══════════════════════════════════════════
索引：
  B+树：内节点只存key → 树矮 → 叶子双向链表 → 范围查询高效
  聚簇索引 = 主键索引（存数据），二级索引（存主键）→ 回表
  联合索引 = 最左前缀 + 范围后失效
  覆盖索引 = 不回表（Using index）
  ICP = 索引层过滤（Using index condition）
  降序索引(8.0) + 函数索引(8.0.13+) → 8.0 索引两大增强

═══════════════════════════════════════════
事务与MVCC：
  ACID：undo(原子/回滚) + 锁+MVCC(隔离) + redo(持久)
  MVCC = trx_id + roll_pointer + undo log版本链 + ReadView
  RC：每次SELECT新ReadView；RR：首次SELECT生成 → 可重复读
  RR防幻读：快照读靠MVCC + 当前读靠Next-Key Lock

═══════════════════════════════════════════
锁：
  Next-Key Lock = Record Lock + Gap Lock → 防幻读
  行锁加在索引上 → 不走索引 → 退化为表锁
  死锁：InnoDB自动检测 + 回滚代价小的事务
  AUTO-INC锁：传统/连续(默认)/交叉 三种模式，分布式用雪花算法
  SKIP LOCKED（跳过被锁行）/ NOWAIT（遇锁报错）→ 8.0+高并发利器

═══════════════════════════════════════════
日志：
  redo log：物理日志 + 顺序写 + WAL → 崩溃恢复
  undo log：逻辑日志 + 版本链 → 回滚 + MVCC
  binlog：逻辑日志 + 追加写 → 主从复制
  2PC：prepare → binlog → commit → 保证一致性
  Group Commit：多事务合并 fsync → 双1配置下仍有高吞吐
  UPDATE流程：查数据 → 写undo → 改BufferPool → 写redo buffer → 2PC提交

═══════════════════════════════════════════
SQL优化：
  EXPLAIN：type(ALL要改) → key(NULL加索引) → rows → Extra(filesort/temporary)
  EXPLAIN ANALYZE(8.0.18+)：实际执行 + 真实耗时 → 验证估算偏差
  慢查询：开日志 → 找SQL → EXPLAIN → 定位 → 优化 → 验证
  深分页：游标分页(WHERE id > last_id) 或 延迟关联 或 ES接管
  Hash Join(8.0.18+)：等值JOIN无索引 O(N×M)→O(N+M)
  Semi-join(5.6+)：IN子查询自动转半连接
  Online DDL：INSTANT(秒级)/INPLACE(不阻塞)/COPY(阻塞)→大表pt-osc
  大表操作：分批提交，每批5000-10000行

═══════════════════════════════════════════
高可用：
  主从：binlog → relay log → 重放(IO Thread + SQL Thread)
  延迟：并行复制 + 强制读主库 + 拆分大事务 + 半同步
  MGR：Paxos协议 + 自动故障转移 + 脑裂保护 → InnoDB Cluster官方方案
  ProxySQL：高性能读写分离 + SQL路由 + 连接池
  延迟复制：误操作恢复的"后悔窗口"
  分库分表：hash/range/一致性hash → 跨库JOIN + 分布式事务 + 全局ID
  分片键：选高频查询列 + 均匀分布 + 避免跨分片
```

## 面试最终 1 分钟总结

```
MySQL InnoDB 的核心公式：

B+树索引（3-4层，千万数据）+ Buffer Pool（内存缓存）→ 读写性能保障
MVCC（快照读不加锁）+ Next-Key Lock（防幻读）→ 高并发保障
redo log（WAL顺序写）+ undo log（回滚+MVCC）+ binlog（主从复制）→ 数据可靠性
2PC（redo+binlog一致性）+ Group Commit（组提交）+ 主从复制/MGR + 分库分表 → 高可用

查询优化三步：EXPLAIN看计划 → 加索引/改写SQL → EXPLAIN ANALYZE验证
高并发三板斧：缓存（Redis抗读）+ MQ（削峰抗写）+ 分库分表（水平扩展）
现代MySQL 8.0：Hash Join + SKIP LOCKED/NOWAIT + EXPLAIN ANALYZE + INSTANT DDL + 降序/函数索引 + MGR + Clone
```

## 易错说法清单

| 错误 | 正确 |
|------|------|
| InnoDB 不支持事务 | InnoDB 是 MySQL 的事务引擎，MyISAM 才不支持 |
| B+ 树内节点也存数据 | 只有叶子节点存数据，内节点只存 key 导航 |
| `INT(11)` 表示 11 位数字 | 11 只是显示宽度，INT 永远是 4 字节 |
| `SELECT *` 性能一样 | 多取不必要的列增加 I/O、网络传输、浪费覆盖索引 |
| COUNT(1) 比 COUNT(*) 快 | InnoDB 中语义几乎相同，优化器处理一致 |
| MVCC 解决了所有并发问题 | MVCC 只解决读写并发，写写并发靠锁 |
| 索引越多越好 | 索引增加写入成本和存储空间，一般 5-6 个上限 |
| RR 级别会幻读 | InnoDB 通过 Next-Key Lock 在 RR 下解决了幻读 |
| redo log 和 binlog 可以互相替代 | 职责不同：redo 崩溃恢复，binlog 复制+恢复，缺一不可 |
| EXPLAIN 显示 1 行就一定快 | EXPLAIN 的 rows 是估算值，统计信息不准时可能偏差巨大。用 EXPLAIN ANALYZE 看真实行数 |
| Online DDL 完全不阻塞 | INPLACE 有短暂的锁窗口（毫秒~秒级），只有 8.0 的 INSTANT 完全不阻塞 |
| UNION 和 UNION ALL 差不多 | UNION 隐含去重（filesort+temporary），性能差距可能几十倍，明确无重复用 ALL |
| NOT IN 没问题 | 子查询含 NULL → 结果永远为空。用 NOT EXISTS 或 LEFT JOIN IS NULL 替代 |
| 双1配置一定很慢 | 高并发 + SSD 下 Group Commit 合并 fsync，双1配置性能仍然可接受 |
| 长事务只影响锁 | 长事务还阻止 purge 清理 undo log → 表空间膨胀 + 版本链变长 → 整体性能下降 |
| INT(1) 和 INT(10) 存储范围不同 | 括号数字只是显示宽度（8.0 已废弃），INT 永远是 4 字节，范围完全相同 |

---

> **版本说明**：本笔记以 MySQL 8.0 + InnoDB 为参考。MySQL 8.0 已移除查询缓存，CHECK 约束真正生效。
> **参考资源**：MySQL 官方文档、InnoDB 技术内幕、MySQL 实战 45 讲
