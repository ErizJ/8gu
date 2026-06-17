# Kubernetes（K8s）面试知识体系（后端开发校招 / 云原生方向终极版）

> 适用于：
>
> - Golang 后端开发
> - Java 后端开发
> - 云原生开发工程师
> - SRE / 运维开发
> - 平台研发工程师
>
> 学习建议：
>
> - ★★★★★：高频必问å
> - ★★★★：常考
> - ★★★：了解即可
>
> 腾讯、阿里云、字节、美团、京东、网易、百度、携程、唯品会等公司近几年越来越喜欢考 K8s，尤其是使用过 Docker、微服务、Go 开发的同学。

---

# 第一章 Kubernetes 基础

## 1.1 什么是 Kubernetes ★★★★★

### Kubernetes 定义

Kubernetes（简称 K8s，K 和 s 之间有 8 个字母）是 Google 开源的容器编排平台，用于自动化部署、扩展和管理容器化应用。它起源于 Google 内部的 Borg 系统，于 2014 年开源，2015 年捐赠给 CNCF（云原生计算基金会），是 CNCF 的第一个毕业项目。

K8s 的核心设计思想是**声明式 API**和**控制循环（Control Loop）**：用户声明期望状态（期望运行多少个 Pod、使用什么镜像），K8s 的控制平面不断比较实际状态与期望状态，并自动调整使实际状态趋近期望状态。

### Kubernetes 解决什么问题

#### 1. 服务部署（Service Deployment）

- **传统方式痛点**：手动 SSH 到服务器、scp 上传二进制/代码、手动启动进程、配置环境变量 → 繁琐、易出错、无法版本化管理
- **K8s 解法**：通过声明式 YAML/JSON 描述应用部署规格（镜像、副本数、资源、环境变量、存储卷等），`kubectl apply -f deployment.yaml` 一键部署

#### 2. 服务扩缩容（Scaling）

- **传统方式痛点**：流量高峰需要手动加机器、改配置、重启 → 响应慢、人工成本高
- **K8s 解法**：
  - 手动扩缩容：`kubectl scale deployment/nginx --replicas=10`
  - 自动扩缩容：HPA（Horizontal Pod Autoscaler）根据 CPU/内存/自定义指标自动调整副本数

#### 3. 服务发现（Service Discovery）

- **传统方式痛点**：服务 IP 变化后需要手动更新调用方的配置文件或注册中心
- **K8s 解法**：内置 Service 资源 + CoreDNS，Pod 可以通过 Service 名称直接访问服务，无需关心后端 Pod IP 变化

#### 4. 故障恢复（Self-Healing）

- **传统方式痛点**：服务挂了需要人工重启；节点挂了需要人工迁移
- **K8s 解法**：
  - Pod 异常退出 → 自动重启（restartPolicy）
  - 节点宕机 → Pod 自动迁移到健康节点
  - 健康检查失败 → 自动摘除流量并重启

#### 5. 负载均衡（Load Balancing）

- **传统方式痛点**：需要额外搭建 Nginx/HAProxy，配置复杂
- **K8s 解法**：Service 自动提供负载均衡，kube-proxy 通过 iptables/IPVS 将流量分发到后端 Pod

#### 6. 滚动更新（Rolling Update）

- **传统方式痛点**：更新需要停服 → 全量替换 → 启动 → 验证 → 恢复流量，风险高、回滚慢
- **K8s 解法**：Deployment 原生支持滚动更新，逐步替换旧版本 Pod，maxSurge/maxUnavailable 控制更新节奏，`kubectl rollout undo` 一键回滚

### 面试题：什么是 K8s？

**标准回答**：

Kubernetes 是一个开源的容器编排平台，由 Google 基于 Borg 的经验开发并捐赠给 CNCF。它提供了一套完整的 API 来控制容器的部署、扩缩容、服务发现、负载均衡、滚动更新和故障恢复。核心架构是 Master-Worker 模式，Master 节点负责调度和管理，Worker 节点负责运行容器。K8s 通过声明式 API 和控制循环实现自动化运维，是目前云原生生态的核心基础设施。

**加分点**：提到 Borg 起源、声明式 API vs 命令式、控制循环（reconcile loop）、CNCF 背景。

### 🔬 深度解析：为什么 Kubernetes 使用声明式 API？

这是面试中的**高频追问点**——面试官会从"什么是声明式 API"一路追问到"为什么这样设计"。

**声明式 API vs 命令式 API**：

| 维度 | 命令式（Imperative） | 声明式（Declarative） |
|------|---------------------|----------------------|
| **思维方式** | "怎么做"（How） | "要什么"（What） |
| **示例** | `kubectl run nginx --image=nginx` | `kubectl apply -f deployment.yaml` |
| **幂等性** | 不保证（重复执行可能出错） | 天然幂等（重复 apply 结果一致） |
| **状态管理** | 用户自己管理状态 | 系统自动管理状态（控制器持续调和） |
| **并发安全** | 多个操作可能冲突 | 通过 etcd 的乐观锁（resourceVersion）解决冲突 |
| **回滚能力** | 需要手动记录历史操作 | 保留历史 ReplicaSet，一键回滚 |

**为什么声明式是更好的选择？**

1. **系统闭环（Closed-loop）**：用户声明期望状态 → 系统持续监控实际状态 → 自动弥合差异。不需要用户持续关注"现在怎么样了"。
2. **天然容错**：命令式操作失败后需要重试逻辑；声明式中控制器会持续 reconcile，任何中间失败都会在下一轮循环中被修正。
3. **基础设施即代码（IaC）**：声明式 YAML 可以存入 Git，实现版本控制、Code Review、CI/CD 集成（GitOps）。
4. **多控制器解耦**：Scheduler、Kubelet、各种 Controller 都独立工作，相互之间不需要知道对方的存在——它们都通过 API Server/etcd 作为唯一协调点。

**面试追问：声明式 API 的代价是什么？**

1. **调试困难**：你写了 YAML，但最终状态由多个控制器共同决定——出问题时需要排查"哪个控制器做了什么"。
2. **最终一致性**：不是立即生效，中间存在短暂的不一致窗口（如滚动更新中旧 Pod 还在接收流量）。
3. **复杂度**：简单的"启动一个容器"在 K8s 中需要经过 API Server → etcd → Scheduler → Kubelet → CRI 多个环节。

**一句话总结**：声明式 API 把"编排的复杂性"从用户转移到了平台，用户只需要说"我要什么"，平台负责"怎么做到"——这是 K8s 区别于传统运维工具的核心设计哲学。

### 面试题：为什么需要 K8s？

**标准回答**：

1. **容器化普及**：Docker 解决了应用打包和隔离的问题，但生产环境中需要管理成百上千个容器，手工操作不可行
2. **微服务架构**：微服务拆分为众多独立服务，每个服务需要独立部署、扩缩容、负载均衡
3. **运维自动化**：人工运维效率低、易出错，需要平台自动化处理部署、扩缩容、故障恢复
4. **基础设施可移植性**：K8s 屏蔽底层基础设施差异，应用可以在任何 K8s 集群上运行（公有云、私有云、混合云）

**加分点**：提到 Dev & Ops 分离 → GitOps 演进、12-Factor App、Immutable Infrastructure 理念。

---

## 1.2 Kubernetes 架构总览 ★★★★★

### 整体架构图

```
┌──────────────────────────────────────────────────────────┐
│                      Control Plane                        │
│  ┌──────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐  │
│  │ API      │ │ Scheduler │ │Controller│ │   etcd   │  │
│  │ Server   │ │           │ │ Manager  │ │          │  │
│  └──────────┘ └───────────┘ └──────────┘ └──────────┘  │
└──────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│                     Worker Node(s)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │ kubelet  │  │kube-proxy│  │ Container Runtime    │   │
│  │          │  │          │  │ (containerd/docker)  │   │
│  └──────────┘  └──────────┘  └──────────────────────┘   │
│                                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│  │  Pod A  │  │  Pod B  │  │  Pod C  │                 │
│  └─────────┘  └─────────┘  └─────────┘                 │
└──────────────────────────────────────────────────────────┘
```

### Master 节点（Control Plane 控制平面）

控制平面是 K8s 集群的大脑，负责集群的全局决策（调度、响应事件）：

| 组件 | 职责 |
|------|------|
| kube-apiserver | 集群统一入口，所有操作都经过它，RESTful API |
| etcd | 分布式 KV 存储，保存集群所有配置和状态数据 |
| kube-scheduler | 负责将 Pod 调度到合适的 Node 上 |
| kube-controller-manager | 运行各种控制器，确保集群状态与期望一致 |

### Worker 节点（Node 数据平面）

Worker 节点是真正运行业务负载的地方：

| 组件 | 职责 |
|------|------|
| kubelet | 节点代理，负责管理本节点上的 Pod 生命周期 |
| kube-proxy | 网络代理，维护节点上的网络规则，实现 Service 负载均衡 |
| Container Runtime | 容器运行时，负责拉取镜像、运行容器（containerd / CRI-O） |

### 控制平面 vs 数据平面

- **控制平面（Control Plane）**：负责"决策"— 调度、编排、管理集群状态
- **数据平面（Data Plane）**：负责"执行" — 运行容器、处理网络流量、存储数据

### 面试题：K8s 整体架构是什么？

**标准回答**：

K8s 采用 Master-Worker 架构（现在官方称为 Control Plane-Node 架构）。控制平面由 4 个核心组件组成：kube-apiserver 是集群统一入口，接收所有 API 请求并写入 etcd；etcd 是分布式 KV 存储，保存集群全量数据；kube-scheduler 负责将未调度的 Pod 绑定到合适的 Node；kube-controller-manager 运行控制器集群，持续进行 reconcile 循环。Worker 节点运行 kubelet（Pod 生命周期管理）、kube-proxy（网络规则维护）和 Container Runtime（容器运行）。

**加分点**：提到声明式 API、list-watch 机制、控制循环（Reconcile Loop）、组件间通过 API Server 解耦。

---

## 1.3 Kubernetes 核心组件 ★★★★★

### Control Plane 组件详解

#### kube-apiserver

**kube-apiserver** 是 K8s 集群的**唯一入口**，所有对内对外的操作都必须经过它：

- 提供 RESTful API（kubectl、Dashboard、SDK 都调用它）
- 认证（Authentication）、授权（Authorization）、准入控制（Admission Control）
- 数据写入 etcd，是 etcd 的唯一直接客户端
- 提供 **List-Watch** 机制：其他组件通过长连接 watch API Server 的资源变化，实现事件驱动

#### etcd

**etcd** 是 CoreOS 开发的分布式 KV 存储，K8s 用它存储所有集群数据：

- 所有资源对象（Pod、Service、Deployment、ConfigMap 等）都存储在 etcd 中
- 使用 **Raft** 共识算法保证数据一致性和高可用
- 通常部署 3/5 个节点（奇数个），容忍 (N-1)/2 个节点故障
- 支持 Watch 机制，API Server 通过 etcd 的 Watch 实现资源变更通知
- **etcd 是 K8s 集群的"单一事实来源（Source of Truth）"**

#### kube-scheduler

**kube-scheduler** 负责将新创建的、未绑定 Node 的 Pod 调度到合适的节点：

```
调度流程：Filter（过滤） → Score（打分） → Bind（绑定）
```

- **Filter（预选/过滤）**：排除不满足条件的节点（资源不足、端口冲突、NodeSelector 不匹配、Taint 不兼容等）
- **Score（优选/打分）**：对剩余节点打分（LeastRequestedPriority、BalancedResourceAllocation 等策略）
- **Bind（绑定）**：选择得分最高的节点，将 Pod 的 `spec.nodeName` 设置为该节点

#### kube-controller-manager

**kube-controller-manager** 运行一组控制器，每个控制器负责一种资源的**控制循环（Reconcile Loop）**：

```
for {
    实际状态 := 获取资源的实际状态()
    期望状态 := 获取资源的期望状态()
    if 实际状态 != 期望状态 {
        执行操作使实际状态趋向期望状态()
    }
}
```

核心控制器包括：

| 控制器 | 职责 |
|--------|------|
| Node Controller | 监控节点状态，处理节点故障 |
| ReplicaSet Controller | 确保 Pod 副本数符合期望 |
| Deployment Controller | 管理 Deployment 的滚动更新 |
| Service Controller | 为 Service 分配 ClusterIP（如果是云 LoadBalancer 还需要创建外部 LB） |
| EndpointSlice Controller | 维护 Service 与 Pod 的映射关系 |
| Job Controller | 管理 Job/CronJob 的执行 |

### Node 组件详解

#### kubelet

**kubelet** 是每个 Node 上的"工头"，负责管理本节点上 Pod 的生命周期：

- 通过 API Server 的 List-Watch 获取分配给本节点的 Pod 列表
- 调用 Container Runtime 拉取镜像、创建/启动/停止容器
- 执行 Pod 中定义的探针（Liveness、Readiness、Startup Probe）
- 向 API Server 上报 Node 和 Pod 的状态
- 管理 Volume 的挂载

#### kube-proxy

**kube-proxy** 是每个 Node 上的网络代理，实现 Service 的负载均衡：

- 监听 API Server 中 Service 和 EndpointSlice 的变化
- 在节点上创建网络规则（iptables 规则或 IPVS 规则）
- 将访问 Service ClusterIP 的流量转发到后端 Pod

**iptables 模式 vs IPVS 模式**：

| 维度 | iptables | IPVS |
|------|----------|------|
| 转发模型 | 规则链顺序匹配 | 哈希表直接查找 |
| 性能（大规模） | O(n) 线性下降 | O(1) 稳定 |
| 负载均衡算法 | 随机（概率性） | 多种（rr/lc/dh/sh） |
| 连接追踪 | iptables conntrack | IPVS conntrack |
| 适用规模 | < 1000 Service | > 1000 Service |

#### Container Runtime

**Container Runtime** 是真正运行容器的组件。K8s 通过 **CRI（Container Runtime Interface）** 抽象接口与各种运行时交互：

- **Docker**：最早的默认运行时（通过 dockershim 适配，1.24 后废弃）
- **containerd**：CNCF 毕业项目，目前 K8s 最常用的运行时（也是 Docker 的底层运行时）
- **CRI-O**：专为 K8s 设计的轻量级 CRI 实现

### 面试题：K8s 核心组件有哪些？

**标准回答**：

控制平面：kube-apiserver（集群入口，REST API，认证授权准入）、etcd（分布式存储，Raft 共识）、kube-scheduler（Pod 调度，Filter → Score → Bind）、kube-controller-manager（运行各类控制器，Reconcile 循环）。

Node 节点：kubelet（Pod 生命周期管理，探针检查，状态上报）、kube-proxy（网络规则维护，Service 负载均衡）、Container Runtime（拉镜像、运行容器，containerd/CRI-O）。

### 面试题：kubelet 的作用是什么？

**标准回答**：

kubelet 是运行在每个 Node 上的核心代理，主要职责：

1. **Pod 生命周期管理**：通过 API Server 获取分配给本节点的 Pod，调用 CRI 创建容器
2. **健康检查**：执行 Pod 中定义的 Liveness/Readiness/Startup Probe
3. **状态上报**：定期向 API Server 上报 Node 和 Pod 的状态
4. **资源管理**：管理 cgroup、预留系统资源、驱逐 Pod（资源不足时）
5. **Volume 管理**：挂载 Pod 需要的 Volume

### 面试题：kube-apiserver 的作用是什么？

**标准回答**：

kube-apiserver 是 K8s 集群的**唯一入口**和**中枢神经**：

1. **统一入口**：所有内部组件和外部用户只能通过 API Server 操作集群
2. **认证/授权/准入**：三层安全防线，确保只有合法的请求才能操作资源
3. **数据持久化**：是 etcd 的唯一客户端，所有资源对象经过它写入 etcd
4. **List-Watch 枢纽**：提供资源变更的 Watch 接口，其他组件通过长连接获取变更通知
5. **解耦各组件**：Scheduler、Controller、Kubelet 之间不直接通信，都通过 API Server

### 🔬 深度解析：List-Watch 机制 —— K8s 的事件驱动核心

List-Watch 是 K8s 所有组件**实时感知资源变化**的基础机制，也是面试中区分"用过 K8s"和"理解 K8s"的关键考点。

**什么是 List-Watch？**

```
组件启动时：
  List（全量同步）：向 API Server 发起 List 请求，获取某类资源的全量快照
        │
        ▼
  Watch（增量同步）：基于 List 返回的 resourceVersion，建立长连接
        持续接收资源的 ADD / MODIFY / DELETE 事件
        │
        ▼
  组件本地缓存（Informer）：将事件写入本地 Store，避免每次查询都访问 API Server
```

**为什么需要 List-Watch 而不是轮询？**

| 方案 | 实时性 | API Server 压力 | 网络开销 |
|------|--------|----------------|---------|
| 轮询（Polling） | 取决于间隔（间隔短 = 压力大） | 高（N 个组件 × M 种资源 = N×M 次查询/秒） | 高 |
| List-Watch | ⭐ 近乎实时 | ⭐ 低（只在资源变化时推送事件） | 低 |

**HTTP 长连接实现**：Watch 基于 HTTP 分块传输编码（Chunked Transfer Encoding），API Server 将每个资源变更事件序列化为 JSON，持续推送到客户端。连接断开后客户端会从上次的 resourceVersion 重新 Watch。

**Informer 机制（客户端库）**：

```
┌─────────────── Informer ────────────────────────────────────┐
│                                                              │
│  ┌──────────┐    ┌───────────┐    ┌────────────┐           │
│  │ Reflector│───▶│   Store   │───▶│ Workqueue  │───▶ Handle│
│  │(List-    │    │ (本地缓存) │    │ (事件队列)  │    │(业务   │
│  │ Watch)   │    │           │    │            │    │ 逻辑)  │
│  └──────────┘    └───────────┘    └────────────┘           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

- **Reflector**：封装 List-Watch，从 API Server 获取数据并写入 Store
- **Store（Indexer）**：线程安全的本地缓存，支持索引查询（按 label/name 等）
- **Workqueue**：解耦生产（事件接收）和消费（事件处理），支持重试和去重
- **Resync 机制**：定期（默认不开启）重新 List 全量数据，防止漏事件

**面试追问：如果 Watch 连接断开怎么办？**

1. 客户端检测到连接断开（TCP 超时 / HTTP 流中断）
2. 使用上次收到事件的 `resourceVersion` 重新发起 Watch
3. 如果版本过于陈旧（etcd 已经压缩了历史版本 → "410 Gone" 错误）
4. 客户端执行 **re-list**——重新 List 全量数据，更新本地缓存，再重新 Watch

**面试追问：etcd 的 Watch 和 API Server 的 Watch 是什么关系？**

- etcd 本身也支持 Watch 机制（基于 Raft 日志）
- API Server 的 Watch **不是**简单透传 etcd Watch
- API Server 在内存中维护了一份资源缓存（watch cache），Watch 请求由 API Server 直接服务，不需要每次都穿透到 etcd
- 写操作：API Server → 写 etcd → etcd Watch 通知 API Server 更新缓存 → API Server 分发给所有 Watch 客户端

**面试追问：K8s 为什么设计成组件间不直接通信？**

这是 K8s **解耦架构**的核心：
1. **API Server 是唯一协调点（Single Source of Truth via API Server）**：所有状态通过 API Server + etcd 作为中介，避免组件间点对点通信的复杂性
2. **任何组件都可以替换**：Scheduler 挂了可以重启，Controller Manager 可以 Leader 选举——因为它们都通过 API Server 获取状态，彼此独立
3. **水平扩展友好**：API Server 可以部署多个副本（无状态），Scheduler/Controller Manager 通过 Leader 选举保证单实例工作
4. **避免循环依赖**：Scheduler 不需要知道 Controller 做了什么，Controller 也不需要知道 Kubelet 做了什么——它们只看 API Server 中的期望和实际状态

### 🔬 API Priority and Fairness（APF）——API Server 的流量控制 ★★★

**APF（API Priority and Fairness）**是 K8s 1.20+ 引入的 API Server 请求优先级和公平性机制（1.29 GA），取代了之前的简易 `--max-requests-inflight` 限流。

**解决的问题**：如果一个 Controller 或用户发送大量请求（如 List 全量 Pod），会挤占 API Server 资源，导致其他关键请求（如 kubelet 心跳、Scheduler 调度）被延迟或超时。

**核心概念**：

```
请求进入 → 按 FlowSchema 分类 → 分配到 PriorityLevel → 公平排队 → 处理
```

| 概念 | 说明 |
|------|------|
| **FlowSchema** | 请求分类规则（按 user、group、verb、resource 等匹配请求） |
| **PriorityLevel** | 优先级级别，每个级别有独立的并发限制 |
| **Queuing** | 每个 PriorityLevel 内有公平排队（shuffle sharding），防止同一用户的请求独占 |

**内置 PriorityLevel 示例**：

| PriorityLevel | 并发份额 | 典型请求 |
|---------------|---------|----------|
| `system` | 高份额 | Node 心跳、kubelet 状态更新 |
| `leader-election` | 小但专用 | Controller Manager / Scheduler 的 Leader 选举 |
| `workload-high` | 高份额 | 用户 Pod 的创建/更新（通过 kubectl） |
| `workload-low` | 低份额 | 大量 List/Watch |
| `global-default` | 默认份额 | 未匹配任何 FlowSchema 的请求 |

**面试追问：API Server 怎么防止被流量打爆？**

APF（API Priority and Fairness）提供了两层保护：
1. **优先级**：关键请求（node/kubelet 心跳）分配高优先级和独立并发配额，不会被用户请求挤占
2. **公平性**：同一优先级内，不同用户/Namespace 的请求公平排队，防止单个用户独占 API Server

配合 `--max-mutating-requests-inflight` 和 `--max-requests-inflight`（传统限流，已逐步被 APF 取代）。

### 1.4 Namespace ★★★★

**Namespace** 是 K8s 中实现**资源隔离**的机制，将集群划分为多个虚拟子集群：

```bash
kubectl get namespaces
# default          # 默认命名空间（用户资源默认在这）
# kube-system      # 系统组件（kube-dns、kube-proxy、CNI 等）
# kube-public      # 公共资源（集群信息 ConfigMap）
# kube-node-lease  # Node 心跳租约
```

**核心特性**：

| 特性 | 说明 |
|------|------|
| **资源隔离** | 不同 Namespace 的资源相互不可见（kubectl get pods 只看到当前 NS） |
| **网络隔离** | 配合 NetworkPolicy 实现网络隔离 |
| **资源配额** | 配合 ResourceQuota 限制每个 Namespace 的资源用量 |
| **RBAC 作用域** | Role/RoleBinding 是 Namespace 范围的 |
| **DNS 隔离** | Service DNS：`<svc>.<ns>.svc.cluster.local`，跨 NS 需加命名空间 |

**哪些资源是 Namespace 级别的？哪些不是？**

| Namespace 级别 | 集群级别 |
|----------------|----------|
| Pod、Service、Deployment、ConfigMap、Secret、PVC、Ingress、Role/RoleBinding | Node、PV、ClusterRole/ClusterRoleBinding、StorageClass、Namespace 本身 |

**面试题：Namespace 有什么作用？**

**标准回答**：

Namespace 是 K8s 的**逻辑隔离**机制（不是物理隔离！），用于将集群划分为多个虚拟子集群。核心作用：

1. **多租户隔离**：不同团队/项目使用不同 Namespace，资源互不可见
2. **资源配额**：配合 ResourceQuota 限制每个 NS 的资源用量，防止资源争抢
3. **RBAC 权限控制**：不同的 NS 赋予不同用户不同权限
4. **环境隔离**：dev / staging / prod 使用不同 NS

**注意**：Namespace 提供的是**逻辑隔离**，并非物理隔离（同集群中不同 NS 的 Pod 默认仍可以通过 IP 通信），真正的网络隔离需要 NetworkPolicy。

---

### 1.5 Labels 与 Annotations ★★★★

#### Labels（标签）

**Labels** 是附加在 K8s 对象上的**键值对**，用于**标识、组织和选择**资源：

```yaml
metadata:
  labels:
    app: nginx
    env: production
    version: "1.21"
    tier: frontend
```

**核心特性**：

| 特性 | 说明 |
|------|------|
| **Selector 选择** | Service/ReplicaSet 通过 Label Selector 选择 Pod |
| **组织资源** | `kubectl get pods -l env=production` |
| **调度约束** | NodeSelector/NodeAffinity 基于 Labels |
| **分类标记** | 给资源打标签用于分组管理 |

**Label Selector 两种语法**：

```bash
# Equality-based（等值选择）
app=nginx
env!=production
tier in (frontend, backend)

# Set-based（集合选择）
environment in (production, staging)
tier notin (frontend, backend)
```

#### Annotations（注解）

**Annotations** 也是键值对，但用于存储**非标识性的元数据**，不能用于 Selector 选择：

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "Update to version 2.0"
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    build-version: "git-abc123"
    contact: "team-platform@company.com"
```

**典型用途**：

| 用途 | 示例 |
|------|------|
| **构建信息** | Git commit、构建时间、CI 流水线 ID |
| **工具配置** | Prometheus 抓取配置、Ingress 配置 |
| **回滚记录** | `kubernetes.io/change-cause` |
| **联系信息** | 负责人邮箱、文档链接 |
| **部署策略** | 金丝雀发布权重 |

#### Labels vs Annotations 对比

| 维度 | Labels | Annotations |
|------|--------|-------------|
| **用途** | 标识和选择资源 | 存储非标识性元数据 |
| **Selector** | ✅ 支持（kubectl / Service / RS 选择） | ❌ 不支持 |
| **索引** | K8s 内部会索引 Labels 用于高效查询 | 不索引 |
| **大小限制** | 较小（key ≤ 63 字符，value ≤ 63 字符） | 较大（无 key 长度限制，value 更大） |
| **典型场景** | app/env/version/tier 等分类标记 | 构建信息/监控配置/描述信息 |

**面试题：Labels 和 Annotations 区别？**

**标准回答**：

| Labels | Annotations |
|--------|-------------|
| 用于**标识和选择**对象 | 用于**描述和配置**对象 |
| 支持 Selector 查询（`-l app=nginx`） | 不支持 Selector |
| Service/ReplicaSet 通过 Label 匹配 Pod | 存储构建信息、监控配置、联系人等 |
| 被 K8s 内部索引，查询高效 | 不被索引 |

**一句话**：Labels 是用来"找"资源的（K8s 用它做关联），Annotations 是用来"描述"资源的（存储额外信息）。

### 1.6 K8s API 版本策略 ★★★

K8s API 遵循严格的版本演进策略，这对于升级规划和 CRD 设计很重要。

**API 版本阶段**：

| 阶段 | 后缀 | 含义 | 能否移除 |
|------|------|------|----------|
| **Alpha** | `v1alpha1`, `v2alpha2` | 实验性功能，可能被大幅修改或**移除**，默认禁用 | ✅ 随时可移除 |
| **Beta** | `v1beta1`, `v2beta2` | 功能相对稳定，默认启用，但仍可能有小幅调整 | ✅ 有 9 个月或 3 个版本（取较长）的弃用周期 |
| **Stable (GA)** | `v1`, `v2` | 正式发布，**永远不会被移除** | ❌ API 版本可被标记为 deprecated 但不移除 |

**面试追问：K8s 如何保证 API 向后兼容？**

1. **GA API 永远不删除**：即使标记 deprecated，API 仍然可用
2. **弃用周期**：Beta API 至少保证 9 个月或 3 个 minor 版本的支持
3. **转换机制**：不同版本的 CRD 之间通过 conversion webhook 自动转换
4. **`kubectl convert`**：命令行工具可以在不同 API 版本间转换

**与面试的关系**：当你说你写过一个 Operator 时，面试官可能追问 CRD 的 API 版本演进策略——你能否说出 `storage version`、`served`、`deprecated` 的含义。

---

# 第二章 Pod ★★★★★

## 2.1 什么是 Pod

### Pod 定义

**Pod** 是 Kubernetes 中**最小的、最基本的部署和调度单元**，是一个或多个容器的组合。同一个 Pod 中的容器：

- **共享网络命名空间**：共享 IP 地址和端口空间（通过 `localhost` 通信）
- **共享存储卷**：可以挂载相同的 Volume
- **共享 IPC 命名空间**：可以通过 SystemV 信号量或 POSIX 共享内存通信
- **共享生命周期**：一起调度、一起启动、一起终止

```
┌────────────────────────── Pod ──────────────────────────┐
│  IP: 10.244.1.5                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │  Container A │  │  Container B │  │  Container C │     │
│  │  (主容器)     │  │  (Sidecar)  │  │  (Init)     │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
│       ▲                ▲                                 │
│       └── localhost ───┘                                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │              Shared Volume (emptyDir)            │    │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### 最小调度单位

**为什么 K8s 不直接管理容器？**

K8s 不直接调度容器，而是以 Pod 为单位，原因：

1. **紧密耦合需求**：有些容器需要紧密协作（如应用容器 + 日志收集 Sidecar），它们需要共享网络和存储
2. **调度原子性**：Pod 作为原子调度单元，要么整体成功，要么整体失败
3. **简化管理**：不需要单独管理每个容器的网络、存储、生命周期

### 面试题：为什么 K8s 不直接管理容器？

**标准回答**：

1. **耦合需求**：实际应用中，一个服务可能需要多个紧密协作的容器（如应用 + 日志收集 Sidecar + 服务网格 Proxy），它们需要共享网络、存储和生命周期
2. **调度原子性**：Pod 是最小调度单位，保证紧密耦合的容器一起调度到同一节点
3. **设计哲学**：Pod 是 K8s 对"进程组"的抽象，Docker 容器是单个进程，而实际应用中往往是多进程协作
4. **复用Docker特性**：Pod 内的容器共享网络命名空间、IPC、Volume，这是 Docker 原生支持的模式

---

## 2.2 Pod 生命周期 ★★★★★

### Pod 的五种状态（Phase）

```
          ┌──────────┐
          │  Pending │  ← Pod 已创建，等待调度或镜像拉取
          └────┬─────┘
               │ 调度成功，镜像拉取完成
               ▼
          ┌──────────┐
          │  Running │  ← Pod 已绑定到 Node，所有容器已创建
          └────┬─────┘
               │                      │
          ┌────▼─────┐          ┌────▼─────┐
          │ Succeeded│          │  Failed  │
          │(正常退出)│          │(异常退出)│
          └──────────┘          └──────────┘
          
          ┌──────────┐
          │  Unknown │  ← 无法获取 Pod 状态（通常节点失联）
          └──────────┘
