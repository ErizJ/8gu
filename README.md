# 8gu

后端开发面试八股文笔记 —— 涵盖中高级后端工程师面试的高频考点，系统化整理，适合面试前快速复习与查漏补缺。

## 📚 内容索引

| 分类 | 文件 | 说明 |
|------|------|------|
| 🐹 Go 语言 | [go.md](go.md) | 类型系统与数据结构（Slice/Map/Channel/Interface）、并发（Goroutine/GMP/Sync/Context）、内存管理（分配器/逃逸分析/GC）、标准库（net/http/reflect/Netpoller）、工程实践（Modules/测试/pprof）、并发 Bug 排查、面试高频 28 问、LeetCode Go 实现 |
| 🔧 Go 框架 | [gin_gorm_grpc_gozero.md](gin_gorm_grpc_gozero.md) | Gin（Radix Tree 路由/中间件洋葱模型）、GORM（Scope 链/Preload 反 N+1/事务）、gRPC（Protobuf 编码/HTTP2 多路复用/四种服务类型）、go-zero（API Gateway/RPC 脚手架/断路器/过载保护） |
| 🗄️ MySQL | [mysql.md](mysql.md) | 架构与存储引擎、索引（B+Tree/覆盖索引/最左前缀/key_len 计算）、事务与 MVCC（ReadView/RC vs RR）、锁（Record/Gap/Next-Key Lock）、日志（redo log/binlog 两阶段提交）、复制（GTID）、Online DDL、面试高频问题 |
| 🔴 Redis | [redis.md](redis.md) | 数据结构（SDS/skiplist/渐进式 rehash）、过期与淘汰策略（惰性+定期/LRU/LFU）、持久化（RDB COW/AOF rewrite）、高可用（Sentinel 故障转移/Cluster 16384 slot）、分布式锁（Redisson Watchdog/Redlock）、缓存问题（穿透/击穿/雪崩）、面试高频问题 |
| 📨 Kafka | [kafka.md](kafka.md) | 架构与存储（Partition 顺序 IO/稀疏索引/PageCache）、生产者（攒批/幂等/事务 exactly-once）、消费者（Rebalance 协议/ CooperativeSticky）、ISR 与 HW 水位线、零拷贝 sendfile、KRaft 共识、面试高频问题 |
| 🐳 Docker | [docker.md](docker.md) | 基础概念（镜像/容器/仓库）、底层原理（Namespace 隔离/Cgroups 限制/OverlayFS 联合文件系统）、镜像构建与优化（多阶段构建/BuildKit）、网络（Bridge/Host/Overlay）、存储（Volume/Bind Mount）、Dockerfile 最佳实践、安全（Seccomp/AppArmor/Rootless）、容器运行时与 K8s 关系、面试 28 问 |
| ☸️ Kubernetes | [k8s.md](k8s.md) | 架构与组件（API Server/Scheduler/Controller）、Pod 生命周期与探针、工作负载（Deployment 滚动更新/StatefulSet/DaemonSet/Job）、网络（Service/kube-proxy/Ingress/Gateway API）、存储（PV/PVC/StorageClass）、调度（亲和性/Taint/Priority）、RBAC 与安全、HPA 弹性伸缩、Helm/Operator、故障排查、面试 22 场景 |
| 🐧 Linux | [linux.md](linux.md) | 系统基础（目录结构/文件权限）、常用命令（文件管理/文本处理/搜索）、进程管理（ps/top/kill/Signal）、内存管理（虚拟内存/free/vmstat/OOM）、磁盘（df/du/iostat）、网络（IP/端口/DNS/tcpdump）、日志分析（grep/awk）、性能排查（strace/perf/sar）、Shell 脚本、IO 模型（epoll/零拷贝/Reactor）、高并发内核调优、10 大排错场景 |
| 💻 操作系统 | [os.md](os.md) | 内核基础（用户态/内核态/系统调用/中断）、进程线程协程（fork COW/僵尸进程/上下文切换）、调度算法、IPC（管道/共享内存/信号量/Socket）、同步机制（Mutex/SpinLock/CAS/Futex/MESI/RCU）、死锁（条件/预防/银行家算法）、虚拟内存（多级页表/TLB/缺页/页面置换）、内存分配（malloc/mmap/伙伴系统/Slab/OOM）、文件系统（Page Cache/硬链接软链接）、IO 模型（epoll/Reactor/Go Netpoll/GMP）、30 场景题 |
| 🌐 计算机网络 | [cn.md](cn.md) | 网络模型（OSI/TCP/IP）、TCP（三次握手/四次挥手/TIME_WAIT/可靠传输/流量控制/拥塞控制/Nagle）、UDP、HTTP（报文/状态码/缓存/版本演进/WebSocket）、HTTPS（TLS 握手/证书链）、DNS、IP 与 NAT、ARP、认证（Cookie/Session/JWT/OAuth2）、安全（SYN Flood/XSS/CSRF/CORS）、代理与 CDN、Socket 编程与 IO 模型（epoll/Reactor/Go netpoll/零拷贝）、高并发服务器设计、网络排错、面试 35 问 |
| 🧠 LeetCode | [lc.md](lc.md) | 按算法分类（数组/哈希/双指针/滑动窗口/栈/链表/二叉树/DFS/BFS/回溯/图论/二分/贪心/DP/单调栈/前缀和/堆/Trie/并查集/位运算）、Hot100 题号索引、算法选型决策树、面试速查表 |
| 🤖 AI Agent | [ai_agent.md](ai_agent.md) | LLM 基础（Transformer/Token/推理参数）、Prompt 工程（CoT/ReAct/Structured Output）、Tool Calling 与 Function Calling、RAG 全链路（Embedding/Chunk/向量库/Rerank/Hybrid Search）、MCP 协议与 A2A、Fine-tuning（LoRA/量化/vLLM）、Agent 架构（单 Agent/Multi-Agent/记忆系统/规划策略/工作流模式）、主流框架（LangChain/LangGraph/AutoGen/CrewAI）、安全（Prompt Injection/Guardrails）、评估与 Tracing、生产级架构、面试 28 问 |
| 🔍 RAG | [rag.md](rag.md) | 基础原理与五步流程、Embedding（对比学习/相似度/模型选型）、向量数据库（ANN HNSW/IVF/PQ 与 8 种方案对比）、Chunking 策略与 Prompt 构造、Pipeline 设计（离线 Ingestion + 在线 Query）、优化（检索/生成/系统延迟/成本/缓存）、进阶（Hybrid Search RRF/Reranker/Multi-Hop/Agent RAG）、工业级系统设计（架构/容错/并发/数据更新）、40 道面试题含追问 |
| 🔌 MCP | [mcp.md](mcp.md) | 架构（Host/Client/Server + JSON-RPC 2.0）、Transport（STDIO/Streamable HTTP/SSE）、Tool vs Resource 区别、ReAct tool-calling 生命周期、能力协商与 Sampling、安全最佳实践 |

## 🎯 使用建议

- **系统复习**：按分类从上到下逐篇阅读，每篇包含背景 → 核心概念 → 面试应答
- **面试速查**：面试前 30 分钟快速浏览各篇摘要和面试问答题
- **查漏补缺**：针对薄弱环节精读对应章节，结合源码加深理解

## 📝 笔记风格

每篇笔记遵循统一结构：**为什么需要（背景）→ 是什么（核心概念）→ 怎么用（实践）→ 面试怎么答（应答模板）**，帮助从理解原理到面试输出的完整闭环。

---

持续更新中 🚀
