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
> - ★★★★★：高频必问
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

### 2.6 三大探针对比（Liveness / Readiness / Startup Probe）★★★★★

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

### 2.7 imagePullPolicy（镜像拉取策略）★★★

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

### 2.8 Pod 优雅退出（Graceful Shutdown）★★★★

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
| Kubernetes 默认 | 否 | 是（1.10 之前默认） | **是（1.11+ 默认）** |

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

- **冷却时间（Cooldown）**：默认扩容后 3 分钟内不缩容（`--horizontal-pod-autoscaler-downscale-stabilization`，默认 5 分钟）
- 防止频繁扩缩容（flapping）

### 面试题：HPA 如何工作？

**标准回答**：

HPA 是 K8s 的自动扩缩容机制：

1. **指标来源**：从 Metrics Server（资源指标）或 Prometheus Adapter（自定义指标）获取 Pod 指标
2. **计算**：`期望副本数 = ceil(当前副本数 × 当前指标值 / 目标指标值)`
3. **执行**：通过 API Server 更新 Deployment/StatefulSet 的 replicas 字段
4. **冷却**：扩容后 3 分钟不缩容，缩容后 5 分钟不扩容（默认值）

**前提条件**：
- Pod 必须设置 resource **Requests**（HPA 根据 Requests 计算使用率百分比）
- 部署 Metrics Server（`kubectl top` 依赖）
- 如果是自定义指标，需要 Prometheus Adapter

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

## 场景 20：线上 K8s 集群出现故障如何排查？

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

> **最后**：K8s 面试不是考察你能否背出所有概念，而是考察你**是否理解容器编排的本质问题**（服务发现、负载均衡、故障恢复、声明式管理），以及**能否将理论和实际问题结合起来**。祝你面试顺利！
