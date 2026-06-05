# Docker 面试知识体系（后端开发校招终极版）

> 适用于：
> - Golang 后端开发
> - 云原生开发工程师
> - DevOps 工程师
> - 平台研发工程师
>
> 学习建议：
> - ★★★★★：高频必问
> - ★★★★：常考
> - ★★★：了解即可
>
> Docker 是 Kubernetes 的基础，也是当前后端部署和微服务架构中的核心技术。

---

# 第一章 Docker 基础

## 1.1 什么是 Docker ★★★★★

### Docker 定义

Docker 是一个开源的**容器化平台**，用于开发、打包、部署和运行应用程序。它通过**操作系统级虚拟化**技术，将应用及其所有依赖（代码、运行时、系统工具、系统库等）打包到一个可移植的**容器**中，确保应用在任何环境中都能一致运行。

**核心思想**：Build Once, Run Anywhere（一次构建，到处运行）。

![Docker Logo](https://www.docker.com/wp-content/uploads/2022/03/horizontal-logo-monochromatic-white.png)

### Docker 解决的问题

| 问题 | Docker 如何解决 |
|------|----------------|
| **环境不一致** | 将应用和依赖打包在一起，消除"在我机器上能跑"问题 |
| **应用隔离** | 每个容器有独立的进程空间、网络、文件系统，互不干扰 |
| **快速部署** | 容器秒级启动，远超虚拟机分钟级启动 |
| **资源利用率高** | 共享宿主机内核，无需虚拟化整个OS，资源开销极低 |
| **持续集成/部署(CI/CD)** | 镜像版本化，方便构建、测试、部署流水线 |
| **微服务架构** | 每个微服务运行在独立容器中，解耦、独立扩缩容 |

### 面试题

**Q: 什么是 Docker？**

Docker 是一个开源容器引擎，基于 Go 语言开发，遵循 Apache 2.0 协议。它让开发者可以将应用及依赖打包到一个轻量级、可移植的容器中，实现环境一致性和快速部署。Docker 底层使用 Linux 的 Namespace 做隔离、Cgroups 做资源限制、UnionFS 做分层存储。

**Q: Docker 有什么优势？**

1. **轻量**：共享宿主机内核，启动快、资源占用少
2. **可移植**：一次构建，到处运行
3. **隔离性**：每个容器独立运行，互不影响
4. **版本化**：镜像分层 + Tag，方便回滚
5. **生态丰富**：Docker Hub、Harbor 等镜像仓库，Kubernetes 编排

**Q: 为什么使用 Docker？**

- 统一开发、测试、生产环境
- 简化部署流程
- 高效利用服务器资源
- 支撑微服务和云原生架构

---

## 1.2 Docker 与虚拟机区别 ★★★★★

### 虚拟机架构

```
+--------------------------------------------------+
|  App A     |  App B     |  App C     |            |
|  Bins/Libs |  Bins/Libs |  Bins/Libs |            |
+--------------------------------------------------+
|  Guest OS  |  Guest OS  |  Guest OS  |            |
+--------------------------------------------------+
|              Hypervisor (VMware/KVM)              |
+--------------------------------------------------+
|              Host Operating System                |
+--------------------------------------------------+
|              Physical Hardware (CPU/Mem/Disk)     |
+--------------------------------------------------+
```

### Docker 架构

```
+--------------------------------------------------+
|  App A     |  App B     |  App C     |            |
|  Bins/Libs |  Bins/Libs |  Bins/Libs |            |
+--------------------------------------------------+
|              Docker Engine                        |
+--------------------------------------------------+
|              Host Operating System                |
+--------------------------------------------------+
|              Physical Hardware (CPU/Mem/Disk)     |
+--------------------------------------------------+
```

### 对比

| 对比维度 | Docker 容器 | 虚拟机 (VM) |
|---------|------------|------------|
| **操作系统** | 共享宿主机内核 | 每个VM有独立 Guest OS |
| **启动速度** | 秒级 | 分钟级 |
| **资源占用** | MB 级别 | GB 级别 |
| **性能** | 接近原生（CPU损耗<2%） | 有较大性能损耗 |
| **隔离级别** | 进程级隔离（弱隔离） | 硬件级隔离（强隔离） |
| **镜像大小** | 几十MB（如Alpine: 5MB） | 几GB |
| **迁移** | 镜像Push/Pull即可 | 需要迁移整个VM |
| **密度** | 一台物理机可运行成百上千个 | 一台物理机只能运行十几个 |
| **持久化** | Volume / Bind Mount | 虚拟磁盘 |

### 面试题

**Q: Docker 和虚拟机有什么区别？**

核心区别：**虚拟机虚拟硬件，Docker 虚拟操作系统**。

- 虚拟机在 Hypervisor 层之上运行完整的 Guest OS，每个 VM 有独立内核
- Docker 容器共享宿主机内核，只隔离用户空间（进程、网络、文件系统等）
- 因此 Docker 更轻量、启动更快、资源利用率更高
- 但 VM 隔离更彻底，安全性更高

**Q: Docker 为什么更轻量？**

1. **无需 Guest OS**：直接复用宿主机内核，省去操作系统开销
2. **分层镜像**：镜像分层复用，公共层共享，减少存储和传输
3. **按需分配资源**：不像 VM 预先分配固定内存/CPU
4. **进程级虚拟化**：本质是宿主机上一个进程组，启动即进程创建

---

## 1.3 Docker 核心概念 ★★★★★

### Image（镜像）

镜像是一个**只读模板**，包含运行应用所需的完整文件系统和配置。镜像是分层构建的，每一层都是只读的。

```
镜像 = 基础层(ubuntu:22.04) + 依赖层(apt install) + 应用层(COPY binary) + 配置层(ENV, CMD)
```

**特性**：
- 分层存储（Layer），每层有唯一ID
- 只读，容器运行时在其上叠加可写层
- 通过 Dockerfile 构建
- 存储在 Registry 中

### Container（容器）

容器是镜像的**运行实例**。可以理解为"镜像 = 类，容器 = 对象"。

```
容器 = 镜像(只读层) + 可写层(Container Layer) + 运行时状态(进程、网络、挂载)
```

**特性**：
- 创建时在镜像层之上添加可写层（Copy-on-Write）
- 删除容器时，可写层数据丢失（除非挂载Volume）
- 可以启动、停止、删除、暂停
- 每个容器有独立的 PID、网络、挂载命名空间

### Registry（镜像仓库）

Registry 是存储和分发 Docker 镜像的服务。

- **Docker Hub**：官方公共 Registry（类似 GitHub）
- **Harbor**：企业级私有 Registry（VMware开源）
- **阿里云容器镜像服务**：国内常用
- **私有 Registry**：`docker run -d -p 5000:5000 registry:2`

### Repository（仓库）

Repository 是 Registry 中一组相关镜像的集合。

```
Registry: docker.io
  └── Repository: library/nginx
       ├── Tag: 1.25 (指向镜像层ID: sha256:abc...)
       ├── Tag: 1.24
       └── Tag: latest
```

### Tag（标签）

Tag 是镜像的版本标识，默认是 `latest`。

```
docker pull nginx:1.25.0
# registry/repository:tag → docker.io/library/nginx:1.25.0
```

### 面试题

**Q: Image 和 Container 区别？**

| Image | Container |
|-------|-----------|
| 静态的、只读的文件系统模板 | 动态的运行实例 |
| 类比：类 (Class) | 类比：对象 (Instance) |
| 分层存储，每层不可变 | 在镜像层上加可写层 |
| 可被多个容器共享 | 每个容器有独立可写层 |
| 存储在 Registry | 运行在 Docker Engine 上 |

**Q: Docker 镜像是什么？**

Docker 镜像是分层构建的只读文件系统模板。每一层是一个文件系统变更（如安装软件、复制文件），层之间通过 UnionFS 叠加。镜像包含应用代码、运行时、系统工具、系统库和配置。镜像不可变，只能通过构建新镜像来修改。

---

# 第二章 Docker 底层原理 ★★★★★

## 2.1 Docker 整体架构

Docker 采用 **Client-Server (C/S) 架构**：

```
┌────────────────────────────────────────────────────┐
│              Docker Client (docker CLI)            │
│           docker build / pull / run / push          │
└──────────────────┬─────────────────────────────────┘
                   │ REST API (Unix Socket / TCP)
                   ▼
┌────────────────────────────────────────────────────┐
│           Docker Daemon (dockerd)                   │
│  ┌──────────────────────────────────────────────┐  │
│  │          containerd (容器运行时管理)          │  │
│  │  ┌──────────────────────────────────────────┐│  │
│  │  │     runc (OCI 容器运行时)                 ││  │
│  │  │  - Namespace 隔离                        ││  │
│  │  │  - Cgroups 资源限制                      ││  │
│  │  │  - UnionFS 文件系统                      ││  │
│  │  └──────────────────────────────────────────┘│  │
│  └──────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

### 各组件职责

| 组件 | 职责 |
|------|------|
| **Docker Client** | 用户交互，发送命令给 Daemon |
| **Docker Daemon (dockerd)** | 管理镜像、容器、网络、数据卷 |
| **containerd** | 容器生命周期管理（拉取镜像、创建/启动/停止容器） |
| **runc** | OCI 标准容器运行时，实际创建容器（调用 Linux 内核 Namespace/Cgroups） |
| **containerd-shim** | 守护容器进程，允许 containerd 重启而不影响运行中的容器 |

### 面试题

**Q: Docker 架构是什么？**

Docker 是 C/S 架构。Docker Client 通过 REST API 与 Docker Daemon 通信。Daemon 管理镜像构建、容器编排、网络和数据卷。Daemon 通过 containerd 管理容器生命周期，最终由 runc 调用 Linux 内核特性（Namespace、Cgroups、UnionFS）创建和运行容器。

---

## 2.2 Namespace（命名空间） ★★★★★

Namespace 是 Linux 内核提供的**资源隔离**机制，让每个进程组看到独立的系统资源视图。Docker 利用 Namespace 实现容器之间的隔离。

### 六种 Namespace

| Namespace | 内核版本 | 隔离内容 | Docker 中的应用 |
|-----------|---------|---------|----------------|
| **PID** | 2.6.24 | 进程ID隔离 | 容器内进程 PID=1，看不到宿主机其他进程 |
| **Network** | 2.6.29 | 网络设备、IP、端口、路由 | 每个容器有独立网卡、IP、路由表 |
| **Mount** | 2.4.19 | 文件系统挂载点 | 容器只能看到自己的文件系统 |
| **IPC** | 2.6.19 | 进程间通信（信号量、消息队列） | 容器间 IPC 资源隔离 |
| **UTS** | 2.6.19 | 主机名和域名 | 每个容器有独立的 hostname |
| **User** | 3.8 | 用户和组ID映射 | 容器内 root 映射到宿主机普通用户 |

### PID Namespace

```
宿主机视角：
PID 1: systemd
PID 1000: dockerd
PID 2000: container-nginx (PID 1 inside container)

容器内视角：
PID 1: nginx
PID 10: nginx worker
# 看不到宿主机的其他进程
```

### Network Namespace

每个容器有独立的：
- 网络接口（eth0 等）
- IP 地址（如 172.17.0.2）
- 路由表
- iptables 规则
- 端口空间（容器内 80 端口 vs 宿主机 8080 端口互不冲突）

### 面试题

**Q: Docker 如何实现资源隔离？**

Docker 使用 Linux 内核的 **Namespace** 机制实现资源隔离：
- PID Namespace：进程隔离
- Network Namespace：网络隔离
- Mount Namespace：文件系统隔离
- IPC Namespace：进程间通信隔离
- UTS Namespace：主机名隔离
- User Namespace：用户权限隔离

每个容器启动时，Docker 会创建一组新的 Namespace，容器内的进程只能看到自己 Namespace 内的资源。

**Q: Namespace 有哪些类型？**

六种：PID、Network、Mount、IPC、UTS、User。（见上表）

---

## 2.3 Cgroups（控制组） ★★★★★

Cgroups (Control Groups) 是 Linux 内核提供的**资源限制和监控**机制，Docker 用它来限制容器可使用的 CPU、内存、IO 等资源。

### Cgroups 子系统

| 子系统 | 功能 |
|--------|------|
| **cpu** | 限制 CPU 使用率（cfs_period / cfs_quota） |
| **cpuacct** | CPU 使用统计 |
| **cpuset** | 绑定 CPU 核心和内存节点 |
| **memory** | 限制内存使用量，触发 OOM |
| **blkio** | 限制块设备 IO |
| **devices** | 控制设备访问权限 |
| **freezer** | 暂停/恢复进程组 |
| **net_cls** | 标记网络包，配合 tc 限流 |

### CPU 限制

```bash
# 限制容器只能使用 1.5 个 CPU 核心
docker run --cpus="1.5" myapp

# 绑定到 CPU 0 和 CPU 1
docker run --cpuset-cpus="0-1" myapp

# CPU 权重（相对值，默认1024）
docker run --cpu-shares=512 myapp
```

### Memory 限制

```bash
# 限制内存 512MB
docker run --memory="512m" myapp

# 限制内存+swap共1GB
docker run --memory="512m" --memory-swap="1g" myapp

# 内存超限时，内核 OOM Killer 会杀死容器进程
```

### 面试题

**Q: Docker 如何限制资源？**

Docker 使用 Linux 内核的 **Cgroups** 机制限制容器资源：
- 通过 `cfs_period` 和 `cfs_quota` 限制 CPU 使用率
- 通过 `memory.limit_in_bytes` 限制内存上限
- 通过 `blkio` 限制磁盘 IOPS/BPS
- `docker run` 的 `--cpus`、`--memory` 等参数最终写入 Cgroups 配置文件

**Q: Cgroups 原理是什么？**

Cgroups 将进程分组，对每组进程的资源使用进行统计和限制。每个子系统在 `/sys/fs/cgroup/` 下有对应的层级目录，Docker 为每个容器创建子目录，将容器进程 PID 写入 `tasks` 文件，资源限制参数写入对应的配置文件。内核根据这些配置对进程组实施资源管控。

---

## 2.4 Union File System（联合文件系统） ★★★★★

UnionFS 是一种**分层、轻量级、高性能**的文件系统，它将多个目录叠加挂载到同一个挂载点，呈现为一个统一的文件系统视图。

Docker 镜像的分层存储和容器的可写层都基于 UnionFS 实现。

### OverlayFS

OverlayFS 是当前 Docker 默认的存储驱动（取代了 AUFS）。

```
┌─────────────────────────────────────┐
│          Container Mount            │
│          (统一视图)                   │
├─────────────────────────────────────┤
│  UpperDir (可写层 - 容器修改)        │  ← 所有写操作写入这里
├─────────────────────────────────────┤
│  LowerDir (只读层 - 镜像层)          │
│  ┌─────────────────────────────────┐│
│  │  Layer 3: 应用代码 (app:v2)     ││
│  │  Layer 2: 依赖 (pip install)    ││
│  │  Layer 1: 基础系统 (ubuntu)     ││ ← 层层叠加
│  └─────────────────────────────────┘│
└─────────────────────────────────────┘
```

**Copy-on-Write (写时复制)**：
- 读取文件：从上往下查找，第一个匹配即返回
- 修改文件：将文件从 LowerDir 复制到 UpperDir，在 UpperDir 修改
- 删除文件：在 UpperDir 创建 "whiteout" 文件标记删除
- 新建文件：直接写入 UpperDir

### 分层存储的优势

1. **复用公共层**：多个镜像共享相同的基础层，节省磁盘空间
2. **加速构建**：未变化的层直接使用缓存，只重建变更层
3. **快速分发**：拉取镜像时已有层不需要重复下载
4. **版本管理**：每个 Tag 指向不同的层组合

### 面试题

**Q: Docker 镜像为什么体积小？**

1. **分层复用**：多个镜像共享基础层（如 ubuntu 层），不用每个镜像存一份完整 OS
2. **不包含内核**：复用宿主机内核，无需打包操作系统内核
3. **精简基础镜像**：Alpine 镜像只有 ~5MB（使用 musl libc 替代 glibc）
4. **按需打包**：只包含应用所需依赖，不像 VM 包含完整 OS

**Q: OverlayFS 原理是什么？**

OverlayFS 将多个目录合并为一个挂载点：
- **LowerDir**：只读层，可以有多个，按优先级排序
- **UpperDir**：可写层，所有修改写入这里
- **MergedDir**：统一视图，对用户透明
- **WorkDir**：内部工作目录

读取时从 MergedDir 读取（自动从 LowerDir/UpperDir 查找），写入时采用 Copy-on-Write 策略。

---

# 第三章 Docker Image 镜像 ★★★★★

## 3.1 镜像结构

### 只读层 (Read-Only Layers)

镜像由多个只读层堆叠而成，每一层是一个文件系统变更的增量：

```
sha256:abc123  ←  Layer 4: COPY app binary
sha256:def456  ←  Layer 3: RUN go build
sha256:ghi789  ←  Layer 2: RUN apt install
sha256:jkl012  ←  Layer 1: FROM ubuntu:22.04
```

每一层有唯一的 **Content-Addressable ID**（基于内容哈希），确保不可篡改。

### 可写层 (Container Layer)

容器启动时，Docker 在镜像层之上添加一个薄的可写层。容器运行期间的所有修改（写文件、删除、安装软件）都发生在这个可写层。

**特点**：
- 使用 Copy-on-Write 策略
- 容器删除时可写层随之删除
- 数据持久化需要使用 Volume

### Layer

Dockerfile 中每条指令（FROM、RUN、COPY 等）通常会创建一个新的镜像层。

### 面试题

**Q: Docker 镜像为什么分层？**

1. **复用共享层**：多个镜像可以共享相同的基础层，节省存储空间和传输带宽
2. **构建缓存**：未变化的层直接复用缓存，加速构建
3. **增量分发**：推送/拉取镜像时只传输变化的层
4. **版本管理**：通过层ID管理，实现内容寻址和完整性校验

---

## 3.2 镜像构建流程

### Docker Build

```
docker build -t myapp:v1 .
```

Docker build 的执行流程：

1. **解析 Dockerfile**：读取指令序列
2. **拉取基础镜像**：`FROM` 指定的镜像
3. **逐条执行指令**：每条指令创建一个临时容器 → 执行 → commit 为新的镜像层
4. **缓存判断**：检查该指令及其上下文是否变化，命中则复用已有层
5. **生成最终镜像**：所有层叠加，打上 Tag

### Layer Cache

```
FROM node:18                    ←  Layer 1: 检查基础镜像是否变化
WORKDIR /app                    ←  Layer 2: 检查指令是否变化
COPY package*.json ./           ←  Layer 3: 检查文件内容是否变化
RUN npm install                 ←  Layer 4: 如果 Layer 3 未变，此层用缓存
COPY . .                        ←  Layer 5: 源码变化，此层及之后全部重建
```

**缓存失效规则**：
- 如果某一层的变化被检测到，该层及之后所有层缓存失效
- 合理安排 Dockerfile 指令顺序可以最大化缓存利用率

### 面试题

**Q: Docker Build 发生了什么？**

Docker build 过程：
1. Docker Client 将构建上下文（build context）发送给 Docker Daemon
2. Daemon 逐条解析 Dockerfile 指令
3. 每条指令在一个临时容器中执行
4. 执行完成后 `docker commit` 为新的镜像层
5. 下一条指令基于上一层继续
6. 最后为镜像打上 Tag
7. 清理中间容器

整个过程实现了**不可变基础设施**的理念——每次构建生成全新的镜像。

### Build Context（构建上下文）★★★

**定义**：`docker build` 时，Docker Client 会将指定目录（通常是 `.`）下的所有文件打包发送给 Docker Daemon，这个打包的文件集合就是 **Build Context**。

```bash
docker build -t myapp:v1 .
#                          ↑ 这个 . 就是 Build Context 的根目录
```

**工作流程**：
```
docker build -t myapp:v1 .
      │
1. Docker Client 将当前目录打包为 tar
      │
2. 通过 REST API 发送给 Docker Daemon
      │  ├── 注意：会发送目录下所有文件！
      │  └── 如果 .git/、node_modules/ 没被排除，也会发送
      │
3. Daemon 解压后逐条执行 Dockerfile 指令
      │
4. COPY / ADD 只能从 Build Context 中复制文件
      │  ├── 不能 COPY ../parent-dir/file（超出 Context 范围）
      │  └── 不能 COPY /etc/hosts（不能用绝对路径）
```

**常见问题**：
```bash
# 问题：发送了几百MB的 node_modules
docker build -t myapp:v1 .

# 输出：
# Sending build context to Docker daemon  543.2MB  ← 太慢了！
```

**.dockerignore 的作用**：

```
# .dockerignore —— 排除不需要发送到 Build Context 的文件
.git
.gitignore
node_modules
*.log
.DS_Store
README.md
test/
.build/
dist/
```

**面试题**：

**Q: Build Context 是什么？**

Build Context 是 `docker build` 时传递给 Docker Daemon 的文件集合。Client 会打包指定目录发送给 Daemon，Daemon 在 Build Context 范围内执行 COPY/ADD 指令。最佳实践：使用 `.dockerignore` 排除不需要的文件，减少传输大小，加速构建。

**Q: 为什么 COPY 不能使用绝对路径？**

因为 COPY 只能从 Build Context 中复制文件，Build Context 是发送到 Daemon 的一个 tar 包，不包含宿主机上的绝对路径。COPY 的源路径必须是相对于 Build Context 的相对路径。

---

## 3.3 镜像优化 ★★★★★

### 多阶段构建 (Multi-Stage Build)

```dockerfile
# Stage 1: 构建阶段（大镜像，包含编译工具）
FROM golang:1.21 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .

# Stage 2: 运行阶段（小镜像，只有二进制）
FROM alpine:3.19
RUN apk add --no-cache ca-certificates tzdata
COPY --from=builder /app/server /app/server
USER nobody
EXPOSE 8080
CMD ["/app/server"]
```

**效果**：将构建依赖留在第一阶段，最终镜像只包含运行时所需，体积从 ~800MB 降到 ~15MB。

### 减少层数

```dockerfile
# 不好：多个 RUN 产生多层
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN rm -rf /var/lib/apt/lists/*

# 好：合并为一个 RUN，在同一个层完成
RUN apt-get update && \
    apt-get install -y curl vim && \
    rm -rf /var/lib/apt/lists/*
```

### Alpine 镜像

Alpine Linux 是一个面向安全的轻量级 Linux 发行版：
- 大小约 5MB
- 使用 musl libc + busybox 替代 glibc + coreutils
- 默认无 Shell、无 package manager（需 `apk add`）
- 可能带来 glibc/musl 二进制兼容性问题（Go 静态编译无此问题）

### 面试题

**Q: 如何减小镜像体积？**

1. **多阶段构建**：分离构建环境和运行环境，最终镜像只包含运行时产物
2. **使用 Alpine 基础镜像**：5MB vs ubuntu 的 77MB
3. **减少层数**：合并 RUN 指令，清理包管理缓存
4. **使用 `.dockerignore`**：排除不需要的文件（node_modules、.git、测试文件）
5. **精确复制**：只 COPY 必要的文件，不整个目录复制
6. **使用 Distroless 镜像**：Google 的精简镜像，连 Shell 都没有
7. **静态编译**：如 Go 程序 `CGO_ENABLED=0` 编译为静态二进制，可以用 `FROM scratch`

### Go 应用镜像最佳实践 ★★★★

Go 语言天然适合容器化，因为可以编译为**纯静态二进制**，不依赖任何动态库。

**编译优化**：

```dockerfile
# ✅ 生产级 Go 构建
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" -o /app/server .

# -ldflags="-w -s" 的作用：
# -w: 去掉 DWARF 调试信息（缩小 ~30%）
# -s: 去掉符号表（strip symbols）
# 效果：20MB 的二进制 → ~13MB
```

**FROM scratch 极限精简**：

```dockerfile
# ✅ 最极致：FROM scratch（空镜像，0 字节）
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-w -s" -o server .

FROM scratch
# scratch 没有任何文件，需要手动复制 CA 证书（HTTPS 请求必需）
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

| 基础镜像 | 镜像大小 | 说明 |
|---------|---------|------|
| `golang:1.21` | ~800MB | 含 Go SDK，仅用于构建 |
| `ubuntu:22.04` + 二进制 | ~80MB | 完整 Linux 环境 |
| `alpine:3.19` + 二进制 | ~15MB | 推荐，有基本工具 |
| `scratch` + 二进制 + CA | ~7MB | 极简，无 Shell、无包管理 |

**scratch 的注意事项**：
- 没有 Shell（不能 `docker exec /bin/sh` 进入调试）
- 没有 CA 证书（HTTPS 请求需要手动复制）
- 没有时区信息（需要复制 `/usr/share/zoneinfo` 或用代码设置）
- 适合对安全要求极高、攻击面最小化的场景

**二进制体积进一步优化**：

| 方法 | 效果 |
|------|------|
| `-ldflags="-w -s"` | 去掉调试信息和符号表 |
| `GOOS=linux GOARCH=amd64` | 交叉编译，不需要 VM |
| `CGO_ENABLED=0` | 禁用 CGO，纯 Go 静态链接 |
| UPX 压缩 | 可再压缩 50-70%（但启动时需解压，略增内存和启动时间） |

**面试题**：

**Q: Go 程序的 Docker 镜像为什么可以做到这么小？**

Go 编译为纯静态二进制，不依赖 libc（设置 `CGO_ENABLED=0`）。可以用 `FROM scratch`（空镜像）作为基础镜像，只放入编译好的二进制和 CA 证书，最终镜像体积只有 5-8MB。对比 Java/Node.js 需要 JDK/JRE/Node 运行时几百 MB，Go 在容器化方面有天然优势。

---

## 3.4 镜像仓库

### Docker Hub

官方公共镜像仓库：`https://hub.docker.com/`

```bash
docker pull nginx                    # docker.io/library/nginx:latest
docker pull bitnami/redis            # 第三方维护的镜像
docker push myusername/myapp:v1      # 推送自己的镜像
```

### Harbor

Harbor 是 VMware 开源的**企业级私有镜像仓库**：

**核心功能**：
- 镜像复制（跨数据中心同步）
- 漏洞扫描（Trivy / Clair 集成）
- 访问控制（项目、角色、LDAP）
- 镜像签名与内容信任
- 垃圾回收和配额管理
- Helm Chart 仓库
- CVE 漏洞报告

### 私有仓库

```bash
# 运行私有 Registry
docker run -d -p 5000:5000 --name registry registry:2

# 推送镜像到私有仓库
docker tag myapp:v1 localhost:5000/myapp:v1
docker push localhost:5000/myapp:v1
```

### 面试题

**Q: Harbor 有什么作用？**

Harbor 是企业级私有镜像仓库，提供：
1. **安全扫描**：自动扫描镜像中的已知漏洞 (CVE)
2. **访问控制**：项目级别的 RBAC 权限管理
3. **镜像复制**：跨数据中心/跨云同步镜像
4. **内容信任**：镜像签名，确保镜像未被篡改
5. **审计日志**：记录所有操作
6. **高可用**：支持主从复制、多实例部署

---

# 第四章 Docker Container 容器 ★★★★★

## 4.1 容器生命周期

```
          docker create
               │
               ▼
    ┌────── Created ──────┐
    │                     │
    │  docker start       │
    ▼                     │
  Running ──docker stop──► Stopped
    │  │                   │
    │  │  docker pause     │  docker rm
    │  ▼                   ▼
    │ Paused             (removed)
    │  │
    │  │  docker unpause
    │  ▼
    │ Running
    │
    │  docker kill / OOM / 进程崩溃
    ▼
   Dead ──────────────────► (removed)
```

### 容器状态

| 状态 | 说明 |
|------|------|
| **Created** | 已创建但未启动（`docker create`） |
| **Running** | 正在运行（`docker start` / `docker run`） |
| **Paused** | 暂停中，进程被冻结（`docker pause`） |
| **Restarting** | 正在重启（`--restart=always` 策略触发） |
| **Exited** | 已退出，主进程结束 |
| **Dead** | 被强制杀死，无法恢复 |

### 面试题

**Q: Docker 容器有哪些状态？**

Created、Running、Paused、Restarting、Exited、Dead。

---

## 4.2 容器启动流程 ★★★★★

### docker run 全过程

```
docker run -d -p 8080:80 --name web nginx
```

**详细步骤**：

```
1. docker CLI 解析命令参数
      │
2. Docker Client 通过 REST API 发送请求到 Docker Daemon
      │
3. Daemon 检查本地是否有 nginx 镜像
      ├── 有 → 跳到步骤5
      └── 没有 → 从 Registry 拉取镜像（分层下载，已有层跳过）
      │
4. Daemon 调用 containerd 创建容器
      │
5. containerd 调用 runc
      │
6. runc 创建容器运行时环境：
   ├── 创建 Namespace (PID/Network/Mount/IPC/UTS/User)
   ├── 创建 Cgroups (CPU/Memory/IO 限制)
   ├── 挂载镜像层 (UnionFS: 只读层 + 可写层)
   ├── 配置网络 (分配IP、创建 veth pair、连接网桥)
   └── 设置根文件系统 (pivot_root)
      │
7. runc 启动容器初始化进程 (tini/dumb-init 或 用户指定的 CMD/ENTRYPOINT)
      │
8. runc 退出，containerd-shim 接管容器进程
      │
9. 容器进入 Running 状态
```

### 面试题

**Q: docker run 发生了什么？**

1. Docker Client 发送请求给 Docker Daemon
2. Daemon 检查本地是否有镜像，无则从 Registry 拉取
3. containerd 接收创建容器请求
4. runc 创建 Namespace 做隔离、创建 Cgroups 做资源限制
5. 通过 OverlayFS 挂载镜像层 + 可写层
6. 配置网络（创建 veth pair、分配 IP、NAT）
7. 启动容器进程（CMD/ENTRYPOINT 指定的命令）
8. containerd-shim 接管进程，容器进入 Running 状态

---

## 4.3 容器数据管理

### Volume（数据卷）

由 Docker 管理的存储，独立于容器的生命周期。

```bash
# 创建 Volume
docker volume create mysql_data

# 挂载到容器
docker run -d \
  -v mysql_data:/var/lib/mysql \
  mysql:8.0

# Volume 存储在宿主机: /var/lib/docker/volumes/mysql_data/_data
```

**特点**：
- 由 Docker 管理，与宿主机文件系统解耦
- 容器删除了 Volume 不删除（除非 `docker rm -v`）
- 可以被多个容器共享
- 支持 Volume Driver 扩展（NFS、Ceph等）

### Bind Mount

将宿主机目录直接挂载到容器。

```bash
docker run -d \
  -v /home/user/app:/app \
  -v /home/user/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx
```

**特点**：
- 宿主机文件系统的任意路径
- 容器可以修改宿主机文件
- 适合开发和配置文件注入
- 路径与宿主机绑定，移植性差

### Tmpfs

将数据存储在宿主机内存中（非持久化）。

```bash
docker run -d --tmpfs /tmp nginx
```

### 面试题

**Q: Docker 数据如何持久化？**

三种方式：
1. **Volume**（推荐）：Docker 管理的卷，存储在 `/var/lib/docker/volumes/`
2. **Bind Mount**：挂载宿主机指定目录到容器
3. **Tmpfs**：存储在内存中，容器停止即消失

生产环境推荐 Volume，因为：
- 与容器生命周期解耦
- 可跨容器共享
- 支持存储驱动扩展

---

## 4.4 重启策略 ★★★★

Docker 提供 4 种重启策略，控制容器在退出时是否自动重启。

```bash
docker run --restart=no myapp        # 默认：不自动重启
docker run --restart=always myapp    # 总是重启
docker run --restart=on-failure:3 myapp  # 仅非零退出码时重启，最多3次
docker run --restart=unless-stopped myapp  # 除非手动stop，否则重启
```

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| **no** | 不自动重启（默认） | 一次性任务、调试 |
| **always** | 无论退出码如何，总是重启 | 常驻服务（Web 服务） |
| **on-failure[:N]** | 仅退出码非0时重启，可限制重试次数 | 需要容错但避免无限重启 |
| **unless-stopped** | 总是重启，除非被 `docker stop` 手动停止 | 生产环境首选 |

**区别**：`always` 和 `unless-stopped` 的细微差别：
- `always`：即使 Daemon 重启也会自动启动容器（Docker 启动时拉起来）
- `unless-stopped`：如果容器被手动 `docker stop`，Daemon 重启后不会自动启动

### 面试题

**Q: Docker 重启策略有哪些？**

no（不重启）、always（总是重启）、on-failure（非零退出码重启）、unless-stopped（除非手动停止，否则重启）。生产环境推荐 `unless-stopped`，一次性任务用默认 `no`。

---

## 4.5 docker stop 与 docker kill ★★★★★

### 区别

| 维度 | `docker stop` | `docker kill` |
|------|--------------|---------------|
| **发送信号** | SIGTERM (15) | SIGKILL (9) |
| **应用感知** | 应用可捕获，做优雅退出 | 应用无法捕获，立即杀死 |
| **默认等待** | 等待 10 秒后强制 SIGKILL | 立即杀死 |
| **自定义信号** | 不支持 | `docker kill -s SIGNAL` |

### docker stop 流程

```
docker stop <container>
      │
1. 向容器 PID 1 发送 SIGTERM 信号
      │
2. 等待容器退出（默认 10 秒 timeout）
      │
3. 如果 10 秒内未退出 → 发送 SIGKILL 强制杀死
      │
4. 容器状态变为 Exited

# 自定义 timeout
docker stop -t 30 <container>  # 等待 30 秒
```

### 面试题

**Q: docker stop 和 docker kill 区别？**

- `docker stop`：发送 SIGTERM，给应用 10 秒时间做优雅关闭（释放连接、保存状态），超时后 SIGKILL
- `docker kill`：直接发送 SIGKILL，立即杀死进程，无法捕获
- Go 程序应在 `signal.Notify` 中处理 SIGTERM 来实现优雅关闭

---

## 4.6 容器 PID 1 问题 & 僵尸进程 ★★★★

### PID 1 的特殊性

在 Linux 中，PID 1（init 进程）有三个特殊职责：

1. **接收信号**：内核只向 PID 1 发送信号
2. **回收孤儿进程**：父进程退出后，子进程变为孤儿，由 PID 1 收养并回收
3. **防止僵尸进程**：子进程退出后，父进程需调用 `wait()` 回收；如果父进程不回收，子进程成为僵尸进程

### Shell 形式的问题

```dockerfile
# ❌ Shell 形式：PID 1 是 /bin/sh，不是你的应用
CMD java -jar app.jar

# 实际运行：
# PID 1: /bin/sh -c "java -jar app.jar"
# PID 7: java -jar app.jar

# 问题1：docker stop 发送 SIGTERM → /bin/sh 不转发给 java → 10秒后 SIGKILL
# 问题2：java 进程产生僵尸子进程时，/bin/sh 不回收
```

### Exec 形式

```dockerfile
# ✅ Exec 形式：PID 1 就是你的应用
CMD ["java", "-jar", "app.jar"]
# PID 1: java -jar app.jar

# 优点：SIGTERM 直接发送给应用
# 缺点：应用必须自己处理 SIGTERM 和回收子进程
```

### tini / dumb-init（推荐）

```dockerfile
# ✅ 最佳实践：使用轻量级 init 进程
FROM alpine:3.19
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["/app/server"]
# PID 1: tini（正确处理信号转发 + 僵尸进程回收）
# PID 7: /app/server
```

```bash
# 或者用 docker run --init
docker run --init myapp
# 自动注入 tini 作为 PID 1
```

### 僵尸进程原理

```
父进程 fork 子进程
  → 子进程退出
  → 子进程保留在进程表中（状态为 Z），释放大部分资源
  → 父进程调用 wait() / waitpid() 回收
  → 子进程彻底清除
  → 如果父进程不回收 → 子进程永久僵尸 → 耗尽 PID 资源
```

### 面试题

**Q: 为什么容器需要 tini / dumb-init？**

1. **信号转发**：Shell 形式的 PID 1 是 `/bin/sh`，不会转发 SIGTERM 给子进程，导致无法优雅退出
2. **僵尸进程回收**：正常应用通常不处理 `wait()`，子进程退出后会变成僵尸进程
3. tini 是一个极小的 init 进程，专门解决这两个问题：正确转发信号 + 回收僵尸进程
4. Docker 1.13+ 提供 `--init` 参数自动注入 tini

**Q: Shell 形式和 Exec 形式的本质区别是什么？**

- Shell 形式：`CMD command arg` → 实际运行 `["/bin/sh", "-c", "command arg"]`，PID 1 是 Shell
- Exec 形式：`CMD ["command", "arg"]` → 直接运行，PID 1 就是 command
- Shell 形式适合简单场景，Exec 形式是推荐做法，信号处理正确

---

## 4.7 容器退出码详解

| 退出码 | 含义 | 常见原因 |
|--------|------|---------|
| **0** | 正常退出 | 程序正常结束 |
| **1** | 应用错误 | 程序异常退出 |
| **2** | 误用 Shell 内置命令 | 语法错误 |
| **126** | 命令无法执行 | 权限不足，非可执行文件 |
| **127** | 命令未找到 | 路径错误，依赖缺失 |
| **128+N** | 被信号 N 终止 | |
| **137** | SIGKILL (128+9) | OOM Killer、docker kill |
| **139** | SIGSEGV (128+11) | 段错误，内存越界 |
| **143** | SIGTERM (128+15) | docker stop 发送 |

```bash
# 查看容器退出码
docker inspect -f '{{.State.ExitCode}}' <container_id>
```

---

# 第五章 Docker 网络 ★★★★★

## 5.1 Docker 网络模型

Docker 使用 **CNM (Container Network Model)** 网络模型：

```
┌──────────────────────────────────────────┐
│             Sandbox (网络命名空间)         │
│  ┌──────────────────────────────────────┐ │
│  │         Endpoint (veth pair一端)      │ │
│  └──────────────┬───────────────────────┘ │
└─────────────────┼─────────────────────────┘
                  │ 连接
┌─────────────────┼─────────────────────────┐
│  ┌──────────────▼───────────────────────┐ │
│  │        Network (网桥/Overlay)        │ │
│  │         docker0 / overlay / ...      │ │
│  └──────────────────────────────────────┘ │
│               Docker Network              │
└──────────────────────────────────────────┘
```

### 网络模式

| 网络模式 | 说明 | 使用场景 |
|---------|------|---------|
| **bridge** | 默认模式，通过 docker0 网桥 + NAT 通信 | 单机容器通信 |
| **host** | 直接使用宿主机网络栈 | 高性能网络场景 |
| **none** | 无网络，只有 lo 接口 | 安全隔离、自定义网络 |
| **overlay** | 跨宿主机容器通信（VXLAN） | Docker Swarm / 多主机 |
| **macvlan** | 容器直接使用物理网卡 MAC 地址 | 需要直接接入物理网络 |

### 面试题

**Q: Docker 网络模式有哪些？**

Bridge（默认网桥模式）、Host（宿主机网络）、None（无网络）、Overlay（跨主机网络VXLAN）、Macvlan（直接接入物理网络）。

---

## 5.2 Bridge 网络 ★★★★★

### docker0 网桥

```
宿主机网络栈
┌─────────────────────────────────────────────────────┐
│  Physical NIC (eth0) 10.0.0.10                      │
│                                                     │
│  docker0 网桥 172.17.0.1/16                         │
│  ┌─────────────────────────────────────────────┐    │
│  │       veth pair                              │    │
│  │  ┌──────────┐    ┌──────────┐               │    │
│  │  │ Container│    │ Container│               │    │
│  │  │ A        │    │ B        │               │    │
│  │  │eth0      │    │eth0      │               │    │
│  │  │172.17.0.2│    │172.17.0.3│               │    │
│  │  └────┬─────┘    └────┬─────┘               │    │
│  │       │               │                      │    │
│  │  ┌────▼───────────────▼─────┐               │    │
│  │  │  vethXXXXX     vethYYYYY │  ← veth pair宿主机端
│  │  └──────────────────────────┘               │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  iptables NAT (SNAT / DNAT / MASQUERADE)            │
└─────────────────────────────────────────────────────┘
```

### veth pair

veth pair 是 Linux 虚拟以太网设备对，像一根网线连接两端：
- 一端插入容器的 Network Namespace（作为 eth0）
- 一端插入 docker0 网桥

### NAT（网络地址转换）

**容器访问外网 (SNAT/MASQUERADE)**：
```
容器(172.17.0.2) → docker0 → iptables MASQUERADE → eth0(宿主机IP) → 外网
```

**外网访问容器 (DNAT/端口映射)**：
```
外网 → 宿主机:8080 → iptables DNAT → 172.17.0.2:80
```

### 面试题

**Q: Docker 容器如何访问外网？**

通过 SNAT（源地址转换）：
1. 容器发送数据包到 docker0 网桥
2. iptables MASQUERADE 规则将源IP从 172.17.0.x 替换为宿主机 IP
3. 数据包通过宿主机网卡发出
4. 回包时反向转换目的IP，送回容器

**Q: Bridge 网络原理是什么？**

docker0 是一个 Linux 虚拟网桥（类似交换机），所有使用 bridge 模式的容器通过 veth pair 连接到 docker0。同一网桥上的容器可以直接通过 IP 通信（二层转发）。容器访问外网通过 iptables NAT 实现，外部访问容器通过端口映射（DNAT）实现。

---

## 5.3 Host 网络

### 特点

容器直接使用宿主机网络栈，不创建独立的 Network Namespace。

```bash
docker run --network host nginx
# 容器内 nginx 直接监听宿主机的 80 端口
```

- 没有独立 IP，与宿主机共享网络栈
- 不需要端口映射（`-p` 无效）
- 性能最高（无网络转换开销）
- 端口冲突风险（容器端口 = 宿主机端口）

### 使用场景

- 高性能网络需求（如高频交易、实时流处理）
- 需要监听大量端口
- 网络监控工具

---

## 5.4 Overlay 网络 ★★★★

### 跨主机通信

Overlay 网络允许不同宿主机上的容器直接通信（通常在 Docker Swarm 中使用）。

```
Host A                          Host B
┌──────────────┐               ┌──────────────┐
│ Container A  │               │ Container B  │
│ 10.0.0.2     │               │ 10.0.0.3     │
└──────┬───────┘               └──────┬───────┘
       │ veth                          │ veth
┌──────▼──────────────────────────────▼──────────┐
│             Overlay Network (10.0.0.0/24)      │
│              VXLAN Tunnel                       │
└──────────────────────┬─────────────────────────┘
                       │
              宿主机物理网络 (192.168.1.0/24)
```

### VXLAN

VXLAN (Virtual Extensible LAN) 将二层以太网帧封装在 UDP 包中，在物理网络上传输。

```
原始以太网帧
  → 封装 VXLAN Header (VNI 网络标识符)
  → 封装 UDP Header
  → 封装 IP Header (源宿主机IP → 目标宿主机IP)
  → 通过物理网络发送
  → 目标宿主机解封装
  → 送到目标容器
```

### 面试题

**Q: Overlay 网络如何实现？**

通过 VXLAN 隧道技术在物理网络之上构建虚拟网络。不同宿主机上的容器发送数据包时，VXLAN 将原始二层帧封装到 UDP 报文中，通过物理网络发送到目标宿主机，目标宿主机解封装后交付给目标容器。这样容器之间感觉自己在一个二层网络中，实际上它们的通信跨越了物理主机。

---

## 5.5 容器通信

### 容器与容器

**同一自定义网络**（推荐）：
```bash
docker network create mynet
docker run --network mynet --name web nginx
docker run --network mynet --name app myapp
# app 可以直接 ping web（DNS解析容器名）
```

**默认 Bridge 网络**：只能通过 IP 通信，无法通过容器名。

### 容器与宿主机

- 容器访问宿主机：`host.docker.internal`（Mac/Win）或宿主机 docker0 IP
- 宿主机访问容器：通过端口映射 `-p 8080:80`

### 容器与外部网络

通过 iptables MASQUERADE 实现 SNAT，容器访问外网。

### 面试题

**Q: 两个容器如何通信？**

- **同一台宿主机**：
  - 自定义网络：通过容器名 + Docker 内置 DNS 解析直接通信
  - 默认 Bridge：通过容器 IP（172.17.0.x）通信
- **不同宿主机**：通过 Overlay 网络（VXLAN 隧道）或宿主机端口映射

推荐做法：创建自定义 Bridge 网络，容器通过容器名互相访问。

---

# 第六章 Docker 存储 ★★★★

## 6.1 Volume

### 生命周期

- **创建**：`docker volume create myvol`
- **挂载到容器**：`docker run -v myvol:/data`
- **列出**：`docker volume ls`
- **查看详情**：`docker volume inspect myvol`
- **删除**：`docker volume rm myvol`（未被任何容器使用时）
- **清理**：`docker volume prune`（删除所有未使用的 Volume）

Volume 独立于容器的生命周期，容器删除后 Volume 仍然存在。

### 使用方式

```bash
# 命名 Volume
docker run -v mydata:/var/lib/mysql mysql:8.0

# 匿名 Volume（Docker 自动生成名称）
docker run -v /var/lib/mysql mysql:8.0

# docker-compose 中使用
services:
  mysql:
    image: mysql:8.0
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

### 面试题

**Q: Volume 和 Bind Mount 区别？**

| 维度 | Volume | Bind Mount |
|------|--------|------------|
| **管理** | Docker 管理 | 宿主机文件系统 |
| **存储位置** | `/var/lib/docker/volumes/` | 宿主机任意路径 |
| **可移植性** | 好（跨平台） | 差（依赖宿主机路径） |
| **权限** | Docker 管理 | 原有文件权限 |
| **性能** | 较好 | 取决于底层文件系统 |
| **容器间共享** | 容易 | 可以但不推荐 |
| **持久化** | 容器删除后保留 | 取决于宿主机 |

---

## 6.2 Bind Mount

### 目录挂载

```bash
# 挂载宿主机目录
docker run -v /opt/app/config:/etc/app/config:ro nginx

# 挂载当前目录
docker run -v $(pwd):/app node:18 npm start

# 开发环境热重载
docker run -v $(pwd)/src:/app/src -v /app/node_modules node:18 npm run dev
```

**注意**：Bind Mount 会覆盖容器内原有内容。

---

## 6.3 Tmpfs

### 内存存储

```bash
docker run --tmpfs /tmp:rw,size=128M nginx
```

数据存储在宿主机内存中，不写入磁盘，容器停止即丢失。适合缓存、临时文件。

---

## 6.4 数据持久化

### MySQL

```bash
docker run -d \
  --name mysql \
  -v mysql_data:/var/lib/mysql \
  -v mysql_config:/etc/mysql/conf.d \
  -e MYSQL_ROOT_PASSWORD=secret \
  mysql:8.0
```

### Redis

```bash
docker run -d \
  --name redis \
  -v redis_data:/data \
  redis:7-alpine redis-server --appendonly yes
```

### Kafka

```bash
docker run -d \
  --name kafka \
  -v kafka_data:/var/lib/kafka/data \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  confluentinc/cp-kafka:latest
```

### 面试题

**Q: 为什么数据库容器必须挂载 Volume？**

1. **数据持久化**：容器删除时，容器可写层的数据随之删除，Volume 中的数据保留
2. **性能**：直接写可写层有 Copy-on-Write 开销，Volume 直接写宿主机文件系统
3. **备份恢复**：Volume 可以独立备份、恢复、迁移
4. **数据共享**：多个容器可以共享同一个 Volume
5. **容器升级**：升级镜像时只需重建容器，数据不受影响

---

# 第七章 Dockerfile ★★★★★

## 7.1 Dockerfile 基础

Dockerfile 是构建镜像的文本文件，包含一系列指令。

### 常用指令

| 指令 | 说明 | 示例 |
|------|------|------|
| **FROM** | 指定基础镜像 | `FROM golang:1.21-alpine` |
| **RUN** | 执行命令（构建时） | `RUN apt-get update && apt-get install -y curl` |
| **COPY** | 复制文件（构建上下文→镜像） | `COPY . /app` |
| **ADD** | 复制文件 + 自动解压tar/URL下载 | `ADD app.tar.gz /app` |
| **CMD** | 容器启动时的默认命令 | `CMD ["/app/server"]` |
| **ENTRYPOINT** | 容器入口点 | `ENTRYPOINT ["/app/server"]` |
| **EXPOSE** | 声明容器监听的端口 | `EXPOSE 8080` |
| **ENV** | 设置环境变量 | `ENV APP_ENV=production` |
| **WORKDIR** | 设置工作目录 | `WORKDIR /app` |
| **ARG** | 构建参数（构建时可用） | `ARG VERSION=1.0` |
| **VOLUME** | 声明挂载点 | `VOLUME /data` |
| **USER** | 指定运行用户 | `USER nobody` |
| **LABEL** | 添加元数据 | `LABEL version="1.0"` |
| **HEALTHCHECK** | 健康检查 | `HEALTHCHECK --interval=30s CMD curl -f http://localhost/ || exit 1` |

### 最简 Dockerfile 示例

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/server /server
EXPOSE 8080
CMD ["/server"]
```

### ARG 与 ENV ★★★★★

| 维度 | ARG | ENV |
|------|-----|-----|
| **作用时机** | 仅构建时（`docker build`） | 构建时 + 运行时 |
| **可见范围** | Dockerfile 内部，构建参数 | 容器运行时环境变量 |
| **可被覆盖** | `docker build --build-arg VERSION=2.0` | `docker run -e APP_ENV=staging` |
| **默认值** | `ARG VERSION=1.0` | `ENV APP_ENV=production` |
| **多阶段** | 每个阶段独立，需重新声明 | 全局可用（FROM 前声明则全局） |
| **在 CMD/ENTRYPOINT 中** | ❌ 不可用 | ✅ 可用（作为环境变量） |

```dockerfile
# ARG：构建时用，运行时不可见
ARG GO_VERSION=1.21
FROM golang:${GO_VERSION}-alpine AS builder
ARG APP_VERSION          # 每个 FROM 后需重新声明
RUN echo "Building version ${APP_VERSION}"

# ENV：运行时也可用
ENV APP_ENV=production
ENV DB_HOST=mysql DB_PORT=3306

# 运行时覆盖 ENV
# docker run -e APP_ENV=staging -e DB_HOST=prod-db myapp
```

**面试题**：

**Q: ARG 和 ENV 区别？**

- ARG：仅在 `docker build` 期间可用，用于向 Dockerfile 传递构建参数（版本号、基础镜像版本等）
- ENV：构建时和运行时都可用，在容器内作为环境变量存在
- ARG 可以用 `--build-arg` 覆盖，ENV 可以用 `-e` 覆盖
- ARG 不应该用于传递敏感信息（会出现在 `docker history` 中），敏感信息用 `--secret` 或 BuildKit secrets

### HEALTHCHECK 健康检查 ★★★★

```dockerfile
# 基础用法：curl 检查 HTTP 端点
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# 检查 MySQL 是否可用
HEALTHCHECK --interval=10s --timeout=5s --retries=5 \
  CMD mysqladmin ping -h localhost || exit 1

# 禁用从基础镜像继承的 HEALTHCHECK
HEALTHCHECK NONE
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--interval` | 30s | 检查间隔 |
| `--timeout` | 30s | 单次检查超时 |
| `--start-period` | 0s | 启动缓冲期（此期间失败不计入重试） |
| `--retries` | 3 | 连续失败 N 次后标记为 unhealthy |

**容器状态**：
- `healthy`：检查通过
- `unhealthy`：连续失败达到 retries 次
- `starting`：在 start-period 内

```bash
# 查看健康状态
docker ps                            # STATUS 列显示 (healthy) / (unhealthy)
docker inspect --format='{{json .State.Health}}' <container_id> | jq
```

**面试题**：

**Q: Docker HEALTHCHECK 和 Kubernetes Probe 是什么关系？**

- Docker HEALTHCHECK 是容器级别的健康检查，Docker Daemon 执行
- Kubernetes 的 liveness/readiness probe 由 Kubelet 执行
- K8s 可以不依赖容器内 HEALTHCHECK，直接在 Pod spec 中定义 probe
- 如果两者都有，K8s probe 会覆盖 Docker HEALTHCHECK 的判断
- HEALTHCHECK 主要是 Docker Swarm / 纯 Docker 场景使用的

---

## 7.2 CMD 与 ENTRYPOINT ★★★★★

### 区别

| 维度 | CMD | ENTRYPOINT |
|------|-----|------------|
| **作用** | 设置默认命令和参数 | 设置容器入口点 |
| **可被覆盖** | `docker run` 参数完全覆盖 | `docker run` 参数作为追加参数 |
| **主要用途** | 提供默认行为，允许覆盖 | 定义固定的执行程序 |
| **组合使用** | CMD 提供默认参数给 ENTRYPOINT | ENTRYPOINT 执行程序，CMD 提供默认参数 |

### 三种写法

```dockerfile
# Shell 形式（通过 /bin/sh -c 执行）
CMD echo hello
ENTRYPOINT /app/server

# Exec 形式（推荐，直接执行，信号正确传递）
CMD ["/app/server", "--port", "8080"]
ENTRYPOINT ["/app/server"]

# 组合使用（ENTRYPOINT + CMD）
ENTRYPOINT ["/app/server"]
CMD ["--port", "8080"]
# 运行: /app/server --port 8080
# docker run myapp --port 3000 → /app/server --port 3000（CMD被覆盖）
# docker run myapp help → /app/server help（CMD被覆盖为help）
```

### 使用场景

```dockerfile
# CMD: 允许用户覆盖命令
FROM ubuntu
CMD ["bash"]
# docker run -it myimage bash   ← 默认
# docker run -it myimage sh     ← 覆盖

# ENTRYPOINT: 固定执行程序
FROM alpine
ENTRYPOINT ["curl"]
CMD ["--help"]
# docker run myimage                    → curl --help
# docker run myimage https://example.com → curl https://example.com
```

### 面试题

**Q: CMD 和 ENTRYPOINT 区别？**

- **CMD**：提供默认命令，`docker run` 后的参数会覆盖 CMD
- **ENTRYPOINT**：定义容器固定入口，`docker run` 后的参数作为追加参数传给 ENTRYPOINT
- 组合使用：ENTRYPOINT 定义可执行程序，CMD 定义默认参数，用户可以通过 `docker run` 覆盖默认参数

建议：使用 Exec 形式（`["executable", "arg1"]`），因为可以正确处理 Unix 信号（SIGTERM等）。

### Shell 形式 vs Exec 形式 ★★★★★

这是 Docker 面试中非常高频的考点，直接影响容器的**优雅退出**。

```dockerfile
# Shell 形式
CMD echo hello
CMD /app/server --port 8080
# 底层实际执行：/bin/sh -c "echo hello"
# 底层实际执行：/bin/sh -c "/app/server --port 8080"

# Exec 形式（JSON 数组，推荐）
CMD ["echo", "hello"]
CMD ["/app/server", "--port", "8080"]
# 底层直接调用 exec，无 Shell 中间层
```

| 维度 | Shell 形式 | Exec 形式 |
|------|-----------|-----------|
| **PID 1** | `/bin/sh`（不是你的应用） | 你的应用 |
| **SIGTERM** | /bin/sh 接收，**不转发**给子进程 | 直接发给应用 |
| **docker stop** | 应用无法优雅退出，10秒后被 SIGKILL | 应用收到 SIGTERM，可优雅退出 |
| **变量替换** | 自动展开（`$PATH`、`${VAR}`） | 不展开，需在 Shell 中处理 |
| **需要 Shell 特性** | ✅ 管道、重定向可用 | ❌ 如需 Shell 特性需手动 `/bin/sh -c` |
| **镜像大小** | 略大（需要 /bin/sh） | 更小（FROM scratch 可用） |
| **僵尸进程** | 可能产生（/bin/sh 不回收） | 取决于应用 |
| **推荐度** | ❌ 不推荐 | ✅ 推荐 |

**信号传递测试**：

```bash
# Shell 形式 —— docker stop 时应用收不到 SIGTERM
docker run -d --name test-shell myapp:shell
time docker stop test-shell
# real 0m10.050s  ← 等了整整 10 秒！超时后被 SIGKILL

# Exec 形式 —— 立即优雅退出
docker run -d --name test-exec myapp:exec
time docker stop test-exec
# real 0m0.380s  ← 应用收到 SIGTERM，立即优雅退出
```

**Go 程序中的信号处理**：

```go
// main.go —— 在容器中优雅退出
func main() {
    // 创建 HTTP 服务
    srv := &http.Server{Addr: ":8080"}

    // 监听 OS 信号
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        <-quit
        log.Println("Shutting down server...")
        // 给正在处理的请求 30 秒完成
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()
        srv.Shutdown(ctx)
    }()

    srv.ListenAndServe()
}
```

**Dockerfile 对应写法**：

```dockerfile
# ✅ Exec 形式 —— SIGTERM 直接发给 Go 程序，优雅退出
CMD ["/app/server"]

# ❌ Shell 形式 —— SIGTERM 发给 /bin/sh，Go 程序无感知
CMD /app/server
```

**面试题**：

**Q: Shell 形式和 Exec 形式有什么区别？为什么推荐 Exec 形式？**

1. Shell 形式底层通过 `/bin/sh -c` 执行，PID 1 是 Shell 进程而非应用；Exec 形式直接 exec 应用，PID 1 就是应用
2. `docker stop` 发送的 SIGTERM 信号只发给 PID 1。Shell 形式的 PID 1 是 Shell（不会转发信号给子进程），所以应用收不到 SIGTERM，10秒后被 SIGKILL 强杀
3. Exec 形式的 PID 1 就是应用，直接收到 SIGTERM，可以优雅关闭
4. Go 程序应该在 `signal.Notify` 中处理 SIGTERM，配合 Exec 形式实现零停机优雅退出

---

## 7.3 COPY 与 ADD ★★★★

### 区别

| COPY | ADD |
|------|-----|
| 只复制本地文件/目录 | 额外支持远程URL和tar自动解压 |
| 简单、明确、可预测 | 功能多但行为不直观 |
| **推荐使用** | 仅在需要自动解压时使用 |

### 使用示例

```dockerfile
# COPY: 简单复制
COPY app /app
COPY --chown=1000:1000 app /app
COPY --from=builder /app/binary /app/binary

# ADD: 自动解压 tar.gz
ADD app.tar.gz /app/        # 自动解压到 /app/
ADD https://example.com/file /tmp/   # 下载到 /tmp/file（不推荐，用 RUN curl/wget）

# 推荐：用 RUN 替代 ADD 的 URL 功能
RUN curl -o /tmp/file https://example.com/file && \
    echo "hash /tmp/file" | sha256sum -c
```

### 面试题

**Q: COPY 和 ADD 区别？**

- **COPY**：仅从构建上下文复制文件/目录到镜像，语义简单
- **ADD**：除 COPY 功能外，还支持：
  1. 自动解压 tar 文件
  2. 从远程 URL 下载文件

**最佳实践**：优先使用 COPY，仅在需要自动解压 tar 文件时使用 ADD。远程下载使用 RUN curl/wget 更可控（可以清理缓存、校验哈希）。

---

## 7.4 Dockerfile 优化 ★★★★★

### 优化策略

#### 1. 多阶段构建

```dockerfile
# ❌ 单阶段：镜像包含所有编译工具
FROM golang:1.21
COPY . .
RUN go build -o server .
CMD ["./server"]
# 镜像 ~800MB

# ✅ 多阶段：最终镜像只有二进制
FROM golang:1.21 AS builder
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM alpine:3.19
COPY --from=builder /go/server /server
CMD ["/server"]
# 镜像 ~15MB
```

#### 2. 减少镜像层

```dockerfile
# ❌ 多层
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN rm -rf /var/lib/apt/lists/*

# ✅ 合并为一层
RUN apt-get update && \
    apt-get install -y curl vim && \
    rm -rf /var/lib/apt/lists/*
```

#### 3. 利用构建缓存

```dockerfile
# ✅ 先复制依赖文件（变化少，利用缓存）
COPY go.mod go.sum ./
RUN go mod download

# 再复制源码（变化频繁）
COPY . .
RUN go build -o server .
```

**原则**：将变化频率低的指令放在前面，变化频率高的放在后面。

#### 4. 使用 .dockerignore

```
# .dockerignore
.git
node_modules
*.log
.DS_Store
README.md
test/
.build/
```

#### 5. 选择合适的基础镜像

| 镜像 | 大小 | 用途 |
|------|------|------|
| `scratch` | 0B | 纯静态二进制 |
| `alpine:3.19` | ~5MB | 需要基本工具 |
| `distroless/static` | ~2MB | Google 维护，安全精简 |
| `ubuntu:22.04` | ~77MB | 需要完整 glibc 环境 |

#### 6. 使用非 Root 用户

```dockerfile
FROM alpine:3.19
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
CMD ["/app/server"]
```

### 面试题

**Q: Dockerfile 如何优化？**

1. **多阶段构建**：分离构建和运行环境，减小最终镜像
2. **减少层数**：合并 RUN 指令，清理包管理器缓存
3. **利用缓存**：先复制依赖文件，后复制源码
4. **使用 .dockerignore**：排除不需要的文件
5. **选择精简基础镜像**：Alpine / Distroless
6. **非 Root 运行**：安全加固
7. **特定版本标签**：不要用 `latest`，确保可重现构建

---

# 第八章 Docker Compose ★★★★

## 8.1 Compose 基础

Docker Compose 是一个多容器应用编排工具，通过一个 YAML 文件定义和运行多容器应用。

### 核心功能

- 一键启动/停止多个容器
- 容器间网络自动创建
- 依赖管理（depends_on）
- 环境变量管理
- 数据卷管理

### 面试题

**Q: Docker Compose 有什么作用？**

Docker Compose 用于定义和运行多容器 Docker 应用：
1. 通过 `docker-compose.yml` 定义所有服务、网络和数据卷
2. 一条命令启动所有服务：`docker compose up -d`
3. 自动创建网络，容器间可通过服务名互相访问
4. 简化开发环境和 CI/CD 中的多容器管理
5. 支持环境变量、扩展、profiles 等高级特性

---

## 8.2 Compose 文件结构

```yaml
version: "3.8"

services:           # 定义所有服务（容器）
  web:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - mysql
      - redis
    environment:
      - DB_HOST=mysql
    volumes:
      - ./config:/app/config

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - mysql_data:/var/lib/mysql

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes

volumes:            # 定义数据卷
  mysql_data:

networks:           # 定义网络
  default:
    driver: bridge
```

### 核心概念

| 关键字 | 说明 |
|--------|------|
| `services` | 服务定义（每个服务对应一个容器） |
| `volumes` | 命名数据卷 |
| `networks` | 自定义网络 |
| `build` | 从 Dockerfile 构建 |
| `image` | 使用已有镜像 |
| `ports` | 端口映射 |
| `depends_on` | 启动顺序依赖 |
| `environment` | 环境变量 |
| `env_file` | 从文件加载环境变量 |
| `volumes`（在service中） | 挂载卷或目录 |
| `command` | 覆盖默认命令 |
| `restart` | 重启策略 |

---

## 8.3 Compose 实战

### 完整 Go 服务栈

```yaml
version: "3.8"

services:
  # Go 应用
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=mysql
      - DB_PORT=3306
      - DB_USER=root
      - DB_PASSWORD=secret
      - DB_NAME=myapp
      - REDIS_ADDR=redis:6379
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped

  # MySQL
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: myapp
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  # Redis
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    restart: unless-stopped

  # Nginx 反向代理
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    restart: unless-stopped

volumes:
  mysql_data:
  redis_data:
```

---

# 第九章 Docker 资源管理 ★★★★★

## 9.1 CPU 限制

### 限制方式

```bash
# 方式1：限制 CPU 核心数
docker run --cpus="1.5" myapp    # 最多使用 1.5 个核

# 方式2：绑定特定 CPU 核心
docker run --cpuset-cpus="0-2" myapp  # 只能使用 CPU 0,1,2

# 方式3：CPU 权重（相对值，只在CPU竞争时生效）
docker run --cpu-shares=512 myapp  # 默认1024，值越小优先级越低

# 方式4：CFS 周期精确限制
docker run --cpu-period=100000 --cpu-quota=50000 myapp  # 每100ms使用50ms = 0.5核
```

**原理**：
```
--cpus 参数
  → 设置 CFS (Completely Fair Scheduler) 周期
  → cpu.cfs_period_us = 100000 (100ms)
  → cpu.cfs_quota_us = 150000 (--cpus=1.5 → 150ms)
  → 每100ms周期内，进程组最多使用150ms CPU
  → 相当于1.5个CPU核心
```

### 面试题

**Q: Docker 如何限制 CPU？**

4种方式：
1. `--cpus`：限制可用 CPU 核心数（最常用，底层是 CFS quota/period）
2. `--cpuset-cpus`：绑定到特定CPU核心
3. `--cpu-shares`：CPU 权重（相对值，默认1024）
4. `--cpu-period` + `--cpu-quota`：精确的 CFS 周期限制

底层都是修改 `/sys/fs/cgroup/cpu/` 下的 Cgroups 配置。

---

## 9.2 内存限制 ★★★★★

### 限制方式

```bash
# 限制内存上限 512MB
docker run --memory="512m" myapp

# 限制内存 + Swap 总量 1GB
docker run --memory="512m" --memory-swap="1g" myapp
# 表示：512MB 内存 + 512MB swap = 1GB 总量

# 只限制内存，不限制 swap
docker run --memory="512m" --memory-swap="-1" myapp

# 禁用 swap
docker run --memory="512m" --memory-swap="512m" myapp
```

### OOM (Out of Memory)

```
容器进程申请内存
  → 超过 --memory 限制
  → 内核触发 OOM Killer
  → 杀死容器内进程 (得分最高的进程)
  → 容器退出，状态码 137 (128+9, SIGKILL)
```

**OOM 优先级调整**：
```bash
docker run --memory="512m" --oom-score-adj=500 myapp
# 值越大越容易被 OOM Kill（范围 -1000 ~ 1000）
# 默认值为 0

# 禁用 OOM Killer（不推荐）
docker run --memory="512m" --oom-kill-disable myapp
```

### 面试题

**Q: Docker 如何限制内存？**

1. `--memory` / `-m`：限制容器能使用的物理内存上限
2. `--memory-swap`：限制物理内存 + Swap 总量
3. `--memory-reservation`：软限制（低于此值时尽量不回收）
4. 底层通过 Cgroups memory 子系统实现：
   - `memory.limit_in_bytes`：硬限制
   - `memory.soft_limit_in_bytes`：软限制
   - 超过硬限制触发 OOM Killer

容器的 OOM 不等于宿主机 OOM，只有在容器内部的进程被杀。

---

## 9.3 IO 限制

```bash
# 限制块设备写入速度
docker run --device-write-bps /dev/sda:10mb myapp
docker run --device-read-bps /dev/sda:10mb myapp

# 限制 IOPS
docker run --device-write-iops /dev/sda:100 myapp
docker run --device-read-iops /dev/sda:100 myapp
```

---

## 9.4 容器监控

```bash
# docker stats（实时监控）
docker stats
# 输出：CPU%、MEM USAGE/LIMIT、MEM%、NET I/O、BLOCK I/O、PIDS

# 查看特定容器
docker stats myapp

# 格式化输出
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**生产级监控方案**：cAdvisor + Prometheus + Grafana

```yaml
# cAdvisor: Google 开源的容器监控
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
```

---

# 第十章 Docker 安全 ★★★★

## 10.1 容器逃逸

### 原理

容器逃逸指攻击者从容器内部突破了容器的隔离机制，获得了宿主机或其它容器的访问权限。

**常见逃逸方式**：

1. **特权容器** (`--privileged`)
   ```bash
   # 特权容器拥有所有设备访问权限
   docker run --privileged myapp
   # 攻击者可挂载宿主机磁盘 → escape
   ```

2. **挂载 Docker Socket**
   ```bash
   # 危险！容器内可操作宿主机 Docker
   docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
   # 攻击者可在容器内 docker run --privileged → escape
   ```

3. **内核漏洞** (Dirty Cow, RunC CVE-2019-5736)

4. **不当的 Capability 配置**
   ```bash
   # SYS_ADMIN 配合不当挂载可能导致逃逸
   docker run --cap-add=SYS_ADMIN myapp
   ```

### 风险

- 获取宿主机所有容器访问权限
- 读取/修改宿主机文件系统
- 窃取敏感信息（环境变量、密钥等）
- 横向渗透到集群其他节点

### 面试题

**Q: 什么是容器逃逸？**

容器逃逸是指攻击者突破容器隔离（Namespace + Cgroups），获得宿主机权限或访问其他容器。常见方式包括：特权容器、挂载 Docker Socket、内核漏洞利用、配置不当的 Capabilities。防范措施：最小权限运行、不挂载敏感宿主机资源、定期更新内核和 Docker 版本、使用安全扫描工具。

---

## 10.2 最小权限原则

### 非 Root 运行

```dockerfile
# Dockerfile 中创建非 root 用户
FROM alpine:3.19
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN chown -R appuser:appgroup /app
USER appuser
CMD ["/app/server"]
```

```bash
# 运行时指定用户
docker run --user 1000:1000 myapp
```

### 限制 Linux Capabilities

```bash
# 移除所有 Capability，只添加必需的
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

**容器默认保留的 Capability**（共14项，远少于 root 的38项）：
CHOWN, DAC_OVERRIDE, FSETID, FOWNER, MKNOD, NET_RAW, SETGID, SETUID, SETFCAP, SETPCAP, NET_BIND_SERVICE, SYS_CHROOT, KILL, AUDIT_WRITE

### 只读文件系统

```bash
docker run --read-only --tmpfs /tmp myapp
```

### 禁止权限提升 (no-new-privileges) ★★★

```bash
# 禁止容器内进程通过 setuid/setgid 提升权限
docker run --security-opt no-new-privileges myapp
```

**作用**：
- 即使容器内执行了 setuid 程序（如 `sudo`、`su`），也无法获得新权限
- 防止攻击者通过 suid 提权
- 这是一个内核级别的安全屏障

**Kubernetes 中的对应配置**：

```yaml
# Pod SecurityContext
spec:
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false   # 对应 --security-opt no-new-privileges
      readOnlyRootFilesystem: true      # 对应 --read-only
      runAsNonRoot: true                # 对应 --user 非 root
      runAsUser: 1000
      capabilities:
        drop:
        - ALL                           # 对应 --cap-drop=ALL
        add:
        - NET_BIND_SERVICE              # 对应 --cap-add=NET_BIND_SERVICE
```

**面试题**：

**Q: Docker 和 K8s SecurityContext 之间的关系？**

K8s 的 Pod SecurityContext 底层映射到 Docker 的安全选项：
- `allowPrivilegeEscalation: false` → `--security-opt no-new-privileges`
- `readOnlyRootFilesystem: true` → `--read-only`
- `runAsNonRoot: true` → `--user` 非 0
- `capabilities.drop: [ALL]` → `--cap-drop=ALL`
- `privileged: false` → 不加 `--privileged`

两者目标一致：实现容器的最小权限原则。

---

## 10.3 镜像安全

### 镜像扫描

```bash
# Docker Hub 自动扫描（推送到 Docker Hub 后自动触发）
docker scan myapp:v1

# 使用 Trivy 扫描
trivy image myapp:v1

# Harbor 集成 Clair/Trivy 漏洞扫描
```

### 漏洞修复

1. 使用最新的基础镜像
2. 定期重建镜像获取安全补丁
3. 使用 Distroless 最小化攻击面
4. 固定镜像版本（不要用 `latest`）

---

## 10.4 Secret 管理

```bash
# ❌ 不要在镜像中硬编码密钥
ENV DB_PASSWORD=mysecret123

# ❌ 不要在 docker run 中暴露
docker run -e DB_PASSWORD=mysecret123 myapp  # 会出现在 docker inspect 中

# ✅ 使用 Docker Secret (Swarm)
echo "mysecret123" | docker secret create db_password -
docker service create --secret db_password myapp

# ✅ 运行时从文件读取
docker run -v /secrets/db_password:/run/secrets/db_password:ro myapp
```

---

# 第十一章 Docker 与 Kubernetes ★★★★★

## 11.1 Docker 与 K8s 关系

### Docker vs Kubernetes

| Docker | Kubernetes |
|--------|------------|
| 单机容器引擎 | 集群容器编排平台 |
| 解决单机上容器运行 | 解决跨多机容器调度、管理 |
| 概念：Container、Image、Volume | 概念：Pod、Deployment、Service、Ingress |
| 类比：单个进程 | 类比：操作系统（调度、服务发现、负载均衡） |
| `docker run` | `kubectl run` |

### 关系

```
Kubernetes
  └── 节点 (Node)
       └── Kubelet
            └── Container Runtime (containerd / CRI-O)
                 └── runc
                      └── Container (运行你的应用)
```

### 面试题

**Q: Docker 和 Kubernetes 区别？**

Docker 是容器引擎，负责构建镜像和运行容器（单机）。Kubernetes 是容器编排平台，负责多台机器上的容器调度、自动扩缩容、服务发现、负载均衡、滚动更新、健康检查等。关系：Kubernetes 早期使用 Docker 作为容器运行时，后来移除了对 Docker 的直接依赖（Dockershim），现在通过 CRI 接口对接 containerd 或 CRI-O。

---

## 11.2 容器运行时 (Container Runtime)

### 分类

```
容器运行时
├── 高级运行时 (High-Level)
│   ├── containerd (CNCF 毕业项目)
│   ├── CRI-O (RedHat 主导，专为 K8s 设计)
│   └── Docker (含 dockerd + containerd + runc)
│
└── 低级运行时 (Low-Level / OCI Runtime)
    ├── runc (最常用)
    ├── crun (C语言实现，更快)
    ├── kata-containers (虚拟机级别隔离)
    └── gVisor (用户态内核，Google)
```

### CRI (Container Runtime Interface)

Kubernetes 定义的容器运行时接口标准：

```
Kubelet ── CRI gRPC ──► containerd / CRI-O ──► runc ──► Container
```

### 面试题

**Q: 什么是 Container Runtime？**

容器运行时是负责运行容器的软件。分为：
- **低级运行时（OCI Runtime）**：如 runc，负责创建 Namespace/Cgroups，实际运行容器进程
- **高级运行时**：如 containerd、CRI-O，负责镜像管理、镜像拉取、网络配置等

Kubernetes 通过 CRI 接口与高级运行时交互，运行时再调用 OCI 兼容的低级运行时。

---

## 11.3 Dockershim

### 为什么废弃

Kubernetes 最初只支持 Docker 作为容器运行时。为了支持其他运行时，K8s 引入了 CRI 接口，但 Docker 出现在 CRI 之前，不兼容 CRI。

所以 K8s 维护了一个适配层 **Dockershim**，将 CRI 调用转换为 Docker API：

```
Kubelet → CRI → Dockershim → Docker API (dockerd) → containerd → runc
```

**问题**：
1. Docker 不是 CRI 兼容的，需要额外的适配层
2. Docker 功能远超容器运行时（构建、网络等），K8s 只需要运行容器
3. 维护 Dockershim 增加了 K8s 代码库复杂度
4. Docker 调用链更长，性能不如直接使用 containerd

**K8s 1.20 标记 Dockershim 为弃用，1.24 正式移除。**

### 现在的架构

```
Kubelet → CRI → containerd → runc → Container
```

### 面试题

**Q: Kubernetes 为什么移除 Dockershim？**

1. **Docker 不兼容 CRI**：需要额外维护 Dockershim 适配层，增加复杂度
2. **功能冗余**：K8s 只需要容器运行时，不需要 Docker 的镜像构建、docker-compose 等功能
3. **更短的调用链**：直接使用 containerd 减少中间环节，性能更好
4. **统一标准**：社区趋向 OCI 标准，containerd 和 CRI-O 都是 CRI 原生实现
5. **减少维护负担**：K8s 社区不需要维护 Docker 特有的兼容逻辑

**对用户的影响**：镜像仍然是 OCI 标准，Docker 构建的镜像在 containerd 中完全可用。

---

## 11.4 Containerd ★★★★★

### 架构

```
┌──────────────────────────────────────────┐
│              containerd                    │
│                                           │
│  ┌─────────┐  ┌──────────┐  ┌─────────┐  │
│  │ Content │  │ Snapshot │  │  Task   │  │
│  │ (镜像)  │  │ (文件系统)│  │ (容器)  │  │
│  └────┬────┘  └────┬─────┘  └────┬────┘  │
│       │            │              │       │
│  ┌────▼────────────▼──────────────▼────┐  │
│  │            Metadata (BoltDB)        │  │
│  └─────────────────────────────────────┘  │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │         OCI Runtime (runc)          │  │
│  └─────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

### 工作流程

```
1. ctr image pull nginx:latest    ← 拉取镜像
      │
2. Content Store 存储镜像层      ← 内容寻址存储
      │
3. Snapshotter 准备文件系统      ← OverlayFS 快照
      │
4. ctr container create           ← 创建容器（不启动）
      │
5. ctr task start                 ← 启动容器进程 (调用 runc)
      │
6. containerd-shim 接管进程      ← 允许 containerd 重启
```

### 面试题

**Q: Containerd 和 Docker 区别？**

| Containerd | Docker |
|------------|--------|
| 纯容器运行时 | 完整的容器平台 |
| 只有运行容器、管理镜像的功能 | 还包含镜像构建、CLI、Compose 等 |
| CRI 原生支持 | 需要通过 Dockershim 适配 CRI |
| 更轻量、调用链更短 | 功能更多但更重 |
| Kubernetes 1.24+ 首选运行时 | K8s 已移除 Dockershim 支持 |
| 没有 `docker` CLI | 有完整的 `docker` CLI 和 API |

简化理解：
```
Docker = docker CLI + dockerd + containerd + runc
Containerd = containerd + runc
```

---

# 第十二章 Docker 运维与排查 ★★★★★

## 12.1 查看容器状态

```bash
# 查看运行中的容器
docker ps

# 查看所有容器（包括已停止）
docker ps -a

# 查看容器详细信息（最全的排查工具）
docker inspect <container_id>

# 查看特定字段
docker inspect -f '{{.State.Status}}' <container_id>
docker inspect -f '{{.NetworkSettings.IPAddress}}' <container_id>
docker inspect -f '{{.Mounts}}' <container_id>

# 查看容器内进程
docker top <container_id>

# 查看资源使用统计
docker stats <container_id>

# 查看容器文件系统变化
docker diff <container_id>
```

---

## 12.2 查看日志

```bash
# 查看全部日志
docker logs <container_id>

# 实时跟踪日志（类似 tail -f）
docker logs -f <container_id>

# 查看最近 N 行
docker logs --tail 100 <container_id>

# 带时间戳
docker logs -t <container_id>

# 查看特定时间范围的日志
docker logs --since 2024-01-01T00:00:00 <container_id>
docker logs --until 2024-01-02T00:00:00 <container_id>

# 查看最近 30 分钟的日志
docker logs --since 30m <container_id>
```

### 面试题

**Q: 容器启动失败如何排查？**

排查流程：
1. `docker ps -a`：确认容器状态（Exited / Dead）
2. `docker logs <container_id>`：查看容器日志，关注最后几行的错误信息
3. `docker inspect <container_id>`：查看详细配置和退出码
4. 检查退出码：
   - 退出码 0：正常退出
   - 退出码 1：应用错误
   - 退出码 137：被 SIGKILL 杀死（可能 OOM）
   - 退出码 139：SIGSEGV 段错误
5. 检查资源限制：`docker inspect | grep -A10 Memory`
6. 检查挂载：`docker inspect | grep -A10 Mounts`
7. 尝试交互式启动排查：`docker run -it --entrypoint /bin/sh myapp`
8. 检查镜像是否正确：`docker image inspect myapp`

### 日志驱动 (Log Drivers) ★★★

```bash
# 查看当前日志驱动
docker info --format '{{.LoggingDriver}}'
# 默认输出: json-file
```

| 日志驱动 | 说明 | 适用场景 |
|---------|------|---------|
| **json-file** | 默认驱动，JSON 格式写入宿主机文件 | 单机开发、小规模部署 |
| **journald** | 写入 systemd journal | 使用 systemd 的 Linux 发行版 |
| **syslog** | 发送到 syslog 服务器 | 传统运维 |
| **fluentd** | 发送到 Fluentd 采集器 | ELK/EFK 日志平台 |
| **gelf** | Graylog Extended Log Format | Graylog 日志平台 |
| **awslogs** | 发送到 AWS CloudWatch | AWS 云环境 |
| **splunk** | 发送到 Splunk | 企业级 Splunk 日志分析 |
| **none** | 禁用日志 | 不需要日志的场景 |

```bash
# 指定日志驱动
docker run --log-driver=fluentd \
  --log-opt fluentd-address=localhost:24224 \
  --log-opt tag="myapp.{{.Name}}" \
  myapp
```

**日志轮转（json-file 驱动配置）**：

```bash
docker run \
  --log-opt max-size=10m \      # 单个日志文件最大 10MB
  --log-opt max-file=3 \        # 最多保留 3 个日志文件
  myapp

# 或者在 /etc/docker/daemon.json 全局配置
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**生产环境日志最佳实践**：

1. 应用日志输出到 **stdout/stderr**（不要写文件）
2. Docker 采集 stdout/stderr → 日志驱动（json-file/fluentd）
3. 日志采集 Agent（Filebeat/Fluentd DaemonSet）→ 集中日志平台（ELK/Loki）
4. 确保日志轮转，避免磁盘写满

```
应用 → stdout/stderr
        │
Docker Log Driver (json-file/fluentd)
        │
日志采集 Agent (Filebeat / Fluentd / Promtail)
        │
集中日志平台 (Elasticsearch / Loki)
        │
可视化 (Kibana / Grafana)
```

### 面试题

**Q: Docker 日志如何管理？**

1. 默认 json-file 驱动，日志存储在 `/var/lib/docker/containers/<id>/<id>-json.log`
2. 生产环境配置日志轮转（`max-size` + `max-file`）防止磁盘写满
3. 应用应输出到 stdout/stderr（12-Factor App 原则），而非容器内文件
4. 集中式日志：Docker → Fluentd/Syslog → ELK/Loki → Kibana/Grafana
5. K8s 中使用 Fluentd/Filebeat DaemonSet 采集所有节点容器日志

---

## 12.3 进入容器

### docker exec（推荐）

```bash
# 启动新的 Shell 进程（退出不影响容器运行）
docker exec -it <container_id> /bin/bash
docker exec -it <container_id> /bin/sh  # Alpine 用 sh

# 在容器中执行单个命令
docker exec <container_id> ls /app
docker exec <container_id> cat /etc/hosts
docker exec -e MYVAR=value <container_id> env  # 带环境变量
```

### docker attach

```bash
# 连接到容器的主进程（PID 1）的标准输入输出
docker attach <container_id>
# 退出时（Ctrl+C）会向主进程发送信号，可能导致容器停止
# 需要 --sig-proxy=false 避免
```

### 面试题

**Q: exec 和 attach 区别？**

| docker exec | docker attach |
|-------------|---------------|
| 在容器内启动**新进程** | 连接到容器**主进程**（PID 1） |
| 退出不影响容器运行 | 退出可能导致容器停止 |
| 可以执行任意命令 | 只能看主进程的输出 |
| 常用场景：调试、排查 | 交互式查看容器输出 |

---

## 12.4 查看资源

```bash
# 实时资源使用
docker stats

# 磁盘使用情况
docker system df           # 总览
docker system df -v        # 详细

# 清理无用资源
docker system prune        # 清理所有未使用资源
docker image prune         # 清理悬空镜像
docker container prune     # 清理已停止容器
docker volume prune        # 清理未使用 Volume
docker network prune       # 清理未使用网络
```

---

## 12.5 网络排查

```bash
# 列出所有网络
docker network ls

# 查看网络详情（连接的容器、IP、子网等）
docker network inspect bridge

# 查看容器网络信息
docker inspect -f '{{json .NetworkSettings}}' <container_id> | jq

# 测试网络连通性
docker exec <container1> ping <container2>
docker exec <container1> nslookup <container2>

# 端口映射检查
docker port <container_id>
```

---

## 12.6 存储排查

```bash
# 列出所有 Volume
docker volume ls

# 查看 Volume 详情
docker volume inspect <volume_name>
# 可以看到 Mountpoint: /var/lib/docker/volumes/mydata/_data

# 查看容器挂载
docker inspect -f '{{json .Mounts}}' <container_id> | jq

# 磁盘使用
docker system df -v
```

---

# 第十三章 Docker 面试高频场景题 ★★★★★

## 场景1: Docker 和虚拟机区别？

**答**：核心区别是虚拟化层次不同。Docker 是操作系统级虚拟化，共享宿主机内核，只隔离用户空间；VM 是硬件级虚拟化，每个 VM 有完整 Guest OS。因此 Docker 更轻量（MB级 vs GB级）、启动更快（秒级 vs 分钟级），但 VM 隔离更彻底、安全性更高。

---

## 场景2: Docker 为什么轻量？

**答**：
1. 共享宿主机内核，无需 Guest OS
2. 镜像分层复用，公共层只存一份
3. 进程级虚拟化，创建容器 ≈ 创建进程
4. 基础镜像精简（Alpine 5MB）

---

## 场景3: Docker 如何实现隔离？

**答**：通过 Linux 内核的 **Namespace** 机制：PID Namespace 隔离进程、Network Namespace 隔离网络、Mount Namespace 隔离文件系统、IPC Namespace 隔离进程间通信、UTS Namespace 隔离主机名、User Namespace 隔离用户权限。

---

## 场景4: Namespace 和 Cgroups 分别负责什么？

**答**：Namespace 负责**资源隔离**（看得到什么），让容器只能看到自己的进程、网络、文件系统等资源。Cgroups 负责**资源限制**（能用多少），限制容器可使用的 CPU、内存、IO 等资源。两者配合实现"隔离 + 限流"。

---

## 场景5: Docker 镜像为什么分层？

**答**：
1. **复用**：公共基础层被多个镜像共享，节省空间
2. **缓存**：构建时命中缓存加速
3. **分发**：已有层不需要重复传输
4. **版本管理**：每层独立，内容寻址

---

## 场景6: docker run 发生了什么？

**答**：Client 请求 Daemon → 检查本地镜像 / 从 Registry 拉取 → containerd 创建容器 → runc 创建 Namespace + Cgroups → 挂载文件系统（镜像只读层 + 容器可写层）→ 配置网络 → 启动 CMD/ENTRYPOINT 进程 → containerd-shim 接管。

---

## 场景7: Docker 容器如何访问外网？

**答**：容器通过 docker0 网桥发送数据包，宿主机 iptables 的 MASQUERADE 规则将源 IP 从容器 IP（172.17.0.x）SNAT 为宿主机 IP。回包时反向 NAT 转换目的 IP 送回容器。外部访问容器通过端口映射（-p）+ DNAT 实现。

---

## 场景8: Bridge 网络原理是什么？

**答**：Docker Daemon 创建 docker0 虚拟网桥（相当于交换机）。每个容器通过 veth pair（虚拟网线）连接到网桥。同一网桥上的容器可直接通过 IP 通信（二层转发）。容器名 DNS 解析通过 Docker 内置 DNS（127.0.0.11）实现。

---

## 场景9: Volume 和 Bind Mount 区别？

**答**：Volume 由 Docker 管理（`/var/lib/docker/volumes/`），可移植性好，生命周期独立于容器。Bind Mount 依赖宿主机绝对路径，可移植性差，但适合开发和注入配置文件。生产环境推荐 Volume。

---

## 场景10: CMD 和 ENTRYPOINT 区别？

**答**：CMD 定义默认命令和参数，可被 `docker run` 后的命令覆盖。ENTRYPOINT 定义容器固定入口程序，不会被覆盖，`docker run` 后的参数作为追加参数。组合使用：ENTRYPOINT 定义程序，CMD 定义默认参数（可被覆盖）。

---

## 场景11: COPY 和 ADD 区别？

**答**：COPY 只从构建上下文复制文件/目录。ADD 额外支持自动解压 tar 和远程 URL 下载。最佳实践：优先用 COPY，只在需要自动解压时用 ADD。远程下载用 RUN curl/wget 更可控。

---

## 场景12: 如何优化 Docker 镜像体积？

**答**：
1. 多阶段构建（构建环境和运行环境分离）
2. 使用 Alpine/Distroless 基础镜像
3. 减少层数（合并 RUN 指令）
4. .dockerignore 排除不必要文件
5. 清理包管理器缓存
6. Go 程序 `CGO_ENABLED=0` 静态编译，用 `FROM scratch`

---

## 场景13: Docker 如何限制 CPU 和内存？

**答**：
- CPU：`--cpus` 限制核心数，`--cpuset-cpus` 绑定核心，`--cpu-shares` 权重
- 内存：`--memory` 硬限制（超限 OOM），`--memory-swap` 总内存+swap限制
- 底层都是 Cgroups 机制

---

## 场景14: 容器启动失败如何排查？

**答**：
1. `docker ps -a` 查看状态
2. `docker logs` 查看日志
3. `docker inspect` 查看退出码
4. 交互式启动 `docker run -it --entrypoint sh`
5. 检查资源限制和挂载配置

---

## 场景15: 容器日志如何查看？

**答**：
- `docker logs <id>` 查看全部
- `docker logs -f <id>` 实时跟踪
- `docker logs --tail 100 <id>` 最近100行
- `docker logs --since 30m <id>` 最近30分钟
- `docker logs -t <id>` 带时间戳
- 生产环境部署日志采集 DaemonSet（如 Filebeat/Fluentd → ELK）

---

## 场景16: Docker Compose 有什么作用？

**答**：通过 YAML 文件定义多容器应用，一条命令启动/停止所有服务。自动创建网络，容器间通过服务名通信。支持环境变量、数据卷、依赖管理。适合本地开发和多服务部署。

---

## 场景17: Docker 与 Kubernetes 区别？

**答**：Docker 负责单机上构建和运行容器，Kubernetes 负责跨多台机器的容器编排（调度、扩缩容、服务发现、负载均衡、滚动更新）。Docker 是容器引擎，K8s 是容器编排平台。K8s 已移除 Docker 作为运行时，改用 containerd/CRI-O。

---

## 场景18: Containerd 和 Docker 区别？

**答**：Docker 是完整容器平台（CLI + dockerd + containerd + runc），Containerd 是纯粹的容器运行时。Kubernetes 只需要运行时能力，不需要 Docker 的构建、Compose 等功能，所以直接使用 containerd 更高效。Docker 构建的镜像完全兼容 containerd。

---

## 场景19: 为什么 Kubernetes 弃用 Dockershim？

**答**：Docker 不兼容 CRI 接口，需要额外维护 Dockershim 适配层。这会增加复杂度、性能开销和维护负担。直接使用 CRI 原生实现（containerd/CRI-O）更简洁高效。Docker 构建的镜像仍然是 OCI 标准，完全兼容。

---

## 场景20: 生产环境如何保证 Docker 安全？

**答**：
1. 非 Root 用户运行容器
2. 最小 Capability 原则（`--cap-drop=ALL`）
3. 文件系统只读（`--read-only`）
4. 镜像漏洞扫描（Trivy/Clair）
5. 不使用 `latest` 标签，固定版本
6. 不挂载 Docker Socket 进入容器
7. 使用 Secret 管理敏感信息
8. 限制资源（防 DDoS/OOM）
9. 定期更新基础镜像和 Docker 版本
10. 使用非特权模式（不加 `--privileged`）
11. 网络隔离和防火墙策略
12. 启用 Docker Content Trust（`DOCKER_CONTENT_TRUST=1`）

---

## 场景21: ARG 和 ENV 区别？

**答**：ARG 仅在 `docker build` 构建时可用（通过 `--build-arg` 传入），ENV 在构建时和运行时都可用（通过 `-e` 覆盖）。ARG 用于向 Dockerfile 传入构建参数（如 GO 版本号），ENV 用于配置运行环境变量。ARG 不应传敏感信息（会留在 `docker history`），敏感信息用 BuildKit secrets。

---

## 场景22: Shell 形式和 Exec 形式有什么区别？

**答**：Shell 形式通过 `/bin/sh -c` 执行，PID 1 是 Shell 进程；Exec 形式直接 exec 应用，PID 1 是应用本身。关键区别：`docker stop` 发送 SIGTERM 只到 PID 1 → Shell 形式下应用收不到信号，无法优雅退出（10秒后被强杀）；Exec 形式下应用直接收到信号，可优雅关闭。**推荐 Exec 形式**。

---

## 场景23: docker stop 和 docker kill 区别？

**答**：`docker stop` 发送 SIGTERM（优雅关闭信号），等待 10 秒后如果容器未退出则发送 SIGKILL 强杀。`docker kill` 直接发送 SIGKILL，立即杀死进程。Go 程序应在 `signal.Notify` 中处理 SIGTERM，配合 Exec 形式实现优雅退出。

---

## 场景24: 容器为什么需要 tini/dumb-init？PID 1 问题是什么？

**答**：Linux PID 1（init 进程）需要处理信号转发和子进程回收。直接运行为 PID 1 的应用通常不处理这两件事 → SIGTERM 信号丢失、僵尸进程堆积。tini 是轻量级 init 进程，正确转发信号给应用 + 回收僵尸进程。Docker 1.13+ 提供 `docker run --init` 自动注入 tini。

---

## 场景25: Docker 日志如何管理？

**答**：默认 json-file 驱动，存储在 `/var/lib/docker/containers/<id>/<id>-json.log`。生产环境需配置日志轮转（`max-size=10m --log-opt max-file=3`）防止磁盘写满。应用应遵循 12-Factor App 原则，日志输出 stdout/stderr。集中式采集：Docker → Fluentd/Syslog → ELK/Loki → Kibana/Grafana。

---

## 场景26: Docker HEALTHCHECK 是什么？

**答**：HEALTHCHECK 是 Dockerfile 指令，定义容器健康检查命令。Docker Daemon 定期执行检查命令，根据结果标记容器为 healthy/unhealthy。参数：`--interval`（检查间隔）、`--timeout`（单次超时）、`--retries`（连续失败阈值）、`--start-period`（启动缓冲期）。与 K8s liveness/readiness probe 功能类似，但在 Docker Swarm 或纯 Docker 环境中使用。

---

## 场景27: Build Context 是什么？.dockerignore 有什么作用？

**答**：Build Context 是 `docker build` 时 Client 打包发送给 Daemon 的文件集合。如果有大量不必要文件（node_modules、.git），发送过程会很慢。`.dockerignore` 文件排除不需要的文件，减小 Build Context 大小，加速构建 + 保护敏感文件不被意外打包。

---

## 场景28: Docker 重启策略有哪些？

**答**：4 种：`no`（不重启，默认）、`always`（总是重启，Daemon 重启也会拉起）、`on-failure[:N]`（仅非零退出码时重启，可限制最多 N 次）、`unless-stopped`（除非手动 docker stop，否则重启）。生产常驻服务推荐 `unless-stopped`，一次性任务用默认 `no`。

---

# 校招必背 Top 30（快速背诵版）

| # | 知识点 | 一句话回答 |
|---|--------|-----------|
| 1 | 什么是 Docker | 开源容器平台，将应用及依赖打包成可移植容器，实现"一次构建，到处运行" |
| 2 | Docker 与 VM 区别 | Docker 共享宿主机内核，VM 有独立 Guest OS；Docker 更轻量、启动更快 |
| 3 | Docker 架构 | C/S 架构：Client → REST API → Daemon → containerd → runc → Container |
| 4 | Image 和 Container 区别 | Image 是只读模板（类比类），Container 是运行实例（类比对象） |
| 5 | Docker 为什么轻量 | 共享内核、分层复用、进程级虚拟化、精简基础镜像 |
| 6 | Namespace 原理 | Linux 内核隔离机制，6种：PID、Network、Mount、IPC、UTS、User |
| 7 | Cgroups 原理 | Linux 内核资源限制机制，限制 CPU、内存、IO 等 |
| 8 | OverlayFS 原理 | 联合文件系统，LowerDir(只读) + UpperDir(可写) + MergedDir(统一视图) |
| 9 | Docker 镜像分层 | 每条指令创建一层，层之间共享复用，构建缓存加速 |
| 10 | docker run 流程 | Client→Daemon→拉镜像→创建Namespace/Cgroups→挂载FS→配置网络→启动进程 |
| 11 | 容器生命周期 | Created → Running → Paused/Stopped → Exited → Dead/Removed |
| 12 | Docker 网络模式 | Bridge(默认)、Host、None、Overlay(跨主机VXLAN)、Macvlan |
| 13 | Bridge 网络原理 | docker0 网桥 + veth pair 连接容器，iptables NAT 实现外网访问 |
| 14 | docker0 是什么 | Docker 默认创建的 Linux 虚拟网桥，所有 Bridge 模式容器连接到此 |
| 15 | veth pair 原理 | 虚拟以太网设备对，一端插入容器NS(eth0)，一端插docker0网桥 |
| 16 | 容器访问外网 | SNAT/MASQUERADE：容器IP → iptables NAT → 宿主机IP → 外网 |
| 17 | Volume vs Bind Mount | Volume 由 Docker 管理，可移植；Bind Mount 直接挂载宿主机路径 |
| 18 | 数据持久化 | Volume：独立于容器生命周期；Bind Mount：依赖宿主机路径；Tmpfs：内存 |
| 19 | Dockerfile 常用指令 | FROM、RUN、COPY、ADD、CMD、ENTRYPOINT、ENV、EXPOSE、WORKDIR |
| 20 | CMD vs ENTRYPOINT | CMD 提供默认命令可覆盖；ENTRYPOINT 固定入口，参数追加 |
| 21 | COPY vs ADD | COPY 只复制文件；ADD 额外支持tar解压和URL下载 |
| 22 | Dockerfile 优化 | 多阶段构建、Alpine基础镜像、减少层数、利用缓存、.dockerignore |
| 23 | Docker Compose 作用 | 多容器编排，YAML定义服务，一键启动/停止，服务间DNS通信 |
| 24 | 限制 CPU | `--cpus`(核心数)、`--cpuset-cpus`(绑定核)、`--cpu-shares`(权重) |
| 25 | 限制内存 | `--memory`(硬限制，超限OOM)、`--memory-swap`(总量限制) |
| 26 | 容器逃逸 | 攻击者突破容器隔离获取宿主机权限，防范：非Root、最小Capability、定期更新 |
| 27 | Docker vs K8s | Docker是单机容器引擎；K8s是集群容器编排平台（调度、扩缩容、服务发现） |
| 28 | Containerd vs Docker | Docker = CLI + dockerd + containerd；Containerd = 纯运行时，更轻量 |
| 29 | 为什么移除 Dockershim | Docker不兼容CRI，Dockershim是适配层，增加复杂度和维护成本 |
| 30 | 容器启动失败排查 | `docker logs` 看日志 / `docker inspect` 看状态和退出码 / `docker run -it --entrypoint sh` 调试 |

---

## 校招必背补充 Top 8（新增高频考点）

| # | 知识点 | 一句话回答 |
|---|--------|-----------|
| 31 | Shell vs Exec 形式 | Shell 形式 PID 1 是 /bin/sh（信号丢失）；Exec 形式 PID 1 是应用（信号直达），推荐 Exec |
| 32 | ARG vs ENV | ARG 仅构建时可用（--build-arg）；ENV 构建+运行都可（-e 覆盖）；ARG 不用来传敏感信息 |
| 33 | docker stop vs kill | stop = SIGTERM + 10s超时 + SIGKILL（优雅退出）；kill = 直接 SIGKILL（无法优雅退出） |
| 34 | PID 1 问题 & tini | PID 1 需处理信号转发+僵尸进程回收；tini/dumb-init 做轻量 init，docker run --init 自动注入 |
| 35 | 重启策略 | no/always/on-failure[:N]/unless-stopped；生产常驻服务推荐 unless-stopped |
| 36 | HEALTHCHECK | 容器健康检查指令，Daemon 定期执行标记 healthy/unhealthy；类似 K8s probe |
| 37 | 日志驱动 & 轮转 | 默认 json-file；生产配置 max-size+max-file 防写满磁盘；stdout/stderr → 集中日志平台 |
| 38 | Build Context | docker build 发送给 Daemon 的文件集合；.dockerignore 排除不必要文件加速构建 |

---

# 附录：常用命令速查表

## 容器操作

```bash
docker run -d -p 80:80 --name web nginx      # 后台运行，端口映射，命名
docker ps                                      # 列出运行中的容器
docker ps -a                                   # 列出所有容器
docker stop web                                # 停止容器
docker start web                               # 启动已存在的容器
docker restart web                             # 重启容器
docker rm web                                  # 删除容器
docker rm -f web                               # 强制删除
docker exec -it web /bin/bash                  # 进入容器
docker logs -f web                             # 查看日志
docker inspect web                             # 查看详细信息
docker stats                                   # 资源使用统计
```

## 镜像操作

```bash
docker pull nginx:1.25                         # 拉取镜像
docker images                                  # 列出本地镜像
docker build -t myapp:v1 .                     # 构建镜像
docker tag myapp:v1 myapp:latest                # 打标签
docker push myapp:v1                           # 推送镜像
docker rmi myapp:v1                            # 删除镜像
docker image prune                             # 清理悬空镜像
docker history nginx                           # 查看镜像构建历史
```

## 网络操作

```bash
docker network ls                              # 列出网络
docker network create mynet                    # 创建网络
docker network inspect mynet                   # 查看网络详情
docker network connect mynet web               # 将容器连接到网络
docker network disconnect mynet web            # 断开连接
docker network rm mynet                        # 删除网络
```

## 数据卷操作

```bash
docker volume ls                               # 列出数据卷
docker volume create mydata                    # 创建数据卷
docker volume inspect mydata                   # 查看数据卷详情
docker volume rm mydata                        # 删除数据卷
docker volume prune                            # 清理未使用数据卷
```

## 系统管理

```bash
docker system df                               # 磁盘使用情况
docker system prune -a                         # 清理所有未使用资源
docker system events                           # 实时事件流
docker system info                             # Docker 系统信息
docker version                                 # Docker 版本
```

---

> **学习建议**：
>
> 1. 先理解 Linux Namespace + Cgroups + UnionFS 三大底层技术
> 2. 亲手写 Dockerfile，用多阶段构建优化镜像
> 3. 手动搭建 docker-compose 多服务环境
> 4. 理解 docker run 的完整流程
> 5. 结合 Kubernetes 理解容器编排的必要性
> 6. 背诵 Top 30 的一问一答作为面试速记
>
> 校招面试中，Docker 是后端开发的基础技能。面试官通常从"用过 Docker 吗？"开始，逐步深入底层原理。掌握以上内容即可应对 95% 的 Docker 面试题。
