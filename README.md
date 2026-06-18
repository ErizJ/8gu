# 8gu

后端开发面试八股文笔记 —— 涵盖中高级后端工程师面试的高频考点，系统化整理，适合面试前快速复习与查漏补缺。

## 📚 内容索引

| 分类 | 文件 | 说明 |
|------|------|------|
| 🐹 Go 语言 | [go.md](go.md) | Slice 24B header、GMP 调度与 work stealing、Channel ring buffer 零拷贝、Map 渐进式 evacuate、接口 iface/eface 动态派发、三色 GC 混合写屏障、mcache→mcentral→mheap 三级分配 |
| 🔧 Go 框架 | [gin_gorm_grpc_gozero.md](gin_gorm_grpc_gozero.md) | Gin Radix Tree 路由与中间件洋葱模型、GORM Scope 链与 Preload 反 N+1、gRPC Protobuf Tag 编码与 HTTP/2 多路复用、go-zero API Gateway 断路器与过载保护 |
| 🗄️ MySQL | [mysql.md](mysql.md) | B+Tree 三层 21M 行、MVCC ReadView 可见性判断（RC vs RR）、Next-Key Lock 区间锁降级规则、redo log + binlog 两阶段提交、EXPLAIN key_len 精确字节计算、GTID 自动故障转移、Online DDL 三种算法选型 |
| 🔴 Redis | [redis.md](redis.md) | SDS 5 种 header 与 embstr 64B 对齐、渐进式 rehash（ht[0]→ht[1] + load-factor 动态阈值）、ZSet skiplist P=0.25 晋升 + dict 联合索引、惰性+定期过期双策略、RDB fork COW 与 AOF everysec rewrite、Sentinel SDOWN→ODOWN→Raft 选主、Cluster 16384 hash slot 与 MOVED/ASK 重定向 |
| 📨 Kafka | [kafka.md](kafka.md) | Partition 顺序追加与稀疏索引 O(1) 查找、ISR 动态维护 + HW 水位线、Producer RecordAccumulator batch 攒批、零拷贝 sendfile（DMA→PageCache→DMA）、Consumer Group CooperativeSticky 增量重平衡、幂等 PID+Seq 去重 + 事务 exactly-once、KRaft Raft 元数据替代 ZK |
| 🐳 Docker | [docker.md](docker.md) | Linux Namespaces 6 种隔离 + Cgroups v1/v2 资源限制、OverlayFS merged view + Copy-on-Write + Whiteout 删除标记、OCI Runtime/Image/Distribution 三层规范、Bridge veth pair + iptables MASQUERADE SNAT/DNAT、PID 1 信号转发陷阱与 tini/dumb-init、多阶段构建（build→scratch 缩至 7MB） |
| ☸️ Kubernetes | [k8s.md](k8s.md) | 声明式 API + Control Loop 持续 reconcile、List-Watch + Informer（Reflector→Store→Workqueue）、Pod 创建全链路（API Server→Scheduler→Kubelet CRI/CNI/CSI）、Startup/Liveness/Readiness 三探针、Deployment 滚动更新 maxSurge/maxUnavailable、StatefulSet 稳定网络标识与逆序拆除、APF 流控替代 max-requests-inflight |
| 🐧 Linux | [linux.md](linux.md) | 虚拟内存多级页表 + TLB、epoll 红黑树 + 就绪队列 ET/LT 模式、进程 task_struct + fork COW 只复制页表、零拷贝 sendfile DMA 全程无 CPU、awk 模式-动作 + 关联数组聚合、perf/strace 性能排错工具链 |
| 💻 操作系统 | [os.md](os.md) | 用户态/内核态 syscall 切换与 vDSO 无切换优化、task_struct PCB 与 zombie 僵尸进程、CFS vruntime 红黑树 vs MLFQ 多级队列、Futex CAS 用户态无竞争锁 + WAIT 系统调用、MESI 缓存一致性与 false sharing、RCU 读锁无关 + grace period 延迟释放 |
| 🌐 计算机网络 | [cn.md](cn.md) | TCP 三次握手 SYN Cookie 防 SYN Flood、四次挥手 TIME_WAIT 2MSL 与 CLOSE_WAIT 泄漏、拥塞控制 CUBIC cubic 函数 vs BBR 测带宽+min RTT、HTTP/2 HPACK 哈夫曼头部压缩 + Stream 多路复用、HTTP/3 QUIC 0-RTT 与无队头阻塞、TLS 1.3 1-RTT ECDHE 握手 |
| 🧠 LeetCode | [lc.md](lc.md) | 双指针（碰撞/快慢/分离）、滑动窗口摊销 O(n)、回溯决策树 + 剪枝去重模板、DP 状态转移与背包 LIS 变体、二分搜索左右边界不变式、Dijkstra + Union-Find 路径压缩、单调栈 Next Greater Element、bit XOR lowbit 状态压缩 |
| 🤖 AI Agent | [ai_agent.md](ai_agent.md) | ReAct Thought-Action-Observation 迭代循环、Tool Calling logits 层 token 概率与 tool_choice 控制、Memory 三层架构（短/长/工作记忆 + 向量检索 + 摘要压缩）、Multi-Agent 共享 Memory vs Message Passing 通信、Plan-Execute / Reflection / Tree of Thoughts 规划策略、Agent 全链路 trace 与断点回放调试 |
| 🔍 RAG | [rag.md](rag.md) | Embedding 对比学习 + Cosine 语义检索、ANN HNSW/IVF/PQ 索引原理与向量库选型、Chunking 递归分割 + Overlap + Parent-Child 长文档策略、Hybrid Search RRF 融合 BM25+向量、Rerank Bi-Encoder→Cross-Encoder 两阶段精排、Multi-Hop + Self-RAG + Agent RAG 自适应检索、工业级 RAG 架构设计 + 语义缓存 + 5 层容错降级、40 道面试题标准答案 + 追问 |
| 🔌 MCP | [mcp.md](mcp.md) | Host→Client→Server 三层架构 + JSON-RPC 2.0 协议、STDIO 与 Streamable HTTP SSE 双传输、Tool（可执行动作）vs Resource（只读数据 URI 订阅）、ReAct tool-calling 完整生命周期、能力协商 initialize 阶段 + Sampling 反向 LLM 调用 |

## 🎯 使用建议

- **系统复习**：按分类从上到下逐篇阅读，每篇包含背景 → 核心概念 → 面试应答
- **面试速查**：面试前 30 分钟快速浏览各篇摘要和面试问答题
- **查漏补缺**：针对薄弱环节精读对应章节，结合源码加深理解

## 📝 笔记风格

每篇笔记遵循统一结构：**为什么需要（背景）→ 是什么（核心概念）→ 怎么用（实践）→ 面试怎么答（应答模板）**，帮助从理解原理到面试输出的完整闭环。

---

持续更新中 🚀