```

| 状态 | 含义 | 常见原因 |
|------|------|----------|
| **Pending** | Pod 已创建但未就绪 | 等待调度、镜像拉取中（ImagePullBackOff）|
| **Running** | Pod 已绑定到节点，所有容器已创建，至少一个容器仍在运行 | — |
| **Succeeded** | 所有容器正常终止（exit code = 0）且不会重启 | Job 执行完成 |
| **Failed** | 所有容器已终止，至少一个容器异常退出 | OOMKilled、CrashLoopBackOff、Error |
| **Unknown** | 无法获取 Pod 状态 | Node 失联、kubelet 与 API Server 通信异常 |

**重要区分**：Pod 的 `status.phase` 是整体状态，但还需要结合 `status.containerStatuses` 中的 `waiting/running/terminated` 状态来准确定位问题。

### 容器状态（Container States）——与 Pod Phase 的区别

Pod Phase 是**Pod 级别**的宏观状态，而每个容器有自己的精细状态，通过 `status.containerStatuses[].state` 查看：

```bash
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*].state}'
```

| 容器状态 | 含义 | 常见原因 |
|----------|------|----------|
| **Waiting** | 容器尚未启动，正在等待前置条件 | 拉取镜像中、等待 Init Container 完成、依赖 Secret/ConfigMap 不存在 |
| **Running** | 容器已创建并正在运行 | 正常运行（但不代表探针通过） |
| **Terminated** | 容器已停止运行 | 正常完成（exit 0）、崩溃（exit ≠ 0）、OOMKilled、被驱逐 |

**Waiting 状态的子原因（Reason）**：

| Reason | 含义 |
|--------|------|
| `ContainerCreating` | 容器正在创建（拉镜像、挂载卷、配置网络） |
| `ImagePullBackOff` | 镜像拉取失败，正在退避重试 |
| `ErrImagePull` | 镜像拉取出错（tag 不存在、认证失败） |
| `CrashLoopBackOff` | 容器反复崩溃，kubelet 进入退避重启模式 |
| `CreateContainerConfigError` | Secret/ConfigMap 不存在或格式错误 |

**Terminated 状态的退出码分析**：

| 退出码 | 含义 | 排查方向 |
|--------|------|----------|
| 0 | 正常退出 | 一次性任务完成 OK |
| 1 | 应用错误 | 查看 `--previous` 日志 |
| 137 | SIGKILL（通常 = OOMKilled） | 内存超 Limits，考虑增大 Limits 或优化内存 |
| 139 | SIGSEGV（段错误） | 代码空指针/内存越界 |
| 143 | SIGTERM | 被正常终止信号杀死（可能是 Liveness Probe 超时或手动删除） |

**面试追问：如何区分 Pod Pending 是因为镜像拉取慢 vs 调度失败？**

```bash
# 看容器级别的状态，而非只看 Pod Phase
kubectl get pod <name> -o jsonpath='{.status.containerStatuses[*].state.waiting.reason}'
# "ContainerCreating" + Events 中有 "Pulling image" → 镜像拉取中
# 没有任何 containerStatuses → 尚未调度（pending 但 Scheduler 还没处理或 Filter 失败）
kubectl describe pod <name> | grep -A 5 "Events"
```

### 面试题：Pod 生命周期有哪些状态？

**标准回答**：

Pod 有 5 种 Phase：

1. **Pending**：Pod 已被 API Server 接受，但容器尚未运行（可能正在调度、拉镜像）
2. **Running**：Pod 已绑定到节点，所有容器已创建，至少一个容器仍在运行
3. **Succeeded**：所有容器正常终止（退出码 0），不会重启
4. **Failed**：所有容器终止，至少一个异常退出
5. **Unknown**：无法获取 Pod 状态（节点失联等原因）

---

## 2.3 Pod 创建流程 ★★★★★

### 完整流程图

```
kubectl apply -f pod.yaml
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ 1. API Server 接收请求                               │
│    - 认证（Authentication）                          │
│    - 授权（Authorization）                           │
│    - 准入控制（Admission Control: Mutating → Validating）│
│    - 将 Pod 对象持久化到 etcd（此时 nodeName 为空）    │
└─────────────────────┬───────────────────────────────┘
                      │ Pod 存储到 etcd，状态 Pending
                      ▼
┌─────────────────────────────────────────────────────┐
│ 2. Scheduler 调度                                    │
│    - 通过 List-Watch 发现未绑定 Node 的 Pod           │
│    - Filter：过滤不满足条件的节点                     │
│    - Score：对剩余节点打分                           │
│    - Bind：选择最优节点，设置 Pod.spec.nodeName       │
│    - 更新 etcd（通过 API Server）                     │
└─────────────────────┬───────────────────────────────┘
                      │ nodeName 已设置
                      ▼
┌─────────────────────────────────────────────────────┐
│ 3. Kubelet 创建容器                                  │
│    - Watch 到分配给本节点的 Pod                       │
│    - 调用 CRI 拉取镜像（如果镜像不存在）              │
│    - 调用 CRI 创建容器                               │
│    - 调用 CNI 分配网络（给 Pod 分配 IP）              │
│    - 调用 CSI 挂载存储                               │
│    - 执行探针检查（Startup → Readiness → Liveness）   │
│    - 上报 Pod 状态到 API Server                      │
└─────────────────────────────────────────────────────┘
```

### 面试题：创建一个 Pod 会发生什么？

这是 K8s 面试中**最高频的场景题**之一，需要完整讲述每一步及涉及的核心组件。

**标准回答**：

1. **kubectl → API Server**：kubectl 发送 POST 请求到 API Server，经过认证、授权、准入控制（Mutating Admission 可修改 Pod 配置，如注入 Sidecar；Validating Admission 校验合法性），通过后将 Pod 对象写入 etcd，此时 Pod 状态为 Pending，`nodeName` 为空

2. **Scheduler 调度**：Scheduler 通过 List-Watch 发现未绑定 Node 的 Pod，执行 Filter（过滤）→ Score（打分）→ Bind（绑定），将 `nodeName` 写入 Pod 对象并更新 etcd

3. **Kubelet 创建容器**：对应节点的 Kubelet Watch 到分配给自己的 Pod，依次调用 CRI 拉取镜像、创建容器，调用 CNI 给 Pod 分配网络 IP，调用 CSI 挂载存储卷

4. **状态上报**：Kubelet 执行探针检查，容器就绪后上报 Running 状态到 API Server

---

## 2.4 Init Container

### 定义与特点

**Init Container** 是在 Pod 中、应用容器启动之前运行的**初始化容器**：

- 按顺序**串行**执行（前一个 Init Container 成功退出后才启动下一个）
- 所有 Init Container 都必须成功退出，应用容器才会启动
- 如果 Init Container 失败，Kubelet 会按照 `restartPolicy` 重启它（直到成功）
- Init Container 的镜像可以和应用容器不同（可以用工具镜像做初始化工作）

### 使用场景

| 场景 | 说明 |
|------|------|
| 等待依赖服务就绪 | 如等待数据库、缓存服务可连接后再启动应用 |
| 数据库迁移 | 运行 `flyway migrate` / `rails db:migrate` |
| 文件权限设置 | `chmod` / `chown` 调整挂载卷的权限 |
| 配置文件生成 | 从环境变量/Secret 渲染配置模板 |
| Git 仓库克隆 | 将代码从 Git 仓库克隆到共享 Volume |

### 面试题：Init Container 有什么作用？

**标准回答**：

Init Container 是在应用容器启动前按顺序执行的初始化容器。主要用途：

1. **等待依赖**：等待数据库、中间件等依赖服务就绪（如 `until nc -z db 3306; do sleep 1; done`）
2. **初始化操作**：数据库迁移、文件权限设置、配置渲染
3. **安全隔离**：可以使用与应用容器不同的镜像（如包含敏感工具的镜像），初始化完成后销毁，减少攻击面
4. **前置条件检查**：检查所需的环境变量、密钥是否存在

---

## 2.5 Sidecar 容器 ★★★★★

### 什么是 Sidecar 模式

**Sidecar（边车）模式**是将辅助功能从主应用容器中解耦，以独立容器运行在同一个 Pod 中的设计模式。借鉴了摩托车边车的概念——边车依附于主车，是主车的一部分。

```
┌──────────────── Pod ────────────────┐
│                                     │
│  ┌──────────────┐ ┌──────────────┐ │
│  │  主容器       │ │  Sidecar    │ │
│  │  (应用)       │◄┼► (辅助功能)  │ │
│  │              │ │              │ │
│  └──────────────┘ └──────────────┘ │
│          ▲                ▲         │
│          └── localhost ───┘         │
│                                     │
│  共享: 网络 | 存储 | 生命周期        │
└─────────────────────────────────────┘
```

### Sidecar 常见场景

| 场景 | Sidecar 角色 | 常见实现 |
|------|--------------|----------|
| **日志收集** | 收集应用日志并发送到日志中心 | Filebeat、Fluentd、Fluent Bit |
| **服务网格** | 代理所有进出流量，实现流量管理 | Istio Envoy、Linkerd proxy |
| **配置热更新** | 监听配置变化并通知应用重载 | confd |
| **监控指标** | 采集应用指标并暴露给 Prometheus | Prometheus Exporter |
| **HTTPS 代理** | 为 HTTP 应用提供 TLS 终止 | Nginx、Envoy |

### K8s 1.28+ 原生 Sidecar 支持

从 K8s 1.28 开始，支持 `initContainers` 设置 `restartPolicy: Always` 来实现原生 Sidecar 容器，与 Init Container 的区别是它们在 Pod 整个生命周期中都运行。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
  - name: sidecar-proxy
    image: envoyproxy/envoy
    restartPolicy: Always  # 原生 Sidecar 支持
  containers:
  - name: app
    image: myapp:latest
```

### 面试题：什么是 Sidecar 模式？

**标准回答**：

Sidecar 模式是容器设计模式的一种，将辅助功能（日志收集、监控、服务网格代理、安全加密）以独立容器运行在同一个 Pod 中。好处：

1. **解耦**：主应用和辅助功能独立开发、独立升级、使用不同技术栈
2. **复用**：同一个 Sidecar 镜像可以用于多个应用
3. **故障隔离**：Sidecar 故障重启不影响主容器
4. **资源隔离**：主容器和 Sidecar 可以独立设置资源 Request/Limit
5. **本地通信**：通过 localhost 通信，延迟极低

典型应用：Istio 的 Envoy Sidecar、日志收集的 Filebeat Sidecar。

---

## 2.6 三大探针对比（Liveness / Readiness / Startup Probe）★★★★★

这是高频考点，三种探针分别解决不同的问题：

```
容器启动
    │
    ▼
┌──────────────────┐
│  Startup Probe   │ ← 检测"启动完成了吗？"（慢启动容器专用）
│  成功 → 继续     │    在 Startup Probe 成功之前，
│  失败 → 重启容器  │    Liveness 和 Readiness 不生效
└────────┬─────────┘
         │ 成功
         ▼
┌──────────────────┐     ┌──────────────────┐
│  Liveness Probe  │     │ Readiness Probe  │
│  "活着吗？"       │     │ "能接客吗？"      │
│  失败 → 重启容器  │     │ 失败 → 摘除流量   │
└──────────────────┘     └──────────────────┘
```

| 维度 | Liveness Probe | Readiness Probe | Startup Probe |
|------|---------------|----------------|---------------|
| **目的** | 检测容器是否**存活** | 检测容器是否**就绪**接受流量 | 检测容器是否**启动完成** |
| **失败后果** | **重启容器**（kill → restart） | 从 Service Endpoints **摘除**，不重启 | **重启容器** |
| **适用场景** | 死锁、无限循环、内存泄漏导致无响应 | 依赖未就绪、预热中、临时过载 | 慢启动应用（Java/ML 模型加载） |
| **执行时机** | 容器启动后周期性执行 | 容器启动后周期性执行 | 容器启动后立即执行，成功后不再执行 |
| **对 Service 影响** | 无直接影响（除非容器被重启） | 直接影响（决定是否接收流量） | 间接影响（成功后才开始 Liveness） |
| **典型配置** | `initialDelaySeconds: 30` | `initialDelaySeconds: 5` | `failureThreshold: 30` |

**三种检测方式**：

```yaml
# 1. HTTP GET（最常用）
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: Custom-Header
      value: Awesome

# 2. TCP Socket（检测端口是否打开）
readinessProbe:
  tcpSocket:
    port: 3306

# 3. Exec 命令（执行命令检查退出码）
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
```

**面试题：Liveness Probe 和 Readiness Probe 区别？**

**标准回答**：

| Liveness | Readiness |
|----------|-----------|
| 检测容器**是否存活** | 检测容器**是否准备好接收流量** |
| 失败 → **重启容器** | 失败 → 从 Service 中**摘除 Pod IP** |
| 不影响 Service 流量 | 直接影响 Service 流量分发 |
| 用于恢复死锁/无响应的容器 | 用于保护暂时无法处理请求的容器 |

**关键理解**：Readiness Probe 决定流量是否进入 Pod，Liveness Probe 决定容器是否需要重启。两者独立工作，不要混淆。如果 Readiness 失败但不重启，说明容器可能还在初始化依赖（如等待数据库连接）；如果 Liveness 失败则说明容器有 bug（死锁、内存泄漏等）需要重启。

**Startup Probe 使用场景**：

```yaml
# 慢启动应用（如 Java Spring Boot 启动需要 2 分钟）
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30    # 最多检查 30 次
  periodSeconds: 10       # 每 10 秒一次 → 最多等待 300 秒
```

**Startup Probe 成功之前，Liveness 和 Readiness 不会执行**，避免了慢启动容器被 Liveness Probe 过早杀死的问题。这是 K8s 1.16+ 引入的新特性。

---

## 2.7 imagePullPolicy（镜像拉取策略）★★★

控制 kubelet 何时拉取镜像：

| 策略 | 行为 | 适用场景 |
|------|------|----------|
| **Always** | 每次创建 Pod 都拉取最新镜像 | 开发环境（tag 不变但镜像更新） |
| **IfNotPresent**（默认） | 本地有就用本地的，没有才拉取 | 生产环境（tag 固定不变） |
| **Never** | 只用本地镜像，不拉取 | 离线环境、预加载的镜像 |

**重要**：`latest` tag 的镜像默认使用 `Always` 策略（因为没有具体版本号，K8s 无法判断本地镜像是否最新）。

**面试题**：生产环境为什么不能用 `latest` tag？

1. `latest` 默认拉取策略是 `Always`，每次都拉取 → 启动慢
2. `latest` 指向的镜像可能随时变化 → 无法保证一致性
3. 回滚时无法确定 `latest` 之前是什么版本
4. 无法审计和追溯某个时刻运行的具体版本

---

## 2.8 Pod 优雅退出（Graceful Shutdown）★★★★

Pod 被删除时，K8s 的终止流程如下：

```
① kubectl delete pod / 滚动更新缩容
         │
         ▼
② API Server 更新 Pod 状态为 Terminating
         │
    ┌────┴────┐
    │ 并行执行  │
    ▼         ▼
③ kube-proxy 摘除流量   ④ 发送 SIGTERM 给容器主进程（PID 1）
  （从 Endpoints 移除）        │
    │                          ▼
    │                    ⑤ 应用收到 SIGTERM：
    │                       - 停止接收新请求
    │                       - 等待现有请求处理完成
    │                       - 关闭数据库连接
    │                       - 清理临时文件
    │                          │
    │              ┌───────────┴───────────┐
    │              │  优雅退出时间内完成？   │
    │              └───────────┬───────────┘
    │              ✅ 是              ❌ 否（超时）
    │              │                   │
    │              ▼                   ▼
    │         进程正常退出         ⑥ 发送 SIGKILL 强制杀死
    │         (exit 0)                │
    │                                 ▼
    └──────────────────────────► Pod 删除完成
```

**关键配置**：

```yaml
apiVersion: v1
kind: Pod
spec:
  terminationGracePeriodSeconds: 30   # 优雅退出等待时间（默认 30s）
  containers:
  - name: app
    image: myapp:latest
    lifecycle:
      preStop:                        # 在发 SIGTERM 之前执行
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # 等待 kube-proxy 摘流
```

**三个关键参数**：

| 参数 | 作用 | 默认值 |
|------|------|--------|
| `terminationGracePeriodSeconds` | 从发 SIGTERM 到 SIGKILL 的总等待时间 | 30s |
| `lifecycle.preStop` | 在发 SIGTERM **之前**执行的 Hook（先摘流再杀进程） | 无 |
| `restartPolicy` | 退出后是否重启 | Always（Deployment 下） |

**preStop Hook 的必要性**：

K8s 摘除流量（从 Endpoints 移除）和向容器发 SIGTERM **是并行的**，没有先后顺序保证。如果先收到 SIGTERM 但流量还没摘完，新请求仍然会被路由到正在关闭的 Pod，导致请求失败。

**解决方案**：在 `preStop` 中 `sleep 5` 秒，确保 kube-proxy 有足够时间更新转发规则（通常 1-3 秒完成），然后再让容器退出。

**Go 应用优雅退出示例**：

```go
func main() {
    srv := &http.Server{Addr: ":8080"}

    // 监听 SIGTERM / SIGINT
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        <-quit
        log.Println("Shutting down gracefully...")
        
        // 创建超时 context（小于 terminationGracePeriodSeconds）
        ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
        defer cancel()
        
        if err := srv.Shutdown(ctx); err != nil {
            log.Fatalf("Forced shutdown: %v", err)
        }
    }()

    srv.ListenAndServe()
}
```

**面试题：Pod 被删除时会发生什么？如何实现优雅退出？**

**标准回答**：

Pod 删除流程：

1. API Server 将 Pod 标记为 Terminating
2. **并行执行**：kube-proxy 从 Endpoints 摘除 Pod IP + kubelet 向容器发送 SIGTERM
3. 应用收到 SIGTERM 后停止接收新请求，处理完现有请求，清理资源后退出
4. 如果超过 `terminationGracePeriodSeconds`（默认 30s）还没退出，kubelet 发送 SIGKILL 强制杀死

**优雅退出最佳实践**：
- 配置 `lifecycle.preStop` + `sleep 5`（弥补摘流和发信号的并行问题）
- 应用监听 SIGTERM 信号，实现 graceful shutdown
- 设置合理的 `terminationGracePeriodSeconds`（应大于应用处理完最大耗时请求的时间）

---

## 2.9 Downward API ★★★★

### 定义

**Downward API** 让 Pod 内的容器可以获取 Pod 自身的元数据（如 Pod Name、Namespace、IP、Labels、资源 Requests/Limits），而**不需要调用 API Server**。它让应用感知到自己的运行环境。

### 两种暴露方式

| 方式 | 说明 | 适用场景 |
|------|------|----------|
| **环境变量** | 注入 Pod 元数据到容器环境变量 | 应用启动时需要知道自己是谁 |
| **Volume 文件** | 将元数据写入文件并挂载 | 数据可能变化（如 Labels），文件会自动更新 |

### 可获取的字段

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
    version: v1
  annotations:
    build: "git-abc123"
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # 方式 1：环境变量
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name          # Pod 名称
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace      # Pod 所在 NS
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP            # Pod IP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName           # 所在 Node 名称
    - name: CPU_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.cpu               # CPU Limits
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo
  volumes:
  # 方式 2：Volume 挂载（labels/annotations 只能用这种方式）
  - name: podinfo
    downwardAPI:
      items:
      - path: "labels"
        fieldRef:
          fieldPath: metadata.labels         # 写入 /etc/podinfo/labels
      - path: "annotations"
        fieldRef:
          fieldPath: metadata.annotations    # 写入 /etc/podinfo/annotations
```

### 可获取的值一览

| 类别 | 字段 | 环境变量 | Volume |
|------|------|----------|--------|
| **Pod 信息** | `metadata.name`, `namespace`, `uid` | ✅ | ✅ |
| **Pod 标签/注解** | `metadata.labels`, `metadata.annotations` | ❌ | ✅ |
| **运行时信息** | `status.podIP`, `spec.serviceAccountName`, `spec.nodeName` | ✅ | ✅ |
| **资源信息** | `requests.cpu/memory`, `limits.cpu/memory` | ✅ | ❌ |

### 使用场景

| 场景 | 说明 |
|------|------|
| **日志标识** | 注入 Pod Name + Namespace 到日志，便于追踪 |
| **注册中心** | 用 Pod IP 向外部注册中心（Consul/Eureka）注册 |
| **资源自适应** | 根据 CPU/Memory Limits 调整应用线程池大小 |
| **Prometheus 标签** | metrics 自动打上 Pod 和 Namespace 标签 |
| **配置渲染** | 根据自身 Labels 生成不同配置文件 |

### 面试题：Downward API 是什么？

**标准回答**：

Downward API 让 Pod 容器获取自身元数据（Pod Name、IP、资源 Requests/Limits、Labels 等），而**不需要通过 API Server**。有两种注入方式：

1. **环境变量（`fieldRef` / `resourceFieldRef`）**：注入静态值（name、IP、资源），Pod 启动后不变
2. **Volume 文件（`downwardAPI`）**：注入 labels/annotations，文件内容随 Pod 更新自动同步

**关键区别**：labels 和 annotations 只能通过 Volume 方式获取（因为可能变化），资源限制只能通过环境变量获取。

典型场景：应用需要知道自己的 Pod Name 打日志、需要根据 Limits 调整线程池、需要向注册中心汇报自己的 IP。

---

## 2.10 Ephemeral Containers（临时容器）★★★★

### 定义

**Ephemeral Container** 是 K8s 1.25+ GA 的特性，允许在不重启 Pod 的情况下，向运行中的 Pod **临时注入**一个调试容器。这对排查无法 `kubectl exec` 的 Pod（如极简镜像 `distroless` 或已崩溃容器）至关重要。

### 为什么需要 Ephemeral Containers

传统 `kubectl exec` 的局限：
- **distroless 镜像**：没有 shell、没有 debugging 工具（`curl`、`strace`、`tcpdump`），exec 进去什么都做不了
- **已崩溃的容器**：无法 exec 进入已退出的容器
- **不想污染应用镜像**：为调试而加工具会增大攻击面

Ephemeral Container 解决了这些问题：可以向运行中的 Pod **临时添加**一个带有所有调试工具的容器，用完即弃。

### 使用方式

```bash
# 向运行中的 Pod 注入一个调试容器
kubectl debug -it myapp-pod --image=nicolaka/netshoot --target=myapp

# 现在你在一个完全独立的容器里，但共享了 myapp 容器的进程命名空间
# 可以：
#   strace -p <pid>        # 跟踪应用进程
#   tcpdump -i eth0         # 抓包
#   curl localhost:8080     # 测试连通性
#   cat /proc/<pid>/status  # 查看进程状态
```

### 与普通容器的区别

| 维度 | 普通容器 | Ephemeral Container |
|------|----------|-------------------|
| **能否通过 Pod Spec 声明** | ✅（`spec.containers`） | ❌（只能在运行时添加） |
| **是否可重启** | ✅ | ❌（临时，不能重启） |
| **资源是否可保障** | ✅（Requests/Limits） | ❌（无资源保障） |
| **Pod 删除后** | 一起删除 | 一起删除 |
| **用途** | 业务容器 | 调试/排错 |
| **API 修改方式** | 通过 Deployment 等 | 通过 `/ephemeralcontainers` 子资源直接更新 Pod |

### 面试题：怎么调试一个 distroless 容器的 Pod？

**标准回答**：

使用 `kubectl debug -it <pod> --image=<debug-tools> --target=<container>` 注入一个 Ephemeral Container：

1. 指定一个包含调试工具的镜像（如 `nicolaka/netshoot` 或 `busybox`）
2. 通过 `--target` 参数**共享目标容器的进程命名空间**（可以 `strace` / `tcpdump` 查看目标进程）
3. 调试完成后退出临时容器的 shell，它不会自动重启，Pod 删除时一起清理

**这是 `kubectl exec` 的救急方案**：对于没有 shell 的极简镜像（distroless），Ephemeral Container 是唯一能在运行时调试的方法。

---

## 2.11 Container Lifecycle Hooks（容器生命周期钩子）

### PostStart 与 PreStop

K8s 提供了两个容器生命周期钩子，在容器生命周期的关键时刻执行：

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    lifecycle:
      postStart:            # 容器创建后、ENTRYPOINT 之前立即执行
        exec:
          command: ["/bin/sh", "-c", "echo 'Container started' > /tmp/started"]
      preStop:              # 容器终止前、SIGTERM 之前执行
        exec:
          command: ["/bin/sh", "-c", "nginx -s quit && sleep 10"]
```

| 钩子 | 执行时机 | 典型用途 | 失败后果 |
|------|---------|----------|----------|
| **PostStart** | 容器创建后**立即**执行（与 ENTRYPOINT **并行**，无先后顺序保证） | 通知外部系统、初始化注册 | 容器被 **kill** 然后根据 restartPolicy 重启 |
| **PreStop** | 收到删除请求后**先于 SIGTERM** 执行（**同步**，PreStop 完成后才发 SIGTERM） | 摘流等待、优雅关闭 | 超时后直接 SIGTERM → SIGKILL |

**关键理解**：
- **PostStart 与 ENTRYPOINT 并行执行**：PostStart 可能在 ENTRYPOINT 完成之前或之后执行，不要依赖 ENTRYPOINT 已经运行
- **PreStop 是同步阻塞的**：kubelet 会等待 PreStop 完成（或超时），然后才发 SIGTERM——这正是用来弥补 kube-proxy 摘流和发信号并行问题的关键机制（参见 2.8 节）

### 面试题：PostStart 和 PreStop 区别？

**标准回答**：

| PostStart | PreStop |
|-----------|---------|
| 容器创建后**立即**执行 | 容器终止**前**执行 |
| 与 ENTRYPOINT **并行**，无顺序保证 | **同步阻塞**，完成后才发 SIGTERM |
| 失败 → kill 容器 | 超时 → 跳过，继续终止流程 |
| 用于初始化通知 | 用于优雅退出准备（等待摘流、通知注册中心） |

**关键陷阱**：PostStart 不能依赖应用已启动（因为和 ENTRYPOINT 并行），它适合做轻量级的通知操作，不适合做依赖应用已就绪的操作。

---

# 第三章 Deployment ★★★★★

## 3.1 Deployment 基础

### 为什么需要 Deployment

直接管理 Pod 的问题：

- 手动创建的 Pod（裸 Pod）被删除后不会自动重建
- 无法实现滚动更新
- 无法实现版本回滚
- 没有副本数保障

Deployment 解决了这些问题，它提供了：

- **声明式更新**：描述期望状态，控制器自动调和
- **副本集管理**：保证指定数量的 Pod 副本始终运行
- **滚动更新**：零停机更新应用
- **版本回滚**：快速回退到历史版本
- **暂停/恢复**：支持金丝雀发布

### Deployment → ReplicaSet → Pod 的层级关系

```
┌─────────── Deployment ───────────┐
│  replicas: 3                     │
│  strategy: RollingUpdate         │
│  revisionHistoryLimit: 10        │
└──────────────┬───────────────────┘
               │ 管理
               ▼
┌─────────── ReplicaSet ───────────┐
│  replicas: 3                     │
│  selector: app=nginx             │
│  template: ...                   │
└──────────────┬───────────────────┘
               │ 管理 (保持3个Pod运行)
               ▼
    ┌──────────┼──────────┐
    ▼          ▼          ▼
┌──────┐  ┌──────┐  ┌──────┐
│ Pod1 │  │ Pod2 │  │ Pod3 │
└──────┘  └──────┘  └──────┘
```

每次 Deployment 更新都会创建新的 ReplicaSet，并保留历史 ReplicaSet 用于回滚。

### 面试题：Deployment 和 Pod 区别？

**标准回答**：

- **Pod** 是 K8s 的最小调度单元，是最基础的资源对象
- **Deployment** 是更高级的工作负载资源，管理 Pod 的副本数、滚动更新、回滚
- Deployment 不直接管理 Pod，而是管理 ReplicaSet，再由 ReplicaSet 管理 Pod 副本
- 裸 Pod 删除后不会重建，Deployment 下的 Pod 删除后会自动重建以保持副本数

---

## 3.2 ReplicaSet

### 副本控制

ReplicaSet 的核心职责是**在任何时刻保证指定数量的 Pod 副本在运行**：

- 如果 Pod 数量过多（集群分裂后的幽灵 Pod），删除多余的
- 如果 Pod 数量不足（节点故障、手动删除），创建新的

```
实际副本数 → ReplicaSet Controller 判断 → 调整（创建/删除） → 实际副本数趋向期望副本数
```

ReplicaSet 通过三个要素管理 Pod：

1. **selector**：标签选择器，识别哪些 Pod 归自己管理
2. **replicas**：期望的副本数
3. **template**：创建新 Pod 使用的模板

### 面试题：ReplicaSet 的作用是什么？

**标准回答**：

ReplicaSet 保证指定数量的 Pod 副本在任意时刻都在运行。它通过 selector 识别 Pod，通过 replicas 定义期望数量，通过 template 创建新 Pod。当 Pod 被删除或节点故障导致 Pod 消失时，ReplicaSet 会自动创建新的 Pod 来补足副本数。Deployment 通过管理 ReplicaSet 来实现滚动更新和回滚。

---

## 3.3 滚动更新 ★★★★★

### Rolling Update 原理

滚动更新是 Deployment 默认的更新策略，**逐步用新版本 Pod 替换旧版本 Pod，确保服务不中断**。

```
更新前:
  ReplicaSet-v1: [Pod1-v1] [Pod2-v1] [Pod3-v1]

更新中 (maxSurge=1, maxUnavailable=1):
  步骤1: [Pod1-v1] [Pod2-v1] [Pod3-v1] + [Pod1-v2]  ← 先创建一个 v2
  步骤2: [Pod1-v1] [Pod2-v1]           [Pod1-v2]  ← 删除一个 v1
  步骤3: [Pod2-v1]                     [Pod1-v2] [Pod2-v2] [Pod3-v2]
  步骤4:                               [Pod1-v2] [Pod2-v2] [Pod3-v2]

更新后:
  ReplicaSet-v2: [Pod1-v2] [Pod2-v2] [Pod3-v2]
  ReplicaSet-v1: (保留，用于回滚)
```

### 两个关键参数

| 参数 | 含义 | 默认值 | 示例 |
|------|------|--------|------|
| **maxSurge** | 更新过程中可以超过期望副本数的最大 Pod 数 | 25% | 3 副本 → 最多 4 个 Pod |
| **maxUnavailable** | 更新过程中不可用的最大 Pod 数 | 25% | 3 副本 → 至少 2 个可用 |

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 可以多出 1 个 Pod
      maxUnavailable: 0    # 不允许任何 Pod 不可用（适合生产环境）
```

**重要**：如果 `maxUnavailable: 0` 且只有 1 个 Node，则需要 `maxSurge >= 1`，否则新 Pod 无法调度。

### 面试题：Deployment 如何实现滚动更新？

**标准回答**：

1. 更新 Deployment 的 Pod Template（如修改镜像版本）
2. Deployment Controller 检测到期望状态变化，创建新的 ReplicaSet
3. 新 ReplicaSet 逐步扩容（创建新版本 Pod），旧 ReplicaSet 逐步缩容（删除旧版本 Pod）
4. 扩缩容的节奏由 maxSurge（最大超出副本数）和 maxUnavailable（最大不可用副本数）控制
5. 直到新 ReplicaSet 达到期望副本数，旧 ReplicaSet 缩容到 0
6. 旧 ReplicaSet 保留（数量由 `revisionHistoryLimit` 控制），用于回滚

---

## 3.4 回滚机制 ★★★★★

### 回滚命令

```bash
# 查看历史版本
kubectl rollout history deployment/nginx-deployment

# 查看某个版本的详细信息
kubectl rollout history deployment/nginx-deployment --revision=2

# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

# 回滚到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# 查看回滚状态
kubectl rollout status deployment/nginx-deployment
```

### 回滚原理

每次 Deployment 更新都会创建一个新的 ReplicaSet，旧的 ReplicaSet 不会被删除（数量受 `revisionHistoryLimit` 控制，默认 10）。回滚时，只需要将旧 ReplicaSet 重新扩容，新 ReplicaSet 缩容即可。

### 面试题：K8s 如何回滚版本？

**标准回答**：

K8s Deployment 天然支持版本回滚：

1. 每次更新创建新的 ReplicaSet，历史 ReplicaSet 被保留
2. `kubectl rollout undo deployment/xxx` 回滚到上一个版本
3. `kubectl rollout undo deployment/xxx --to-revision=N` 回滚到指定版本
4. 回滚本质是将旧 ReplicaSet 扩容、新 ReplicaSet 缩容
5. `spec.revisionHistoryLimit`（默认 10）控制保留的历史版本数

**加分点**：说明 `CHANGE-CAUSE` 需要加 `--record` 或设置 annotation `kubernetes.io/change-cause`（新版本中 `--record` 已废弃，建议用 annotation）。

---

# 第三章扩展：其他工作负载资源 ★★★★★

除了 Deployment，K8s 还提供了针对不同场景的工作负载资源。

---

## 3e.1 StatefulSet ★★★★★

### 为什么需要 StatefulSet

**Deployment 的局限**：Deployment 管理的 Pod 是**无状态的、可互换的**：
- Pod 名称随机（如 `nginx-7d8f9c5b-xyz12`）
- 网络标识不固定（每次重建 IP 和名字都变）
- 存储不绑定（除非用同一个 PVC）
- 启动/停止顺序不可控

**StatefulSet 专为有状态应用设计**，提供：
- **稳定的网络标识**（Pod 名称固定：`<statefulset-name>-0`、`<statefulset-name>-1`）
- **稳定的持久存储**（每个 Pod 绑定独立的 PVC，重建后仍绑定同一 PVC）
- **有序的部署和扩缩容**（0 → 1 → 2 顺序启动；N-1 → N-2 → ... 逆序终止）
- **有序的滚动更新**（从最大索引开始逆序更新）

### StatefulSet vs Deployment

| 维度 | Deployment | StatefulSet |
|------|-----------|-------------|
| **Pod 标识** | 随机名称，无身份 | 固定序号：`<name>-0`, `<name>-1` |
| **网络标识** | 不固定 | 稳定的 DNS：`<pod>.<headless-svc>.<ns>.svc.cluster.local` |
| **存储** | 共享 PVC（所有 Pod 共用） | 每个 Pod 独立 PVC（volumeClaimTemplates） |
| **启停顺序** | 并行、无序 | 顺序启动（0→N），逆序停止（N→0） |
| **扩缩容** | 任意 Pod 被删除 | 逆序缩容（先删最高索引） |
| **适用场景** | 无状态 Web 服务 | 数据库、消息队列、分布式存储 |

### StatefulSet 三要素

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql-headless"    # ① Headless Service（必须）
  replicas: 3
  podManagementPolicy: OrderedReady  # ② 启动策略
  volumeClaimTemplates:             # ③ 存储模板
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 100Gi
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
```

### 网络标识详解

配合 **Headless Service**（`clusterIP: None`），每个 Pod 获得稳定 DNS：

```
mysql-0.mysql-headless.default.svc.cluster.local
mysql-1.mysql-headless.default.svc.cluster.local
mysql-2.mysql-headless.default.svc.cluster.local
```

Pod 重启/迁移到其他 Node 后，**DNS 名称和 PVC 绑定不变**。

### podManagementPolicy

| 策略 | 行为 |
|------|------|
| **OrderedReady**（默认） | 严格顺序：0 启动成功 → 再启动 1 → 再启动 2 |
| **Parallel** | 并行启动/停止所有 Pod，但网络标识仍保持 |

### 面试题：StatefulSet 和 Deployment 区别？

**标准回答**：

| Deployment | StatefulSet |
|-----------|-------------|
| 用于**无状态**应用（Web Server、API 网关） | 用于**有状态**应用（数据库、消息队列） |
| Pod 随机命名，身份可互换 | Pod 固定序号命名（`name-0, name-1`），有身份 |
| Pod 重建后 IP 和名字都变 | Pod 重建后**名字和 DNS 不变** |
| 共享 PVC 或不用 PVC | 每个 Pod **独立 PVC**（volumeClaimTemplates） |
| 无序并行启动 | **有序**启动（0→1→2）和停止（2→1→0） |
| 滚动更新随机替换 | 滚动更新**逆序**进行（从最高索引开始） |

**选型原则**：应用有状态（需要数据持久化、节点之间有主从关系、需要固定网络标识）→ StatefulSet；应用无状态（可随意扩缩容、随意替换）→ Deployment。

---

## 3e.2 DaemonSet ★★★★

### 定义

**DaemonSet** 保证**每个 Node 上运行且仅运行一个** Pod 副本（或匹配特定条件的 Node）：

```
集群有 5 个 Node → DaemonSet 自动创建 5 个 Pod（每个 Node 一个）
新增 1 个 Node → 自动在该 Node 上创建一个 Pod
删除 1 个 Node → 该 Node 上的 Pod 被回收
```

### 典型使用场景

| 场景 | 示例 |
|------|------|
| **日志采集** | 每个 Node 运行 Fluentd / Filebeat 收集容器日志 |
| **监控 Agent** | 每个 Node 运行 Prometheus Node Exporter / Datadog Agent |
| **网络插件** | 每个 Node 运行 CNI 插件（Calico node、Flannel、Cilium） |
| **存储插件** | 每个 Node 运行 CSI 插件 |
| **kube-proxy** | 每个 Node 上的网络代理（本身也是 DaemonSet） |

### 配置示例

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:              # 容忍所有 Node 的污点
      - operator: Exists        # 保证每个 Node（包括有 Taint 的）都运行
      containers:
      - name: fluentd
        image: fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log        # 收集宿主机日志
```

### DaemonSet 调度

DaemonSet 绕过 Scheduler 的常规调度流程，由 DaemonSet Controller 直接为每个 Node 创建 Pod（自动添加 `spec.nodeName`），同时也会自动添加 Toleration 以保证能调度到有 Taint 的 Node。

### 面试题：DaemonSet 有什么作用？

**标准回答**：

DaemonSet 保证集群中**每个（符合条件的）Node 上运行一个 Pod 副本**。典型用途：

1. **基础设施守护进程**：每个 Node 必须运行的组件（CNI 网络插件、kube-proxy）
2. **日志收集**：每个 Node 运行日志采集 Agent（Fluentd、Filebeat）
3. **监控采集**：每个 Node 运行 Node Exporter 采集机器指标
4. **存储 CSI 插件**：每个 Node 需要挂载存储的驱动

**与 Deployment 的区别**：Deployment 关注副本数（不管在哪个 Node），DaemonSet 关注每个 Node 覆盖（不管总副本数）。

---

## 3e.3 Job 与 CronJob ★★★★

### Job

**Job** 用于管理**一次性任务**，确保指定数量的 Pod **成功完成**后停止：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 1          # 需要成功完成的 Pod 数
  parallelism: 1          # 并行运行的 Pod 数
  backoffLimit: 6         # 失败重试次数
  activeDeadlineSeconds: 300 # 总超时时间
  template:
    spec:
      restartPolicy: Never   # Job 不能用 Always（否则永远不会停止）
      containers:
      - name: migration
        image: migration-tool:latest
        command: ["./migrate.sh"]
```

**三种 Job 模式**：

| 模式 | completions | parallelism | 说明 |
|------|-------------|-------------|------|
| **单次 Job** | 1 | 1 | 运行一次，成功就结束 |
| **固定完成次数** | N | M | 总共成功 N 次，最多 M 个并行 |
| **工作队列** | 不设置 | M | 每个 Worker 处理队列中的一个任务，直到队列空 |

### CronJob

**CronJob** 在 Job 之上添加**定时调度**功能（类似 Linux crontab）：

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-backup
spec:
  schedule: "0 2 * * *"        # 每天凌晨 2 点执行（5 字段 cron 表达式）
  concurrencyPolicy: Forbid    # 禁止并发执行（Allow/Forbid/Replace）
  startingDeadlineSeconds: 300 # 最晚启动延迟
  successfulJobsHistoryLimit: 3  # 保留成功 Job 记录数
  failedJobsHistoryLimit: 1      # 保留失败 Job 记录数
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: backup-tool:latest
            command: ["./backup.sh"]
```

**Cron 表达式**（5 字段）：

```
# ┌────────── 分钟 (0 - 59)
# │ ┌────────── 小时 (0 - 23)
# │ │ ┌────────── 日 (1 - 31)
# │ │ │ ┌────────── 月 (1 - 12)
# │ │ │ │ ┌────────── 星期 (0 - 6, 0=周日)
# │ │ │ │ │
# * * * * *
 0 2 * * *    # 每天凌晨 2 点
 */5 * * * *  # 每 5 分钟
 0 9 * * 1-5  # 工作日早上 9 点
```

**concurrencyPolicy**：

| 策略 | 行为 |
|------|------|
| **Allow**（默认） | 允许并发执行 |
| **Forbid** | 如果上一个没完成，跳过本次 |
| **Replace** | 如果上一个没完成，取消它并开始新的 |

### 面试题：Job 和 CronJob 区别？

**标准回答**：

- **Job**：一次性任务，运行直到成功完成指定次数
- **CronJob**：在 Job 基础上加了定时器，按 cron 表达式定期创建 Job
- CronJob 本质是 Job 的定时调度器

**适用场景**：Job → 数据库迁移、数据批处理；CronJob → 定时备份、定时清理、定时报表生成。

---

## 3e.4 Pod Disruption Budget（PDB）★★★★

### 定义

**Pod Disruption Budget（PDB）** 是保护应用在**自愿性中断（Voluntary Disruption）**期间保持可用性的安全策略。它限制同时不可用的 Pod 数量，防止运维操作导致服务完全中断。

**自愿性中断 vs 非自愿性中断**：

| 类型 | 原因 | PDB 是否有效 |
|------|------|-------------|
| **自愿性中断** | `kubectl drain`、节点升级、Cluster Autoscaler 缩容、Deployment 滚动更新、Scheduler 抢占 | ✅ PDB 生效 |
| **非自愿性中断** | 节点硬件故障、内核 panic、网络分区、磁盘损坏 | ❌ PDB 不生效（无法预见） |

### 配置示例

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2           # 至少保持 2 个 Pod 可用
  # maxUnavailable: 1       # 或：最多 1 个 Pod 不可用（二选一）
  selector:
    matchLabels:
      app: myapp
```

**`minAvailable` vs `maxUnavailable`**：

| 参数 | 含义 | 示例（3 副本） |
|------|------|---------------|
| `minAvailable: 2` | 任何时候至少 2 个 Pod 可用 | 最多 1 个被驱逐 |
| `maxUnavailable: 1` | 任何时候最多 1 个 Pod 不可用 | 最多 1 个被驱逐 |

### PDB 工作机制

```
kubectl drain node-1
      │
      ▼
Eviction API 检查 PDB
      │
      ├── 满足 PDB（≥ minAvailable） → 驱逐 Pod
      │
      └── 不满足 PDB（< minAvailable） → 拒绝驱逐，drain 阻塞
```

### 面试题：PDB 有什么作用？

**标准回答**：

PDB 保护应用在**计划内运维操作**中不会全部下线：

1. **节点维护**：`kubectl drain` 驱逐 Pod 时受到 PDB 限制——如果驱逐会导致可用 Pod 低于 PDB 阈值，驱逐操作被拒绝
2. **集群自动缩放**：Cluster Autoscaler 缩容节点时也会检查 PDB
3. **自愿性中断保底**：设置 `minAvailable: 1` 至少保证 1 个副本始终在线

**注意**：PDB 只能限制**自愿性中断**，节点突然宕机导致的 Pod 丢失不受 PDB 控制。因此 PDB 是"主动运维保护"，不能替代多副本高可用部署。

**最佳实践**：关键服务至少配置 `minAvailable: 1` 或 `maxUnavailable: 0`。


---

# 第四章 Service 服务发现 ★★★★★

## 4.1 为什么需要 Service

### Pod IP 不固定的问题

- Pod 是**临时性**的，创建时分配 IP，销毁时回收 IP
- Pod 重启/迁移后 IP 会变化
- Deployment 中的 Pod 是动态增减的

如果其他服务直接使用 Pod IP 通信，一旦 Pod 重启就需要更新所有调用方的配置，这在微服务架构中不可行。

### Service 解决的问题

Service 提供了一层**稳定的网络抽象**：

```
Client → Service (ClusterIP: 10.96.0.1, 永久不变) → kube-proxy/iptables → Pod A / Pod B / Pod C
```

- **稳定 IP（ClusterIP）**：Service 创建后分配一个不变的虚拟 IP（除非手动删除 Service）
- **负载均衡**：将请求分发到后端 Pod
- **服务发现**：配合 DNS（CoreDNS），可以通过 Service 名称访问服务（如 `my-svc.default.svc.cluster.local`）
- **解耦前端与后端**：前端不需要关心后端 Pod 的数量和 IP

### 面试题：为什么需要 Service？

**标准回答**：

因为 Pod IP 是临时的、动态变化的（重启、扩缩容、迁移都会改变 IP），而微服务间需要稳定的通信端点。Service 提供了：

1. **稳定 IP**：Service 创建后分配固定 ClusterIP，只要 Service 存在 IP 就不变
2. **负载均衡**：自动将请求分发到健康的 Pod
3. **服务发现**：通过 DNS 名称访问（如 `<service>.<namespace>.svc.cluster.local`）
4. **解耦**：调用方不需要知道后端 Pod 的具体 IP 和数量

---

## 4.2 Service 类型 ★★★★★

### 四种类型对比

| 类型 | 说明 | 访问范围 | 使用场景 |
|------|------|----------|----------|
| **ClusterIP** | 默认类型，分配集群内部 IP | 仅集群内部 | 内部微服务间通信 |
| **NodePort** | 在每个 Node 上开放静态端口 | 集群外部（通过 `NodeIP:NodePort`） | 开发测试环境、简单外部访问 |
| **LoadBalancer** | 通过云厂商 LB 暴露服务 | 集群外部（通过 LB 公网/内网 IP） | 生产环境外部访问 |
| **ExternalName** | 将 Service 映射到外部 DNS 名称 | 集群内部 | 访问集群外部的服务（如外部数据库） |

### ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
  - port: 80        # Service 的端口
    targetPort: 8080 # 后端 Pod 的端口
```

- Service 分配一个虚拟 IP（ClusterIP），只能在集群内部访问
- kube-proxy 通过 iptables/IPVS 将 ClusterIP 流量转发到后端 Pod
- 是最常用的 Service 类型

### NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # 可选，不指定则自动分配 30000-32767
```

- 在每个 Node 上开放指定端口，将流量转发到 Service
- `NodeIP:NodePort` → Service → Pod
- 端口范围：30000-32767（可配置）
- 缺点：端口冲突、负载均衡不灵活（流量到某个 Node 后再转发可能有跨节点开销）

### LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

- 在 NodePort 基础上，自动创建云厂商的负载均衡器（CLB/ELB/ALB）
- 分配公网或内网 IP
- 按云厂商计费

### ExternalName

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

- 不创建 ClusterIP，不设置 selector
- 集群内访问 `external-db` 时，DNS 直接返回 `db.example.com`
- 常用于将外部服务纳入 K8s 服务发现体系

### 面试题：ClusterIP 和 NodePort 区别？

**标准回答**：

- **ClusterIP**：Service 默认类型，分配一个集群内部 IP，**只能在集群内部访问**。流量路径：Client → ClusterIP → kube-proxy(iptables/IPVS) → Pod
- **NodePort**：在 ClusterIP 之上，在每个 Node 上开放一个端口（30000-32767），**可以从集群外部通过 `任意NodeIP:NodePort` 访问**。流量路径：External → NodeIP:NodePort → ClusterIP → kube-proxy → Pod
- NodePort 主要用于开发测试环境或有特殊网络需求的场景，生产环境通常用 LoadBalancer 或 Ingress

---

## 4.3 Service 工作原理 ★★★★★

### 后端 Pod 发现机制

Service 通过 **Label Selector** 选择后端 Pod：

```
Service(selector: app=nginx) → 匹配 → Pod A(labels: app=nginx), Pod B(labels: app=nginx)
```

对应到底层：
1. API Server 创建 **Endpoints**（或 EndpointSlice）对象，包含所有匹配 Pod 的 `IP:Port`
2. kube-proxy 监听 Endpoints 变化，更新本地的网络规则

### kube-proxy 如何实现负载均衡

#### iptables 模式

```
Client → Service ClusterIP (10.96.0.1:80)
         ↓
    iptables 规则（概率性 DNAT）
         ↓
    随机选择一个后端 Pod IP:Port
    Pod A (10.244.1.2:8080)  ← 33%
    Pod B (10.244.2.3:8080)  ← 33%
    Pod C (10.244.3.4:8080)  ← 33%
```

- 在每个 Node 上创建 iptables 规则链
- 通过 `statistic` 模块实现概率性随机负载均衡
- 缺点：规则数量多时 O(n) 线性匹配，性能下降

#### IPVS 模式

```
Client → Service ClusterIP (10.96.0.1:80)
         ↓
    IPVS 虚拟服务器（哈希查找 O(1)）
         ↓
    支持多种调度算法：
    - rr: 轮询
    - lc: 最少连接
    - dh: 目标哈希
    - sh: 源哈希
```

- 使用 Linux IPVS（IP Virtual Server）内核模块
- O(1) 哈希查找，性能更优
- 支持多种负载均衡算法

### 面试题：Service 如何实现负载均衡？

**标准回答**：

Service 通过 kube-proxy 实现负载均衡。kube-proxy 监听 API Server 中 Service 和 Endpoints 的变化，在本节点上创建网络规则：

- **iptables 模式**：创建 iptables 规则链，使用 statistic 模块进行概率性随机分发，O(n) 匹配
- **IPVS 模式**：创建 IPVS 虚拟服务器，使用哈希表 O(1) 查找，支持多种负载均衡算法（rr/lc/dh/sh）

流量路径：Client → Service ClusterIP → （iptables/IPVS 规则）→ 随机选择后端 Pod IP → Pod 的容器端口

还有一个新特性是 **Service Topology** 和 **Traffic Distribution**，可以优先将流量路由到同一节点或同一区域的 Pod，减少跨节点网络开销。

---

### 4.4 Headless Service ★★★★

#### 定义

**Headless Service** 是一种特殊的 Service，设置 `clusterIP: None`，**不分配 ClusterIP**，DNS 查询直接返回后端 Pod 的 IP 列表（而不是 Service 的 VIP）。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None           # ← Headless Service 的关键配置
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
```

#### Headless Service vs 普通 Service

```
普通 Service (ClusterIP):
  DNS: mysql-svc.default.svc.cluster.local → 10.96.0.5 (VIP)
  流量: Client → VIP → kube-proxy → Pod (负载均衡)

Headless Service (clusterIP: None):
  DNS: mysql-headless.default.svc.cluster.local → [10.244.1.2, 10.244.2.3, 10.244.3.4]
  流量: Client → 直接连接某个 Pod IP (无负载均衡)
```

#### 使用场景

| 场景 | 说明 |
|------|------|
| **StatefulSet** | 每个 Pod 需要独立 DNS 记录（`pod-0.svc.ns.svc.cluster.local`） |
| **集群成员发现** | 应用自己实现服务发现（如 Kafka/ZK/etcd 集群需要知道所有节点 IP） |
| **自定义负载均衡** | 客户端自己实现负载均衡策略（如 gRPC 客户端负载均衡） |
| **广播通信** | 需要向所有副本发送请求 |

#### DNS 解析行为

```bash
# 普通 Service: 返回 ClusterIP
nslookup mysql-svc.default.svc.cluster.local
# → 10.96.0.5

# Headless Service: 返回所有 Ready Pod 的 IP
nslookup mysql-headless.default.svc.cluster.local
# → 10.244.1.2
# → 10.244.2.3
# → 10.244.3.4
```

#### 面试题：Headless Service 是什么？和普通 Service 区别？

**标准回答**：

Headless Service 是 `clusterIP: None` 的特殊 Service，**不分配虚拟 IP**，DNS 直接解析到后端 Pod IP 列表。

| 普通 Service | Headless Service |
|-------------|-----------------|
| 分配虚拟 ClusterIP | 不分配 ClusterIP |
| DNS → VIP → kube-proxy 负载均衡 → Pod | DNS → 直接返回 Pod IP 列表 |
| 有负载均衡 | **无负载均衡**（客户端自选 Pod） |
| 适合无状态服务 | 适合有状态服务（StatefulSet） |

**典型场景**：StatefulSet + Headless Service，每个 Pod 获得独立 DNS（`pod-0.svc.ns.svc.cluster.local`），适合数据库主从、Kafka 集群等需要知道其他成员 IP 的场景。

---

# 第五章 Ingress ★★★★★

## 5.1 什么是 Ingress

### 定义

**Ingress** 是 K8s 中管理**七层（HTTP/HTTPS）负载均衡**的资源对象，提供基于 HTTP Host 和 Path 的路由规则。

```
                     ┌─── example.com/api/* ──→ Service-A → Pod-A
Internet → Ingress ──┤
                     └─── example.com/web/* ──→ Service-B → Pod-B
```

### Ingress 解决的问题

| Service (NodePort/LoadBalancer) 的局限 | Ingress 的优势 |
|---------------------------------------|---------------|
| 每个 Service 需要一个 LB（成本高） | 一个 Ingress 可代理多个 Service |
| 仅 TCP/UDP 四层负载均衡 | HTTP/HTTPS 七层路由 |
| 无 Host/Path 路由 | 基于域名和 URL Path 路由 |
| 无 SSL/TLS 终止 | 集中 TLS 终止 |
| 无 URL 重写 | 支持 URL 重写、重定向 |

### Ingress 的核心功能

- **HTTP/HTTPS 路由**：基于域名（Host）和路径（Path）转发到不同 Service
- **TLS/SSL 终止**：集中管理 HTTPS 证书
- **负载均衡**：七层负载均衡
- **URL 重写**：如 `/api(/|$)(.*)` → `/$2`
- **限流、认证**：部分 Ingress Controller 支持

### 面试题：Ingress 和 Service 区别？

**标准回答**：

- **Service**：四层（TCP/UDP）负载均衡，工作在网络层，通过 ClusterIP 暴露服务
- **Ingress**：七层（HTTP/HTTPS）负载均衡，工作在应用层，基于域名和路径路由到不同 Service
- Service 的 LoadBalancer 类型每个服务需要独立的负载均衡器（成本高），而一个 Ingress 可以代理多个后端 Service
- Ingress 需要部署 Ingress Controller 才能工作，它本身只是一个路由规则的定义

---

## 5.2 Ingress Controller

### 为什么需要 Controller

**Ingress 只是一份路由规则定义（YAML），需要 Ingress Controller 来实际执行这些规则**。Ingress Controller 是一个运行在集群中的 Pod（通常以 Deployment 方式部署），它：

1. 监听 Ingress 资源的变化
2. 根据 Ingress 规则生成反向代理配置（如 nginx.conf）
3. 重新加载配置使规则生效
4. 作为流量入口处理所有外部流量

```
┌──────────────────┐
│  Ingress YAML    │ ← 用户定义的路由规则
│  (资源定义)       │
└────────┬─────────┘
         │ Watch
         ▼
┌──────────────────┐
│ Ingress Controller│ ← 实际执行规则的组件
│  (Nginx/Envoy)    │    监听 Ingress 变化 → 生成配置 → Reload
└────────┬─────────┘
         │ 转发流量
         ▼
┌──────────────────┐
│  Service → Pod   │ ← 后端服务
└──────────────────┘
```

### 常见 Ingress Controller

| Controller | 特点 | 适用场景 |
|-----------|------|----------|
| **Nginx Ingress** | K8s 社区最常用，基于 Nginx，功能丰富 | 通用场景 |
| **Traefik** | 自动发现、自动 HTTPS（Let's Encrypt）、Web UI | 中小规模、微服务 |
| **Istio Ingress Gateway** | 服务网格的一部分，集成 Istio 的流量管理能力 | 服务网格场景 |
| **Kong** | API 网关、插件丰富 | API 管理 |
| **HAProxy** | 高性能 | 高性能要求场景 |
| **Contour** | 基于 Envoy，高性能 | 云原生 |

### 面试题：Ingress 为什么需要 Controller？

**标准回答**：

Ingress 只是一个**声明式的路由规则资源对象**，定义了"什么流量去哪个服务"，它本身不具备处理流量的能力。Ingress Controller 是**实际执行这些规则的组件**，它是一个运行在集群中的 Pod，通常包含反向代理（如 Nginx/Envoy）：

1. Controller 通过 List-Watch 监听 Ingress 资源变化
2. 根据 Ingress 规则生成反向代理配置文件（如 nginx.conf）
3. 动态 Reload 配置使规则生效
4. 作为集群流量入口，处理所有外部 HTTP/HTTPS 请求并转发到后端 Service

**类比**：Ingress 是"菜单"，Ingress Controller 是"厨师"，只写菜单没有人做菜是不行的。

---

## 5.3 Gateway API（新一代 Ingress 标准）★★★★

### 定义

**Gateway API** 是 K8s SIG-Network 推出的**下一代七层网络 API**，旨在取代/补充 Ingress。它在设计上更**面向角色（Role-Oriented）**、更**表达力强**、更**可移植**。

```
┌─────────────────────────────────────────────────────┐
│                  Gateway API 架构                     │
│                                                     │
│  GatewayClass ──▶ Gateway ──▶ HTTPRoute / TCPRoute  │
│  (基础设施管理)    (集群运维)    (应用开发者)            │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐│
│  │ 平台团队  │   │ 集群运维  │   │    应用开发者     ││
│  │定义可用   │   │创建具体   │   │   定义路由规则    ││
│  │Controller│   │ Gateway  │   │   (HTTPRoute)    ││
│  └──────────┘   └──────────┘   └──────────────────┘│
└─────────────────────────────────────────────────────┘
```

### Gateway API vs Ingress 对比

| 维度 | Ingress | Gateway API |
|------|---------|-------------|
| **角色分离** | 无（一份 YAML 混合所有配置） | 三层分离（GatewayClass / Gateway / ×Route） |
| **协议支持** | 仅 HTTP/HTTPS | HTTP、TCP、UDP、gRPC、TLS |
| **路由能力** | Host + Path | Host + Path + Header + Method + Query + Weight |
| **流量管理** | 无 | 金丝雀发布（权重分流）、A/B 测试、流量镜像 |
| **扩展性** | 依赖 Annotation（因 Controller 而异，不可移植） | 原生字段（可移植于不同 Controller） |
| **跨命名空间** | 不支持（Ingress 和 Service 必须在同 NS） | 支持（Route 可以引用不同 NS 的 Service） |
| **状态反馈** | 弱 | 丰富的 Conditions（Accepted/Programmed/ResolvedRefs） |

### 核心资源类型

```yaml
# 1. GatewayClass — 定义可用的 Controller 实现（基础设施层）
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller

---
# 2. Gateway — 创建具体的网关实例（集群运维层）
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: infra
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All   # 允许所有 NS 的路由绑定到此 Gateway
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: tls-secret

---
# 3. HTTPRoute — 定义具体的路由规则（应用开发者层）
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-routes
  namespace: team-a
spec:
  parentRefs:
  - name: prod-gateway
    namespace: infra
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: my-service-v1
      port: 8080
  - matches:
    - path:
        type: PathPrefix
        value: /v2
      headers:                     # Ingress 无法做到的 Header 匹配
      - name: x-canary
        value: "true"
    backendRefs:
    - name: my-service-v2
      port: 8080
      weight: 10                   # 金丝雀发布：10% 流量到 v2
```

### 面试题：Gateway API 和 Ingress 区别？为什么需要 Gateway API？

**标准回答**：

Gateway API 是 K8s 官方推出的下一代路由标准，解决了 Ingress 的核心缺陷：

1. **角色分离**：Ingress 一份 YAML 混合了基础设施配置和路由规则；Gateway API 分为 GatewayClass（平台团队）、Gateway（集群运维）、HTTPRoute（开发者）三层，职责清晰
2. **表达力更强**：支持 Header 匹配、权重分流、流量镜像、跨 NS 引用等 Ingress 做不到的能力
3. **可移植性**：Ingress 大量依赖 Annotation（如 `nginx.ingress.kubernetes.io/rewrite-target`），换一个 Ingress Controller 就需要改配置；Gateway API 的所有能力都是标准字段
4. **更多协议**：Ingress 只支持 HTTP，Gateway API 原生支持 TCP/UDP/gRPC/TLS

**一句话**：Ingress 是功能不足的 MVP，Gateway API 是面向角色的、可移植的、表达能力强的下一代方案。目前两者并存，企业逐渐向 Gateway API 迁移。

**注意**：Gateway API 从 K8s 1.19 开始逐步引入，1.0 版本在 2023 年 GA，是未来的标准方向。面试中能提到它说明你关注社区最新动态。

---

# 第六章 ConfigMap 与 Secret ★★★★★

## 6.1 ConfigMap

### 定义

**ConfigMap** 用于将**非敏感**的配置数据与应用程序代码解耦，使应用可以**配置化**部署。

典型配置：
- 数据库连接地址（非密码部分）
- 应用运行参数（日志级别、超时时间、线程池大小）
- 功能开关/特性开关
- 外部服务 URL

### ConfigMap 三种使用方式

#### 1. 环境变量注入

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "mysql.default.svc.cluster.local"
  LOG_LEVEL: "debug"
---
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    envFrom:
    - configMapRef:
        name: app-config        # 注入整个 ConfigMap 的所有 key
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config      # 注入 ConfigMap 中的单个 key
          key: DB_HOST
```

#### 2. 文件挂载（Volume）

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 80;
      server_name example.com;
      location / {
        proxy_pass http://backend:8080;
      }
    }
---
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: nginx
    image: nginx:latest
    volumeMounts:
    - name: config
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf    # 挂载单个文件而不是目录
  volumes:
  - name: config
    configMap:
      name: nginx-config
```

#### 3. 命令行参数

```yaml
spec:
  containers:
  - name: app
    image: myapp:latest
    args:
    - "--log-level=$(LOG_LEVEL)"
    env:
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LOG_LEVEL
```

### ConfigMap 更新

- **环境变量方式**：ConfigMap 更新后，**已运行的 Pod 不会自动更新**环境变量，需要重启 Pod
- **Volume 挂载方式**：ConfigMap 更新后，挂载的文件会在一定时间内（kubelet 同步周期 + TTL）自动更新（不支持 subPath 挂载自动更新）

### 面试题：ConfigMap 如何使用？

**标准回答**：

ConfigMap 有三种使用方式：

1. **环境变量注入**（`envFrom` 注入整个 ConfigMap 或 `valueFrom.configMapKeyRef` 注入单个 key）：适合简单配置值，但更新后需要重启 Pod
2. **文件挂载**（Volume 挂载为文件）：适合配置文件，更新后文件会自动同步（约 1 分钟），尤其适合 Nginx 等需要配置文件的场景
3. **命令行参数**（结合环境变量）：通过 `$(VAR_NAME)` 语法引用

**注意事项**：ConfigMap 有大小限制（默认 1MB，基于 etcd 的存储限制），不适合存储大配置文件；只适合非敏感数据，敏感数据应使用 Secret。

---

## 6.2 Secret ★★★★★

### 定义

**Secret** 用于存储**敏感信息**，如密码、Token、TLS 证书、SSH 密钥等。与 ConfigMap 类似，但增加了编码和一定的安全保护。

### Secret 类型

| 类型 | 用途 | 说明 |
|------|------|------|
| **Opaque** | 通用密钥（默认） | base64 编码的任意键值对 |
| **kubernetes.io/tls** | TLS 证书 | 包含 `tls.crt` 和 `tls.key` |
| **kubernetes.io/dockerconfigjson** | 镜像仓库认证 | `~/.docker/config.json` 内容 |
| **kubernetes.io/basic-auth** | 基础认证 | 包含 `username` 和 `password` |
| **kubernetes.io/service-account-token** | ServiceAccount Token | 自动创建，挂载到 Pod |

### Secret 使用方式

与 ConfigMap 完全相同：环境变量、文件挂载、命令行参数。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=        # base64("admin")
  password: cGFzc3dvcmQ=    # base64("password")
# 或使用 stringData（不需要 base64 编码，K8s 会自动编码）
stringData:
  username: admin
  password: password
```

### Secret vs ConfigMap

| 维度 | ConfigMap | Secret |
|------|-----------|--------|
| **用途** | 非敏感配置 | 敏感信息（密码、证书、Token） |
| **存储编码** | 明文 | base64 编码（注意：不是加密！） |
| **内存** | 普通存储 | 某些实现中加密存储，只在需要时解密到 tmpfs |
| **大小限制** | ~1MB | ~1MB |
| **RBAC** | 可以单独设置访问权限 | 建议与 ConfigMap 分离权限 |
| **更新行为** | 与 ConfigMap 相同 | 与 ConfigMap 相同 |

### 重要安全提示

Secret 的 `data` 字段是 **base64 编码，不是加密**！任何人都可以解码。生产环境建议：

- 结合 **External Secrets Operator**，从 Vault/AWS Secrets Manager 等外部密钥管理服务同步密钥
- 设置严格的 RBAC 权限
- 启用 **Encryption at Rest**（etcd 数据加密）
- 使用 **Sealed Secrets** 安全存储到 Git

### 面试题：Secret 和 ConfigMap 区别？

**标准回答**：

1. **用途不同**：ConfigMap 存储非敏感配置（日志级别、超时时间），Secret 存储敏感信息（密码、Token、TLS 证书）
2. **编码不同**：ConfigMap 明文存储，Secret 的 data 字段使用 base64 编码（但注意 base64 不是加密，只是编码）
3. **安全机制**：Secret 有额外的安全特性——内存中存储（tmpfs，不落盘）、支持 etcd 静态加密、可以设置严格的 RBAC 访问控制
4. **使用方式相同**：都可以通过环境变量、文件挂载、命令行参数注入到容器中

**核心点**：ConfigMap 放明文配置，Secret 放 base64 编码的敏感数据 + 额外的安全机制，但 base64 本身不提供加密保护，生产环境需配合外部密钥管理系统。

---

# 第七章 存储管理 ★★★★

## 7.1 Volume 概述

### 为什么需要 Volume

容器中的文件是**临时的**，容器崩溃后 kubelet 重启容器时，文件会丢失。此外，同一个 Pod 中的多个容器需要共享文件。

K8s Volume 解决了：
- 容器重启后数据保留
- Pod 内多容器共享数据
- 将外部存储挂载到容器中

### 两种基本的临时卷

#### emptyDir

```yaml
volumes:
- name: cache
  emptyDir: {}
  # emptyDir:
  #   medium: Memory   # 使用内存（tmpfs），适合缓存
  #   sizeLimit: 1Gi
```

| 特性 | 说明 |
|------|------|
| **生命周期** | 与 Pod 绑定，Pod 删除卷也删除 |
| **创建时机** | Pod 被调度到 Node 时创建 |
| **初始状态** | 空目录 |
| **使用场景** | 临时缓存、排序中间文件、多容器共享数据 |
| **注意** | 容器重启时 emptyDir 数据不会丢失，Pod 删除才会丢失 |

#### hostPath

```yaml
volumes:
- name: host-data
  hostPath:
    path: /data/app-logs
    type: DirectoryOrCreate
```

| 特性 | 说明 |
|------|------|
| **生命周期** | 与 Node 绑定，Pod 删除数据仍在 Node 上 |
| **风险** | 调度到不同 Node 可能找不到数据；Pod 迁移数据丢失 |
| **使用场景** | 访问宿主机 Docker socket、日志采集、监控 |
| **生产慎用** | 大多数场景应使用 PV/PVC 代替 |

### 面试题：Pod 重启数据为什么会丢失？

**标准回答**：

容器文件系统是**临时层（OverlayFS 的容器层）**，容器删除/重启后该层会被删除。K8s 中 Pod 重启时 kubelet 本质是停止旧容器、创建新容器，新容器的文件系统是全新的。解决方法是使用 Volume：

1. **emptyDir**：Pod 生命周期内的临时存储（Pod 删除前数据不丢）
2. **hostPath**：挂载宿主机目录（Pod 迁移会丢失，不推荐生产使用）
3. **PV/PVC + StorageClass**：持久化存储（生产环境标准方案），使用云盘/NAS/分布式存储

---

## 7.2 PV（Persistent Volume） ★★★★★

### 定义

**PersistentVolume（PV）** 是集群中由管理员预先提供的一块**存储资源**，类似于 Node 是计算资源，PV 是存储资源。

```
PV 是"仓库里的一块存储"：大小固定、有访问模式、独立于 Pod 存在
```

PV 的核心属性：

| 属性 | 说明 |
|------|------|
| **capacity** | 存储容量，如 `storage: 10Gi` |
| **accessModes** | 访问模式：`ReadWriteOnce`、`ReadOnlyMany`、`ReadWriteMany` |
| **persistentVolumeReclaimPolicy** | 回收策略：`Retain`/`Delete`/`Recycle`（已废弃） |
| **storageClassName** | 绑定到哪个 StorageClass |
| **volumeMode** | 卷模式：`Filesystem` / `Block` |

### 访问模式

| 模式 | 缩写 | 含义 |
|------|------|------|
| **ReadWriteOnce** | RWO | 单个 Node 以读写方式挂载 |
| **ReadOnlyMany** | ROX | 多个 Node 以只读方式挂载 |
| **ReadWriteMany** | RWX | 多个 Node 以读写方式挂载（需 NFS/CephFS 等共享存储） |
| **ReadWriteOncePod** | RWOP | 单个 Pod 以读写方式挂载（K8s 1.22+） |

### PV 回收策略

| 策略 | 行为 |
|------|------|
| **Retain** | PVC 删除后 PV 保留，需要手动清理（适合需要数据保护的场景） |
| **Delete** | PVC 删除后 PV 和底层存储都自动删除（云盘常用） |

### 面试题：什么是 PV？

**标准回答**：

PV（PersistentVolume）是集群级别的存储资源抽象，由管理员预先创建（或通过 StorageClass 动态创建），独立于 Pod 的生命周期。它定义了存储的容量、访问模式（RWO/ROX/RWX）、回收策略等。PV 是存储的"供给方"，PVC 是"需求方"，两者通过匹配绑定。

可以类比为：PV 是预先挖好的"池塘"（存储池），PVC 是向池塘申请用水的"水管"，Pod 通过水管喝水。

---

## 7.3 PVC（Persistent Volume Claim） ★★★★★

### 定义

**PersistentVolumeClaim（PVC）** 是用户对存储的**申请**，类似于 Pod 申请计算资源（CPU/内存），PVC 申请存储资源。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
---
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc     # Pod 通过 PVC 使用存储
```

### PV 和 PVC 的绑定流程

```
创建 PVC → K8s 寻找匹配的 PV → 绑定(Bind) → Pod 使用 PVC
```

匹配条件：
1. **storage**：PV 容量 >= PVC 请求量
2. **accessModes**：PV 访问模式兼容 PVC
3. **storageClassName**：一致（除非 PVC 不指定或 PV 无 class）
4. **selector**：如果 PVC 指定了 label selector，还需要标签匹配

绑定后：
- PV 和 PVC 是 **一对一绑定**（一个 PV 只能绑定一个 PVC）
- PVC 删除后 PV 根据回收策略（Retain/Delete）处理

### 面试题：PV 和 PVC 区别？

**标准回答**：

| 维度 | PV | PVC |
|------|-----|------|
| **角色** | 存储的"供给方" | 存储的"需求方" |
| **创建者** | 管理员（或 StorageClass 自动创建） | 用户（开发者） |
| **生命周期** | 独立于 Pod，直到 PVC 删除后根据策略处理 | 与 Pod 间接关联 |
| **类比** | 仓库里的硬盘 | 向仓库申请一块硬盘 |

PV 和 PVC 的设计理念是**关注点分离**：管理员管理存储基础设施，开发者只需声明自己的存储需求（容量、访问模式），不需要关心底层存储实现。

---

## 7.4 StorageClass ★★★★★

### 定义

**StorageClass** 用于**动态供给（Dynamic Provisioning）** PV。不需要管理员预先创建 PV，当 PVC 创建时，StorageClass 自动创建 PV 和底层存储。

```
静态供给（无 StorageClass）：
管理员手动创建 PV → PVC 绑定 PV → Pod 使用 PVC

动态供给（有 StorageClass）：
管理员创建 StorageClass → 用户创建 PVC（指定 StorageClass）
→ Provisioner 自动创建 PV + 底层存储 → 绑定 PVC → Pod 使用
```

### StorageClass 核心配置

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs   # 存储提供者（云厂商或 CSI 驱动）
parameters:                            # 传递给 provisioner 的参数
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete                 # PV 回收策略
allowVolumeExpansion: true            # 是否允许 PVC 扩容
volumeBindingMode: WaitForFirstConsumer  # 绑定模式（延迟绑定）
```

### volumeBindingMode

| 模式 | 行为 |
|------|------|
| **Immediate**（默认） | PVC 创建后立即创建 PV 并绑定，不考虑 Pod 调度 |
| **WaitForFirstConsumer** | 等到有 Pod 使用此 PVC 时才创建 PV，确保 PV 创建在 Pod 所在可用区（避免跨 AZ 挂载失败） |

### 常见 Provisioner

| 环境 | Provisioner |
|------|------------|
| AWS EBS | `kubernetes.io/aws-ebs` |
| Azure Disk | `kubernetes.io/azure-disk` |
| GCE PD | `kubernetes.io/gce-pd` |
| Ceph RBD | `rbd.csi.ceph.com` |
| NFS | `nfs.csi.k8s.io` |
| 本地存储 | `kubernetes.io/no-provisioner`（local PV） |

### 面试题：StorageClass 有什么作用？

**标准回答**：

StorageClass 实现**动态存储供给**。没有 StorageClass 时，管理员需要手动创建 PV（静态供给），效率低且浪费（创建的 PV 可能永远不会被使用）。有了 StorageClass，开发者创建 PVC 时指定 StorageClass，provisioner 会**自动创建底层存储和 PV**，实现按需分配。

核心特性：
1. **动态供给**：按需创建 PV
2. **延迟绑定**（WaitForFirstConsumer）：PV 等 Pod 调度完成后再创建，避免跨可用区挂载
3. **卷扩容**：PVC 可以直接扩容（`allowVolumeExpansion: true`）
4. **多存储类型**：可以创建多个 StorageClass（fast-ssd、standard-hdd），用户按需选择

**PV/PVC/StorageClass 三者关系**：
- **PV**：抽象存储资源（供给方）
- **PVC**：申请存储资源（需求方）
- **StorageClass**：动态创建 PV 的模板/工厂（自动化供给）

---

# 第八章 调度器 Scheduler ★★★★★

## 8.1 Scheduler 原理

### 调度流程（Filter → Score → Bind）

K8s Scheduler 负责将未分配的 Pod 调度到合适的 Node 上，流程分三步：

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  1. Filter   │────▶│  2. Score    │────▶│  3. Bind     │
│  （过滤）     │     │  （打分）     │     │  （绑定）     │
│              │     │              │     │              │
│  排除不满足   │     │  对剩余节点   │     │  选最优节点   │
│  条件的节点   │     │  进行打分排序 │     │  绑定 Pod    │
└──────────────┘     └──────────────┘     └──────────────┘
    N个节点              M个候选              1个最优
    (M ≤ N)             节点                 节点
```

**Scheduler 不是直接调度，而是通过 API Server 更新 Pod 的 `spec.nodeName`**。Kubelet 监听 API Server，发现分配给本节点的 Pod 后创建容器。

#### 1. Filter（预选/过滤）

排除不满足条件的节点：

- **资源不足**：CPU/内存剩余量不满足 Pod 的 Request
- **NodeSelector/NodeAffinity 不匹配**
- **Taint 不兼容**：Pod 不能容忍节点的污点
- **端口冲突**：Pod 需要的 hostPort 已被占用
- **卷冲突**：PV 的拓扑限制（如云盘在特定可用区）
- Node 处于不可调度状态（`Unschedulable` 或 `Cordon`）

#### 2. Score（优选/打分）

对通过过滤的节点进行打分，常用策略：

| 打分策略 | 说明 |
|---------|------|
| **LeastRequestedPriority** | 资源使用率越低的节点分越高（避免热点）|
| **BalancedResourceAllocation** | CPU 和内存使用率均衡的节点分越高 |
| **MostRequestedPriority** | 资源使用率越高的节点分越高（装箱策略，减少节点数）|
| **NodeAffinityPriority** | 符合亲和性规则的节点分更高 |
| **ImageLocalityPriority** | 已有容器镜像的节点分更高（减少拉镜像时间）|
| **PodTopologySpread** | 按拓扑域均匀分布 Pod |

#### 3. Bind（绑定）

选择得分最高的节点，通过 API Server 将 Pod 的 `spec.nodeName` 设置为该节点。

### 调度插件（Scheduling Framework）

K8s 1.15+ 引入了**调度框架（Scheduling Framework）**，将调度过程扩展为多个扩展点（PreFilter、Filter、PostFilter、PreScore、Score、Reserve、Permit、PreBind、Bind、PostBind），每个扩展点都可以注册插件，提供更高的扩展性。

### 面试题：Scheduler 如何选择节点？

**标准回答**：

Scheduler 调度分为三步：

1. **Filter（过滤）**：排除不满足条件的节点——资源不足、NodeSelector 不匹配、Taint 不兼容、端口冲突、卷拓扑限制等
2. **Score（打分）**：对候选节点按多种策略打分——资源利用率（LeastRequested）、资源均衡（BalancedAllocation）、镜像本地性等，加权求和
3. **Bind（绑定）**：选择最高分节点，通过 API Server 设置 Pod 的 `spec.nodeName`

Scheduler 支持自定义调度器和调度插件（Scheduling Framework），可以扩展调度逻辑。核心代码在 `kube-scheduler` 进程中，通过 List-Watch 获取未绑定 Pod 和 Node 状态。

---

## 8.2 NodeSelector

最简单的调度约束，通过节点标签选择节点：

```yaml
spec:
  nodeSelector:
    disktype: ssd       # 只调度到有 disktype=ssd 标签的节点
```

给节点打标签：`kubectl label nodes node1 disktype=ssd`

**局限**：只支持精确匹配（AND），不支持 `in/notin/exists/gt/lt` 等高级表达式。

---

## 8.3 Node Affinity（节点亲和性）

Node Affinity 是 NodeSelector 的增强版，支持更丰富的表达式：

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # 硬亲和（必须满足）
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
      preferredDuringSchedulingIgnoredDuringExecution:  # 软亲和（优先满足）
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - zone-a
```

| 类型 | 行为 |
|------|------|
| **requiredDuringSchedulingIgnoredDuringExecution** | 硬亲和：必须满足，否则 Pod 无法调度 |
| **preferredDuringSchedulingIgnoredDuringExecution** | 软亲和：优先满足，不满足也可以调度 |

**命名解析**：
- `DuringScheduling`：调度时检查 → 调度时生效
- `DuringExecution`：运行时检查 → 运行时生效（目前不支持运行时驱逐，所以只有 `IgnoredDuringExecution`）

---

## 8.4 Pod Affinity / Anti-Affinity（Pod 间亲和性/反亲和性）

根据**已运行的 Pod** 来调度新 Pod：

```yaml
spec:
  affinity:
    podAffinity:              # Pod 亲和：把 Pod 调度到"特定 Pod 所在"的节点/拓扑域
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: "kubernetes.io/hostname"    # 拓扑域：同节点
    podAntiAffinity:          # Pod 反亲和：把 Pod 分散到"特定 Pod 不在"的节点/拓扑域
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - web-server
          topologyKey: "topology.kubernetes.io/zone"  # 拓扑域：跨可用区
```

| 场景 | 使用 | 示例 |
|------|------|------|
| **Pod 亲和** | 把相关服务部署在一起 | 缓存 Pod 和应用 Pod 放在同一节点（降低延迟）|
| **Pod 反亲和** | 把同一服务的 Pod 分散 | 同一 Deployment 的 Pod 分散到不同可用区（高可用）|

**topologyKey** 定义了"拓扑域"的范围：
- `kubernetes.io/hostname`：同节点
- `topology.kubernetes.io/zone`：同可用区
- `topology.kubernetes.io/region`：同区域

---

## 8.5 Taint 与 Toleration ★★★★★

### 概念

**Taint（污点）** 是打在节点上的标记，表示"这个节点不干净"：
- 有 Taint 的节点**排斥**没有对应 Toleration 的 Pod
- Taint 作用对象是 Node，Toleration 作用对象是 Pod

**Toleration（容忍）** 是 Pod 上的声明，表示"我能接受哪些 Taint"：
- 有相应 Toleration 的 Pod 才能被调度到有 Taint 的 Node
- Toleration 不会让 Pod 主动选择有 Taint 的节点，只是"允许"被调度过去

### Taint 结构

```
key=value:effect
```

```bash
# 给节点打 Taint
kubectl taint nodes node1 key1=value1:NoSchedule
```

### 三种 Effect

| Effect | 行为 |
|--------|------|
| **NoSchedule** | 新 Pod **不会调度**到该节点（已运行的 Pod 不受影响）|
| **PreferNoSchedule** | 新 Pod **尽量不调度**到该节点（软约束）|
| **NoExecute** | 新 Pod **不调度**，已运行的 Pod **立即驱逐**（除非有对应 Toleration 且设置了 tolerationSeconds）|

### Toleration 配置

```yaml
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"        # Equal 或 Exists
    value: "value1"
    effect: "NoSchedule"
    tolerationSeconds: 3600  # 仅在 NoExecute 时有效，3600 秒后驱逐
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300   # 节点 NotReady 后等待 300 秒再驱逐
```

### 典型使用场景

| 场景 | Taint | 说明 |
|------|-------|------|
| **专用节点** | `dedicated=group1:NoSchedule` | 只允许特定团队的工作负载 |
| **GPU 节点** | `nvidia.com/gpu:NoSchedule` | 只有需要 GPU 的 Pod 才调度到 GPU 节点 |
| **节点故障驱逐** | `node.kubernetes.io/not-ready:NoExecute` | 节点 NotReady 一定时间后驱逐 Pod（K8s 自动添加） |
| **节点维护** | `node.kubernetes.io/unschedulable:NoSchedule` | `kubectl cordon` 时自动添加 |
| **独占节点** | `Exclusive:NoSchedule` | 配合 Toleration 实现某类 Pod 独占节点 |

### 面试题：Taint 和 Toleration 是什么？

**标准回答**：

Taint 是 Node 上的"污点标记"，表示节点有某种属性或限制；Toleration 是 Pod 上的"容忍声明"，表示 Pod 可以接受相应 Taint 的节点。

- **Taint**：给 Node 打标记（effect: NoSchedule / PreferNoSchedule / NoExecute），排斥不匹配的 Pod
- **Toleration**：给 Pod 声明容忍，允许它调度到有对应 Taint 的 Node

典型应用：
1. **专用节点**：GPU 节点打 Taint，只有需要 GPU 的 Pod 配置 Toleration
2. **节点隔离**：`kubectl cordon` 实际上就是添加 NoSchedule Taint
3. **故障自动驱逐**：Node 故障时 K8s 自动加 NoExecute Taint，未匹配的 Pod 被驱逐
4. **污点容忍**：容忍 `node.kubernetes.io/not-ready` 一段时间后才被驱逐，防止短暂故障导致 Pod 误驱逐

**关键区别**：Node Affinity 是 Pod **主动选择**节点，Taint/Toleration 是 Node **被动排斥** Pod，两者配合使用可以实现精细的调度控制。

---

## 8.6 PriorityClass（优先级）与 Preemption（抢占）★★★★

### 定义

**PriorityClass** 给 Pod 分配优先级数值，**Preemption** 是高优先级 Pod 调度失败时驱逐低优先级 Pod 以腾出资源。

```
Pod A (priority: 1000) + Pod B (priority: 100) + Pod C (priority: 10)

节点资源不足时，Pod C 最先被驱逐
```

### PriorityClass 配置

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000                        # 数值越大优先级越高（范围 -2^31 ~ 10^9）
globalDefault: false               # 是否为默认优先级
description: "Used for critical services"
---
apiVersion: v1
kind: Pod
spec:
  priorityClassName: high-priority  # Pod 引用优先级
  containers:
  - name: app
    image: myapp:latest
```

### 系统内置优先级

| PriorityClass | 值 | 用途 |
|---------------|-----|------|
| `system-node-critical` | 2000001000 | Node 关键组件（CNI、kube-proxy 等） |
| `system-cluster-critical` | 2000000000 | 集群关键组件（CoreDNS、Metrics Server 等） |
| 自定义高优先级 | 1000 ~ 10000 | 核心业务服务 |
| 自定义低优先级 | 0 ~ 100 | 测试/批处理任务 |

### Preemption（抢占）流程

```
1. Scheduler 尝试调度高优先级 Pod → 资源不足，调度失败
       │
2. 寻找可抢占的低优先级 Pod（在目标节点上）
       │
3. 驱逐低优先级 Pod（优雅退出 →
       │
4. 在被释放的资源上调度高优先级 Pod
       │
5. 被驱逐的低优先级 Pod 重新进入调度队列，可能被调度到其他节点
```

**Preemption 遵循的规则**：
- 只驱逐优先级**更低**的 Pod（同级不驱逐）
- 优先驱逐优先级最低的 Pod
- 同类优先级的 Pod 中，QoS 等级越低越先被驱逐（BestEffort > Burstable > Guaranteed）
- 尽可能减少被驱逐的 Pod 数量（最少化影响）

### 面试题：PriorityClass 和 Preemption 是什么？

**标准回答**：

PriorityClass 给 Pod 分配优先级（值越大优先级越高），Preemption 是 Scheduler 的抢占机制：当高优先级 Pod 因资源不足无法调度时，会**驱逐**目标节点上的低优先级 Pod 来腾出资源。

**典型场景**：
1. **核心服务优先保障**：支付服务的 PriorityClass 设为 10000，批处理任务设为 100——资源紧张时优先保障支付
2. **集群组件保护**：system-node-critical 优先级极高（20 亿+），防止用户 Pod 抢占系统 Pod 资源
3. **弹性资源利用**：低优先级的批处理任务使用集群剩余资源，核心服务需要时自动让出

**与 PDB 的关系**：Preemption 会遵守 PDB——如果驱逐会导致可用副本低于 PDB 阈值，Preemption 会被拒绝。这是为了防止"为了保护 Pod A 上线，把 Pod A 的其他副本也杀掉了"。

---

## 8.7 Topology Spread Constraints（拓扑分布约束）★★★★

### 定义

**Topology Spread Constraints** 控制 Pod 在拓扑域（可用区、节点等）之间的**分布均匀度**，比 Pod Anti-Affinity 更灵活。

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                       # 最大倾斜度（数量差）
    topologyKey: topology.kubernetes.io/zone  # 拓扑域
    whenUnsatisfiable: DoNotSchedule  # 不满足时的处理策略
    labelSelector:                    # 匹配哪些 Pod 参与分布计算
      matchLabels:
        app: web-server
```

**核心参数详解**：

| 参数 | 含义 | 示例 |
|------|------|------|
| `maxSkew` | 任意两个拓扑域的 Pod 数量最大差值 | maxSkew=1，zone-a 3 个 Pod，zone-b 至少 2 个 |
| `topologyKey` | 定义拓扑域的维度 | Node label 的 key（如 `zone`、`hostname`） |
| `whenUnsatisfiable` | 无法满足时怎么办 | `DoNotSchedule`（不调度）或 `ScheduleAnyway`（尽量满足） |

**与 Pod Anti-Affinity 的区别**：

| 维度 | Pod Anti-Affinity | Topology Spread Constraints |
|------|-------------------|---------------------------|
| **语义** | "不要把 Pod 调度在一起" | "让 Pod 在拓扑域之间均匀分布" |
| **精确度** | 二元（在/不在） | 数量化（maxSkew 精确控制差值） |
| **扩展** | 每个规则只检查一个拓扑域 | 可以声明多个约束（zone + hostname） |

### 面试题：如何让 Pod 均匀分布在不同可用区？

**标准回答**：使用 `topologySpreadConstraints`，设置 `topologyKey: topology.kubernetes.io/zone` 和 `maxSkew: 1`，确保任意两个可用区的 Pod 数量差不超过 1。

**进阶**：结合 `whenUnsatisfiable: DoNotSchedule`（严格要求，宁可 Pending 也不破坏分布）和 `whenUnsatisfiable: ScheduleAnyway`（尽力而为，分布不均总比没有好）的不同策略。

---

# 第九章 Kubernetes 网络 ★★★★★

## 9.1 K8s 网络模型

### 四层网络通信

K8s 网络有四个层次的通信场景：

```
┌─────────────────────────────────────────────┐
│          同一 Pod 内多容器通信               │
│         (共享网络命名空间 → localhost)        │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│          不同 Pod 之间通信                   │
│    (同一 Node → cni0 网桥                   │
│     不同 Node → CNI 隧道/路由)               │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│          Pod 与 Service 之间通信             │
│   Pod → ClusterIP → kube-proxy → 目标 Pod    │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│        集群外部与 Service 之间通信            │
│  External → NodePort / LoadBalancer / Ingress│
└─────────────────────────────────────────────┘
```

### K8s 网络模型的基本要求

K8s 对网络提出三个基本要求（CNI 规范的基础）：

1. **Pod 可以和任何其他 Pod 直接通信**（不需要 NAT）
2. **Node 可以和任何 Pod 直接通信**（不需要 NAT）
3. **Pod 看到自己的 IP 和其他 Pod 看到它的 IP 是同一个人**（IP 身份一致性）

这就意味着：**所有 Pod 在一个扁平的网络空间中**，跨节点 Pod 通信由 CNI 插件实现。

### 面试题：K8s 网络模型是什么？

**标准回答**：

K8s 网络模型要求所有 Pod 在一个**扁平的、三层可达的网络空间**中：

1. Pod 间可以直接通信（无需 NAT）
2. Node 和 Pod 可以直接通信
3. Pod 看到自己的 IP 和外部看到的 IP 一致

实现这个模型的是 **CNI 插件**（如 Calico/Flannel/Cilium），它们负责：
- 给每个 Pod 分配唯一 IP
- 跨 Node 的 Pod 通信（隧道/VXLAN/BGP 路由）
- 网络策略（NetworkPolicy）

同时，Service 的负载均衡由 **kube-proxy**（iptables/IPVS）实现，不是 CNI 的职责。

---

## 9.2 CNI ★★★★

### 什么是 CNI

**CNI（Container Network Interface）** 是 CNCF 定义的标准网络接口，用于容器运行时和网络插件之间的交互。CNI 插件负责：

- 给 Pod 分配 IP 地址
- 为 Pod 配置网络（创建 veth pair、网桥、路由等）
- Pod 删除时回收 IP 和清理网络

K8s 本身不提供网络实现，通过 CNI 接口让用户自由选择网络方案。

### 三大主流 CNI 插件对比

| 维度 | Flannel | Calico | Cilium |
|------|---------|--------|--------|
| **网络模型** | Overlay（VXLAN/host-gw） | BGP 路由（也可 VXLAN） | eBPF |
| **性能** | 中等（VXLAN 有封装开销） | 高（BGP 路由，少封装） | 最高（eBPF 内核级转发，绕过 iptables） |
| **网络策略** | 不支持 | 支持（丰富的 NetworkPolicy） | 支持（基于 Identity 的策略） |
| **服务网格集成** | 无 | 无 | 内置（Cilium Service Mesh） |
| **可观测性** | 基本 | 好（Flow Log） | 最好（Hubble，L7 可观测） |
| **复杂度** | 低 | 中 | 高 |
| **适用场景** | 简单集群 | 需要 NetworkPolicy | 大规模、可观测性要求高 |

#### Flannel

- 最**简单**的 CNI，适合学习和小集群
- VXLAN 模式：Pod 间跨节点通信通过 VXLAN 隧道封装
- Host-GW 模式：直接在 Node 上添加路由，性能好但要求二层互通
- 不支持 NetworkPolicy

#### Calico

- **最常用**的生产级 CNI
- BGP 模式：在每个 Node 运行 BGP 进程，Node 之间通过 BGP 交换路由，纯三层路由，无隧道封装开销
- IPIP/VXLAN 模式：当底层网络不支持 BGP 时启用隧道封装
- 强大的 NetworkPolicy 支持

#### Cilium

- 基于 **eBPF（extended Berkeley Packet Filter）** 的新一代 CNI
- 数据面绕过 iptables，直接在内核处理网络转发，性能极高
- L7 网络策略（HTTP/gRPC/Kafka 级别）
- 集成 Hubble 实现全面的可观测性（流量可视化、监控）

### 面试题：Calico 和 Flannel 区别？

**标准回答**：

| 维度 | Flannel | Calico |
|------|---------|--------|
| 设计目标 | 简单易用 | 高性能 + 丰富的网络策略 |
| 网络模型 | Overlay（VXLAN） | BGP 路由（三层） |
| 性能 | VXLAN 有封装/解封装开销 | BGP 路由无封装，性能更高 |
| NetworkPolicy | **不支持** | **完整支持** |
| 运维复杂度 | 简单，几乎零配置 | 需配置 BGP（也可 VXLAN 模式） |

**选型建议**：简单环境用 Flannel（学习成本低），生产环境用 Calico（性能 + 安全策略）。Cilium 是未来的方向（eBPF），适合对性能和可观测性有高要求的场景。

---

## 9.3 kube-proxy ★★★★★

### 三种模式对比

| 维度 | userspace（已废弃） | iptables | IPVS |
|------|-------------------|----------|------|
| 转发机制 | kube-proxy 进程转发 | iptables 规则 | IPVS 内核模块 |
| 查找复杂度 | O(1)（但经过用户态） | O(n)（规则链线性匹配） | O(1)（哈希表） |
| 性能 | 低（用户态↔内核态切换） | 中等（规则少时好） | 高（大规模集群） |
| 负载均衡算法 | 随机 | 随机（概率性） | rr/lc/dh/sh/nq/sed 等 |
| Kubernetes 默认 | 否 | **是（kubeadm 默认）** | 否（需显式配置 `--proxy-mode=ipvs`） |

### iptables 模式详解

```
访问 Service ClusterIP:Port
         │
         ▼
PREROUTING → KUBE-SERVICES → KUBE-SVC-XXXX（Service 规则链）
                                   │
                                   │ 概率性选择（statistic 模块）
                                   ▼
                              KUBE-SEP-XXXX（Endpoint 规则链）
                                   │
                                   ▼
                              DNAT → Pod IP:Port
```

- kube-proxy 为每个 Service 创建一条 `KUBE-SVC-XXXX` 链
- 为每个 Endpoint 创建一条 `KUBE-SEP-XXXX` 链
- 使用 `statistic` 模块的 `probability` 实现概率性负载均衡
- **问题**：规则过多时性能线性下降（1000 个 Service × 10 个 Pod = 10000+ 条规则，匹配速度慢）

### IPVS 模式详解

```
访问 Service ClusterIP:Port
         │
         ▼
IPVS 虚拟服务器（Service ClusterIP）
         │
         │ 哈希查找
         ▼
调度算法选择 Real Server（Pod IP:Port）
         │
         ▼
转发到 Pod
```

- 使用 Linux IPVS 内核模块
- 哈希表查找，O(1) 性能，与规则数量无关
- 支持 8 种调度算法：`rr`（轮询）、`lc`（最少连接）、`dh`（目标哈希）、`sh`（源哈希）等
- 需要注意：IPVS 模式下 Service ClusterIP 必须在 kube-ipvs0 接口上绑定，本地访问需要特殊处理

### 面试题：kube-proxy 工作原理？

**标准回答**：

kube-proxy 是每个 Node 上的网络代理，负责实现 Service 的负载均衡：

1. **监听**：通过 List-Watch 监听 API Server 中 Service 和 EndpointSlice 变化
2. **生成规则**：根据变化生成网络转发规则
3. **转发流量**：将到达 Service ClusterIP 的流量转发到后端 Pod

**iptables 模式**：
- 为每个 Service/Endpoint 创建 iptables 规则链
- 通过 statistic 模块概率性选择后端 Pod
- 规则数量多时性能 O(n)，适合中小规模

**IPVS 模式**（推荐）：
- 使用 Linux IPVS 内核模块创建虚拟服务器
- 哈希表 O(1) 查找，性能优于 iptables
- 支持多种调度算法（rr/lc/dh/sh）

**service 流量路径**：Client Pod → Service ClusterIP → kube-proxy 规则（DNAT）→ 后端 Pod IP:Port

---

# 第十章 Kubernetes 高可用 ★★★★★

## 10.1 etcd ★★★★★

### etcd 是什么

**etcd** 是 CoreOS（现属 Red Hat/IBM）开发的**分布式键值存储**，K8s 将**所有集群数据**都存储在 etcd 中：

- 所有 K8s 资源对象（Pod、Service、Deployment、ConfigMap、Secret 等）
- 集群状态、节点信息
- 配置数据

**etcd 是 K8s 集群的唯一状态存储，是集群的"大脑"和"记忆"。**

### Raft 共识算法

etcd 使用 **Raft** 共识算法保证数据一致性：

```
┌──────────────────────────────────────┐
│           Raft 集群 (3 节点)          │
│                                      │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│   │ Leader  │  │Follower │  │Follower│ │
│   │ (Node1) │◄─┤ (Node2) │◄─┤ (Node3)│ │
│   └────┬────┘  └─────────┘  └─────────┘ │
│        │                                │
│   写入流程:                              │
│   ① Client 发送写请求到 Leader          │
│   ② Leader 写入本地日志                  │
│   ③ Leader 复制到 Follower              │
│   ④ 多数节点确认（≥ N/2+1）→ 提交       │
│   ⑤ 返回成功给 Client                  │
└──────────────────────────────────────┘
```

**Raft 核心机制**：

| 概念 | 说明 |
|------|------|
| **Leader 选举** | 只有一个 Leader 处理写请求，Leader 定期发心跳给 Follower |
| **日志复制** | 所有写操作通过 Leader 复制到所有节点日志中 |
| **多数派提交（Quorum）** | 写入需要 ≥ N/2+1 个节点确认才能提交（N=3 时需要 2 个节点）|
| **自动故障恢复** | Leader 宕机后 Follower 自动发起新选举（超时机制）|

**为什么 etcd 集群需要奇数个节点？**

- **3 节点**：容忍 1 个节点故障（2 > 3/2）
- **5 节点**：容忍 2 个节点故障（3 > 5/2）
- 偶数节点不会提高容错能力（4 节点也只能容忍 1 个故障，因为 3 > 4/2 仍是 3）

### 面试题：etcd 为什么可靠？

**标准回答**：

etcd 通过 **Raft 共识算法**实现高可用和强一致性：

1. **多副本**：通常部署 3/5 个节点，容忍少数节点故障
2. **Raft**：任何写操作需多数节点确认才提交，保证数据不丢失
3. **Leader 选举**：Leader 故障后自动选举新 Leader，恢复服务
4. **WAL（Write-Ahead Log）**：所有写操作先写入日志再应用，保证崩溃恢复
5. **快照 + 日志压缩**：定期创建快照，防止日志无限增长

etcd 是 K8s 的"单一事实来源"，它的可靠性直接决定 K8s 集群的可靠性。如果 etcd 不可用，集群将无法创建/更新任何资源（但已运行的 Pod 不受影响）。

---

## 10.2 Controller 工作原理 ★★★★★

### Reconcile 控制循环

K8s 中所有 Controller 都遵循同一个设计模式——**Reconcile Loop（控制循环）**：

```go
// 伪代码
for {
    desiredState := getDesiredState()   // 期望状态（用户声明的 spec）
    actualState := getActualState()     // 实际状态（集群中的 status）
    if desiredState != actualState {
        reconcile(desiredState, actualState) // 执行操作使实际状态趋近期望状态
    }
}
```

这是 K8s **声明式 API** 的底层实现——用户声明"我要 3 个副本"，Controller 不停地比较"现在有几个副本"与"3"，并根据差异执行操作。

### ReplicaSet Controller 示例

```
期望状态: replicas=3
实际状态: running=2（1 个 Pod 挂了）

Controller 逻辑:
  diff = 3 - 2 = 1  → 需要创建 1 个新 Pod
  创建 Pod → 更新 etcd → 等待下一次 reconcile
```

### 面试题：Controller 工作原理？

**标准回答**：

K8s 采用**声明式 API + 控制循环**的模式。每个 Controller 都是一个独立的控制循环：

1. 通过 List-Watch 从 API Server 获取资源的变化
2. 比较资源的 `spec`（期望状态）和 `status`（实际状态）
3. 如果两者不一致，执行 reconcile 操作（创建/更新/删除资源）
4. 操作完成后更新 `status`，等待下一次事件触发

这种设计的好处：
- **天然幂等**：多次执行 reconcile 结果一致，不会因为重复调用产生副作用
- **健壮性**：即使某次 reconcile 失败，下次循环会再次尝试
- **自愈能力**：Controller 不关心 Pod 怎么坏的，只关心现在差几个
- **解耦**：每个 Controller 只管理自己的资源类型
- **可扩展**：任何人都可以编写自定义 Controller（Operator 模式），扩展 K8s 的能力

---

## 10.3 自愈机制 ★★★★★

### K8s 自愈机制层次

```
┌─────────────────────────────────────────────┐
│  第一层：Pod 级别自愈                        │
│  - 容器异常退出 → kubelet 自动重启           │
│  - Liveness Probe 失败 → kubelet 重启容器     │
│  - Readiness Probe 失败 → 摘除 Service 流量   │
└─────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│  第二层：ReplicaSet 级别自愈                  │
│  - Pod 被删除/节点故障 → 自动创建新 Pod       │
│  - 始终保持期望副本数                         │
└─────────────────────────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│  第三层：Node 级别自愈                        │
│  - Node NotReady → Node Controller 驱逐 Pod  │
│  - Node 长时间无响应 → Pod 迁移到健康节点      │
│  - Cluster Autoscaler → 新节点加入            │
└─────────────────────────────────────────────┘
```

### 探针（Probe）

| 探针 | 用途 | 失败后果 |
|------|------|----------|
| **Liveness Probe** | 检测容器是否存活（是否需要重启） | 重启容器 |
| **Readiness Probe** | 检测容器是否准备好接收流量 | 从 Service Endpoints 中移除 |
| **Startup Probe** | 检测容器是否启动完成（慢启动容器） | 失败则重启，成功后才开始 Liveness 检查 |

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30    # 容器启动后 30 秒开始检查
  periodSeconds: 10          # 每 10 秒检查一次
  timeoutSeconds: 5          # 超时 5 秒
  failureThreshold: 3        # 连续失败 3 次才重启

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3        # 连续失败 3 次摘除流量

startupProbe:
  httpGet:
    path: /startup
    port: 8080
  failureThreshold: 30       # 最多检查 30 次
  periodSeconds: 10          # 每 10 秒一次 = 最多等 300 秒
```

### 面试题：K8s 如何实现故障恢复？

**标准回答**：

K8s 有多层自愈机制：

1. **容器级别**：容器崩溃 → kubelet 自动重启（restartPolicy：Always/OnFailure）
2. **探针检查**：
   - Liveness Probe 失败 → 重启容器
   - Readiness Probe 失败 → 暂时摘除 Service 流量，容器不重启
   - Startup Probe 失败 → 重启（保护慢启动应用）
3. **Pod 级别**：ReplicaSet Controller 发现 Pod 数不足 → 自动创建新 Pod
4. **Node 级别**：Node Controller 发现节点 NotReady（默认 5 分钟）→ 驱逐该节点上的 Pod → Scheduler 调度到健康节点
5. **自动扩缩容**：HPA 根据负载自动增加副本数，Cluster Autoscaler 自动增加节点

**关键时间线（Node 故障）**：
- 0s：Node 故障
- ~40s：kubelet 停止上报心跳 → Node 标记为 NotReady
- ~5min（`pod-eviction-timeout`）：Node Controller 开始驱逐 Pod
- ~几分钟：Pod 在健康节点上重新创建

---

# 第十一章 Kubernetes 自动扩缩容 ★★★★★

## 11.1 HPA（Horizontal Pod Autoscaler）

### 定义和工作原理

**HPA** 根据 CPU/内存/自定义指标自动调整 Deployment/StatefulSet 的 Pod 副本数。

```
HPA 工作流程：
  指标采集（Metrics Server/Prometheus Adapter）
       ↓
  HPA Controller 定期查询指标
       ↓
  计算期望副本数：desiredReplicas = ceil(currentReplicas * (currentMetric / desiredMetric))
       ↓
  比较期望副本数与当前副本数
       ↓
  通过 API Server 更新 Deployment replicas
       ↓
  Deployment Controller 执行扩缩容
```

### HPA 配置示例

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70    # 平均 CPU 使用率目标 70%
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80    # 平均内存使用率目标 80%
```

### HPA 缩容保护

- **冷却时间（Cooldown）**：缩容有稳定窗口（`--horizontal-pod-autoscaler-downscale-stabilization`，默认 5 分钟），扩容没有冷却时间（默认 `--horizontal-pod-autoscaler-upscale-stabilization` 为 0）
- 防止频繁扩缩容（flapping）

### 面试题：HPA 如何工作？

**标准回答**：

HPA 是 K8s 的自动扩缩容机制：

1. **指标来源**：从 Metrics Server（资源指标）或 Prometheus Adapter（自定义指标）获取 Pod 指标
2. **计算**：`期望副本数 = ceil(当前副本数 × 当前指标值 / 目标指标值)`
3. **执行**：通过 API Server 更新 Deployment/StatefulSet 的 replicas 字段
4. **冷却**：缩容有 5 分钟稳定窗口（防止抖动），扩容默认无冷却

**前提条件**：
- Pod 必须设置 resource **Requests**（HPA 根据 Requests 计算使用率百分比）
- 部署 Metrics Server（`kubectl top` 依赖）
- 如果是自定义指标，需要 Prometheus Adapter

### K8s 指标管道（Metrics Pipeline）

**这是理解 HPA 和 `kubectl top` 的前提**——指标从哪里来、到哪里去：

```
┌─────────────────────────────────────────────────────────┐
│              资源指标管道（Resource Metrics）            │
│                                                         │
│  kubelet(cAdvisor) ──▶ Metrics Server ──▶ kubectl top   │
│                              │                          │
│                              ▼                          │
│                         HPA (Resource 类型指标)          │
│                         kubectl top pod/node            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│            完整指标管道（Full Metrics Pipeline）          │
│                                                         │
│  kubelet(cAdvisor) ──▶ Prometheus ──▶ Prometheus Adapter│
│                              │              │           │
│                              ▼              ▼           │
│                         Grafana      HPA (自定义指标)     │
│                         (可视化)     (Pods/External类型)  │
└─────────────────────────────────────────────────────────┘
```

| 管道 | 数据源 | 中间层 | 用途 |
|------|--------|--------|------|
| **资源指标管道** | kubelet 内置 cAdvisor | Metrics Server | `kubectl top` + HPA 基于 CPU/内存 |
| **完整指标管道** | cAdvisor + 应用暴露的 Prometheus metrics | Prometheus + Adapter | Grafana 可视化 + HPA 基于自定义指标（QPS、延迟等） |

**面试追问：为什么 HPA 需要 Metrics Server？**

Metrics Server 是 K8s **资源指标管道**的核心组件，它从每个 Node 的 kubelet（内置 cAdvisor）采集 CPU/内存指标，聚合后通过 API Server 的 `/apis/metrics.k8s.io` 接口暴露。HPA Controller 通过这个接口获取 Pod 的 CPU/内存使用量来计算是否扩缩容。**没有 Metrics Server，HPA 无法获取资源指标**。

---

## 11.2 VPA（Vertical Pod Autoscaler）

### 定义

**VPA** 自动调整 Pod 的 **CPU/内存 Requests**（垂直扩缩容），不是调整副本数：

- **Recommender**：根据历史使用情况推荐合理的 Request 值
- **Updater**：驱逐 Pod 使其用新的 Request 重建
- **Admission Controller**：创建新 Pod 时注入推荐值

### 面试题：HPA 和 VPA 区别？

**标准回答**：

| 维度 | HPA | VPA |
|------|-----|-----|
| 方向 | **水平**（增加/减少副本数） | **垂直**（增加/减少单个 Pod 的资源） |
| 调整目标 | replicas | CPU/内存 Requests |
| 适用场景 | 无状态服务（Web 服务） | 有状态服务（数据库）|
| 影响 | 创建/删除 Pod | 重启 Pod（需要重建） |
| 组合使用 | 可以 | HPA+VPA 不建议同时使用（基于资源指标时可能冲突） |

---

## 11.3 Cluster Autoscaler

### 定义

当 Pod 因资源不足无法调度时，Cluster Autoscaler 自动向云厂商申请新节点加入集群；当节点长期利用率低时，自动移除节点。

### 面试题：HPA 和 VPA 区别？

见 11.2 面试题答案。

---

# 第十二章 Kubernetes 资源管理 ★★★★★

## 12.1 Requests（资源请求）

**Requests** 是容器**保证获得**的最小资源量，Scheduler 根据 Requests 来决定 Pod 调度到哪个节点：

```yaml
resources:
  requests:
    cpu: "500m"      # 保证至少 0.5 核 CPU
    memory: "512Mi"  # 保证至少 512Mi 内存
```

- Scheduler 只使用 Requests 做调度决策，不使用 Limits
- 节点总 Requests 超过总容量时，新 Pod 无法调度
- **如果不设置 Requests，K8s 认为 Requests = 0，可能导致节点过载**

## 12.2 Limits（资源限制）

**Limits** 是容器**最多能使用**的资源上限：

```yaml
resources:
  limits:
    cpu: "2000m"     # 最多使用 2 核 CPU（可被 throttle）
    memory: "2048Mi" # 最多使用 2Gi 内存（超过 → OOMKilled）
```

**CPU 超限**：容器可以被 throttle（限制 CPU 时间片），不会杀死
**Memory 超限**：容器直接被 OOMKill（内存无法 throttle）

## 12.3 QoS 等级 ★★★★★

根据 Requests 和 Limits 的设置方式，Pod 被分为三个 QoS 等级：

| QoS 等级 | 条件 | 驱逐优先级 | 说明 |
|----------|------|-----------|------|
| **Guaranteed** | 每个容器的 CPU 和内存都设置了 **Requests == Limits** | 最低（最不容易被驱逐） | 资源完全保障 |
| **Burstable** | 至少一个容器设置了 Requests，但不满足 Guaranteed 条件 | 中等 | 有基本保障 |
| **BestEffort** | 没有任何容器设置 Requests 或 Limits | 最高（最容易被驱逐） | 无保障，使用剩余资源 |

```
驱逐顺序（节点资源不足时）：
  BestEffort → Burstable → Guaranteed（最后被驱逐）
```

### QoS 判断逻辑

```go
// 伪代码
func getQoS(pod) {
    guaranteed := true
    hasRequest := false
    for _, container := range pod.containers {
        if container.resources.requests.cpu == nil || container.resources.requests.memory == nil {
            guaranteed = false
        } else if container.resources.limits.cpu != container.resources.requests.cpu ||
                  container.resources.limits.memory != container.resources.requests.memory {
            guaranteed = false
        }
        if container.resources.requests != nil {
            hasRequest = true
        }
    }
    if guaranteed { return Guaranteed }
    if hasRequest { return Burstable }
    return BestEffort
}
```

### 面试题：QoS 有哪几种等级？

**标准回答**：

K8s 将 Pod 分为三个 QoS 等级，影响 Pod 在资源紧张时的驱逐优先级：

1. **Guaranteed（保障级）**：所有容器都设置了 Requests 且 Requests == Limits。节点 OOM 时最不容易被驱逐
2. **Burstable（弹性级）**：至少一个容器设置了 Requests，但不满足 Guaranteed 条件。有基本资源保障
3. **BestEffort（尽力级）**：没有任何容器设置 Requests 或 Limits。使用节点剩余资源，节点 OOM 时最先被驱逐

**驱逐顺序**：BestEffort → Burstable（按超出量排序） → Guaranteed

**最佳实践**：生产环境关键服务至少设为 Burstable，核心服务（如 etcd）设为 Guaranteed。

---

### 12.4 ResourceQuota（资源配额）★★★

#### 定义

**ResourceQuota** 限制一个 Namespace 的**资源总用量**，防止某个团队/项目占用过多集群资源：

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    # 计算资源限制
    requests.cpu: "20"          # 所有 Pod 的 CPU requests 总和 ≤ 20 核
    requests.memory: "40Gi"     # 所有 Pod 的 memory requests 总和 ≤ 40Gi
    limits.cpu: "40"            # 所有 Pod 的 CPU limits 总和 ≤ 40 核
    limits.memory: "80Gi"       # 所有 Pod 的 memory limits 总和 ≤ 80Gi

    # 对象数量限制
    count/pods: "50"            # 最多 50 个 Pod
    count/services: "20"        # 最多 20 个 Service
    count/secrets: "30"         # 最多 30 个 Secret
    count/configmaps: "40"      # 最多 40 个 ConfigMap
    count/persistentvolumeclaims: "20"  # 最多 20 个 PVC

    # 存储限制
    requests.storage: "500Gi"   # 所有 PVC 请求存储总和 ≤ 500Gi
```

#### 核心作用

| 作用 | 说明 |
|------|------|
| **防止资源抢占** | 团队 A 不会因为 Pod 无限创建而耗尽集群资源 |
| **多租户治理** | 每个 Namespace 分配合理的资源预算 |
| **成本控制** | 限制昂贵资源（GPU、SSD 存储）的使用 |
| **对象爆炸防护** | 限制 ConfigMap/Secret/PVC 等对象的数量 |

#### 与 QoS 的关系

如果创建 Pod 时不指定 Requests/Limits，ResourceQuota 会**拒绝创建**（因为无法统计资源用量）。所以 ResourceQuota 实际上**强制**用户设置合理的资源配置。

---

### 12.5 LimitRange（限制范围）★★★

#### 定义

**LimitRange** 为 Namespace 中的**单个** Pod/容器设置资源上下限，防止创建过大或过小的容器：

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    # 资源上下限
    max:
      cpu: "4"                 # 单个容器 CPU 不能超过 4 核
      memory: "8Gi"            # 单个容器内存不能超过 8Gi
    min:
      cpu: "100m"              # 单个容器 CPU 不能小于 0.1 核
      memory: "128Mi"          # 单个容器内存不能小于 128Mi
    # 默认值（容器不指定 resources 时自动填充）
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "200m"
      memory: "256Mi"
    # limits/requests 的最大比例
    maxLimitRequestRatio:
      cpu: "4"                 # limits.cpu / requests.cpu ≤ 4
      memory: "2"              # limits.memory / requests.memory ≤ 2
```

#### ResourceQuota vs LimitRange

| 维度 | ResourceQuota | LimitRange |
|------|--------------|------------|
| **作用对象** | Namespace 总资源 | **单个** Pod/容器/PVC |
| **限制内容** | 资源总和、对象数量 | 资源上下限、默认值、比例 |
| **核心目的** | 多租户资源预算管理 | 防止极端配置（太大/太小） |
| **典型场景** | 限制团队 A 总共用 20 核 CPU | 限制单个容器最多 4 核，最少 0.1 核 |
| **配合使用** | 宏观约束 | 微观约束 |

---

# 第十三章 Kubernetes 安全 ★★★★

## 13.1 RBAC ★★★★★

### 什么是 RBAC

**RBAC（Role-Based Access Control，基于角色的访问控制）** 是 K8s 中控制用户/ServiceAccount 对资源访问权限的核心机制。

```
谁（Subject） + 能做什么（Verb） + 对什么（Resource）= 权限

User/ServiceAccount → Role/ClusterRole → RoleBinding/ClusterRoleBinding → 权限生效
```

### 四大核心概念

| 概念 | 作用域 | 说明 |
|------|--------|------|
| **Role** | 单个 Namespace | 定义在某个 Namespace 内的权限规则 |
| **ClusterRole** | 整个集群 | 定义集群级别的权限规则（也可用于 Namespace 资源） |
| **RoleBinding** | 单个 Namespace | 将 Role/ClusterRole 绑定到 Subject，作用范围为单个 Namespace |
| **ClusterRoleBinding** | 整个集群 | 将 ClusterRole 绑定到 Subject，作用范围为整个集群 |

### Role 和 RoleBinding 示例

```yaml
# Role: 定义权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]    # 只能查看，不能创建/删除

---
# RoleBinding: 将权限授予用户
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: "alice"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRole 和 ClusterRoleBinding 示例

```yaml
# ClusterRole: 集群级别权限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]

---
# ClusterRoleBinding: 集群级别绑定
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-nodes
subjects:
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

### 常见 RBAC 设计模式

**最小权限原则（Least Privilege）**：只授予完成任务所需的最小权限。

```
开发团队：get, list, watch (只读) + create, update (部署权限) → Role per Namespace
运维团队：* (全部权限) → ClusterRole cluster-admin
CI/CD 系统：create, update, patch, delete (部署相关) → Role per Namespace
监控系统：get, list, watch (只读) → ClusterRole view
```

### 面试题：RBAC 是什么？

**标准回答**：

RBAC 是 K8s 的基于角色的访问控制机制，包含四个核心对象：

1. **Role / ClusterRole**：定义权限规则（能对哪些资源做什么操作）
2. **RoleBinding / ClusterRoleBinding**：将角色绑定到用户/组/ServiceAccount

权限模型：(User/SA/Group) + (RoleBinding → Role) → (verbs on resources)

关键设计：
- Role 是 Namespace 范围的，ClusterRole 是集群范围的
- RoleBinding 可以将 ClusterRole 绑定到某个 Namespace（复用集群角色定义）
- 支持聚合 ClusterRole（`aggregationRule`），多个 ClusterRole 的权限聚合
- 遵循**最小权限原则**

---

## 13.2 ServiceAccount

### 定义

**ServiceAccount** 是 Pod 在 K8s 中的身份标识，用于 Pod 访问 API Server 时的认证：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app-sa    # Pod 使用该 SA
  containers:
  - name: app
    image: myapp
```

- 每个 Namespace 都有默认 ServiceAccount（`default`）
- SA 的 Token 自动挂载到 Pod 的 `/var/run/secrets/kubernetes.io/serviceaccount/token`
- K8s 1.24+ 中 Token 不再自动创建（需手动创建或用 TokenRequest API）
- 结合 RBAC，通过 ServiceAccount + Role/ClusterRole 授予 Pod 特定权限

### 与 User 的区别

| 维度 | User（用户） | ServiceAccount（服务账号） |
|------|-------------|-------------------------|
| 用途 | 人类用户 | Pod 中的进程/应用 |
| 管理 | 外部身份（证书/OIDC） | K8s 内部资源 |
| 命名空间 | 全局 | Namespace 范围 |
| kubectl 操作 | `kubectl config set-credentials` | `kubectl create serviceaccount` |

---

## 13.3 NetworkPolicy

### 定义

**NetworkPolicy** 定义 Pod 间的网络通信规则（类似 Pod 级别的"防火墙"）：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: production
spec:
  podSelector: {}           # 选择所有 Pod（空 = 全部）
  policyTypes:
  - Ingress
  ingress: []               # 拒绝所有入站流量（空规则列表 = 拒绝所有）
```

**默认行为**：
- **没有 NetworkPolicy 时**：所有 Pod 之间可以自由通信
- **有一条 NetworkPolicy 后**：默认拒绝所有，只允许策略中明确允许的流量

**重要前提**：NetworkPolicy 需要 CNI 插件支持（Calico 支持，Flannel 不支持）。

### 面试题（略，了解即可）

---

## 13.4 Pod Security Admission（PSA）★★★★

### 定义

**Pod Security Admission（PSA）** 是 K8s 1.25+ 内置的 Pod 安全准入控制器，取代了已废弃的 PodSecurityPolicy（PSP）。它在 Namespace 级别定义 Pod 的安全标准，由 API Server 的 Admission Controller 直接执行。

### 三种安全级别

```
Privileged ──▶ Baseline ──▶ Restricted
   (宽松)        (中等)        (严格)
```

| 级别 | 说明 | 允许的安全上下文 | 适用场景 |
|------|------|-----------------|----------|
| **Privileged** | 无限制 | 所有（privileged 容器、hostPath、hostNetwork 等） | 系统组件 NS（kube-system） |
| **Baseline** | 防止已知提权 | 禁止 hostPath、hostNetwork、privileged 容器等 | 大多数应用 / 过渡期 |
| **Restricted** | 遵循 Pod 安全最佳实践 | 在 Baseline 基础上进一步限制（如必须 runAsNonRoot、禁止 capabilities） | 安全敏感环境 |

### 配置方式（Namespace 级别标签）

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # 预防策略（enforce）：违反直接拒绝
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.27

    # 审计策略（audit）：违反记录到审计日志但不拒绝
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/audit-version: v1.27

    # 警告策略（warn）：违反给用户返回警告
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: v1.27
```

### 三个策略模式

| 模式 | 行为 | 用途 |
|------|------|------|
| **enforce** | 违反 → **拒绝**创建 Pod | 强制生效 |
| **audit** | 违反 → 记录审计日志，但不拒绝 | 评估影响范围 |
| **warn** | 违反 → 返回警告给用户 | 给开发者提示整改 |

### PSA vs PSP（为什么替换）

| 维度 | PodSecurityPolicy (已废弃) | Pod Security Admission |
|------|--------------------------|----------------------|
| **配置复杂度** | 高（需要 RBAC + PSP 绑定） | 低（Namespace Label 即可） |
| **启用** | 需额外 Admission Controller | K8s 1.23+ 内置，1.25+ GA |
| **细粒度** | 非常细（每字段控制） | 三级简化（Privileged/Baseline/Restricted） |
| **用户体验** | PSP 权限错误令人困惑 | 明确返回违反的字段和原因 |
| **迁移成本** | — | 低（逐 NS 加 label 渐进式迁移） |

### 面试题：Pod Security Admission 是什么？

**标准回答**：

Pod Security Admission（PSA）是 K8s 1.23+ 内置的 Pod 安全准入控制器，在 Namespace 级别通过 Label 定义三种安全级别（Privileged / Baseline / Restricted），配合三种策略模式（enforce / audit / warn）来实现 Pod 的安全生产准入。它取代了已废弃的 PodSecurityPolicy（PSP），极大简化了配置复杂度——从"每个用户绑定 PSP + RBAC"变为"给 Namespace 打 Label"。

**关键点**：PSA 是声明式的、Namespace 粒度的、渐进式可迁移的——可以先用 `audit` 和 `warn` 评估影响，再逐步开启 `enforce`。

---

## 13.5 Admission Webhooks（准入 Webhook）★★★★

### 定义

**Admission Webhook** 是 K8s 准入控制（Admission Control）的扩展机制，允许用户在资源**持久化到 etcd 之前**执行自定义的校验或修改逻辑。Admission Webhook 是 K8s 实现安全合规和自动化注入的核心手段。

### 准入控制流程（三层防线）

```
API 请求
    │
    ▼
① Authentication（认证）：你是谁？
    │
    ▼
② Authorization（授权）：你能做什么？（RBAC）
    │
    ▼
③ Admission Control（准入控制）：
    │
    ├── Mutating Webhooks（修改型）   ← 可以修改资源（如注入 Sidecar）
    │     按字母顺序串行调用
    │
    ├── Object Schema Validation      ← API Server 内建校验
    │
    └── Validating Webhooks（验证型）  ← 只能通过/拒绝（如检查合规性）
          按字母顺序串行调用
          任何一个拒绝 → 请求被拒绝
    │
    ▼
持久化到 etcd
```

### 两种 Webhook 对比

| 维度 | MutatingAdmissionWebhook | ValidatingAdmissionWebhook |
|------|--------------------------|---------------------------|
| **执行时机** | 先执行（在 Validating 之前） | 后执行 |
| **能力** | **修改**资源对象 | **通过/拒绝**请求，不能修改 |
| **典型场景** | 注入 Sidecar（Istio）、注入环境变量、添加默认资源 Limits | 安全合规检查、镜像来源验证、资源配额硬限制 |
| **调用顺序** | 多个 Mutating Webhook 串行调用 | 多个 Validating Webhook 串行调用 |
| **失败策略** | `Fail`：Webhook 不可用 → 拒绝请求；`Ignore`：Webhook 不可用 → 跳过 |

### 配置示例

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
- name: sidecar-injector.example.com
  clientConfig:
    service:
      name: sidecar-injector-svc
      namespace: default
      path: "/mutate"
    caBundle: <base64-encoded-ca-cert>
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  failurePolicy: Fail          # Webhook 挂了 → 拒绝所有 Pod 创建
  namespaceSelector:
    matchLabels:
      sidecar-injection: enabled  # 只在打了标签的 NS 中生效
  sideEffects: None
```

### 使用 Webhook 的典型场景

| 场景 | Webhook 类型 | 说明 |
|------|-------------|------|
| **Istio Sidecar 注入** | Mutating | 自动注入 `istio-proxy` 容器 |
| **镜像仓库白名单** | Validating | 拒绝使用非白名单镜像仓库的 Pod |
| **强制资源 Limits** | Mutating | Pod 未指定 Limits 时自动添加默认值 |
| **禁止 latest tag** | Validating | 拒绝使用 `:latest` 镜像 |
| **Pod Security 准入** | Validating | K8s 1.25+ 替代 PSP（PodSecurityPolicy） |
| **自动添加 Toleration** | Mutating | 系统自动添加节点容忍 |
| **密钥注入** | Mutating | 从 Vault/AWS Secrets Manager 注入密钥到 Pod |

### 面试题：Admission Webhook 是什么？Mutating 和 Validating 区别？

**标准回答**：

Admission Webhook 是 K8s 准入控制的扩展机制，在资源持久化到 etcd **之前**进行拦截：

1. **Mutating Webhook**：先执行，可以**修改**资源对象（如 Istio 注入 Envoy Sidecar、自动添加默认资源 Limits）
2. **Validating Webhook**：后执行，只能**通过或拒绝**请求（如镜像仓库白名单、禁止 latest tag、安全合规检查）

**关键点**：
- Webhook 是一个 HTTPS 回调——API Server 将 AdmissionReview 请求发送到外部 Webhook 服务，等待响应
- `failurePolicy` 决定 Webhook 不可用时的行为：`Fail`（拒绝请求，安全优先）vs `Ignore`（跳过检查，可用性优先）
- 这是 K8s **可扩展性** 的基石之一——允许你在不修改 K8s 源码的前提下，自定义集群的安全策略

---

# 第十四章 Kubernetes 运维与排查 ★★★★★

## 14.1 kubectl 常用命令

### 资源查看

```bash
# 查看所有资源
kubectl get pods                   # 查看 Pods
kubectl get pods -o wide           # 显示更多信息（IP、Node）
kubectl get pods -n <namespace>    # 查看指定命名空间
kubectl get pods --all-namespaces  # 查看所有命名空间
kubectl get pods -l app=nginx      # 通过标签过滤  -l 是 --selector
kubectl get pods -o yaml           # 以 YAML 格式输出
kubectl get pods -o json           # 以 JSON 格式输出
kubectl get pods -w                # Watch 模式，实时查看变化

kubectl get all                    # 查看所有常用资源
kubectl get deploy,svc,cm,secret   # 查看多种资源
```

### 详情查看

```bash
kubectl describe pod <pod-name>     # 查看 Pod 详细信息（含 Events）
kubectl describe node <node-name>   # 查看节点信息
kubectl describe deployment <name>  # 查看 Deployment 信息

kubectl logs <pod-name>             # 查看 Pod 日志（单容器）
kubectl logs <pod-name> -c <container>  # 查看指定容器日志
kubectl logs <pod-name> --tail=50   # 查看最后 50 行
kubectl logs <pod-name> -f          # 实时跟踪日志
kubectl logs <pod-name> --previous  # 查看上一个崩溃容器的日志（排查 CrashLoopBackOff）

kubectl top pods                    # 查看 Pod 资源使用（需要 Metrics Server）
kubectl top nodes                   # 查看 Node 资源使用
```

### 操作命令

```bash
kubectl create -f pod.yaml          # 创建资源
kubectl apply -f deployment.yaml    # 声明式创建/更新
kubectl delete -f pod.yaml          # 删除资源
kubectl delete pod <name>           # 按名称删除
kubectl delete pod <name> --force --grace-period=0  # 强制立即删除

kubectl exec -it <pod> -- /bin/bash # 进入容器
kubectl exec -it <pod> -- /bin/sh   # 进入容器（无 bash）

kubectl port-forward pod/<name> 8080:80  # 端口转发到本地

kubectl cp <pod>:/path/file ./local-file # 从 Pod 拷贝文件

kubectl edit deployment/<name>      # 在线编辑资源
kubectl scale deployment/<name> --replicas=5  # 扩缩容

kubectl rollout history deployment/<name>    # 查看部署历史
kubectl rollout undo deployment/<name>       # 回滚
kubectl rollout restart deployment/<name>    # 重启所有 Pod（滚动重启）
kubectl rollout pause deployment/<name>      # 暂停滚动更新（金丝雀发布）
kubectl rollout resume deployment/<name>     # 恢复滚动更新（金丝雀发布继续）

kubectl set image deployment/<name> app=myapp:v2  # 直接更新镜像（比 edit 更快）

kubectl label node <node> key=value  # 给节点打标签
kubectl taint nodes <node> key=value:NoSchedule # 给节点打污点

kubectl cordon <node>               # 标记节点不可调度
kubectl drain <node> --ignore-daemonsets  # 驱逐节点上的所有 Pod
kubectl uncordon <node>              # 恢复节点可调度
```

---

## 14.2 Pod 故障排查 ★★★★★

### CrashLoopBackOff

**现象**：Pod 反复启动 → 崩溃 → 重启 → 崩溃

**排查步骤**：

```bash
# 1. 查看 Pod 状态和重启次数
kubectl describe pod <name> | grep -A 10 "State:"

# 2. 查看上一个崩溃容器的日志（关键！）
kubectl logs <pod-name> --previous

# 3. 查看当前日志
kubectl logs <pod-name>

# 4. 查看 Events
kubectl describe pod <name> | grep -A 30 "Events:"
```

**常见原因**：

| 原因 | 解决方案 |
|------|----------|
| 应用代码 panic/异常退出 | 修复代码 bug |
| 环境变量/配置不完整 | 检查 ConfigMap/Secret 是否正确挂载 |
| 依赖服务不可用（数据库等） | 增加 Init Container 等待依赖，或应用增加重试 |
| OOMKilled（内存超限） | 增加内存 Limits 或优化内存使用 |
| 启动命令错误 | 检查 `command` 和 `args` 配置 |
| 健康检查配置不当 | 调整 Liveness Probe 的 initialDelaySeconds |

### ImagePullBackOff / ErrImagePull

**现象**：Pod 无法拉取镜像

**排查步骤**：

```bash
# 查看具体错误
kubectl describe pod <name> | grep -A 5 "Failed"
```

**常见原因**：

| 原因 | 解决方案 |
|------|----------|
| 镜像 tag 不存在 | 确认镜像 tag 正确 |
| 镜像仓库需要认证 | 创建 `imagePullSecrets` |
| 网络不通 | 检查 Node 到镜像仓库的网络连通性 |
| 仓库地址错误 | 确认 image 名称完整（私有仓库需加完整地址） |

### OOMKilled

**现象**：Pod 被系统杀死，状态显示 OOMKilled

**原因**：容器使用的内存超过了 Limits 设定的上限

**排查**：

```bash
kubectl describe pod <name> | grep "OOMKilled"
kubectl top pods
```

**解决**：
- 增加 `memory.limits`
- 优化应用内存使用
- 排查内存泄漏

### Pending

见下文场景题 2。

### 面试题：Pod 一直 Pending 怎么排查？

见第 17 章场景题 2。

---

## 14.3 服务无法访问排查 ★★★★★

### 排查链路

服务不可用排查遵循从外到内、从逻辑到物理的顺序：

```
Ingress → Service → Endpoints → Pod → 应用本身
```

**排查清单**：

```bash
# 1. Pod 是否正常运行？
kubectl get pods -l app=<label>

# 2. Pod 的 Readiness 是否就绪？（READY 列 1/1）
kubectl get pods

# 3. Service 是否有 Endpoints？
kubectl get endpoints <service-name>
# Endpoints 为空 → Service selector 与 Pod labels 不匹配

# 4. Service 的端口映射是否正确？
kubectl describe service <service-name>
# 确认 Port 和 TargetPort 正确

# 5. 从另一个 Pod 测试网络连通性
kubectl run test --rm -it --image=busybox -- wget -O- http://<service>:<port>

# 6. 检查 DNS 解析
kubectl run test --rm -it --image=busybox -- nslookup <service>.<namespace>.svc.cluster.local

# 7. 检查 NetworkPolicy
kubectl get networkpolicies

# 8. Ingress 是否正确配置？
kubectl describe ingress <name>
kubectl logs -n ingress-nginx <ingress-controller-pod>
```

### 常见故障原因速查表

| 现象 | 可能原因 | 排查命令 |
|------|----------|----------|
| Service 无 Endpoints | selector 不匹配 | `kubectl get endpoints` |
| Service DNS 解析失败 | CoreDNS 故障 | `kubectl get pods -n kube-system \| grep coredns` |
| 跨 Namespace 无法访问 | 使用了不完整的 service 名 | 使用完整 FQDN: `<svc>.<ns>.svc.cluster.local` |
| NodePort 外部无法访问 | Node 安全组/防火墙拦截 | 检查云厂商安全组规则 |
| Ingress 无响应 | Ingress Controller 未运行 | `kubectl get pods -n ingress-nginx` |

---

## 14.4 etcd 故障排查（略，了解即可）

etcd 故障通常表现为 API Server 不可用（无法 kubectl）：
- 检查 etcd 集群成员状态：`etcdctl member list`
- 检查 Raft 集群健康：`etcdctl endpoint health`
- 常见故障：多数节点宕机（丧失 Quorum）、磁盘满、证书过期

---

## 14.5 节点 NotReady 排查

```bash
# 查看节点状态
kubectl get nodes

# 查看节点详细信息
kubectl describe node <node-name>

# 关键检查项：
# - Conditions: MemoryPressure / DiskPressure / PIDPressure / Ready
# - kubelet 是否运行？
# - 容器运行时是否正常？
# - 网络是否正常？
```

**常见原因**：

| 原因 | 检查方法 |
|------|----------|
| kubelet 挂了 | SSH 到 Node：`systemctl status kubelet` |
| 磁盘满 | `df -h`，检查 `/var/lib/kubelet` |
| 内存不足导致 kubelet OOM | `dmesg \| grep -i oom` |
| Docker/containerd 故障 | `systemctl status containerd` |
| CNI 插件故障 | 检查 CNI Pod 日志 |
| 节点网络不通 | `ping` API Server |

---

## 14.6 K8s Events —— 排查的第一入口 ★★★★

### Events 是什么

**Events** 是 K8s 对资源状态变化的**时间线记录**。每当资源发生重要变化（Pod 被调度、镜像拉取完成、探针失败、节点失联），相关组件会向 API Server 写入一条 Event。Events 是 `describe` 命令的最终信息来源。

### Events 特性

| 特性 | 说明 |
|------|------|
| **资源关联** | 每一条 Event 关联到一个具体的 K8s 资源（通过 `involvedObject` 字段） |
| **保留时间** | 默认**保留 1 小时**（`--event-ttl`），过期自动清除 |
| **不可变性** | Event 写入后不可修改（immutable） |
| **写入者** | kubelet、Scheduler、Controller Manager、用户都可以写入 Event |

### Event 结构

```bash
kubectl describe pod myapp-7d8f9c5b-abc12 | grep -A 30 "Events:"

# Events:
#   Type    Reason     Age   From               Message
#   ----    ------     ----  ----               -------
#   Normal  Scheduled  5m    default-scheduler  Successfully assigned default/myapp to node-1
#   Normal  Pulling    5m    kubelet            Pulling image "myapp:v1.0"
#   Normal  Pulled     4m    kubelet            Successfully pulled image
#   Normal  Created    4m    kubelet            Created container myapp
#   Normal  Started    4m    kubelet            Started container myapp
#   Warning  Unhealthy  1m    kubelet            Readiness probe failed: dial tcp 10.244.1.5:8080
```

| 字段 | 含义 |
|------|------|
| **Type** | `Normal`（正常事件）或 `Warning`（异常事件） |
| **Reason** | 机器可读的原因代码（`Pulling`、`Pulled`、`Scheduled` 等） |
| **From** | 哪个组件发出的（`kubelet`、`default-scheduler` 等） |
| **Message** | 人类可读的详细信息 |

### Event 常用命令

```bash
# 看所有 Namespace 的 Events（全局视角，时间倒序）
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# 看某个 Namespace 的 Events
kubectl get events -n default

# 只看 Warning（快速定位问题）
kubectl get events --all-namespaces --field-selector type=Warning

# 看某个资源的 Events
kubectl get events --field-selector involvedObject.name=myapp-pod

# Watch 模式（实时盯着变化）
kubectl get events -w
```

### 面试题：Pod 出问题了，Events 和日志有什么区别？

**标准回答**：

| Events | kubectl logs |
|--------|-------------|
| 告诉你**K8s 对这个 Pod 做了什么**（调度、拉镜像、探针失败） | 告诉你**应用代码输出了什么** |
| 由 K8s 组件写入（kubelet / Scheduler） | 由容器进程写入（stdout/stderr） |
| 回答"Pod 为什么创建失败了" | 回答"应用代码报什么错了" |
| 保留 1 小时 | 取决于日志采集系统 |

**排查顺序**：先看 Events 定位问题类型 → 再看 Logs 定位应用层根因。Event 告诉你"怎么死的"，Log 告诉你"为什么死的"。

---

# 第十四章扩展：后端开发日常 K8s 操作手册 ★★★★★

> 本章聚焦**后端开发工程师的日常工作流**——不是你该背哪些命令，而是每次遇到某个场景时你该怎么做。
>
> 适用于：日常开发、联调测试、排查问题、发布上线。

---

## 补充 1：看——查看应用状态（最高频）

### 场景：我刚部署了服务，怎么确认它跑起来了？

```bash
# 第一步：看 Pod 有没有在 Running，READY 列是 n/n
kubectl get pods -l app=myapp

# 输出示例：
# NAME                      READY   STATUS    RESTARTS   AGE
# myapp-7d8f9c5b-abc12      1/1     Running   0          30s

# 第二步：如果有 Pod 不是 Running，看发生了什么
kubectl describe pod myapp-7d8f9c5b-abc12
# 重点看最后的 Events 部分——
#   "Pulling image" → 在拉镜像，等着
#   "Failed to pull image" → 镜像问题
#   "0/3 nodes are available" → 资源不够或调度约束不满足

# 第三步：看应用日志，确认它没报错
kubectl logs -l app=myapp --tail=50
```

### 场景：Pod 运行了，但我想知道它用了多少资源

```bash
# 看 Pod 的 CPU/内存实时用量
kubectl top pods -l app=myapp

# 看具体哪个容器吃得多
kubectl top pods <pod-name> --containers

# 看节点还剩多少资源（决定能不能再部署）
kubectl top nodes
```

### 场景：我想知道当前有哪些 Pod、哪些 Service，一目了然

```bash
# 看当前 Namespace 的所有资源
kubectl get all

# 看指定 app 的全部关联资源
kubectl get all -l app=myapp

# 宽格式——显示 IP 和 Node 信息
kubectl get pods -o wide

# 按某个字段排序（如按重启次数）
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'
```

---

## 补充 2：改——更新和部署（日常发布）

### 场景：我要更新镜像版本

```bash
# 方式 1：直接改镜像（最快，适合开发环境）
kubectl set image deployment/myapp myapp=myapp:v1.2.3

# 方式 2：在线编辑（适合改多个字段）
kubectl edit deployment/myapp

# 方式 3：apply 新的 YAML（最规范，适合生产环境）
kubectl apply -f deployment-v2.yaml

# 方式 4：使用 kustomize / helm 更新（团队规范通常用这个）
helm upgrade myapp ./chart --set image.tag=v1.2.3
kustomize build ./overlays/prod | kubectl apply -f -
```

### 场景：更新过程中我想盯着看，确认不出问题

```bash
# 实时看滚动更新状态
kubectl rollout status deployment/myapp

# 输出示例：Waiting for rollout to finish: 1 old replicas are pending termination...
#         → deployment "myapp" successfully rolled out

# 同时开另一个终端看 Pod 变化
kubectl get pods -l app=myapp -w
# -w 是 watch 模式，Pod 状态变化会实时刷新
```

### 场景：更新出问题了，我要回滚

```bash
# 看有哪些历史版本
kubectl rollout history deployment/myapp

# 看某个版本的 Pod 模板详情
kubectl rollout history deployment/myapp --revision=3

# 回滚到上一个版本
kubectl rollout undo deployment/myapp

# 回滚到指定版本
kubectl rollout undo deployment/myapp --to-revision=2

# 回滚后盯一下状态
kubectl rollout status deployment/myapp
```

### 场景：我要暂停更新做金丝雀验证

```bash
# 先发布新版本，但只创建 1 个新 Pod 就暂停
kubectl set image deployment/myapp myapp=myapp:v2.0
kubectl rollout pause deployment/myapp  # 立即暂停！

# 等新 Pod 就绪后验证（看日志、打流量、检查指标）
kubectl logs <new-pod-name>
curl http://<new-pod-ip>:8080/health

# 确认 OK → 继续滚动
kubectl rollout resume deployment/myapp

# 确认有问题 → 回滚
kubectl rollout undo deployment/myapp
```

---

## 补充 3：调——调试和诊断（日常最花时间的部分）

### 场景：我想进容器里看看

```bash
# 标准做法：开 shell
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- /bin/sh    # 有些镜像没有 bash

# 进指定容器（Pod 里有多个容器时）
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# 只执行一条命令，不进交互式 shell
kubectl exec <pod-name> -- curl -s localhost:8080/health
kubectl exec <pod-name> -- cat /etc/config/app.yaml
kubectl exec <pod-name> -- env | grep DB_
```

### 场景：我的服务还没暴露出去，我想本地调一下

```bash
# 端口转发——把 Pod 的端口映射到本地
kubectl port-forward pod/<pod-name> 8080:8080
# 现在 curl localhost:8080 就是直接访问 Pod 了

# 转发 Service
kubectl port-forward svc/<service-name> 5432:5432
# 本地连 localhost:5432 等于连上了 K8s 里的数据库 Service

# 后台运行，方便长期调试
kubectl port-forward svc/myapp 8080:8080 &
```

### 场景：只是临时测试一下 Service 是否通

```bash
# 用临时 Pod 测试网络连通性（用完即删）
kubectl run test-conn --rm -it --image=busybox --restart=Never -- \
  wget -O- http://myapp-svc.default.svc.cluster.local:8080/health

# 测试 DNS 解析
kubectl run test-dns --rm -it --image=busybox --restart=Never -- \
  nslookup myapp-svc.default.svc.cluster.local

# 测试 TCP 端口通不通
kubectl run test-port --rm -it --image=busybox --restart=Never -- \
  nc -zv myapp-svc.default.svc.cluster.local 8080
```

### 场景：容器一直崩溃，我要看上一次崩溃的日志

```bash
# 这是 CrashLoopBackOff 排查的第一步！
kubectl logs <pod-name> --previous

# 如果容器 crash 太快日志来不及写，加 -f 追
kubectl logs <pod-name> --previous -f
```

### 场景：极简镜像（distroless）进不去，怎么调试？

```bash
# kubectl debug 注入一个带工具的临时容器
kubectl debug -it <pod-name> --image=nicolaka/netshoot --target=<container-name>

# 进入后你可以：
#   strace -p 1           # 跟踪主进程的系统调用
#   tcpdump -i eth0        # 抓包看网络流量
#   curl localhost:8080     # 从 Pod 内部测试端口
#   cat /proc/1/status     # 看主进程状态
```

---

## 补充 4：配——配置和密钥管理

### 场景：我要新增配置项或修改配置

```bash
# 从文件创建 ConfigMap
kubectl create configmap myapp-config --from-file=app.yaml --dry-run=client -o yaml | kubectl apply -f -

# 从字面量创建（简单的 key=value）
kubectl create configmap myapp-config --from-literal=LOG_LEVEL=debug --from-literal=DB_HOST=mysql

# 修改已有的 ConfigMap
kubectl edit configmap myapp-config

# 创建密钥（密码类）
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=MyP@ssw0rd

# 从文件创建 TLS 密钥
kubectl create secret tls my-tls --cert=cert.pem --key=key.pem

# 查看 Secret 的值（base64 解码）
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

### 场景：ConfigMap 更新后 Pod 没生效

```bash
# 环境变量方式注入的 → 必须重启 Pod 才能生效！
kubectl rollout restart deployment/myapp

# Volume 挂载方式 → 等 1 分钟左右 kubelet 会自动同步到文件
# 如果没生效 → 检查是否用了 subPath（subPath 不会自动更新）
kubectl exec <pod-name> -- cat /etc/config/app.yaml  # 看文件内容确认
```

---

## 补充 5：扩——扩缩容操作

### 场景：流量上来了，我要多加几个副本

```bash
# 手动扩缩容
kubectl scale deployment/myapp --replicas=5

# 看扩容是否到位
kubectl get pods -l app=myapp -w

# 查看 HPA 状态和当前指标
kubectl get hpa
kubectl describe hpa myapp-hpa
```

### 场景：我要重启某个服务（不是更新，只是重启）

```bash
# 滚动重启——一个一个 Pod 重建，不影响可用性（推荐）
kubectl rollout restart deployment/myapp

# 粗暴方式——直接删 Pod（ReplicaSet 会自动创建新的）
kubectl delete pod <pod-name>   # 不推荐，因为 Pod 立刻没了
```

---

## 补充 6：实战工作流——从开发到上线的完整操作链

### 工作流 1：本地开发 → 部署到测试环境 → 联调

```bash
# 1. 构建镜像（tag 用 commit hash，不用 latest）
docker build -t registry.example.com/myapp:$(git rev-parse --short HEAD) .
docker push registry.example.com/myapp:$(git rev-parse --short HEAD)

# 2. 部署到 K8s 测试 Namespace
kubectl set image deployment/myapp myapp=registry.example.com/myapp:$(git rev-parse --short HEAD) -n staging

# 3. 盯着更新完成
kubectl rollout status deployment/myapp -n staging

# 4. 看日志确认没报错
kubectl logs -l app=myapp -n staging --tail=100

# 5. 端口转发到本地联调
kubectl port-forward svc/myapp 8080:8080 -n staging

# 6. 本地用 curl / Postman 调接口
curl localhost:8080/api/users
```

### 工作流 2：生产环境发布（谨慎操作）

```bash
# 1. 先看当前状态
kubectl get deploy,svc,ingress -l app=myapp -n production

# 2. 备份当前版本号（万一要回滚）
kubectl get deployment myapp -n production -o jsonpath='{.spec.template.spec.containers[0].image}'
# → myapp:v1.5.0

# 3. 更新镜像
kubectl set image deployment/myapp myapp=myapp:v1.6.0 -n production

# 4. 盯着滚动更新
kubectl rollout status deployment/myapp -n production

# 5. 检查新 Pod 健康状态
kubectl get pods -l app=myapp -n production -o wide

# 6. 抽查新 Pod 日志
kubectl logs <new-pod> -n production --tail=20

# 7. 如果发现问题 → 立刻回滚
kubectl rollout undo deployment/myapp -n production
kubectl rollout status deployment/myapp -n production
```

### 工作流 3：灰度/金丝雀发布（配合 pause/resume）

```bash
# 1. 更新镜像
kubectl set image deployment/myapp myapp=myapp:v2.0

# 2. 立即暂停（只创建 1 个新 Pod）
kubectl rollout pause deployment/myapp

# 3. 验证新 Pod
NEW_POD=$(kubectl get pods -l app=myapp --sort-by=.metadata.creationTimestamp -o name | tail -1)
kubectl logs $NEW_POD --tail=20

# 4. 给新 Pod 单独打流量验证
kubectl port-forward $NEW_POD 8081:8080
curl localhost:8081/api/health

# 5. OK → 全量滚动
kubectl rollout resume deployment/myapp
kubectl rollout status deployment/myapp

# 5'. 有问题 → 回滚
kubectl rollout undo deployment/myapp
```

### 工作流 4：紧急排查——用户报障后第一步做什么

```bash
# 1. 确定影响范围：哪个 Namespace、哪个 Service、哪个 Deployment
kubectl get pods --all-namespaces --field-selector=status.phase!=Running

# 2. 查看最近的集群事件（全局视角）
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20

# 3. 找到问题 Pod，看 Events 和日志
kubectl describe pod <problem-pod>
kubectl logs <problem-pod> --tail=100
kubectl logs <problem-pod> --previous   # 如果 Pod 刚重启过

# 4. 检查资源是否耗尽
kubectl top nodes
kubectl top pods -l app=<app-label>

# 5. 检查依赖是否正常（Service、ConfigMap、Secret 是否存在）
kubectl get svc,endpoints,configmap,secret -n <namespace>

# 6. 快速扩容器维持服务可用
kubectl scale deployment <name> --replicas=<n+2> -n <namespace>
```

---

## 补充 7：必知的 kubectl 小技巧

### 提高效率的别名和插件

```bash
# 常用别名（写入 ~/.bashrc 或 ~/.zshrc）
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias kpf='kubectl port-forward'
alias krs='kubectl rollout restart deployment'

# 用 k 代替 kubectl
k get pods          # 省 7 个字母
k get deploy,svc    # 一次看多种资源

# kubectl 插件管理工具 krew
kubectl krew install ctx     # 快速切换集群
kubectl krew install ns      # 快速切换 Namespace
kubectl krew install tail    # 多 Pod 日志聚合查看

# 用法：
kubectl ctx                    # 列出所有集群，上下键选择
kubectl ns                     # 列出所有 Namespace，上下键选择
kubectl tail -l app=myapp      # 同时看所有 myapp Pod 的日志
```

### 不懂资源怎么写？用 explain 查文档

```bash
# 不需要开浏览器查文档，kubectl 内置 API 文档
kubectl explain pods
kubectl explain pods.spec
kubectl explain pods.spec.containers
kubectl explain deployment.spec.strategy.rollingUpdate

# 带 --recursive 看字段树
kubectl explain pods --recursive
```

### 看看集群有哪些 API 资源可用

```bash
# 列出所有 API 资源（包括 CRD）
kubectl api-resources

# 只列出带特定名字的
kubectl api-resources | grep -i autoscaling

# 看资源的 shortNames 和 API Group
kubectl api-resources -o wide

# 列出所有 API 版本
kubectl api-versions
```

### RBAC 调试：我能做什么？

```bash
# 当前用户能不能在某个 NS 创建 Pod？
kubectl auth can-i create pods -n default

# 当前用户能不能删某个 Deployment？
kubectl auth can-i delete deployment/myapp -n production

# ServiceAccount 能不能 list 某个资源？
kubectl auth can-i get pods --as=system:serviceaccount:default:myapp-sa

# 列出当前用户的所有权限（管理员才能用）
kubectl auth can-i --list
```

### 安全发布：apply 前先 diff

```bash
# 看看 apply 会改什么（不实际执行）
kubectl diff -f deployment.yaml

# 输出示例：
# -  image: myapp:v1.5.0
# +  image: myapp:v1.6.0
# 一看就知道这次改动会更新镜像版本，确认无误后再 apply
kubectl apply -f deployment.yaml
```

### 清理无用资源：apply 的 prune 模式

```bash
# apply 的同时清理 YAML 中已不存在的资源
kubectl apply -f ./manifests/ --prune \
  -l app=myapp \
  --prune-whitelist=apps/v1/Deployment \
  --prune-whitelist=v1/Service \
  --prune-whitelist=v1/ConfigMap
# 这样从 YAML 目录删掉的文件，对应的 K8s 资源也会被清理
```

### 快速获取某个字段（不用 grep 一大串）

```bash
# jsonpath 提取特定字段
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
# → 10.244.1.5 10.244.2.8 10.244.3.3

# 获取所有 Pod 的镜像版本
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].image}{"\n"}{end}'

# 获取 Deployment 当前副本数
kubectl get deploy myapp -o jsonpath='{.status.readyReplicas}'

# 获取 Secret 值并解码
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d

# 如果你装了 jq（推荐）
kubectl get pods -o json | jq '.items[] | {name: .metadata.name, ip: .status.podIP}'
```

### 批量操作

```bash
# 重启某个 Namespace 下所有的 Deployment
kubectl get deploy -n staging -o name | xargs kubectl rollout restart -n staging

# 删除所有 Evicted 状态的 Pod
kubectl get pods --all-namespaces --field-selector=status.phase=Failed \
  -o json | jq -r '.items[] | "\(.metadata.namespace) \(.metadata.name)"' | \
  while read ns pod; do kubectl delete pod $pod -n $ns; done

# 复制文件到/从 Pod
kubectl cp <pod-name>:/app/logs/app.log ./app.log     # 从 Pod 拷出
kubectl cp ./config.yaml <pod-name>:/app/config.yaml  # 拷入 Pod
```

---

## 补充 8：常见操作事故及预防

| 操作 | 危险后果 | 正确做法 |
|------|----------|----------|
| `kubectl delete pod` 在生产 NS | Pod 立刻没了，正在处理的请求丢失 | 用 `kubectl rollout restart` 滚动重启 |
| `kubectl apply` 贴错 YAML | 误改 Service selector 导致流量全断 | apply 前先 `diff`：`kubectl diff -f xxx.yaml` |
| `kubectl drain` 忘加 `--ignore-daemonsets` | drain 卡住报错 | 加上 `--ignore-daemonsets --delete-emptydir-data` |
| 在错误集群执行操作 | 把测试环境操作打到了生产 | 用 `kubectl ctx` 切换，生产集群终端用红色提示 |
| 端口转发忘了关 | 本地端口被占用，安全风险 | `jobs` 查看后台任务，`kill %n` 关闭，或前台运行不乱后台 |
| `kubectl delete ns` | 整个 Namespace 资源全删 | 先确认环境：`kubectl config current-context`，生产永远不直接删 NS |

---

# 第十五章 Docker 与 Kubernetes ★★★★★

## 15.1 Docker 基础

### 三大核心概念

| 概念 | 说明 | 类比 |
|------|------|------|
| **Image（镜像）** | 应用的只读模板，包含运行环境和代码 | 类（Class） |
| **Container（容器）** | 镜像的运行实例，有独立的文件系统和网络 | 对象（Object / Instance） |
| **Registry（镜像仓库）** | 存储和分发镜像的地方 | GitHub（代码仓库） |

### Dockerfile 常用指令

```dockerfile
FROM golang:1.21-alpine      # 基础镜像
WORKDIR /app                  # 工作目录
COPY . .                      # 拷贝文件
RUN go build -o main .       # 构建时执行
EXPOSE 8080                   # 声明端口
CMD ["./main"]               # 容器启动命令
```

### Image 分层

Docker 镜像是**分层构建**的，每条 Dockerfile 指令创建一个新的层：

```
Layer 1: FROM golang:1.21-alpine (基础层)
Layer 2: COPY go.mod go.sum ./
Layer 3: RUN go mod download
Layer 4: COPY . .
Layer 5: RUN go build -o main .
```

分层的好处：
- **缓存复用**：未变化的层直接使用缓存，加速构建
- **共享**：不同镜像可以共享相同的层（如都基于 alpine）
- **增量传输**：只需拉取新的层

---

## 15.2 Docker 与 K8s 关系

### 面试题：Docker 和 Kubernetes 区别？

**标准回答**：

| 维度 | Docker | Kubernetes |
|------|--------|------------|
| **定位** | 容器引擎，创建和运行**单个**容器 | 容器编排平台，管理**成百上千**容器 |
| **功能** | 镜像构建（build）、运行（run）、分发（push/pull） | 部署、扩缩容、服务发现、负载均衡、滚动更新、故障恢复 |
| **范围** | 单机 | 集群（多台机器） |
| **类比** | 单个进程管理器 | 整个数据中心的操作系统 |
| **关系** | K8s 曾使用 Docker 作为容器运行时 | K8s 通过 CRI 调用容器运行时 |

**一句话总结**：Docker 让容器可以运行，Kubernetes 让容器可以在生产中可靠地运行。

---

## 15.3 容器运行时与 CRI

### 为什么 K8s 弃用 Dockershim

**时间线**：
- 早期：K8s 直接集成 Docker
- K8s 1.5：引入 CRI（Container Runtime Interface），但 Docker 不兼容 CRI → 开发了 dockershim（垫片）
- K8s 1.20：官方宣布弃用 dockershim
- K8s 1.24：完全移除 dockershim

**弃用原因**：

| 原因 | 说明 |
|------|------|
| **维护负担** | dockershim 是 K8s 代码库内的一层额外代码，需要单独维护 |
| **中间层开销** | `CRI → dockershim → Docker daemon → containerd → runc`，链路太长 |
| **社区统一** | containerd 已经是 CNCF 项目，直接支持 CRI |
| **K8s 不需要 Docker 全部功能** | K8s 只需要创建/运行容器的能力，不需要 `docker build`、`docker-compose` 等 |

**新架构**：

```
K8s (kubelet)
    │
    ▼ CRI
┌──────────────┐
│  containerd   │  ← 直接对接 CRI，无中间层
│  (或 CRI-O)   │
└──────┬───────┘
       │
       ▼
     runc (OCI 运行时)
       │
       ▼
   容器进程
```

### 重要澄清

**Docker 构建的镜像仍然可以在 K8s 中使用！** K8s 只是不再通过 dockershim 调用 Docker daemon 来运行容器，但 Docker 镜像符合 OCI 标准，可以被任何 CRI 运行时（containerd/CRI-O）运行。

### 面试题：为什么 K8s 弃用 Dockershim？

**标准回答**：

K8s 1.24 移除 dockershim 的原因：

1. **减少中间层**：dockershim 是 CRI 到 Docker 的适配层，增加了一层调用链（CRI → dockershim → Docker → containerd → runc），维护成本高
2. **社区统一**：containerd 和 CRI-O 原生支持 CRI，不需要垫片
3. **K8s 只需要 CRI 接口**：Docker 的镜像构建、docker-compose 等功能 K8s 不需要
4. **性能优化**：去掉中间层减少延迟和故障点

**影响**：Docker 构建的镜像仍然可用（符合 OCI 标准），只是运行时不再经过 Docker daemon。用户通常无需做任何改动，镜像名称不变。

---

# 第十六章 云原生进阶 ★★★★

## 16.1 Helm ★★★★★

### 定义

**Helm** 是 K8s 的**包管理器**（类似 apt/yum/brew），用于管理和部署 K8s 应用。

### 核心概念

| 概念 | 说明 | 类比 |
|------|------|------|
| **Chart** | K8s 应用的打包格式，包含模板、默认值、元数据 | apt 中的 .deb 包 |
| **Release** | Chart 在集群中的运行实例（一次安装 = 一个 Release） | apt 安装后的软件实例 |
| **Repository** | Chart 的仓库（如 https://charts.bitnami.com） | apt 源 |
| **Values** | 自定义配置值，覆盖 Chart 中的默认值 | 安装时传参 |

### Helm 解决的问题

**没有 Helm 时**：
- 部署一个应用需要管理多个 YAML 文件（Deployment、Service、ConfigMap、Secret、Ingress 等）
- 多环境需要复制粘贴并手动修改 YAML
- 没有版本管理 → 难以回滚
- 无法复用和分享部署模板

**有 Helm 后**：
- 一个 Chart 打包所有资源
- 通过 Values 文件区分环境
- `helm rollback` 一键回滚
- 社区 Helm Chart 仓库直接复用

### 核心命令

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami  # 添加仓库
helm repo update                                           # 更新仓库索引
helm search repo nginx                                     # 搜索 Chart
helm install my-release bitnami/nginx                      # 安装
helm upgrade my-release bitnami/nginx --values prod.yaml   # 升级
helm rollback my-release 1                                 # 回滚
helm uninstall my-release                                  # 卸载
helm list                                                  # 查看已安装
helm template ./chart -f values.yaml                       # 渲染模板（不安装，用于调试）
```

### 面试题：Helm 有什么作用？

**标准回答**：

Helm 是 K8s 的包管理器（类似 Linux 的 apt/yum）：

1. **打包**：将应用的所有 K8s 资源（Deployment、Service、ConfigMap 等）打包成 Chart
2. **模板化**：通过 Go Template 将 YAML 参数化，用不同的 values.yaml 适配不同环境（dev/staging/prod）
3. **版本管理**：每个 Release 都有版本号，`helm rollback` 一键回滚
4. **生态复用**：ArtifactHub 上有大量社区 Chart，无需从零开始编写
5. **依赖管理**：Chart 可以声明对其他 Chart 的依赖

---

## 16.2 Operator ★★★★

### 定义

**Operator** 是将人类运维知识编码为代码的 K8s 扩展模式，用于管理复杂有状态应用。

```
Operator = CRD（Custom Resource Definition）+ Controller
```

### 核心概念

| 概念 | 说明 |
|------|------|
| **CRD（Custom Resource Definition）** | 扩展 K8s API，定义自己的资源类型（如 MySQLCluster） |
| **CR（Custom Resource）** | CRD 的实例（如创建一个具体的 MySQL 集群） |
| **Custom Controller** | 监听 CR 的变化并执行操作（创建 Pod、配置主从等） |

### 解决的问题

**Deployment 的局限**：Deployment 只能管理无状态应用的 Pod 副本，无法处理：
- 数据库主从复制
- 数据备份恢复
- 配置文件同步
- 故障转移
- 版本升级（如 MySQL 5.7 → 8.0）

**Operator 将专家的运维知识变为自动化代码**，让 K8s 能管理这些有状态应用。

### 常见 Operator

| Operator | 管理对象 |
|----------|----------|
| Prometheus Operator | Prometheus + Alertmanager + Grafana |
| etcd Operator | etcd 集群 |
| Strimzi | Kafka 集群 |
| Zalando Postgres Operator | PostgreSQL 集群 |
| MySQL Operator | MySQL 集群 |
| Elastic Cloud on K8s (ECK) | Elasticsearch + Kibana |

### 面试题：Operator 是什么？

**标准回答**：

Operator 是 K8s 的扩展模式，由 CRD（自定义资源）+ Custom Controller 组成：

1. **CRD**：扩展 K8s API，定义新的资源类型（如定义 `MySQLCluster` 资源）
2. **Custom Controller**：监听 CR 变化，执行自动化运维操作（创建实例、主从复制、备份恢复、故障切换）

Operator 将 DBA/运维专家的经验编码为代码，使 K8s 能够**自动化管理复杂有状态应用**，超越了 Deployment 只能管理无状态 Pod 的局限。

**类比**：Deployment 是自动挡（管 Pod 副本），Operator 是全自动驾驶（管数据库集群的全部生命周期）。

---

## 16.2b CRD 深入：如何编写一个自定义控制器 ★★★★

### CRD 定义详解

CRD 是扩展 K8s API 的方式，以下是一个完整的 CRD 示例：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mysqlclusters.database.example.com   # <plural>.<group>
spec:
  group: database.example.com                # API Group
  names:
    plural: mysqlclusters                    # 复数名（URL 路径）
    singular: mysqlcluster                   # 单数名（CLI 使用）
    shortNames: ["mc"]                       # 短名
    kind: MySQLCluster                       # Go 结构体名
  scope: Namespaced                          # Namespaced 或 Cluster
  versions:
  - name: v1
    served: true                            # 是否启用此版本
    storage: true                           # 是否存储版本
    subresources:
      status: {}                            # 启用 /status 子资源
    schema:
      openAPIV3Schema:                      # 基于 OpenAPI v3 的校验
        type: object
        properties:
          spec:
            type: object
            required: ["version", "replicas"]
            properties:
              version:
                type: string
                description: "MySQL version"
              replicas:
                type: integer
                minimum: 1
                maximum: 10
          status:
            type: object
            properties:
              readyReplicas:
                type: integer
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type: {type: string}
                    status: {type: string}
                    reason: {type: string}
                    lastTransitionTime: {type: string, format: date-time}
    additionalPrinterColumns:               # kubectl get 时的额外列
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Ready
      type: integer
      jsonPath: .status.readyReplicas
```

### 🔬 自定义 Controller 源码级解析（Informer + Workqueue 模式）

这是面试中**区分"用过 Controller"和"写过 Controller"**的核心考点。

```go
// 标准 Controller 的五步模式
package controller

import (
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/util/workqueue"
)

// Step 1: 定义 Controller 结构体
type MySQLController struct {
    indexer  cache.Indexer                    // 本地缓存（Store）
    queue    workqueue.RateLimitingInterface   // 工作队列（限速 + 重试）
    informer cache.Controller                 // Informer（封装 List-Watch）
    kubeClient kubernetes.Interface           // K8s 客户端
}

// Step 2: 注册事件处理器（Event Handlers）
func (c *MySQLController) addEventHandler() {
    c.informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            key, _ := cache.MetaNamespaceKeyFunc(obj)
            c.queue.Add(key)                  // 加入队列（异步处理）
        },
        UpdateFunc: func(old, new interface{}) {
            // 只有 spec 变化时才入队（避免 status 更新触发死循环）
            if oldSpec != newSpec {
                key, _ := cache.MetaNamespaceKeyFunc(new)
                c.queue.Add(key)
            }
        },
        DeleteFunc: func(obj interface{}) {
            key, _ := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
            c.queue.Add(key)
        },
    })
}

// Step 3: 启动 Controller
func (c *MySQLController) Run(stopCh <-chan struct{}) {
    // 启动 Informer（开始 List-Watch）
    go c.informer.Run(stopCh)

    // 等待缓存同步完成（确保本地 Store 和 etcd 一致）
    if !cache.WaitForCacheSync(stopCh, c.informer.HasSynced) {
        return
    }

    // 启动 Worker 处理队列
    go wait.Until(c.runWorker, time.Second, stopCh)
}

// Step 4: Worker 从队列取任务并处理
func (c *MySQLController) runWorker() {
    for c.processNextItem() {
    }
}

func (c *MySQLController) processNextItem() bool {
    // 从队列取 key（阻塞）
    key, quit := c.queue.Get()
    if quit {
        return false
    }
    defer c.queue.Done(key)   // 标记处理完成

    // 实际的 Reconcile 逻辑
    err := c.syncHandler(key.(string))
    if err == nil {
        c.queue.Forget(key)   // 成功 → 不再重试
    } else {
        c.queue.AddRateLimited(key) // 失败 → 重新入队（限速退避）
    }
    return true
}

// Step 5: 核心 Reconcile 逻辑
func (c *MySQLController) syncHandler(key string) error {
    // 从本地缓存获取 CR 对象
    obj, exists, err := c.indexer.GetByKey(key)
    if err != nil {
        return fmt.Errorf("fetching object: %v", err)
    }
    if !exists {
        // 对象已被删除 → 清理相关资源
        return c.cleanup(key)
    }

    mysql := obj.(*MySQLCluster)

    // === Reconcile 循环 ===
    // 1. 创建 Service（如果不存在）
    // 2. 创建 StatefulSet（如果不存在或 spec 变化）
    // 3. 检查 Pod 状态
    // 4. 更新 CR 的 status.readyReplicas

    actualPods := c.countRunningPods(mysql)
    if mysql.Spec.Replicas != actualPods {
        // 更新 StatefulSet replicas
    }

    // 更新 status 子资源
    mysql.Status.ReadyReplicas = actualPods
    c.updateStatus(mysql)

    return nil
}
```

### Controller 模式的五个关键设计决策

| 设计决策 | 为什么这样 | 替代方案的问题 |
|----------|-----------|---------------|
| **本地缓存（Store/Indexer）** | 避免每次 Reconcile 都查 API Server，减少 API Server 压力 | 无缓存 → 大规模集群下 API Server 被打爆 |
| **Workqueue** | 解耦事件接收和处理，提供重试和限速；处理失败不影响事件接收 | 同步处理 → 一个失败阻塞所有事件 |
| **Status 子资源** | spec 和 status 分离，防止 Controller 修改 spec 导致死循环 | spec/status 混合 → 无法区分用户意图和当前状态 |
| **幂等 Reconcile** | 多次执行结果一致；Controller 重启后重新 Reconcile 所有对象 | 非幂等 → 重启后状态错乱 |
| **Finalizer** | 资源删除前的清理钩子（如释放外部负载均衡器、清理外部数据库） | 无 Finalizer → 资源删除后外部资源泄漏 |

### 面试题：如何编写一个自定义 Controller？关键组件有哪些？

**标准回答**：

编写自定义 Controller 需要五个核心组件：

1. **Informer**：封装 List-Watch，监听 CR 变化，将事件写入本地缓存（Store）
2. **Store（Indexer）**：线程安全的本地缓存，提供按 key/标签的快速查询，避免每次处理都访问 API Server
3. **Workqueue**：事件队列，解耦事件接收和处理，提供限速重试、去重
4. **Reconcile Loop**：从 Workqueue 取 key → 从 Store 获取对象 → 比较 spec 和 status → 执行操作 → 更新 status
5. **Finalizer**：资源删除前的清理钩子，确保外部资源（如云负载均衡器）被正确清理

**关键代码模式**：
- `AddEventHandler` 只在 spec 变化时入队（status 变化不入队，避免死循环）
- 处理失败时 `AddRateLimited`（限速重试），成功时 `Forget`
- `WaitForCacheSync` 确保启动时本地缓存与 etcd 同步完成
- 使用 `kubebuilder` 或 `operator-sdk` 框架可以简化 Controller 编写

---

## 16.2c Kubebuilder 与 controller-runtime ★★★

### 定义

**Kubebuilder** 是 K8s SIG 官方维护的 Operator 开发框架，基于 **controller-runtime** 库，提供了 Controller 开发的脚手架和最佳实践。**Operator SDK** 是 Red Hat 维护的类似框架，底层也基于 controller-runtime。

```bash
# 创建一个 Operator 项目
kubebuilder init --domain example.com
kubebuilder create api --group database --version v1 --kind MySQLCluster

# 生成的文件结构：
# ├── api/v1/mysqlcluster_types.go     # CRD 结构体定义
# ├── controllers/mysqlcluster_controller.go  # Reconcile 逻辑
# └── config/crd/                      # CRD YAML
```

### Reconcile 方法签名

```go
// controller-runtime 定义的 Reconcile 接口
func (r *MySQLClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // req.NamespacedName = {Namespace: "default", Name: "my-mysql"}

    // 获取 CR
    var mysql databasev1.MySQLCluster
    if err := r.Get(ctx, req.NamespacedName, &mysql); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil  // CR 已删除，无需处理
        }
        return ctrl.Result{}, err      // 其他错误 → 重试
    }

    // Reconcile 逻辑
    // ...

    return ctrl.Result{RequeueAfter: time.Minute}, nil  // 1 分钟后再次 Reconcile
}
```

### 面试题：Kubebuilder 和 controller-runtime 是什么？

**标准回答**：

Kubebuilder 是 K8s SIG 维护的 Operator 开发框架，基于 controller-runtime 库：

1. **代码生成**：自动生成 CRD 类型定义、DeepCopy 方法、Controller 脚手架
2. **controller-runtime**：封装了 Informer、Workqueue、Leader Election 等通用逻辑
3. **EnqueueRequestFromMapFunc**：允许跨资源触发 Reconcile（如 Service 变化触发 Deployment Controller Reconcile）
4. **内置 Webhook 支持**：轻松创建 Mutating/Validating Webhook

**核心价值**：开发者只需关注 Reconcile 业务逻辑，框架处理所有基础设施（Informer 管理、队列、Leader 选举、健康检查）。

---

## 16.3 Service Mesh（服务网格）★★★

### 解决的问题

微服务架构中，服务间通信的挑战：

| 挑战 | 传统方案 | 服务网格方案 |
|------|----------|-------------|
| 服务发现 | 代码集成 SDK | Sidecar 代理（透明） |
| 负载均衡 | 客户端库 | Sidecar 代理 |
| 重试/超时/熔断 | 代码中实现 | Sidecar 代理声明式配置 |
| 灰度发布 | 复杂路由配置 | VirtualService/DestinationRule |
| mTLS | 代码集成证书管理 | Sidecar 自动 mTLS |
| 可观测性 | 各服务自行上报 | Sidecar 统一采集 |

### Istio 架构

```
┌───────────────── Pod ──────────────────┐
│                                        │
│  ┌──────────┐     ┌──────────────────┐ │
│  │ App 容器  │◄───►│ Envoy Sidecar   │ │
│  │ (业务代码)│     │ (istio-proxy)   │ │
│  └──────────┘     └────────┬─────────┘ │
│                            │  iptables  │
└────────────────────────────┼────────────┘
                             │
                             ▼
                   网络 → 自动 mTLS + 流量管理
```

| 组件 | 职责 |
|------|------|
| **Envoy** | Sidecar 代理，拦截所有进出 Pod 的流量 |
| **Istiod** | 控制平面，下发配置到 Envoy、管理证书 |
| **Ingress/Egress Gateway** | 集群入口/出口流量管理 |

### 面试题：Istio 解决什么问题？

**标准回答**：

Istio 是服务网格实现，通过 Sidecar（Envoy）代理模式将服务间通信的复杂性从应用代码中剥离：

1. **流量管理**：智能路由、灰度发布（金丝雀/蓝绿）、故障注入、超时重试
2. **安全**：自动 mTLS（双向 TLS）、细粒度访问控制
3. **可观测性**：分布式追踪（Jaeger/Zipkin）、指标采集（Prometheus）、流量可视化（Kiali）
4. **策略**：限流、熔断、访问控制

核心价值是**应用零侵入**——业务代码不需要 SDK，不需要修改，只需注入 Sidecar 即可获得以上能力。

---

# 第十七章 K8s 面试高频场景题 ★★★★★

## 场景 1：创建一个 Pod 会发生什么？

**参见 2.3 节，此处提供面试精简版答案**：

1. **kubectl → API Server**：认证 → 授权 → 准入控制（Mutating + Validating）→ 写入 etcd（状态 Pending）
2. **Scheduler Watch** 到未绑定的 Pod → Filter（过滤）→ Score（打分）→ Bind（设置 nodeName）
3. **目标 Node 的 Kubelet** Watch 到分配 → CRI 拉镜像 → CNI 分配 IP → CSI 挂载存储 → 启动容器
4. **状态上报**：Kubelet 执行探针 → 上报 Running 到 API Server → 更新 etcd

**关键组件**：API Server、etcd、Scheduler、Kubelet、CRI、CNI、CSI

---

## 场景 2：Pod 一直 Pending 怎么办？

**排查步骤**：

```bash
# 1. 最直接的入口
kubectl describe pod <name> | grep -A 20 "Events:"

# 2. 常见原因排查
```

| 原因 | Events 关键字 | 解决方法 |
|------|--------------|----------|
| 没有可用 Node | `no nodes available` | 检查 Node 状态、是否被 cordon |
| 资源不足 | `insufficient cpu/memory` | 减少 Request 或扩容 Node |
| Taint 不兼容 | `node(s) had taint` | 添加 Toleration 或移除 Taint |
| NodeSelector 不匹配 | `didn't match NodeSelector` | 检查节点标签 |
| PVC 未绑定 | `persistentvolumeclaim not found` | 检查 PVC 和 StorageClass |
| 镜像拉取中 | `Pulling image` | 等一下，大镜像拉取慢 |
| CPU/内存 Request 为 0 | 没有明确报错 | 设置合理的 Requests |

---

## 场景 3：Pod 一直 CrashLoopBackOff 怎么办？

**排查步骤**：

```bash
# 1. 查看崩溃容器的日志（关键！）
kubectl logs <pod> --previous

# 2. 查看 Events
kubectl describe pod <pod> | grep -A 20 "Events:"

# 3. 检查退出码
kubectl describe pod <pod> | grep "Exit Code"
```

| 退出码 | 含义 | 排查方向 |
|--------|------|----------|
| 0 | 正常退出 | 检查应用逻辑（如一次性任务完成后退出） |
| 1 | 一般错误 | 查看 `--previous` 日志找 stack trace |
| 137 | SIGKILL（通常 OOMKilled） | 内存超过 Limits，增加 limits 或优化内存 |
| 139 | SIGSEGV（段错误） | 代码有 bug，如空指针 |
| 143 | SIGTERM | 收到终止信号（可能是 Liveness Probe 超时）|

**常见根因**：
- 启动命令错误（`command`/`args` 配置问题）
- 依赖服务不可达（数据库/Redis 连接失败但无重试）
- 配置缺失（ConfigMap/Secret 未创建或内容错误）
- OOMKilled
- Liveness Probe 配置过于激进（initialDelaySeconds 太小）

---

## 场景 4：Service 无法访问怎么办？

**排查链路**：

```bash
# 1. 检查 Pod 是否 Ready
kubectl get pods -l app=<label>

# 2. 检查 Service → Endpoints
kubectl get endpoints <svc-name>
# 如果 ENDPOINTS 为空 → selector 不匹配 Pod labels

# 3. 检查端口映射
kubectl describe service <svc-name>
# Port (Service 端口) → TargetPort (Pod 端口)

# 4. 从测试 Pod 访问
kubectl run test --rm -it --image=busybox -- wget -O- http://<svc>:<port>

# 5. 检查 DNS
kubectl run test --rm -it --image=busybox -- nslookup <svc>.<ns>.svc.cluster.local

# 6. 检查网络策略
kubectl get networkpolicies
```

**速查表**：

| 现象 | 原因 | 检查 |
|------|------|------|
| Endpoints 为空 | selector 不匹配 | `k get pods --show-labels` vs `k describe svc` |
| DNS 解析失败 | CoreDNS 故障 | `k get pods -n kube-system` |
| 端口不通 | targetPort 错误 | `k describe svc` 与容器监听端口对比 |
| 跨 Namespace 失败 | 用了短名 | 使用 `<svc>.<ns>.svc.cluster.local` |

---

## 场景 5：Ingress 不生效怎么办？

**排查步骤**：

```bash
# 1. Ingress Controller 是否运行？
kubectl get pods -n ingress-nginx

# 2. Ingress 资源配置是否正确？
kubectl describe ingress <name>

# 3. Ingress Controller 日志
kubectl logs -n ingress-nginx <controller-pod>

# 4. 后端 Service 和 Pod 是否正常？
kubectl get svc,endpoints -n <namespace>

# 5. DNS 解析
nslookup <domain>
```

**常见原因**：
- Ingress Controller 未安装/未运行
- Ingress 中 backend serviceName/servicePort 错误
- Service 的 Endpoints 为空
- TLS Secret 未创建或格式错误
- Ingress 的 `ingressClassName` 不匹配

---

## 场景 6：K8s 如何实现服务发现？

**回答**：

K8s 有两层服务发现机制：

1. **环境变量**：Pod 启动时注入 Service 的 IP 和端口到环境变量（只在同 Namespace 的 Pod 创建顺序依赖中使用，不推荐）

2. **DNS（CoreDNS）**：**最主要的方式**
   - 每个 Service 自动获得 DNS 记录：`<service>.<namespace>.svc.cluster.local`
   - Pod 的 `/etc/resolv.conf` 中配置 DNS 搜索域
   - 同 Namespace 可以直接用 Service 名访问
   - 跨 Namespace 用 `<service>.<namespace>` 访问

3. **Headless Service**：设置 `clusterIP: None`，DNS 直接返回 Pod IP 列表（用于 StatefulSet）

---

## 场景 7：Deployment 如何实现滚动更新？

**回答**：

参见 3.3 节（滚动更新原理）。

1. 修改 Pod Template（如更新镜像 → Deployment Controller 创建新 ReplicaSet
2. 新 RS 逐步扩容，旧 RS 逐步缩容
3. maxSurge（最多超出多少 Pod）和 maxUnavailable（最多多少不可用）控制节奏
4. 旧 RS 保留用于回滚
5. `kubectl rollout undo` 回滚（本质是将旧 RS 重新扩容）

---

## 场景 8：K8s 如何实现故障恢复？

**回答**：

参见 10.3 节（自愈机制）。

1. 容器级：Liveness Probe 失败 → kubelet 重启容器
2. Pod 级：ReplicaSet Controller 发现副本不足 → 创建新 Pod
3. Node 级：Node 失联 ~5 分钟 → Node Controller 驱逐 Pod → Scheduler 调度到健康节点
4. 流量级：Readiness Probe 失败 → kube-proxy 从 Service Endpoints 移除

---

## 场景 9：Scheduler 如何选择节点？

**回答**：

参见 8.1 节（调度器原理）。

Filter（过滤不满足条件的） → Score（对候选节点打分排序） → Bind（绑定最优节点）

---

## 场景 10：Taint 和 Toleration 有什么作用？

**回答**：

参见 8.5 节。

Taint 给节点打污点排斥 Pod，Toleration 给 Pod 声明容忍允许被调度到有 Taint 的节点。典型场景：GPU 节点专属、节点维护隔离、故障自动驱逐。

---

## 场景 11：PV、PVC、StorageClass 区别？

**回答**：

参见 7.2-7.4 节。

| 概念 | 角色 | 类比 |
|------|------|------|
| **PV** | 存储的"供给方"，集群级别的存储资源 | 仓库里的硬盘 |
| **PVC** | 存储的"需求方"，用户申请存储 | 向仓库申请硬盘 |
| **StorageClass** | 动态供给的"模板"，按需创建 PV | 自动化的硬盘工厂 |

---

## 场景 12：etcd 为什么使用 Raft？

**回答**：

etcd 是分布式 KV 存储，需要保证：
1. **强一致性**：任何时刻所有节点看到的数据一致（Raft 保证线性一致性写）
2. **高可用**：少数节点故障不影响服务（Raft 的多数派 Quorum 机制）
3. **自动故障恢复**：Leader 宕机自动选举（Raft Leader Election）

没有 Raft（或其他共识算法），etcd 在分布式环境下会出现脑裂、数据不一致等问题。Raft 相比 Paxos 更易理解和实现。

---

## 场景 13：HPA 如何自动扩容？

**回答**：

参见 11.1 节。

1. Metrics Server 采集 Pod 的 CPU/内存指标
2. HPA Controller 定期计算：`期望副本 = ceil(当前副本 × 当前指标值 / 目标指标值)`
3. 更新 Deployment replicas → 触发 ReplicaSet 扩缩容
4. 缩容有冷却时间防止抖动

---

## 场景 14：RBAC 如何实现权限控制？

**回答**：

参见 13.1 节。

1. 定义 Role/ClusterRole（哪些资源 + 哪些操作）
2. 创建 RoleBinding/ClusterRoleBinding（角色 → 用户/SA/组）
3. API Server 在每次请求时校验：认证 → 授权（RBAC） → 准入控制

---

## 场景 15：Calico 和 Flannel 区别？

**回答**：

参见 9.2 节。

| Flannel | Calico |
|---------|--------|
| Overlay 网络（VXLAN） | BGP 路由（也可 VXLAN） |
| 简单，学习和部署容易 | 功能丰富，运维稍复杂 |
| **不支持 NetworkPolicy** | **完整支持 NetworkPolicy** |
| 适合学习和小集群 | 适合生产环境 |

---

## 场景 16：iptables 和 IPVS 区别？

**回答**：

参见 9.3 节。

| iptables | IPVS |
|----------|------|
| O(n) 规则链线性匹配 | O(1) 哈希表查找 |
| 概率性随机负载均衡 | 多种调度算法（rr/lc/dh/sh） |
| 规则多时性能下降 | 大规模稳定 |
| 无需额外内核模块 | 需要 IPVS 内核模块 |

---

## 场景 17：K8s 如何实现高可用？

**回答**：

1. **控制平面高可用**：
   - API Server：多副本 + 前面 LB
   - etcd：3/5 节点 Raft 集群
   - Controller Manager + Scheduler：Leader 选举（同一时刻只有一个在工作）

2. **数据平面高可用**：
   - 多 Worker Node
   - Pod 多副本分散到不同 Node/可用区（Pod Anti-Affinity）
   - 自动故障恢复（10.3 节）

3. **存储高可用**：
   - Ceph/NFS 等多副本共享存储
   - 云盘自动跨可用区复制

---

## 场景 18：为什么 Pod IP 会变化？

**回答**：

Pod 是**临时性**资源：
- 每个 Pod 创建时从 CNI 网络插件获取 IP
- Pod 删除/重启后创建的是**新 Pod**，分配**新 IP**
- ReplicaSet 扩容的新 Pod 也分到新 IP

这就是为什么需要 **Service**：提供稳定的 ClusterIP 和 DNS 名称，屏蔽 Pod IP 的变化。

---

## 场景 19：Docker 和 Kubernetes 区别？

**回答**：

参见 15.2 节。
- Docker：容器引擎（单机运行容器）
- K8s：容器编排平台（集群管理容器）
- 关系：K8s 使用 CRI 兼容的容器运行时（containerd/CRI-O）来运行 Docker 格式的镜像

---

## 场景 20：节点 NotReady 怎么排查？

**排查步骤**：

```bash
# 1. 查看节点状态
kubectl get nodes
kubectl describe node <node-name>

# 2. 检查 Conditions（关键！）
# Conditions 中有 MemoryPressure / DiskPressure / PIDPressure / Ready / NetworkUnavailable
kubectl describe node <node> | grep -A 5 "Conditions:"

# 3. SSH 到问题节点
systemctl status kubelet
journalctl -u kubelet --since "10 min ago"
systemctl status containerd  # 或 docker

# 4. 检查资源
df -h              # 磁盘是否满
free -m            # 内存是否耗尽
top -bn1 | head    # CPU 负载

# 5. 检查网络
ip route           # 路由表是否正常
ping <api-server-ip>  # 到 API Server 的连通性
```

| 常见根因 | 症状 | 修复 |
|----------|------|------|
| kubelet 进程停止 | `systemctl status kubelet` 显示 inactive | `systemctl restart kubelet` |
| 磁盘满 | `DiskPressure=True` | 清理 `/var/lib/kubelet`、旧镜像、日志 |
| 内存不足 | `MemoryPressure=True` | 扩容节点或驱逐部分 Pod |
| 容器运行时故障 | containerd 异常 | `systemctl restart containerd` |
| CNI 插件异常 | 新 Pod 网络不可用 | 检查 CNI Pod 日志，重启 CNI DaemonSet |
| API Server 不可达 | kubelet 无法上报心跳 | 检查网络/防火墙/API Server LB |

## 场景 21：etcd 集群故障如何排查和恢复？

**排查步骤**：

```bash
# 1. 检查 etcd 集群健康
etcdctl --endpoints=<ep1>,<ep2>,<ep3> endpoint health
etcdctl --endpoints=<ep1>,<ep2>,<ep3> endpoint status

# 2. 检查 Raft Leader
etcdctl member list

# 3. 检查 etcd 性能
etcdctl check perf

# 4. 检查 etcd 日志
journalctl -u etcd --since "10 min ago"
```

**常见故障及处理**：

| 故障 | 现象 | 根因 | 恢复 |
|------|------|------|------|
| **多数节点宕机（丧失 Quorum）** | API Server 返回 503，`kubectl` 无法创建/更新 | 超过半数 etcd 节点不可用 | 恢复故障节点，极端情况下需强制重建集群 |
| **磁盘满** | etcd 写入失败，集群不可用 | 日志文件过大或数据目录满 | 执行 `etcdctl compact` + `etcdctl defrag` |
| **证书过期** | API Server 无法连接 etcd | etcd 的 TLS 证书过期 | 更新证书并重启 etcd |
| **Leader 频繁切换** | 集群性能急剧下降 | 磁盘 I/O 延迟高或网络抖动 | 使用 SSD、排查网络延迟 |
| **DB 大小过大** | etcd 性能下降 | 历史数据过多 | 定期 compaction + defragmentation |

**etcd 运维最佳实践**：

```bash
# 定期压缩历史版本（建议 CronJob）
etcdctl compact <revision>

# 碎片整理（压缩后执行）
etcdctl defrag

# 备份（必须定期执行！）
etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db

# 从快照恢复
etcdctl snapshot restore /backup/etcd-backup.db \
    --name=<new-node> \
    --initial-cluster=<new-node>=https://<ip>:2380 \
    --initial-advertise-peer-urls=https://<ip>:2380
```

**面试追问：etcd 数据太大怎么办？**

etcd 默认存储限制 2GB（可配置 `--quota-backend-bytes`）。解决方案：
1. **碎片整理**：`etcdctl defrag` 回收已删除 key 的空间
2. **压缩历史**：`etcdctl compact` 压缩旧版本（K8s 默认保留最新版本，但历史版本可能堆积）
3. **清理无用资源**：删除不需要的 K8s 资源（旧 ReplicaSet、完成的 Job 等）
4. **调整保留策略**：减少 `--auto-compaction-retention`

## 场景 22：线上 K8s 集群出现故障如何排查？

**回答**：

**排查框架**：

```
1. 确定影响范围（单个 Pod / 单个 Service / 整个 Namespace / 整个集群？）
2. 检查控制平面（API Server / etcd / Scheduler / Controller Manager 是否正常？）
3. 检查节点状态（kubectl get nodes，是否有 NotReady 节点？）
4. 检查网络组件（CNI Pod / kube-proxy / CoreDNS 是否正常？）
5. 按链路排查（Ingress → Service → Pod → 应用日志）
```

**常用命令**：

```bash
kubectl get nodes                          # 节点状态
kubectl get pods -n kube-system            # 系统组件状态
kubectl describe pod <name>                # Pod 详情 + Events
kubectl logs <pod> --previous              # 崩溃容器日志
kubectl top nodes / pods                   # 资源使用
kubectl get events --sort-by='.lastTimestamp'  # 集群事件
journalctl -u kubelet                      # kubelet 日志（SSH 到 Node）
```

---

# 校招必背 TOP 40（速记版）

以下是校招面试中最高频的 40 个问题及核心要点，每条一句话 + 关键词，方便快速复习。

> ★ 标记为本次补充的新增高频考点（原 30 → 扩展至 40）

| # | 问题 | 核心答案要点 |
|---|------|------------|
| 1 | **什么是 Kubernetes** | Google 开源的容器编排平台，声明式 API + 控制循环，自动化部署/扩缩容/故障恢复 |
| 2 | **K8s 架构** | Control Plane（API Server/etcd/Scheduler/Controller Manager）+ Worker Node（kubelet/kube-proxy/CRI） |
| 3 | **kube-apiserver 作用** | 集群唯一入口，认证授权准入，数据写入 etcd，提供 List-Watch 机制 |
| 4 | **kubelet 作用** | 节点代理，管理 Pod 生命周期，执行探针，上报状态，调用 CRI/CNI/CSI |
| 5 | **Pod 是什么** | 最小调度单元，共享网络/存储/IPC 命名空间，可以包含多个紧密耦合容器 |
| 6 | **Pod 生命周期** | Pending → Running → Succeeded/Failed，还有 Unknown 状态 |
| 7 | **创建 Pod 流程** | kubectl → API Server（认证/授权/准入）→ etcd → Scheduler（Filter/Score/Bind）→ Kubelet 创建 → 上报 Running |
| 8 | **Liveness vs Readiness vs Startup** ★ | Liveness 决定重启（检测存活），Readiness 决定流量（检测就绪），Startup 保护慢启动（成功后才开始 Liveness） |
| 9 | **Deployment 原理** | 管理 ReplicaSet → 管理 Pod 副本，滚动更新（新建 RS 逐步扩容、旧 RS 逐步缩容） |
| 10 | **ReplicaSet 原理** | 保证指定数量 Pod 副本始终运行，通过 selector/replicas/template 三要素管理 |
| 11 | **滚动更新原理** | 创建新 RS → 新 RS 扩容 + 旧 RS 缩容，maxSurge/maxUnavailable 控制节奏 |
| 12 | **StatefulSet vs Deployment** ★ | Deployment 无状态（Pod 可互换、随机名）；StatefulSet 有状态（固定序号、稳定 DNS、独立 PVC、有序启停） |
| 13 | **DaemonSet 作用** ★ | 每个 Node 运行一个 Pod 副本，用于日志采集、监控 Agent、CNI 网络插件等守护进程 |
| 14 | **Job vs CronJob** ★ | Job 一次性任务直到成功完成；CronJob 按 cron 表达式定时创建 Job（定时备份/清理） |
| 15 | **Service 原理** | 稳定 IP + 负载均衡 + 服务发现，通过 selector 匹配 Pod，kube-proxy 创建转发规则 |
| 16 | **ClusterIP vs NodePort** | ClusterIP 仅集群内部访问；NodePort 每个 Node 开放端口可从外部访问 |
| 17 | **Headless Service** ★ | `clusterIP: None`，DNS 直接返回 Pod IP 列表，无负载均衡，用于 StatefulSet + 集群成员发现 |
| 18 | **Ingress 原理** | 七层 HTTP/HTTPS 路由（Host + Path），需要 Ingress Controller 实际执行规则 |
| 19 | **ConfigMap vs Secret** | ConfigMap 存非敏感明文配置；Secret 存敏感信息（base64 编码 + 额外安全机制） |
| 20 | **Labels vs Annotations** ★ | Labels 用于标识和 Selector 选择资源；Annotations 存储非标识性元数据（构建信息/监控配置） |
| 21 | **Namespace 作用** ★ | 逻辑隔离多租户，配合 ResourceQuota 限制资源，配合 RBAC 限制权限，DNS 隔离（跨 NS 需 FQDN） |
| 22 | **PV vs PVC** | PV 是存储供给方（管理员创建/StorageClass 动态创建），PVC 是需求方（用户申请存储） |
| 23 | **StorageClass 原理** | 动态供给 PV 的模板，PVC 创建时自动创建底层存储和 PV，实现按需分配 |
| 24 | **Scheduler 调度流程** | Filter（过滤不满足条件的节点）→ Score（对候选节点打分）→ Bind（绑定最优节点） |
| 25 | **Taint 与 Toleration** | Taint（节点污点，排斥 Pod）← → Toleration（Pod 容忍，允许调度到有 Taint 的节点） |
| 26 | **Pod Affinity / Anti-Affinity** | 亲和：把 Pod 调度到一起（低延迟）；反亲和：把 Pod 分散（高可用） |
| 27 | **K8s 网络模型** | 所有 Pod 扁平三层网络，互通无 NAT，Pod IP 全局唯一，由 CNI 插件实现 |
| 28 | **Calico vs Flannel** | Flannel VXLAN 简单无网络策略；Calico BGP 路由高性能 + NetworkPolicy |
| 29 | **kube-proxy 原理** | 监听 Service/Endpoints 变化 → 创建 iptables/IPVS 规则 → DNAT 到后端 Pod |
| 30 | **etcd 与 Raft** | 分布式 KV 存储 + Raft 共识（Leader 选举 + 日志复制 + 多数派提交），K8s 唯一数据源 |
| 31 | **Controller 工作原理** | Reconcile Loop：不断比较 spec（期望）和 status（实际），不一致则执行操作 |
| 32 | **HPA 原理** | 根据 CPU/内存指标计算期望副本数 = ceil(当前 × 当前指标 / 目标指标)，更新 Deployment |
| 33 | **Requests vs Limits** | Requests 是资源保证（调度依据），Limits 是资源上限（超限 OOM/Throttle） |
| 34 | **QoS 等级** | Guaranteed（R=L）→ Burstable（有 R）→ BestEffort（无 R 无 L），OOM 时按此顺序驱逐 |
| 35 | **ResourceQuota vs LimitRange** ★ | ResourceQuota 限制 Namespace 资源总和；LimitRange 限制单个 Pod/容器资源上下限和默认值 |
| 36 | **RBAC 原理** | Role/ClusterRole 定义权限 + RoleBinding/ClusterRoleBinding 绑定到 User/SA/Group |
| 37 | **imagePullPolicy** ★ | Always 每次拉取，IfNotPresent 本地优先，Never 只用本地镜像；生产环境不用 latest tag |
| 38 | **Helm 原理** | K8s 包管理器，Chart 模板化打包 + Values 参数化 + Release 版本管理 + rollback |
| 39 | **Operator 原理** ★ | CRD（自定义资源）+ Custom Controller，将运维知识编码为代码，自动化管理有状态应用 |
| 40 | **Docker vs Kubernetes** | Docker 单机运行容器（容器引擎），K8s 集群管理容器（编排平台）；K8s 通过 CRI 调用 containerd 运行 OCI 镜像 |

---

# 附录：面试技巧

### 回答问题的 STAR 框架（改编版）

**技术问题可以用以下结构回答**：

1. **是什么**（定义）：一句话概括核心概念
2. **为什么**（动机）：它解决了什么问题
3. **怎么做**（原理/流程）：核心工作机制
4. **举例/场景**：实际使用场景或遇到过的问题

### 场景题的万能排查公式

```
1. 确定现象和影响范围
2. 检查最外层（Ingress/Service/Pod 状态）
3. 检查关键组件日志
4. 检查配置（selector、端口、资源）
5. 测试连通性
6. 缩小范围，定位根因
```

### 加分项

- 提到相关开源项目（如 Istio、Prometheus、Helm、ArgoCD）
- 提到 CNCF Landscape 和技术趋势
- 提到实际项目中的踩坑经验和解决方案
- 提到源码层面理解（如 Scheduler Framework、Controller 的 workqueue）
- 关注 K8s 最新特性（如 Sidecar Container 原生支持、Gateway API）

---

# 第十八章 ⭐ 面试速记版（1 页总结）

> 以下内容可以**打印成 1 页纸**，面试前 10 分钟快速过一遍。

## K8s 核心架构速记

```
Control Plane: API Server(统一入口) + etcd(状态存储/Raft) + Scheduler(Filter→Score→Bind) + Controller Manager(Reconcile循环)
Worker Node: kubelet(Pod生命周期/探针/CRI/CNI/CSI) + kube-proxy(iptables/IPVS) + Container Runtime(containerd)
```

## 核心设计哲学（3 句话回答任何"为什么"问题）

| 问题 | 一句话回答 |
|------|-----------|
| 为什么用声明式 API？ | 用户说"要什么"，系统自动做"怎么做"——控制和复杂度在平台侧，用户侧简单 |
| 为什么用控制器模式？ | 持续比较 spec 和 status，不一致就调和——天然容错、天然幂等、天然自愈 |
| 为什么用 Pod？ | 紧密协作的容器共享网络/存储/生命周期，Pod 是原子调度单位 |
| 为什么用 Service？ | Pod IP 会变，Service 提供稳定的虚拟 IP 和 DNS，解耦前端和后端 |
| 为什么用 etcd？ | 集群唯一状态存储，Raft 保证强一致性和高可用 |
| 为什么组件间不直接通信？ | 都通过 API Server 作为唯一协调点——组件解耦、可替换、可水平扩展 |
| 为什么用控制器而不是人工？ | 机器永远比人快、比人精确、不会忘——自愈的本质是自动化的 Reconcile Loop |
| 为什么需要 Ingress Controller？ | Ingress 只是路由规则（YAML），Controller 是实际执行规则的反向代理进程 |
| 为什么 K8s 弃用 dockershim？ | 去掉中间层（CRI→dockershim→dockerd→containerd 链路），containerd 已原生支持 CRI |

## 最核心的 5 个流程

| 流程 | 关键步骤 | 考察概率 |
|------|----------|----------|
| **创建 Pod** | kubectl → API Server(认证/授权/准入) → etcd(Pending) → Scheduler(Filter/Score/Bind) → Kubelet(CRI/CNI/CSI) → Running | ★★★★★ |
| **Service 转发** | Client → ClusterIP → kube-proxy(iptables/IPVS 规则) → DNAT → Pod IP:Port | ★★★★★ |
| **滚动更新** | 改 Pod Template → 创建新 RS → 新 RS 扩容 + 旧 RS 缩容(maxSurge/maxUnavailable 控制节奏) → 旧 RS 保留(回滚用) | ★★★★★ |
| **节点故障恢复** | Node NotReady(~40s) → Node Controller 标记(~5min) → 驱逐 Pod → Scheduler 重新调度 → Kubelet 创建新 Pod | ★★★★★ |
| **HPA 扩缩容** | Metrics Server 采集 → Controller 计算: `ceil(当前×当前值/目标值)` → 更新 Deployment replicas → RS 扩缩容 | ★★★★ |

## 必背对比表

| 对比 | 选 A | 选 B |
|------|------|------|
| Deployment vs StatefulSet | 无状态 Web 服务 | 有状态(数据库/消息队列) |
| ConfigMap vs Secret | 非敏感配置(明码) | 敏感数据(base64+安全机制) |
| iptables vs IPVS | 小规模(<1000 Service) | 大规模(>1000 Service)，O(1) |
| Flannel vs Calico | 简单/学习用，无网络策略 | 生产/高性能/支持 NetworkPolicy |
| ClusterIP vs NodePort | 集群内通信 | 集群外访问(开发测试) |
| Ingress vs Gateway API | 当前主流/功能有限 | 下一代标准/角色分离/更强表达 |
| HPA vs VPA | 水平扩缩(增减副本) | 垂直扩缩(调整资源 Request) |
| Liveness vs Readiness | 重启容器(存活检测) | 摘除流量(就绪检测) |
| Job vs CronJob | 一次性任务 | 定时任务 |

---

# 第十九章 ⭐ 高频面试题一句话回答模板

> 以下模板可以直接用于面试回答的第一句话，然后展开细节。

## 基础概念类

| 问题 | 一句话回答模板 |
|------|---------------|
| 什么是 K8s？ | "Kubernetes 是 Google 开源的容器编排平台，基于声明式 API 和控制循环，自动化管理容器的部署、扩缩容、服务发现、负载均衡和故障恢复。" |
| K8s 架构？ | "控制平面（API Server + etcd + Scheduler + Controller Manager）负责决策，Worker Node（kubelet + kube-proxy + 容器运行时）负责运行容器，组件间通过 API Server + List-Watch 解耦通信。" |
| 什么是 Pod？ | "Pod 是 K8s 最小调度单位，包含一个或多个共享网络和存储的容器，是紧密协作容器的原子调度单元。" |
| 什么是 Deployment？ | "Deployment 是管理无状态应用的工作负载资源，通过 ReplicaSet 控制 Pod 副本数，支持声明式更新、滚动更新和版本回滚。" |
| 什么是 Service？ | "Service 是 Pod 的稳定网络抽象，提供固定的 ClusterIP 和 DNS，通过 kube-proxy 实现后端 Pod 的负载均衡。" |

## 组件类

| 问题 | 一句话回答模板 |
|------|---------------|
| API Server 作用？ | "API Server 是集群唯一入口，负责认证/授权/准入控制，所有数据通过它写入 etcd，是组件间 List-Watch 解耦通信的枢纽。" |
| etcd 是什么？ | "etcd 是分布式 KV 存储，基于 Raft 共识算法保证强一致性，存储集群所有状态数据，是 K8s 的单一事实来源。" |
| Scheduler 如何工作？ | "Scheduler 通过 Filter（过滤不满足条件的节点）→ Score（对候选节点打分）→ Bind（绑定最优节点），将未调度的 Pod 分配到合适的 Node。" |
| kubelet 作用？ | "kubelet 是节点代理，通过 CRI 管理容器生命周期、执行探针、上报节点和 Pod 状态到 API Server。" |
| kube-proxy 原理？ | "kube-proxy 监听 Service 和 Endpoints 变化，在节点上创建 iptables/IPVS 规则，将访问 Service ClusterIP 的流量 DNAT 到后端 Pod。" |

## 机制类

| 问题 | 一句话回答模板 |
|------|---------------|
| Reconcile Loop？ | "控制器不断比较 spec（期望状态）和 status（实际状态），发现差异就执行操作使两者趋于一致——这是 K8s 声明式 API 的底层实现。" |
| 滚动更新原理？ | "创建新 ReplicaSet 逐步扩容，同时旧 ReplicaSet 逐步缩容，maxSurge 控制超出数、maxUnavailable 控制不可用数，旧 RS 保留用于回滚。" |
| Service 负载均衡？ | "kube-proxy 通过 iptables 规则链（概率性随机）或 IPVS 虚拟服务器（哈希查找 O(1)）将 Service ClusterIP 流量转发到后端 Pod。" |
| HPA 如何扩缩容？ | "HPA Controller 定期查询 Metrics Server 获取 Pod 指标，按公式 ceil(当前副本×当前值/目标值) 计算期望副本数，更新 Deployment replicas。" |
| RBAC 权限模型？ | "Role 定义权限规则 + RoleBinding 将角色绑定到用户/SA，API Server 在每次请求时进行授权检查，遵循最小权限原则。" |

## 场景排错类

| 问题 | 一句话回答模板 |
|------|---------------|
| Pod Pending？ | "先用 kubectl describe pod 看 Events：常见原因有资源不足、Taint 不兼容、NodeSelector 不匹配、PVC 未绑定、镜像拉取中。" |
| CrashLoopBackOff？ | "先用 kubectl logs --previous 看上一个崩溃容器的日志，常见原因有启动命令错误、依赖不可达、OOMKilled、Liveness Probe 配置过于激进。" |
| Service 不通？ | "按链路排查：Pod Ready？→ Service Endpoints 非空？→ 端口映射正确？→ DNS 解析正常？→ NetworkPolicy 允许？→ 测试 Pod curl 验证。" |
| 节点 NotReady？ | "kubectl describe node 看 Conditions，SSH 到节点检查 kubelet/containerd 状态、磁盘/内存/CUP 资源、网络连通性。" |

## 设计原理类（应对追问）

| 追问 | 一句话回答模板 |
|------|---------------|
| K8s 为什么是声明式而不是命令式？ | "声明式把编排复杂性从用户转移到平台——用户说我要什么，系统负责怎么做到。声明式天然幂等、支持 IaC/GitOps、容错性更好。" |
| K8s 为什么要抽象出 Pod？ | "因为实际应用中多容器经常需要紧密协作（日志采集、服务网格代理），共享网络和存储——Pod 是'进程组'的抽象，原子调度。" |
| etcd 为什么用 Raft？ | "Raft 保证强一致性和高可用：任何写操作需多数节点确认才提交，Leader 挂了自动选举——没有 Raft 就无法保证分布式存储的一致性。" |
| kube-proxy 为什么有 iptables 和 IPVS 两种模式？ | "iptables 小规模够用但规则多时 O(n) 性能下降，IPVS 用哈希表 O(1) 查找、支持多种调度算法——1000+ Service 时 IPVS 明显优于 iptables。" |
| K8s 弃用 dockershim 为什么？ | "为了去掉中间层：CRI → dockershim → dockerd → containerd 链路太长，containerd 已原生支持 CRI，去掉 dockershim 减少维护成本和延迟。" |
| 控制器为什么用 Workqueue 而不是直接处理事件？ | "解耦事件接收和处理：事件回调中不应该做耗时操作，Workqueue 支持重试、限速、去重——即使处理失败也不丢事件。" |

---

# 第二十章 ⭐ 面试高频考点 Checklist

> 面试前逐项打勾，确保全覆盖。

## 必须能说清楚的概念（每个 2 分钟讲透）

- [ ] Pod 是什么？为什么是原子调度单位？
- [ ] Pod 生命周期（5 个 Phase + 容器 3 个 State）
- [ ] Liveness / Readiness / Startup Probe 区别（各解决什么问题）
- [ ] PostStart vs PreStop Hook（执行时机和并行/同步区别）
- [ ] Deployment → ReplicaSet → Pod 层级关系
- [ ] 滚动更新原理（maxSurge / maxUnavailable）
- [ ] StatefulSet vs Deployment（什么场景用哪个）
- [ ] Service 类型（ClusterIP / NodePort / LoadBalancer / ExternalName / Headless）
- [ ] Service 如何实现负载均衡（iptables vs IPVS，注意 IPVS 并非默认）
- [ ] Ingress vs Gateway API（角色分离，新一代标准）
- [ ] Scheduler 调度流程（Filter → Score → Bind）
- [ ] Node Affinity vs Taint/Toleration（主动选择 vs 被动排斥）
- [ ] PriorityClass & Preemption（优先级抢占机制）
- [ ] PV / PVC / StorageClass（静态 vs 动态供给，WaitForFirstConsumer）
- [ ] ConfigMap 和 Secret 区别（使用方式、安全差异）
- [ ] Downward API（环境变量 vs Volume 方式获取元数据）
- [ ] kube-proxy 工作原理（iptables vs IPVS 模式）
- [ ] Controller 模式（Reconcile Loop + Informer + Workqueue）
- [ ] etcd 和 Raft 共识（Leader 选举、日志复制、多数派提交）
- [ ] RBAC（Role / ClusterRole / RoleBinding / ClusterRoleBinding）
- [ ] Pod Security Admission（PSA，三级安全标准）
- [ ] Admission Webhook（Mutating vs Validating）
- [ ] HPA 扩缩容原理（指标管道：Metrics Server vs Prometheus Adapter）

## 必须能画流程图的流程（面试可能让画图）

- [ ] Pod 创建全流程（kubectl → API Server → etcd → Scheduler → Kubelet → CRI/CNI/CSI）
- [ ] Pod 删除流程（Terminating → SIGTERM → preStop Hook → SIGKILL）
- [ ] Service 流量路径（Client → ClusterIP → kube-proxy → DNAT → Pod）
- [ ] 滚动更新流程（新 RS 扩容 + 旧 RS 缩容）
- [ ] 节点故障恢复时间线（NotReady 40s → 驱逐 5min → 重新调度）

## 必须能排查的故障场景

- [ ] Pod Pending → describe + Events + 容器状态 Waiting Reason
- [ ] CrashLoopBackOff → logs --previous + 退出码分析
- [ ] ImagePullBackOff → describe + 检查 imagePullSecrets + 镜像 tag
- [ ] Service 不通 → Endpoints + 端口 + DNS + 测试 Pod curl + NetworkPolicy
- [ ] 节点 NotReady → describe Conditions + SSH + kubelet 日志 + 磁盘/内存
- [ ] etcd 故障 → endpoint health + member list + compaction/defrag + 备份恢复
- [ ] distroless 容器调试 → kubectl debug + Ephemeral Container
- [ ] HPA 不工作 → Metrics Server 是否运行 + Requests 是否设置 + 指标是否正确

## 加分项（提到即加分）

- [ ] 声明式 API vs 命令式 API 的设计哲学
- [ ] List-Watch 机制和 Informer 模式（Reflector/Store/Workqueue）
- [ ] Controller 的 Workqueue（限速重试、去重）和 Indexer
- [ ] Gateway API（新一代 Ingress，角色分离三层模型）
- [ ] Admission Webhook（Mutating + Validating）
- [ ] Pod Security Admission（PSA，替代 PSP）
- [ ] CRD + Operator（Kubebuilder / controller-runtime）
- [ ] Cilium + eBPF（新一代 CNI，绕过 iptables）
- [ ] Service Mesh（Istio + Envoy Sidecar）
- [ ] Ephemeral Containers（kubectl debug）
- [ ] API Priority and Fairness（APF，API Server 流量控制）
- [ ] K8s 源码理解（Scheduler Framework / controller-runtime）
- [ ] etcd 运维（compaction / defrag / snapshot / restore）

---

> **最后**：K8s 面试不是考察你能否背出所有概念，而是考察你**是否理解容器编排的本质问题**（服务发现、负载均衡、故障恢复、声明式管理），以及**能否将理论和实际问题结合起来**。祝你面试顺利！
