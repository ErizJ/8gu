# MCP（Model Context Protocol）面试知识体系

> 适用于：
>
> - AI Agent 开发工程师
> - Golang 后端开发
> - Python 后端开发
> - 大模型应用开发
> - 智能体（Agent）开发
> - RAG 开发工程师
>
> MCP 是 Anthropic 于 2024 年提出并开源的协议，目前已经成为 AI Agent 生态的重要标准协议，类似于 AI 世界的「USB 接口标准」。

---

## 目录

- [第一章 MCP 基础](#第一章-mcp-基础)
  - [1.1 什么是 MCP](#11-什么是-mcp)
  - [面试题：什么是 MCP？](#面试题什么是-mcp)
  - [面试题：MCP 为什么被称为 AI 世界的 USB？](#面试题mcp-为什么被称为-ai-世界的-usb)
- [第二章 MCP 整体架构](#第二章-mcp-整体架构)
  - [2.1 MCP 架构图](#21-mcp-架构图)
  - [2.2 MCP 核心角色](#22-mcp-核心角色)
  - [面试题：Host、Client、Server 的区别？](#面试题hostclientserver-的区别)
- [第三章 MCP 通信机制](#第三章-mcp-通信机制)
  - [3.1 JSON-RPC 2.0](#31-json-rpc-20)
  - [面试题：MCP 为什么使用 JSON-RPC？](#面试题mcp-为什么使用-json-rpc)
  - [面试题：Notification 和 Request 的区别？](#面试题notification-和-request-的区别)
- [第四章 MCP Transport（传输层）](#第四章-mcp-transport传输层)
  - [4.1 STDIO Transport](#41-stdio-transport)
  - [4.2 HTTP Transport（请求-响应模式）](#42-http-transport请求-响应模式)
  - [4.3 SSE Transport（Server-Sent Events）](#43-sse-transportserver-sent-events)
  - [4.4 Streamable HTTP Transport（2025 推荐方案）](#44-streamable-http-transport2025-推荐方案)
  - [4.5 四种 Transport 全景对比](#45-四种-transport-全景对比)
- [第五章 MCP Tool](#第五章-mcp-tool)
  - [5.1 什么是 Tool](#51-什么是-tool)
  - [5.2 Tool 定义（Tool Schema）](#52-tool-定义tool-schema)
  - [5.3 Tool 调用流程](#53-tool-调用流程)
  - [面试题：MCP Tool 如何定义？](#面试题mcp-tool-如何定义)
  - [面试题：MCP Tool 调用流程是什么？](#面试题mcp-tool-调用流程是什么)
  - [面试回答模板（Tool 章节综合）](#面试回答模板tool-章节综合)
- [第六章 MCP Resource](#第六章-mcp-resource)
  - [6.1 什么是 Resource](#61-什么是-resource)
  - [6.2 Resource URI](#62-resource-uri)
  - [6.3 Resource 订阅机制](#63-resource-订阅机制resourcessubscribe)
  - [面试题：Resource 和 Tool 的区别？](#面试题resource-和-tool-的区别)
  - [面试题：MCP 如何实现资源变更通知？](#面试题mcp-如何实现资源变更通知)
- [第七章 MCP Prompt](#第七章-mcp-prompt)
  - [7.1 什么是 Prompt 资源](#71-什么是-prompt-资源)
  - [面试题：MCP Prompt 有什么作用？](#面试题mcp-prompt-有什么作用)
- [第八章 MCP 生命周期](#第八章-mcp-生命周期)
  - [8.1 完整生命周期](#81-完整生命周期)
  - [8.2 初始化阶段（Initialize）](#82-初始化阶段initialize)
  - [8.3 正常操作阶段的方法](#83-正常操作阶段的方法)
  - [8.4 Content 类型详解](#84-content-类型详解)
  - [8.5 Tool Call 取消机制](#85-tool-call-取消机制)
  - [8.6 MCP Sampling（Server → LLM 反向调用）](#86-mcp-samplingserver-llm-反向调用)
  - [8.7 MCP Roots（根目录声明）](#87-mcp-roots根目录声明)
  - [8.8 MCP Logging（日志机制）](#88-mcp-logging日志机制)
  - [8.9 MCP Elicitation（用户交互）](#89-mcp-elicitation用户交互)
- [第九章 MCP Server 开发](#第九章-mcp-server-开发)
  - [9.1 MCP Server 核心结构](#91-mcp-server-核心结构)
  - [9.2 Python MCP Server 开发（FastMCP）](#92-python-mcp-server-开发)
  - [9.3 Golang MCP Server 开发](#93-golang-mcp-server-开发)
  - [9.4 TypeScript MCP Server 开发](#94-typescript-mcp-server-开发)
  - [面试题：如何开发一个 MCP Server？](#面试题如何开发一个-mcp-server)
- [第十章 MCP 与 Function Calling](#第十章-mcp-与-function-calling)
  - [面试题：MCP 和 Function Calling 的区别？](#面试题mcp-和-function-calling-的区别)
  - [面试回答模板（MCP vs Function Calling）](#面试回答模板mcp-vs-function-calling)
- [第十一章 MCP 与 Agent](#第十一章-mcp-与-agent)
  - [面试题：MCP 与 Agent 是什么关系？](#面试题mcp-与-agent-是什么关系)
- [第十二章 MCP 与 RAG](#第十二章-mcp-与-rag)
  - [面试题：MCP 能否替代 RAG？](#面试题mcp-能否替代-rag)
  - [面试回答模板（MCP vs RAG）](#面试回答模板mcp-vs-rag)
- [第十三章 MCP 安全机制](#第十三章-mcp-安全机制)
  - [13.1 权限控制](#131-权限控制)
  - [13.2 沙箱机制](#132-沙箱机制)
  - [13.3 数据隔离](#133-数据隔离)
  - [13.4 OAuth 2.0 授权](#134-oauth-20-授权mcp-authorization)
  - [面试题：MCP 如何保证安全？](#面试题mcp-如何保证安全)
- [第十四章 MCP 生态](#第十四章-mcp-生态)
  - [面试题：常见 MCP Server 有哪些？](#面试题常见-mcp-server-有哪些)
- [第十五章 MCP 高级设计](#第十五章-mcp-高级设计)
  - [15.1 Tool Router（工具路由）](#151-tool-router工具路由)
  - [15.2 多 MCP Server 架构](#152-多-mcp-server-架构)
  - [15.3 MCP Gateway（统一入口）](#153-mcp-gateway统一入口)
  - [15.4 MCP Proxy（代理层）](#154-mcp-proxy代理层)
  - [面试题：Tool Router 的设计要点？](#面试题tool-router-的设计要点)
  - [面试题：MCP Proxy 有哪些应用场景？](#面试题mcp-proxy-有哪些应用场景)
  - [面试题：如何管理多个 MCP Server？](#面试题如何管理多个-mcp-server)
- [第十六章 MCP 面试场景题](#第十六章-mcp-面试场景题)
  - [场景 1：请介绍一下什么是 MCP？](#场景-1请介绍一下什么是-mcp)
  - [场景 2：MCP 为什么重要？](#场景-2mcp-为什么重要为什么最近这么火)
  - [场景 3：MCP 和 Function Calling 有什么区别？](#场景-3mcp-和-function-calling-有什么区别能互相替代吗)
  - [场景 4：Tool 和 Resource 有什么区别？](#场景-4tool-和-resource-有什么区别什么时候用哪个)
  - [场景 5：MCP 的连接初始化流程是怎样的？](#场景-5mcp-的连接初始化流程是怎样的)
  - [场景 6：STDIO 和 Streamable HTTP 有什么区别？](#场景-6stdio-和-streamable-http-传输有什么区别怎么选)
  - [场景 7：如果要你开发一个 MCP Server，你会怎么做？](#场景-7如果要你开发一个-mcp-server你会怎么做)
  - [场景 8：如何实现一个 MySQL MCP Server？](#场景-8如何实现一个-mysql-mcp-server需要注意什么)
  - [场景 9：MCP 在 Agent 架构中处于什么位置？](#场景-9mcp-在-agent-架构中处于什么位置如何支持-agent-运行)
  - [场景 10：企业中有几十个 MCP Server，如何统一管理？](#场景-10企业中有几十个-mcp-server如何统一管理)
  - [场景 11：MCP 系统如何保证安全性？](#场景-11mcp-系统如何保证安全性)
  - [场景 12：MCP 能不能替代 API Gateway？](#场景-12mcp-能不能替代-api-gateway)
  - [场景 13：MCP 和 LangChain 是什么关系？](#场景-13mcp-和-langchain-是什么关系)
  - [场景 14：MCP 和 RAG 有什么区别？](#场景-14mcp-和-rag-有什么区别能替代-rag-吗)
  - [场景 15：请设计一个企业级 MCP 平台架构](#场景-15请设计一个企业级-mcp-平台架构)
- [第十七章 MCP 日常开发使用指南](#第十七章-mcp-日常开发使用指南)
  - [17.1 Claude Desktop 中使用 MCP](#171-claude-desktop-中使用-mcp)
  - [17.2 VS Code / Cursor 中使用 MCP](#172-vs-code--cursor-中使用-mcp)
  - [17.3 MCP Inspector（调试工具）](#173-mcp-inspector调试工具)
  - [17.4 命令行中使用 MCP](#174-命令行中使用-mcp)
  - [17.5 日常开发工作流](#175-日常开发工作流)
  - [17.6 常见问题排查](#176-常见问题排查)
  - [17.7 MCP Server 性能优化](#177-mcp-server-性能优化)
  - [17.8 MCP 在 CI/CD 中的应用](#178-mcp-在-cicd-中的应用)
  - [17.9 日常使用速查](#179-日常使用速查)
  - [面试题：MCP 日常开发中的常见问题？](#面试题mcp-日常开发中的常见问题)
- [MCP 面试必背 Top 20](#mcp-面试必背-top-20)
  - [Q1 什么是 MCP ~ Q20 企业级架构](#q1-什么是-mcp)
- [附录：MCP 协议参考](#附录mcp-协议参考)
  - [A.1 核心 JSON-RPC 方法速查表](#a1-核心-json-rpc-方法速查表)
  - [A.2 MCP 协议版本](#a2-mcp-协议版本)
  - [A.3 推荐学习资源](#a3-推荐学习资源)

---

# 第一章 MCP 基础

## 1.1 什么是 MCP ★★★★★

### MCP 全称

**Model Context Protocol**（模型上下文协议）

### MCP 定义

MCP 是一套**开放的、标准化的协议**，用于连接：

- **大模型（LLM）**：Claude、GPT、Gemini 等
- **工具（Tools）**：可被模型调用的执行能力，如查询数据库、调用 API
- **数据源（Data Sources）**：模型可以读取的上下文资源，如文件、数据库记录
- **外部服务（Services）**：GitHub、Slack、Jira、Notion 等 SaaS 服务

### 核心思想

MCP 的核心思想是：**让 AI 模型通过一个统一的标准协议，安全、可控地访问外部世界的数据和工具。**

类比理解：

- **USB 协议**：任何 USB 设备插到电脑上都能被识别 → MCP 让任何工具/数据源都能被 AI 模型识别和调用
- **HTTP 协议**：任何浏览器都能访问任何网站 → MCP 让任何 AI 应用都能使用任何 MCP Server

---

### MCP 解决的问题

#### 在 MCP 出现之前（N × M 问题）

```text
ChatGPT ──── 需要开发 ──── MySQL 插件
    │                        │
    ├── 需要开发 ──── Redis 插件
    │                        │
    ├── 需要开发 ──── GitHub 插件
    │                        │
    ├── 需要开发 ──── Jira 插件
    │                        │
    └── 需要开发 ──── Notion 插件

Claude ──── 又要重新开发 ──── MySQL 插件
    │                            │
    ├── 又要重新开发 ──── Redis 插件
    ...（每个 LLM × 每个工具 = N × M 组合爆炸）
```

**痛点：**
- 每个 LLM 平台都要为每个外部系统单独开发适配器
- 开发者锁定在特定 LLM 生态中
- 工具无法跨平台复用
- 维护成本极高（N 个模型 × M 个工具 = N×M 个集成）

#### 有了 MCP 之后（N + M 问题）

```text
任何 LLM（Claude / GPT / Gemini / ...）
         │
         ▼
    ┌─────────┐
    │   MCP   │  ← 统一协议层（标准接口）
    └─────────┘
         │
    ┌────┼────┬────┬────┬────┬────┐
    ▼    ▼    ▼    ▼    ▼    ▼    ▼
  MySQL Redis GitHub Jira Notion Slack 本地文件
```

**收益：**
- **一次开发，到处使用**：开发一个 MCP Server，所有支持 MCP 的 LLM 都能使用
- **厂商解耦**：工具开发者不需要关心底层是哪个 LLM
- **生态繁荣**：社区可以贡献各种 MCP Server，形成工具市场
- **维护成本降为 N+M**：每个工具只需维护一个 MCP Server，每个 LLM 只需实现一次 MCP Client

---

### 面试题：什么是 MCP？

**标准回答：**

> MCP（Model Context Protocol）是 Anthropic 于 2024 年开源的一套标准协议，用于连接大模型与外部工具、数据源和服务。它解决了 AI 应用与外部系统集成的 N×M 组合爆炸问题，通过统一的协议层实现了「一次开发，到处使用」。MCP 基于 JSON-RPC 2.0 协议，定义了 Host、Client、Server 三层架构，支持 STDIO 和 HTTP 等多种传输方式。它被称为 AI 世界的「USB 接口标准」，因为任何支持 MCP 的工具都可以被任何支持 MCP 的 LLM 应用即插即用。

---

### 面试题：MCP 为什么被称为 AI 世界的 USB？

**标准回答：**

> 1. **标准化接口**：就像 USB 定义了统一的物理和电气接口标准，MCP 定义了统一的通信协议和数据格式
> 2. **即插即用**：USB 设备插上就能用，MCP Server 启动后 LLM 就能发现并调用其工具
> 3. **跨平台兼容**：USB 设备可以在 Windows/Mac/Linux 上使用，MCP Server 可以在 Claude/GPT/Gemini 等任何支持 MCP 的应用中使用
> 4. **生态效应**：USB 催生了庞大的外设生态，MCP 正在催生庞大的 AI 工具生态
> 5. **屏蔽底层差异**：USB 屏蔽了硬件协议差异，MCP 屏蔽了不同 LLM 的 API 差异

---

# 第二章 MCP 整体架构 ★★★★★

## 2.1 MCP 架构图

```text
┌──────────────────────────────────────────────┐
│                   User                        │
│              ("查一下天气")                     │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│                 LLM / AI Model                │
│          (Claude / GPT / Gemini)              │
│                                                │
│   理解用户意图 → 决定调用哪个工具 → 生成结果     │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│               MCP Host                        │
│   (Claude Desktop / Cursor / 自定义Agent)      │
│                                                │
│   管理多个 MCP Client 的生命周期                │
│   路由 LLM 的工具调用请求到正确的 Server         │
└──────┬───────────────────┬───────────────────┘
       │                   │
       ▼                   ▼
┌──────────────┐   ┌──────────────┐
│  MCP Client  │   │  MCP Client  │
│  (进程内)     │   │  (进程内)     │
│              │   │              │
│  负责与一个   │   │  负责与一个   │
│  MCP Server  │   │  MCP Server  │
│  通信        │   │  通信        │
└──────┬───────┘   └──────┬───────┘
       │                   │
       │  JSON-RPC         │  JSON-RPC
       │  (STDIO/HTTP)     │  (STDIO/HTTP)
       │                   │
       ▼                   ▼
┌──────────────┐   ┌──────────────┐
│  MCP Server  │   │  MCP Server  │
│  (GitHub)    │   │  (MySQL)     │
│              │   │              │
│  Tools:      │   │  Tools:      │
│  - search_   │   │  - query_    │
│    repos     │   │    mysql     │
│  - create_   │   │  - list_     │
│    issue     │   │    tables    │
│              │   │              │
│  Resources:  │   │  Resources:  │
│  - repo://   │   │  - mysql://  │
│  - issue://  │   │  - table://  │
└──────────────┘   └──────────────┘
```

---

## 2.2 MCP 核心角色 ★★★★★

### 1. MCP Host（宿主应用）

**定义**：运行 AI Agent 的应用程序。Host 是用户直接交互的应用层。

**职责**：
- 管理多个 MCP Client 的生命周期（启动、停止、重启）
- 接收用户的输入，发送给 LLM
- 接收 LLM 的工具调用请求，路由到正确的 MCP Client
- 将工具执行结果返回给 LLM 进行下一步推理

**典型 Host 示例**：
- **Claude Desktop**：Anthropic 官方桌面应用
- **Cursor / Windsurf**：AI 编程 IDE
- **VS Code + Copilot**：微软生态
- **自定义 Agent 应用**：企业内部 Agent 平台
- **LangChain / CrewAI**：Agent 开发框架

```text
# Claude Desktop 的 mcp.json 配置示例
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/workspace"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

---

### 2. MCP Client（客户端）

**定义**：Host 内部的一个组件，负责与**一个** MCP Server 建立一对一连接并通信。

**职责**：
- 与 MCP Server 建立连接（STDIO / HTTP / SSE）
- 发送 JSON-RPC 请求
- 接收 JSON-RPC 响应
- 处理连接生命周期（initialize → ready → shutdown）
- 维护与一个 Server 的会话状态

**关键特征**：
- **一对一关系**：一个 Client 对应一个 Server
- **Host 内含多个 Client**：一个 Host 可以管理多个 Client，每个连接不同的 Server
- **协议翻译层**：Client 将 Host 内部的工具调用请求翻译为标准 MCP JSON-RPC 消息

```text
MCP Host
├── MCP Client A ──── STDIO ──── MCP Server (GitHub)
├── MCP Client B ──── STDIO ──── MCP Server (MySQL)
└── MCP Client C ──── HTTP ───── MCP Server (Remote Redis)
```

---

### 3. MCP Server（服务端）

**定义**：提供具体工具能力的服务进程。每个 Server 封装了对某个外部系统或数据源的操作。

**职责**：
- 暴露 Tools（可被 LLM 调用的函数）
- 暴露 Resources（可被 LLM 读取的数据）
- 暴露 Prompts（预定义的提示词模板）
- 处理来自 Client 的 JSON-RPC 请求
- 执行具体的业务逻辑（查询数据库、操作文件、调用 API）

**典型 MCP Server 示例**：

| Server | 功能 | Tools 示例 |
|--------|------|-----------|
| Filesystem | 文件系统操作 | `read_file`, `write_file`, `list_directory` |
| GitHub | GitHub 仓库操作 | `search_repositories`, `create_issue`, `create_pull_request` |
| PostgreSQL | 数据库操作 | `query`, `list_tables`, `describe_table` |
| Redis | 缓存操作 | `get`, `set`, `del`, `keys` |
| Brave Search | 网络搜索 | `brave_search`, `brave_local_search` |
| Puppeteer | 浏览器控制 | `navigate`, `click`, `screenshot`, `evaluate` |
| Slack | 团队协作 | `send_message`, `list_channels`, `search_messages` |
| Notion | 知识库 | `search_pages`, `create_page`, `update_block` |

---

### 面试题：Host、Client、Server 的区别？

**标准回答：**

> - **Host**：运行 AI Agent 的应用（如 Claude Desktop、Cursor），负责整体 Agent 逻辑和用户交互，管理多个 Client 的生命周期
> - **Client**：Host 内部的通信组件，与一个 MCP Server 建立一对一连接，负责 JSON-RPC 消息的收发。Host 通过多个 Client 连接多个 Server
> - **Server**：提供具体工具能力的独立进程，封装对外部系统（数据库、API、文件系统）的操作，暴露 Tools、Resources、Prompts
>
> **记忆口诀**：Host 是房子，Client 是插座，Server 是电器。一个房子可以有多个插座，每个插座插一个电器。

---

# 第三章 MCP 通信机制 ★★★★★

## 3.1 JSON-RPC 2.0

### 为什么选择 JSON-RPC？

MCP 基于 **JSON-RPC 2.0** 协议进行通信。选择 JSON-RPC 的原因：

| 考量维度 | JSON-RPC 的优势 |
|---------|----------------|
| **简单性** | 协议极简，易于理解和实现，没有 HTTP/REST 的复杂语义 |
| **语言无关** | JSON 是通用数据格式，任何语言都有成熟的 JSON 库 |
| **传输无关** | JSON-RPC 不绑定传输层，可以在 STDIO、HTTP、WebSocket 等多种传输上运行 |
| **无状态** | 请求-响应模式简单清晰，适合工具调用的场景 |
| **成熟稳定** | JSON-RPC 2.0 是成熟的工业标准，有丰富的工具链支持 |
| **双向通信** | 支持 Request（请求-响应）和 Notification（单向通知）两种模式 |

### JSON-RPC 2.0 消息格式

#### Request（请求）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "Beijing",
      "unit": "celsius"
    }
  }
}
```

**字段说明：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `jsonrpc` | string | 是 | 固定值 `"2.0"` |
| `id` | number/string | 是 | 请求唯一标识，用于匹配响应。Notification 无此字段 |
| `method` | string | 是 | 调用的方法名，如 `tools/list`、`tools/call` |
| `params` | object/array | 否 | 方法参数 |

#### Response - Success（成功响应）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "北京当前温度：25°C，晴"
      }
    ]
  }
}
```

#### Response - Error（错误响应）

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32601,
    "message": "Method not found",
    "data": "The method 'tools/call_unknown' does not exist"
  }
}
```

**标准 JSON-RPC 错误码：**

| 错误码 | 含义 | 说明 |
|--------|------|------|
| `-32700` | Parse error | JSON 解析失败 |
| `-32600` | Invalid Request | 请求格式无效 |
| `-32601` | Method not found | 方法不存在 |
| `-32602` | Invalid params | 参数无效 |
| `-32603` | Internal error | 服务器内部错误 |
| `-32000` ~ `-32099` | Server error | 自定义服务端错误 |

**MCP 协议自定义错误码（非 JSON-RPC 标准）：**

| 错误码 | 含义 | 说明 |
|--------|------|------|
| `-32800` | Request cancelled | 请求被取消（用户中断或超时） |
| `-32801` | Server not initialized | Server 尚未完成初始化，拒绝非 initialize 请求 |
| `-32802` | Capability not supported | 请求的能力（如 sampling）未被 Client 声明支持 |

#### Notification（通知，无响应）

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "abc123",
    "progress": 50,
    "total": 100
  }
}
```

**特点**：
- 没有 `id` 字段
- 服务器**不应**返回响应
- 用于单向通知场景（进度更新、状态变更等）

---

### 面试题：MCP 为什么使用 JSON-RPC？

**标准回答：**

> 1. **简洁性**：JSON-RPC 协议设计极简，只有 Request、Response、Notification 三种消息类型，学习成本低
> 2. **传输无关**：JSON-RPC 不绑定特定传输层，MCP 可以在 STDIO、HTTP、SSE、WebSocket 上使用同一套协议语义
> 3. **语言无关**：JSON 是通用数据格式，Go、Python、TypeScript、Java 等都有成熟的 JSON 库，降低各语言的实现门槛
> 4. **双向通信**：支持 Request-Response 模式和单向 Notification 模式，能同时满足工具调用和进度通知的需求
> 5. **成熟生态**：JSON-RPC 2.0 是成熟的工业标准，有丰富的工具链和调试工具，不是 MCP 自己发明的轮子

---

### 面试题：Notification 和 Request 的区别？

**标准回答：**

> | 维度 | Request | Notification |
> |------|---------|-------------|
> | `id` 字段 | 必须有 | 必须没有 |
> | 响应 | 服务器必须返回 Response | 服务器不应返回任何响应 |
> | 用途 | 查询、命令（需要结果） | 事件通知、进度更新（不需要结果） |
> | 可靠性 | 有响应确认 | 发后即忘（fire-and-forget） |
> | MCP 示例 | `tools/call`、`tools/list` | `notifications/progress`、`notifications/cancelled` |
>
> **核心区别**：Request 是「我问你答」，Notification 是「我告诉你，不用回」。

---

# 第四章 MCP Transport（传输层） ★★★★★

## 4.1 STDIO Transport ★★★★★

### 工作原理

```text
┌──────────────┐          ┌──────────────┐
│  MCP Host    │          │  MCP Server  │
│  (父进程)     │          │  (子进程)     │
│              │  stdin   │              │
│              ├─────────►│              │
│              │          │              │
│              │  stdout  │              │
│              │◄─────────┤              │
└──────────────┘          └──────────────┘
```

**流程：**
1. Host 启动 Server 作为**子进程**
2. Host 通过子进程的 **stdin** 发送 JSON-RPC 消息
3. Server 通过自己的 **stdout** 返回 JSON-RPC 消息
4. Server 的 **stderr** 用于日志输出（不干扰协议通信）

### 启动方式

```bash
# Host 内部执行类似命令
npx -y @modelcontextprotocol/server-filesystem /path/to/workspace
```

### 消息格式

STDIO 传输中，每条 JSON-RPC 消息遵循 **行分隔（newline-delimited）** 格式：

```text
Content-Length: 256\r\n
\r\n
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{...}}
```

**格式规则：**
- 先发送 `Content-Length: <字节数>` 头
- 行分隔符必须使用 **`\r\n`（CRLF）**，这是 MCP spec 的明确要求，使用 `\n` 可能导致协议解析失败
- 空行（`\r\n\r\n`）分隔头与体
- Body 是一行完整的 JSON（不换行）
- 每条消息独立，不依赖前后消息

### 优缺点

| 优点 | 缺点 |
|------|------|
| 实现简单，无需网络配置 | 只能本地通信，不支持远程 |
| 进程隔离，安全可靠 | 每个 Server 一个进程，资源占用大 |
| 没有网络延迟 | 无法跨机器共享 Server |
| 天然适合本地工具 | 进程管理复杂（启动、重启、崩溃恢复） |
| 无需端口管理 | 不适合多 Host 共享同一 Server |

### 使用场景

- **本地 IDE 工具**：Cursor 连接本地文件系统 MCP
- **本地 Agent**：个人 AI 助手连接本地数据库
- **开发调试**：开发阶段测试 MCP Server
- **安全敏感场景**：需要进程隔离的工具调用

---

### 面试题：STDIO Transport 如何工作？

**标准回答：**

> STDIO Transport 是 MCP 最基础的传输方式。Host 启动 MCP Server 作为子进程，通过子进程的 **stdin** 发送 JSON-RPC 请求，Server 通过 **stdout** 返回 JSON-RPC 响应。消息格式使用 `Content-Length` 头 + JSON body 的行分隔格式，stderr 用于日志输出不干扰协议通信。这种方式的优点是简单、安全、适合本地场景，缺点是不支持远程通信和跨 Host 共享。

---

## 4.2 HTTP Transport（请求-响应模式）

### 工作原理

HTTP Transport 将 JSON-RPC 消息封装在标准 HTTP POST 请求中：

```text
┌──────────────┐                              ┌──────────────┐
│  MCP Client  │                              │  MCP Server  │
│  (Host 内)    │                              │  (HTTP 服务)  │
│              │   POST /mcp                  │              │
│              │   Content-Type: application/json  │         │
│              │──────────────────────────────►│              │
│              │                              │              │
│              │   HTTP/1.1 200 OK            │              │
│              │   Content-Type: application/json  │         │
│              │◄──────────────────────────────│              │
└──────────────┘                              └──────────────┘
```

**请求示例（Client → Server）：**
```http
POST /mcp HTTP/1.1
Host: mcp-server.example.com
Content-Type: application/json
Content-Length: 156

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {"city": "Beijing"}
  }
}
```

**响应示例（Server → Client）：**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 128

{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {"type": "text", "text": "北京当前温度：25°C，晴"}
    ]
  }
}
```

### HTTP Transport 的特点分析

| 特性 | 说明 | 影响 |
|------|------|------|
| **请求-响应模式** | 每次工具调用 = 一次 HTTP 请求 | 实现简单，但无法服务端推送 |
| **无状态** | 每个 HTTP 请求独立，Server 不保留会话状态 | 水平扩展容易，但需要外部会话管理 |
| **同步阻塞** | Client 发送请求后等待 HTTP 响应 | 适合短时操作，长时操作需要超时处理 |
| **标准 HTTP** | 使用标准 HTTP 协议和端口（通常是 443/80） | 防火墙友好，无需特殊端口 |
| **TLS 支持** | 天然支持 HTTPS 加密 | 传输安全有保障 |
| **单工通信** | 仅支持 Client → Server 的单向请求 | **无法支持 Server 主动推送** Notification |

### HTTP Transport 的局限性

1. **不支持服务端推送**：Server 无法主动向 Client 发送 `notifications/progress` 或 `notifications/resources/updated`
2. **不支持流式响应**：对于长时间运行的工具，Client 必须等待完整响应，无法获取中间进度
3. **连接开销**：每次工具调用都需要建立 TCP + TLS 连接（HTTP/1.1 可复用但仍有开销）
4. **无法取消**：一旦发出请求，无法方便地发送取消信号（需额外端点）

### 适用场景

- ✅ 简单的工具调用（查询 API、读取配置）
- ✅ 不需要流式响应/进度通知的场景
- ✅ 需要穿透企业防火墙/代理（标准 HTTPS 端口）
- ✅ 无服务器/函数计算场景（AWS Lambda 式的 MCP Server）
- ❌ 长时间运行的工具（需要进度反馈）
- ❌ 需要 Server 主动推送的场景（资源变更通知）
- ❌ 需要双向流式通信的复杂 Agent 场景

### 面试题：HTTP Transport 有什么优缺点？

**标准回答：**

> HTTP Transport 是 MCP 的远程通信方式，将 JSON-RPC 消息封装在 HTTP POST 请求中。
>
> **优点**：
> 1. **实现简单**：标准 HTTP，任何语言都有成熟的 HTTP 库
> 2. **防火墙友好**：使用标准端口（80/443），不被企业防火墙拦截
> 3. **天然 TLS**：通过 HTTPS 即可获得传输加密
> 4. **无状态扩展**：每个请求独立，容易水平扩展
>
> **缺点**：
> 1. **不支持服务端推送**：Server 无法主动发送 Notification
> 2. **不支持流式响应**：Client 必须等待完整响应，无法获取中间进度
> 3. **连接开销**：每次调用建立连接（相比 STDIO 和 WebSocket 的持久连接）
> 4. **无法取消**：难以中断正在执行的工具调用
>
> 正因为这些局限，MCP 后续推出了 Streamable HTTP 作为推荐的远程传输方案。

---

## 4.3 SSE Transport（Server-Sent Events）

### 设计背景

MCP 早期版本中，为了解决 HTTP Transport "无法支持服务端推送"的缺陷，引入了 SSE（Server-Sent Events）作为传输方式。SSE 是 HTML5 标准的一部分，允许服务器通过单一的 HTTP 长连接向客户端推送事件流。

### 双通道工作原理

SSE Transport 需要维护**两条独立的 HTTP 连接**：

```text
┌──────────────┐                              ┌──────────────┐
│  MCP Client  │                              │  MCP Server  │
│              │   GET /sse (建立长连接)        │              │
│              │   Accept: text/event-stream   │              │
│              │──────────────────────────────►│              │
│              │                              │              │
│              │   ◄── SSE Event Stream ────   │              │
│              │   (Server → Client 推送通道)   │              │
│              │                              │              │
│              │   POST /message              │              │
│              │   (Client → Server 请求通道)   │              │
│              │──────────────────────────────►│              │
│              │                              │              │
│              │   ◄── HTTP Response ────      │              │
│              │   (对此请求的响应)              │              │
└──────────────┘                              └──────────────┘
```

**通道 1 — GET /sse（Server → Client 推送通道）：**
- 建立 SSE 长连接，保持打开状态
- Server 通过此通道**推送**消息给 Client：
  - `notifications/progress`（进度通知）
  - `notifications/resources/updated`（资源变更）
  - `notifications/tools/list_changed`（工具列表变更）
  - 任何需要主动通知 Client 的消息

**通道 2 — POST /message（Client → Server 请求通道）：**
- Client 每次需要调用工具时，发送 POST 请求到此端点
- 携带 JSON-RPC Request（`tools/call`、`tools/list` 等）
- Server 返回 JSON-RPC Response

### 为什么 MCP 逐渐放弃 SSE？

| SSE 的问题 | 详细说明 | 实际影响 |
|-----------|---------|----------|
| **双通道复杂性** | 需要同时维护 GET/SSE（收）和 POST（发）两个连接，Client 必须管理两个独立 HTTP 连接的生命周期 | 代码量增加约 40%，调试困难 |
| **连接状态耦合** | 两个通道需要关联到同一个 session（通过 sessionId 参数），Client 需要在两个请求间同步状态 | 重连逻辑复杂，状态不一致导致消息丢失 |
| **代理兼容性差** | 很多 HTTP 代理/负载均衡器（如 Nginx 默认配置）对 SSE 长连接支持不佳，会在 60s 无活动时断开 | 需要额外配置 `proxy_read_timeout` 等参数 |
| **不支持请求-响应匹配** | SSE 通道的消息是单向推送的，无法将 Server 对某个请求的响应与请求精确匹配（需应用层实现 correlation ID） | 增加了协议层的复杂性 |
| **移动端支持弱** | iOS 的 `URLSession` 和 Android 的 `HttpURLConnection` 对 SSE 的支持不如 WebSocket 成熟 | 移动端 Agent 应用开发受限 |
| **无法复用连接** | 每个 Client 需要独立的 SSE 连接，100 个 Client = 100 个 SSE 长连接 | Server 端连接数压力大 |
| **重连机制简陋** | SSE 标准的 `Last-Event-ID` 重连机制简单，断线期间的消息会丢失 | 可靠性不足 |

### 面试题：SSE Transport 的工作原理和存在的问题？

**标准回答：**

> SSE Transport 采用**双通道**模型：
>
> **工作原理**：
> 1. **GET /sse 通道**：Client 建立 SSE 长连接，Server 通过此通道持续推送 Notification（进度、资源变更等）
> 2. **POST /message 通道**：Client 每次工具调用时发送 POST 请求，Server 返回响应
>
> **核心问题（为什么不再推荐用于新项目）**：
> 1. **双通道实现复杂**：Client 需要管理两条独立连接，session 关联逻辑繁琐
> 2. **代理不友好**：HTTP 代理/负载均衡器对 SSE 长连接支持差，容易超时断开
> 3. **请求-响应不匹配**：两个通道分离，需要额外机制关联请求和响应
> 4. **连接不可复用**：每个 Client 独占一个 SSE 连接，扩展性差
> 5. **移动端支持弱**：iOS/Android 的 SSE 实现不如 WebSocket 成熟
>
> 因此 MCP 在 2025 年推出 Streamable HTTP 作为替代方案。

---

## 4.4 Streamable HTTP Transport（2025 推荐方案） ★★★★★

### 背景

MCP 协议在 2025 年 3 月（协议版本 `2025-03-26`）的更新中引入了 **Streamable HTTP** 作为**推荐的传输方式**，逐步替代 SSE。

### 核心变化

从 SSE 的 **双通道** 模型简化为 **单一端点 + 流式响应** 模型：

```text
Client ──── POST /mcp ────► Server
           (Accept: text/event-stream)
Client ◄── streaming ────── Server
           (同一个 HTTP 连接上流式返回)
```

### 工作原理

1. Client 发送 POST 请求到单一端点（如 `/mcp`）
2. 请求头中声明支持流式响应：`Accept: text/event-stream`
3. Server 在同一个 HTTP 连接上以 SSE 格式流式返回结果
4. 支持请求-响应的精确匹配（因为在一个连接上）
5. Server 可以在此连接上主动推送 Notification

### Streamable HTTP vs SSE 对比

| 维度 | SSE Transport | Streamable HTTP |
|------|--------------|-----------------|
| 连接数 | 2 个（GET/SSE + POST） | 1 个（单一 POST 端点） |
| 复杂度 | 高（session 关联、双通道管理） | 低（单一端点，简单直接） |
| 代理友好 | 差（SSE 长连接代理支持不好） | 好（标准 HTTP POST，代理普遍支持） |
| 请求-响应匹配 | 需应用层实现 | 同一连接上天然匹配 |
| 服务端推送 | SSE 通道 | 同一 POST 连接的流式响应 |
| 移动端支持 | 弱 | 好（标准 HTTP） |
| 可恢复性 | 困难 | 支持断点续传（通过 `Last-Event-ID`） |
| 向后兼容 | - | 兼容旧版 SSE Client（可降级） |

### Streamable HTTP 的优势总结

1. **单端点架构**：只需一个 HTTP 端点，大幅简化 Client 和 Server 的实现
2. **代理友好**：标准 HTTP POST，通过任何 HTTP 代理/负载均衡器都没有兼容性问题
3. **连接效率**：一个连接完成双向通信，减少连接开销
4. **可恢复**：支持断线重连和事件恢复
5. **渐进增强**：不支持流式的 Client 可以降级为普通 HTTP 请求-响应模式
6. **全平台支持**：任何支持 HTTP 的平台都可以使用

---

### 面试题：MCP 为什么从 SSE 转向 Streamable HTTP？

**标准回答：**

> 1. **简化架构**：SSE 需要双通道（GET + POST），实现复杂；Streamable HTTP 只需单一 POST 端点
> 2. **代理兼容性**：SSE 长连接在代理/负载均衡器中支持不佳，Streamable HTTP 使用标准 HTTP POST，无兼容问题
> 3. **请求匹配**：Streamable HTTP 在一个连接上完成双向通信，天然支持请求-响应的精确匹配
> 4. **可恢复性**：Streamable HTTP 支持断点续传（Last-Event-ID），比 SSE 更可靠
> 5. **全平台兼容**：标准 HTTP 在所有平台（包括移动端）都有成熟支持
> 6. **向后兼容**：Streamable HTTP 可以降级为普通 HTTP 请求-响应模式，兼容旧版 Client

---

## 4.5 四种 Transport 全景对比 ★★★★★

### 对比总表

| 维度 | STDIO | HTTP | SSE | Streamable HTTP |
|------|-------|------|-----|-----------------|
| **连接数** | 1（父子进程管道） | 每次请求一个 | 2（GET+POST） | 1（POST 复用） |
| **通信方向** | 双向 | 单向（C→S） | 双向（分离通道） | 双向（同一通道） |
| **服务端推送** | ✅ stdout | ❌ 不支持 | ✅ SSE 通道 | ✅ 流式响应 |
| **远程访问** | ❌ 仅本地 | ✅ | ✅ | ✅ |
| **代理友好** | N/A | ✅ | ❌ 差 | ✅ 好 |
| **流式响应** | ✅ 持续 stdout | ❌ | ✅ | ✅ |
| **实现复杂度** | 低 | 低 | 高 | 中 |
| **安全性** | 进程隔离 | TLS | TLS | TLS |
| **移动端** | ❌ | ✅ | ❌ 弱 | ✅ |
| **推荐场景** | 本地 IDE 工具 | 简单远程调用 | 遗留系统 | **首选远程方案** |

### 选型决策树

```text
是否需要远程访问？
├── 否 → 用 STDIO Transport
│        - 本地开发工具
│        - 个人 Agent
│        - 安全敏感场景（进程隔离）
│
└── 是 → 用 Streamable HTTP（推荐）
         - 生产环境首选
         - 多用户共享 MCP Server
         - 需要服务端推送的 Agent 场景
         
         例外：如果环境不支持流式响应
         → 降级到普通 HTTP Transport
```

### 面试题：四种 Transport 如何选型？

**标准回答（30秒版）：**

> MCP 有四种 Transport：STDIO 用于本地工具（简单、进程隔离），HTTP 用于简单远程调用，SSE 是过渡方案（2024-11-05 版本中引入，不推荐新项目使用），Streamable HTTP 是远程场景的推荐首选——单端点双向流式通信，代理友好，全平台支持。

**标准回答（1分钟版）：**

> MCP Transport 选型的核心原则是：
> - **本地用 STDIO**：Host 启动 Server 子进程，通过 stdin/stdout 通信。零网络配置、进程隔离、最安全。适合 IDE 内置工具和个人 Agent。
> - **远程用 Streamable HTTP**：所有远程场景的首选。单一 POST 端点完成双向流式通信，支持服务端推送、断点续传、代理友好。解决了 SSE 双通道复杂和代理兼容性差的问题。
> - **过渡方案**：HTTP Transport（简单请求-响应，不支持推送）和 SSE Transport（不推荐新项目使用，Streamable HTTP 为首选替代）仅用于兼容旧系统。
>
> 实际项目中，本地开发用 STDIO 调试，生产环境部署为 Streamable HTTP 服务。

---

# 第五章 MCP Tool ★★★★★

## 5.1 什么是 Tool

**Tool（工具）** 是 MCP 中最核心的概念，代表 **LLM 可以调用的一个具体能力**。

### 本质

Tool = 一个可以被 AI 模型发现、理解和调用的**函数**

```text
用户："帮我查一下北京今天的天气"

LLM 推理：需要调用 get_weather 工具
        ↓
生成工具调用：
{
  "name": "get_weather",
  "arguments": {
    "city": "Beijing",
    "unit": "celsius"
  }
}
        ↓
MCP Server 执行 → 返回结果
        ↓
LLM 基于结果生成回复："北京今天晴，25°C"
```

---

## 5.2 Tool 定义（Tool Schema）

### 核心组成

每个 Tool 必须定义以下要素：

| 要素 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 工具唯一标识，LLM 通过 name 选择调用哪个工具 |
| `description` | string | 是 | 工具功能描述，LLM 通过 description 理解工具的用途和适用场景 |
| `inputSchema` | JSON Schema | 是 | 工具的参数定义（类型、必填项、约束），LLM 据此生成正确的参数 |
| `annotations` | object | 否 | 工具的元数据标注，帮助 Client/Host 做 UI 展示和安全控制 |

### annotations 字段详解

`annotations` 是可选的元数据对象，用于给 Host 提供 UI 提示和安全提示：

| annotation | 类型 | 说明 |
|-----------|------|------|
| `title` | string | 工具的人类可读标题（用于 UI 展示） |
| `readOnlyHint` | boolean | 提示该工具是只读的（不修改任何数据） |
| `destructiveHint` | boolean | 提示该工具可能执行破坏性操作（删除、覆盖） |
| `idempotentHint` | boolean | 提示该工具是幂等的（重复调用结果相同） |
| `openWorldHint` | boolean | 提示该工具可能连接外部世界（网络 API 等） |

### 完整示例

```json
{
  "name": "get_weather",
  "description": "获取指定城市的当前天气信息。返回温度、湿度、天气状况和风速。适用于查询实时天气。",
  "inputSchema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "城市名称，支持中文和英文，例如 'Beijing' 或 '北京'"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "温度单位，celsius（摄氏度）或 fahrenheit（华氏度）",
        "default": "celsius"
      },
      "include_forecast": {
        "type": "boolean",
        "description": "是否包含未来3天预报",
        "default": false
      }
    },
    "required": ["city"]
  },
  "annotations": {
    "title": "天气查询",
    "readOnlyHint": true,
    "destructiveHint": false,
    "idempotentHint": true,
    "openWorldHint": true
  }
}
```

### Tool 定义要点

**1. name 命名规范：**
- 使用 `snake_case`（如 `get_weather`、`search_repos`）
- 使用动词开头，清晰表达动作
- 全局唯一

**2. description 编写原则：**
- **越详细越好**：LLM 完全依赖 description 理解工具的用途
- **说明适用场景**："适用于查询实时天气" vs "适用于查询历史天气"
- **说明限制条件**："最多返回 10 条结果"
- **说明副作用**："此操作会在服务器上创建文件"
- **使用自然语言**：面向 LLM 的描述，不需要像 API 文档那样正式

**3. inputSchema 设计原则：**
- 使用 JSON Schema 标准定义
- 每个参数都要有清晰的 `description`
- 合理使用 `enum` 限制选项
- 设置合适的 `default` 值
- 明确 `required` 字段

---

### 面试题：MCP Tool 如何定义？

**标准回答：**

> MCP Tool 通过三个核心要素定义：
> 1. **name**：工具的唯一标识符，使用 snake_case 命名，动词开头（如 `get_weather`）
> 2. **description**：详细的功能描述，包括适用场景、限制条件、副作用。这是 LLM 理解工具用途的唯一依据，必须写得足够详细
> 3. **inputSchema**：使用 JSON Schema 标准定义工具的参数，包括类型、枚举值、必填项、默认值
>
> 这三个要素通过 `tools/list` 方法暴露给 LLM。LLM 通过 `name` 选择工具，通过 `description` 理解何时使用，通过 `inputSchema` 生成正确的调用参数。

---

## 5.3 Tool 调用流程 ★★★★★

### 完整调用流程

```text
步骤1: 用户输入
    │
    "帮我查一下北京今天的天气，并保存到文件"
    │
    ▼
步骤2: LLM 分析（ReAct 循环）
    │
    分析：需要先查天气，再保存文件
    决策：调用 get_weather 工具
    │
    ▼
步骤3: tools/list（发现阶段，仅初始化时执行一次）
    │
    Host → MCP Server:
    {
      "jsonrpc": "2.0",
      "id": 1,
      "method": "tools/list"
    }
    │
    MCP Server → Host:
    {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "tools": [
          { "name": "get_weather", "description": "...", "inputSchema": {...} },
          { "name": "write_file", "description": "...", "inputSchema": {...} }
        ]
      }
    }
    │
    ▼
步骤4: 工具选择（LLM 决策）
    │
    LLM 从可用工具列表中分析：
    - get_weather: ✓ 可以获取天气
    - write_file: ✗ 暂时不需要（先获取数据）
    决策：先调用 get_weather
    │
    ▼
步骤5: tools/call（调用阶段）
    │
    Host → MCP Server:
    {
      "jsonrpc": "2.0",
      "id": 2,
      "method": "tools/call",
      "params": {
        "name": "get_weather",
        "arguments": {
          "city": "Beijing",
          "unit": "celsius"
        }
      }
    }
    │
    ▼
步骤6: 工具执行（MCP Server）
    │
    MCP Server 执行 get_weather 的处理逻辑：
    - 调用天气 API
    - 解析结果
    - 格式化输出
    │
    ▼
步骤7: 返回结果
    │
    MCP Server → Host:
    {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "content": [
          {
            "type": "text",
            "text": "北京当前温度：25°C，晴，湿度：45%，风速：3级"
          }
        ]
      }
    }
    │
    ▼
步骤8: LLM 再次推理
    │
    获得天气数据后：
    - 评估：数据已经拿到 ✓
    - 下一步：需要保存到文件 → 调用 write_file
    │
    ▼
步骤9: 第二次 tools/call
    │
    Host → MCP Server:
    {
      "jsonrpc": "2.0",
      "id": 3,
      "method": "tools/call",
      "params": {
        "name": "write_file",
        "arguments": {
          "path": "/tmp/weather.txt",
          "content": "北京当前温度：25°C，晴，湿度：45%"
        }
      }
    }
    │
    ▼
步骤10: 生成最终回复
    │
    LLM 综合所有结果：
    "已为您查询北京天气（25°C，晴）并保存到 /tmp/weather.txt"
```

### 调用流程中的关键点

1. **工具发现是懒加载的**：`tools/list` 通常在初始化阶段调用一次，缓存结果
2. **LLM 自主决策**：LLM 根据任务的语义理解，自主决定调用哪个工具、以什么参数调用、按什么顺序调用
3. **多轮调用**：复杂任务可能涉及多个工具的多轮调用（ReAct 循环：Reasoning + Acting）
4. **错误处理**：工具调用失败时，LLM 会收到错误信息并尝试调整策略

### MCP Tool 调用的 ReAct 循环

```text
┌─────────────────────────────────────────┐
│           ReAct 循环（Reasoning + Acting） │
│                                           │
│  用户输入                                 │
│     ↓                                     │
│  ┌─────────┐                             │
│  │ Thought │ ← 分析当前状态，思考下一步   │
│  └────┬────┘                             │
│       ↓                                   │
│  ┌─────────┐                             │
│  │ Action  │ ← 调用 Tool（tools/call）    │
│  └────┬────┘                             │
│       ↓                                   │
│  ┌──────────┐                            │
│  │Observation│ ← 接收 Tool 返回的结果      │
│  └────┬─────┘                            │
│       ↓                                   │
│  ┌──────────┐                            │
│  │ 任务完成？ │                           │
│  └────┬─────┘                            │
│   是  │  否 → 回到 Thought               │
│    ↓                                      │
│  生成最终回答                             │
└─────────────────────────────────────────┘
```

---

### 面试题：MCP Tool 调用流程是什么？

**标准回答：**

> MCP Tool 调用遵循 **工具发现 → 工具选择 → 工具调用 → 结果返回 → 再次推理** 的循环流程：
>
> 1. **工具发现（tools/list）**：初始化时 Host 从 Server 获取所有可用 Tool 的列表（name、description、inputSchema）
> 2. **工具选择（LLM 决策）**：LLM 根据用户意图和 Tool 的 description，自主选择最合适的工具并生成参数
> 3. **工具调用（tools/call）**：Host 通过 JSON-RPC 向 Server 发送 `tools/call` 请求
> 4. **工具执行（Server 处理）**：Server 执行实际的业务逻辑（查数据库、调 API 等）
> 5. **结果返回**：Server 将执行结果以 `content` 数组形式返回
> 6. **再次推理（LLM）**：LLM 基于结果判断是否需要继续调用其他工具，形成 ReAct 循环直到任务完成

---

### 面试回答模板（Tool 章节综合）

**30 秒版（说给面试官的第一句话）：**

> "MCP 的 Tool 本质上是 LLM 可以调用的标准化函数。每个 Tool 由三个要素定义：name（唯一标识，snake_case 命名，动词开头）、description（详细描述用途、场景、限制、副作用——这是 LLM 选择工具的唯一依据）、inputSchema（JSON Schema 定义参数类型和约束）。Tool 通过 `tools/list` 被发现，通过 `tools/call` 被调用，结果以 `content` 数组返回。整个调用流程遵循 ReAct 循环直到任务完成。"

**1 分钟版（边画图边讲）：**

> "MCP Tool 的调用流程可以分为发现阶段和调用阶段。发现阶段在初始化后执行一次：Client 调用 `tools/list` 获取所有 Tool 的 name、description 和 inputSchema，LLM 据此建立'工具认知'。调用阶段是一个 ReAct 循环：LLM 收到用户请求后，在 Thought 阶段分析需要什么工具，在 Action 阶段通过 `tools/call` 发送工具名和参数，在 Observation 阶段接收 `content` 数组结果，然后判断是否需要继续调用。这里的关键是——description 写得越详细，LLM 选择工具的准确率越高，所以这是 MCP Server 开发中最重要的工作。"

**3 分钟版（完整面试答辩）：**

> "Let me walk through the complete lifecycle of an MCP Tool.
>
> **First, Tool Definition**: Every tool has three mandatory components. `name` follows snake_case, starts with a verb like `get_weather` or `search_repos`. `description` is the most critical — it's the ONLY thing the LLM reads to decide when to use this tool. A good description includes: what the tool does, when to use it (and when not to), any limitations (like 'max 10 results'), and any side effects (like 'this will create a file'). `inputSchema` uses JSON Schema to define parameters with types, enums, defaults, and required fields.
>
> **Second, Discovery via tools/list**: After initialization, the Client sends a `tools/list` request. The Server responds with the complete list of all tools and their schemas. This is cached — it only happens once per connection.
>
> **Third, the ReAct Loop**: This is where the magic happens. The LLM receives a user request like '查一下北京天气并保存到文件'. It enters a Thought-Action-Observation cycle:
> - Thought: 'I need weather data first, then save it'
> - Action: calls `get_weather` with `{city: 'Beijing', unit: 'celsius'}`
> - Observation: receives `{content: [{type: 'text', text: '25°C, 晴'}]}`
> - Thought: 'Got weather data. Now I need to save it'
> - Action: calls `write_file` with `{path: '/tmp/weather.txt', content: '...'}`
> - Observation: receives success confirmation
> - Final Answer: '已查询并保存'
>
> The critical design principle: **each tool should do ONE thing** (atomic operation), so the LLM can compose them flexibly. If a tool does too much, it becomes hard for the LLM to use correctly.

---

# 第六章 MCP Resource ★★★★★

## 6.1 什么是 Resource

**Resource（资源）** 是 MCP 中代表**可被 LLM 读取的上下文数据**的概念。

### Tool vs Resource 的核心区别

| 维度 | Tool | Resource |
|------|------|----------|
| **本质** | 动作 / 操作 | 数据 / 状态 |
| **方向** | LLM → 执行 → 获得结果 | LLM ← 读取 ← 获取数据 |
| **副作用** | 可能有（写入、删除、修改） | 无（只读） |
| **类比** | 函数 / RPC 调用 | 文件 / REST GET |
| **访问方式** | `tools/call` | `resources/read` |
| **发现方式** | `tools/list` | `resources/list` |
| **典型示例** | `query_mysql`, `create_issue` | `file://readme.md`, `mysql://users/table` |

### 简单理解

```text
Tool 是 "做什么"
Resource 是 "有什么"

示例：
- Tool: "帮我查数据库"（执行 SQL 查询）
- Resource: "这个数据库有哪些表"（读取元数据）
```

---

## 6.2 Resource URI

### URI 格式

MCP Resource 使用 URI 来唯一标识每个资源：

```text
scheme://authority/path

示例：
- file:///home/user/project/readme.md    # 本地文件
- mysql://database/users                  # 数据库表
- redis://cache/session:*                 # 缓存键
- github://owner/repo/issues/42          # GitHub Issue
- notion://workspace/page/abc123         # Notion 页面
- slack://team/channel/general           # Slack 频道
```

### Resource 定义示例

```json
{
  "uri": "file:///home/user/project/src/main.go",
  "name": "main.go",
  "description": "项目的主入口文件，包含 main 函数和路由配置",
  "mimeType": "text/x-go"
}
```

### Resource 模板（动态 URI）

支持 URI 模板来处理动态资源：

```json
{
  "uriTemplate": "mysql://{database}/{table}",
  "name": "Database Table",
  "description": "数据库表中的数据",
  "mimeType": "application/json"
}
```

LLM 可以基于模板生成具体的 URI：`mysql://production/users`

---

### 面试题：Resource 和 Tool 的区别？

**标准回答：**

> | 维度 | Tool | Resource |
> |------|------|----------|
> | 语义 | 动作（Action） | 数据（Data） |
> | 类比 | RPC 调用 / 函数 | REST GET / 文件读取 |
> | 副作用 | 可能有写入/删除 | 无（只读） |
> | 调用方式 | `tools/call` | `resources/read` |
> | 核心方法 | `tools/list`, `tools/call` | `resources/list`, `resources/read` |
>
> **一句话总结**：Tool 是「能做什么」，Resource 是「有什么数据」。Tool 用于执行操作，Resource 用于获取上下文。

---

## 6.3 Resource 订阅机制（resources/subscribe） ★★★★

### 什么是 Resource 订阅

`resources/subscribe` 允许 **Client 订阅某个 Resource 的变更通知**。当 Resource 的内容发生变化时，Server 主动推送 `notifications/resources/updated` 给 Client。

### 工作流程

```text
Client                                    Server
  │                                          │
  │  1. Client 订阅某个资源                    │
  │─────────────────────────────────────────►│
  │  {                                       │
  │    "method": "resources/subscribe",      │
  │    "params": {                           │
  │      "uri": "file:///project/config.yaml" │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  2. Server 确认订阅                       │
  │◄─────────────────────────────────────────│
  │  { "result": {} }                        │
  │                                          │
  │  ... 一段时间后，文件被外部程序修改 ...     │
  │                                          │
  │  3. Server 推送变更通知                   │
  │◄─────────────────────────────────────────│
  │  {                                       │
  │    "method": "notifications/resources/   │
  │               updated",                  │
  │    "params": {                           │
  │      "uri": "file:///project/config.yaml" │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  4. Client 重新读取资源                   │
  │─────────────────────────────────────────►│
  │  { "method": "resources/read",           │
  │    "params": {                           │
  │      "uri": "file:///project/config.yaml" │
  │    }                                     │
  │  }                                       │
```

### 使用场景

| 场景 | 说明 |
|------|------|
| **配置文件监控** | 订阅 `config.yaml`，配置变更时 Agent 自动感知 |
| **日志文件追踪** | 订阅 `app.log`，实时获取最新日志内容 |
| **数据库变更通知** | 订阅 `mysql://{db}/{table}`，数据变更时触发 Agent 重新分析 |
| **协作场景** | 多人编辑同一文件时，Agent 感知文件变化 |

### 与 `resources/list_changed` 的区别

| 维度 | resources/updated | resources/list_changed |
|------|-------------------|------------------------|
| **粒度** | 单个资源的内容变更 | 整个资源列表的结构变更 |
| **触发条件** | 文件内容变化 | 新增/删除资源 |
| **携带信息** | URI | 无（Client 需重新 `resources/list`） |
| **需要订阅** | 是（需先 `resources/subscribe`） | 否（Client 初始化后自动生效） |

### 面试题：MCP 如何实现资源变更通知？

**标准回答：**

> MCP 通过订阅-推送模型实现资源变更通知：
> 1. **订阅**：Client 通过 `resources/subscribe` 订阅感兴趣的 Resource URI
> 2. **推送**：当 Resource 内容发生变化时，Server 主动推送 `notifications/resources/updated`（携带变化的 URI）
> 3. **重新读取**：Client 收到通知后，通过 `resources/read` 获取最新内容
>
> 与之平行的还有 `notifications/resources/list_changed`：当资源列表结构变化时（新增/删除资源），Server 通知 Client 重新调用 `resources/list`。前者针对内容变更，后者针对结构变更。

---

# 第七章 MCP Prompt ★★★★

## 7.1 什么是 Prompt 资源

**Prompt 资源** 是 MCP Server 可以提供的**预定义提示词模板**。

### 作用

- 提供可复用的提示词模板（如代码审查、单元测试生成）
- 封装领域知识（如特定框架的最佳实践）
- 标准化常见任务的交互方式
- 支持参数化（如 `review_code` 接受 `language` 参数）

### Prompt 定义示例

```json
{
  "name": "code_review",
  "description": "对代码进行全面的 Code Review",
  "arguments": [
    {
      "name": "language",
      "description": "代码语言",
      "required": true
    },
    {
      "name": "focus_areas",
      "description": "关注领域：security, performance, readability",
      "required": false
    }
  ]
}
```

### Prompt 模板内容

```markdown
你是一名资深 {language} 代码审查专家。请对以下代码进行全面审查：

重点关注：
- 代码正确性
- 安全问题
{focus_areas}

请按以下格式输出审查结果：
1. 严重问题（必须修复）
2. 改进建议
3. 最佳实践检查
```

---

### 面试题：MCP Prompt 有什么作用？

**标准回答：**

> MCP Prompt 是预定义的提示词模板，有三个核心作用：
> 1. **模板复用**：提供标准化的提示词模板（代码审查、测试生成等），避免每次重复编写
> 2. **领域知识封装**：将特定领域的专业知识（安全审查规范、代码风格指南）固化到模板中
> 3. **参数化定制**：支持参数（语言、关注领域），同一个模板适用于不同场景
>
> Prompt 本质上是 Server 提供的「Prompt 服务」，让用户无需记住复杂提示词即可获得高质量的输出。

---

# 第八章 MCP 生命周期 ★★★★★

## 8.1 完整生命周期

```text
┌──────────────────────────────────────────────────┐
│              MCP 连接生命周期                      │
│                                                    │
│  [启动]                                            │
│     │                                              │
│     ▼                                              │
│  ┌──────────┐                                     │
│  │ 1. 启动  │ Host 启动 Server 子进程              │
│  └────┬─────┘                                     │
│       │                                           │
│       ▼                                           │
│  ┌──────────────┐                                 │
│  │ 2. Initialize│ 握手 & 能力协商                  │
│  │    (握手)     │ Client ↔ Server 交换版本和能力   │
│  └────┬─────────┘                                 │
│       │                                           │
│       ▼                                           │
│  ┌──────────────┐                                 │
│  │ 3. Ready     │ Server 就绪                      │
│  │    (就绪)     │ 发送 initialized 通知            │
│  └────┬─────────┘                                 │
│       │                                           │
│       ▼                                           │
│  ┌──────────────┐                                 │
│  │ 4. 正常操作   │ 工具调用 & 资源读取               │
│  │              │ ← tools/list                     │
│  │              │ ← tools/call                     │
│  │              │ ← resources/list                 │
│  │              │ ← resources/read                 │
│  │              │ → notifications/progress          │
│  └────┬─────────┘                                 │
│       │                                           │
│       ▼                                           │
│  ┌──────────────┐                                 │
│  │ 5. Shutdown  │ 关闭连接                         │
│  │    (关闭)     │ Server 清理资源并退出            │
│  └──────────────┘                                 │
└──────────────────────────────────────────────────┘
```

## 8.2 初始化阶段（Initialize）★★★★★

### 详细流程

```text
Client                                    Server
  │                                          │
  │  1. initialize 请求                       │
  │─────────────────────────────────────────►│
  │  {                                       │
  │    "method": "initialize",               │
  │    "params": {                           │
  │      "protocolVersion": "2024-11-05",    │
  │      "capabilities": {                   │
  │        "roots": {...},                   │
  │        "sampling": {...}                 │
  │      },                                  │
  │      "clientInfo": {                     │
  │        "name": "claude-desktop",         │
  │        "version": "1.0.0"               │
  │      }                                   │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  2. initialize 响应                       │
  │◄─────────────────────────────────────────│
  │  {                                       │
  │    "result": {                           │
  │      "protocolVersion": "2024-11-05",    │
  │      "capabilities": {                   │
  │        "tools": {},                      │
  │        "resources": {},                  │
  │        "prompts": {}                     │
  │      },                                  │
  │      "serverInfo": {                     │
  │        "name": "github-mcp-server",      │
  │        "version": "1.2.0"               │
  │      }                                   │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  3. initialized 通知                      │
  │◄─────────────────────────────────────────│
  │  {                                       │
  │    "method": "notifications/initialized" │
  │  }                                       │
  │                                          │
  │  ✓ 连接就绪，可以开始正常操作              │
```

### 能力协商（Capabilities）

Client 和 Server 在初始化时相互声明自己支持的能力：

**Client 可声明的能力：**
- `roots`：提供根目录信息
- `sampling`：支持 LLM 采样请求
- `experimental`：实验性功能

**Server 可声明的能力：**
- `tools`：提供工具调用能力
- `resources`：提供资源读取能力
- `prompts`：提供提示词模板
- `logging`：支持日志功能
- `experimental`：实验性功能

---

## 8.3 正常操作阶段的方法

### Client → Server（请求）

| 方法 | 类别 | 说明 |
|------|------|------|
| `tools/list` | Tool | 获取所有可用工具列表 |
| `tools/call` | Tool | 调用指定工具 |
| `resources/list` | Resource | 获取所有可用资源列表 |
| `resources/read` | Resource | 读取指定资源的内容 |
| `resources/templates/list` | Resource | 获取资源 URI 模板 |
| `resources/subscribe` | Resource | 订阅资源变更通知 |
| `prompts/list` | Prompt | 获取所有可用 Prompt 列表 |
| `prompts/get` | Prompt | 获取指定 Prompt 的内容 |
| `logging/setLevel` | Logging | 设置 Server 日志级别 |
| `ping` | 心跳 | 检查连接是否存活 |

### Server → Client（通知）及双向通知

| 方法 | 方向 | 说明 |
|------|------|------|
| `notifications/progress` | S→C | 工具执行进度通知 |
| `notifications/cancelled` | S→C | 请求取消通知 |
| `notifications/logging` | S→C | 日志消息推送通知 |
| `notifications/resources/updated` | S→C | 资源内容变更通知 |
| `notifications/resources/list_changed` | S→C | 资源列表变更通知 |
| `notifications/tools/list_changed` | S→C | 工具列表变更通知 |
| `notifications/prompts/list_changed` | S→C | Prompt 列表变更通知 |
| `notifications/roots/list_changed` | C→S | Roots 列表变更通知 |

---

### 面试题：MCP 初始化流程是什么？

**标准回答：**

> MCP 初始化分为三步：
> 1. **Client 发送 `initialize` 请求**：携带协议版本、自身能力声明、客户端信息（名称 + 版本）
> 2. **Server 返回 `initialize` 响应**：确认协议版本、声明自身能力（tools/resources/prompts/logging）、返回服务端信息
> 3. **Server 发送 `notifications/initialized` 通知**：告知 Client 初始化完成，可以开始正常操作
>
> **核心是第二步的能力协商**：Client 告诉 Server「我能做什么」，Server 告诉 Client「我提供什么」。双方根据能力交集决定后续可用的功能。

---

### 面试题：tools/list 和 tools/call 的区别？

**标准回答：**

> | 维度 | tools/list | tools/call |
> |------|-----------|------------|
> | 目的 | 发现可用工具 | 执行具体工具 |
> | 频率 | 初始化时调用一次（可缓存） | 每次需要执行工具时调用 |
> | 参数 | 无（或分页参数） | 工具名 + 参数 |
> | 返回 | Tool 列表（name + description + inputSchema） | 工具执行结果（content 数组） |
> | 类比 | API 文档 / Swagger | API 调用 |
>
> `tools/list` 告诉 LLM「你能用什么」，`tools/call` 让 LLM「实际去用」。

---

## 8.4 Content 类型详解 ★★★★★

### 什么是 Content

Content 是 MCP 中**工具执行结果和资源读取结果的标准化数据格式**。每次 `tools/call` 的返回结果和 `resources/read` 的返回结果都使用 Content 数组表示。

### Content 的四种类型

MCP 协议定义了以下 Content 类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| **text** | 纯文本内容 | 对话回复、查询结果、错误信息 |
| **image** | 图片内容（base64 编码） | 图表、截图、照片 |
| **resource** | 嵌入的资源引用（URI + MIME 类型） | 引用文件路径或数据库记录 |
| **embedded resource** | 嵌入的完整资源内容（含 URI、MIME 类型和实际数据） | 同时提供资源标识和其完整内容 |

### 1. Text Content

最常用的类型，表示纯文本或 Markdown 格式的文本：

```json
{
  "type": "text",
  "text": "北京当前温度：25°C，晴，湿度：45%"
}
```

**字段说明**：
- `type`：固定值 `"text"`
- `text`：文本内容（支持 Markdown 格式）
- `annotations`（可选）：元数据标注

### 2. Image Content

用于返回图片，数据以 base64 编码：

```json
{
  "type": "image",
  "data": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk",
  "mimeType": "image/png"
}
```

**字段说明**：
- `type`：固定值 `"image"`
- `data`：base64 编码的图片数据（不含 `data:image/png;base64,` 前缀）
- `mimeType`：图片 MIME 类型（`image/png`、`image/jpeg`、`image/gif`、`image/webp` 等）

**使用场景**：
- 生成图表/统计图
- 返回截图（Puppeteer MCP Server）
- OCR 识别结果的可视化标注

### 3. Resource Content（资源引用）

返回资源的引用信息（URI + MIME 类型），不包含实际内容：

```json
{
  "type": "resource",
  "resource": {
    "uri": "file:///reports/q1-sales-2026.xlsx",
    "mimeType": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "text": "Q1 销售报表"
  }
}
```

**使用场景**：
- 工具返回"找到以下相关文件"，LLM 后续通过 `resources/read` 获取内容
- 减少大文件传输（按需读取）

### 4. Embedded Resource Content

同时包含资源引用和完整内容：

```json
{
  "type": "resource",
  "resource": {
    "uri": "file:///config/settings.json",
    "mimeType": "application/json",
    "text": "{\"theme\": \"dark\", \"lang\": \"zh-CN\"}"
  }
}
```

**与 Resource 的区别**：Embedded Resource 的 `resource` 对象中除了 `uri` 和 `mimeType`，还包含 `text` 或 `blob`（base64 编码的二进制数据），即资源内容和引用一起返回。

### Content 数组的灵活性

一次工具调用可以返回**多种类型的 Content** 混合：

```json
{
  "result": {
    "content": [
      {
        "type": "text",
        "text": "已生成销售趋势图："
      },
      {
        "type": "image",
        "data": "iVBORw0KGgo...",
        "mimeType": "image/png"
      },
      {
        "type": "text",
        "text": "数据来源：mysql://sales/q1_orders，共 1,234 条记录"
      }
    ]
  }
}
```

### 面试题：MCP Content 有哪几种类型？

**标准回答：**

> MCP Content 有四种类型：
> 1. **text**：纯文本/Markdown 内容，最常用
> 2. **image**：图片内容（base64 编码 + MIME 类型），用于返回图表、截图
> 3. **resource**：资源引用（URI + MIME 类型），LLM 可后续通过 `resources/read` 获取
> 4. **embedded resource**：完整嵌入的资源（引用 + 实际数据一起返回）
>
> 一次返回可以混合多种类型，比如同时返回文字说明和图表图片。

---

## 8.5 Tool Call 取消机制 ★★★★

### 什么是 Tool Call 取消

当用户中断 LLM 的回复（点击停止按钮），或者 Agent 发现当前策略不可行时，需要**取消正在执行的工具调用**。

### 取消流程

```text
Client                                    Server
  │                                          │
  │  1. tools/call 请求（带进度 token）        │
  │─────────────────────────────────────────►│
  │  {                                       │
  │    "method": "tools/call",               │
  │    "params": {                           │
  │      "name": "long_running_task",        │
  │      "_meta": {                          │
  │        "progressToken": "token-abc123"   │
  │      }                                   │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  2. 用户点击取消/Agent 决定中断           │
  │                                          │
  │  3. 取消通知                              │
  │─────────────────────────────────────────►│
  │  {                                       │
  │    "method": "notifications/cancelled",  │
  │    "params": {                           │
  │      "requestId": 42,                    │
  │      "reason": "User cancelled"          │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  4. Server 停止执行                       │
  │     清理资源，返回部分结果或错误           │
  │                                          │
  │  5. Error Response（可选）                │
  │◄─────────────────────────────────────────│
  │  {                                       │
  │    "id": 42,                             │
  │    "error": {                            │
  │      "code": -32800,                     │
  │      "message": "Request cancelled"      │
  │    }                                     │
  │  }                                       │
```

### 实现要点

1. **Server 端需要周期性检查取消信号**：长时间运行的工具（如大数据查询）应在执行循环中检查取消状态
2. **优雅中断**：取消不是暴力 kill，Server 应清理资源（关闭 DB 连接、删除临时文件）
3. **超时自动取消**：Client 可设置超时，到期自动发送取消通知
4. **`progressToken` 关联**：取消通知通过 `requestId`（即原始请求的 `id`）关联到被取消的请求

### Go 实现示例（Server 端检查取消）

```go
func handleLongRunningTask(ctx context.Context, args map[string]any) (*mcp.CallToolResult, error) {
    for i := 0; i < 100; i++ {
        // 每次迭代检查是否被取消
        select {
        case <-ctx.Done():
            // 清理资源
            cleanup()
            return &mcp.CallToolResult{
                IsError: true,
                Content: []mcp.Content{
                    mcp.TextContent{Type: "text", Text: "任务已被取消"},
                },
            }, nil
        default:
            // 继续执行
        }
        // 执行实际工作...
        time.Sleep(100 * time.Millisecond)
    }
    return successResult(), nil
}
```

### 面试题：MCP 如何取消正在执行的工具调用？

**标准回答：**

> MCP 通过 `notifications/cancelled` 通知来取消工具调用：
> 1. Client 发送 `tools/call` 请求时，可以在 `_meta.progressToken` 中传递进度追踪标识
> 2. 当用户取消或 Agent 决定中断时，Client 发送 `notifications/cancelled` 通知，携带被取消请求的 `requestId`
> 3. Server 收到取消通知后，应**优雅中断**执行——清理资源（关闭连接、删除临时文件），停止耗时操作
> 4. Server 端实现要点：长时间运行的工具需要在执行循环中**周期性检查 `ctx.Done()`**，Go 中通过 `context.Context` 实现
>
> 取消不是暴力 kill，Server 有责任做资源清理。

---

## 8.6 MCP Sampling（Server → LLM 反向调用） ★★★★★

### 什么是 Sampling

Sampling 是 MCP 的一项高级能力，允许 **MCP Server 反向请求 LLM 生成文本**。

在正常的 MCP 流程中，是 LLM → Server（LLM 调用工具）。而 Sampling 是 Server → LLM（工具执行过程中需要 LLM 帮忙推理）。

### 使用场景

```text
场景 1：智能摘要
  Client → Server: tools/call "list_recent_issues"（返回 200 个 Issue）
  Server → LLM (Sampling): "请把以下 200 个 Issue 按优先级分类并生成摘要"
  Server → Client: 返回分类后的摘要结果

场景 2：代码审查
  Client → Server: tools/call "analyze_code"（读取仓库代码）
  Server → LLM (Sampling): "请审查以下代码的安全漏洞"
  Server → Client: 返回审查结果
```

### Sampling 的工作流程

```text
Client                                    Server
  │                                          │
  │  1. Server 需要 LLM 推理                  │
  │     （Server 代码内部）                    │
  │                                          │
  │  2. Server 发送 sampling/createMessage   │
  │◄─────────────────────────────────────────│
  │  {                                       │
  │    "method": "sampling/createMessage",   │
  │    "params": {                           │
  │      "messages": [                       │
  │        {                                │
  │          "role": "user",                │
  │          "content": {                   │
  │            "type": "text",              │
  │            "text": "请总结以下内容..."    │
  │          }                              │
  │        }                                │
  │      ],                                 │
  │      "maxTokens": 1000                  │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  3. Client 调用 LLM API                  │
  │     （实际调用 Claude/GPT）               │
  │                                          │
  │  4. Client 返回 LLM 生成结果              │
  │─────────────────────────────────────────►│
  │  {                                       │
  │    "result": {                           │
  │      "role": "assistant",               │
  │      "content": {                       │
  │        "type": "text",                  │
  │        "text": "总结：主要有三类问题..."   │
  │      },                                  │
  │      "model": "claude-sonnet-4-6",      │
  │      "stopReason": "endTurn"            │
  │    }                                     │
  │  }                                       │
```

### 能力声明

Sampling 需要 Client 在初始化时声明支持：

```json
{
  "method": "initialize",
  "params": {
    "capabilities": {
      "sampling": {}  // Client 声明支持 Sampling
    }
  }
}
```

如果 Client 未声明 `sampling` 能力，Server 的 `sampling/createMessage` 请求将被 Client 拒绝。

### Sampling 的限制

1. **需要 Client 授权**：Client 可以限制 Sampling 的 `maxTokens`、允许的 model
2. **只读 LLM 调用**：Sampling 不改变 LLM 的对话历史，是独立的生成请求
3. **成本考量**：每次 Sampling 都会消耗 LLM API 的 token 配额
4. **安全性**：Server 不应通过 Sampling 注入恶意提示词

### 面试题：MCP Sampling 是什么？和 Function Calling 有什么不同？

**标准回答：**

> Sampling 是 MCP Server **反向调用 LLM** 的机制：
> - **方向不同**：Function Calling 是 LLM → Server（LLM 调用工具），Sampling 是 Server → LLM（工具执行中需要 LLM 推理）
> - **用途不同**：Sampling 用于工具内部需要 LLM 帮助的场景，如"查到的 200 条数据请 LLM 做摘要"、"读取的代码请 LLM 做审查"
> - **由 Client 控制**：Server 发送 `sampling/createMessage` 请求，Client 决定是否以及如何调用 LLM
> - **需要能力声明**：Client 必须在 initialize 时声明 `sampling` 能力，否则 Server 不能使用
>
> Sampling 让 MCP Server 从"被动执行"变为"主动请求 AI 帮助"，是构建智能 Agent 工具的关键能力。

---

## 8.7 MCP Roots（根目录声明） ★★★★

### 什么是 Roots

Roots 是 MCP Client 向 Server 声明的**允许访问的文件系统根路径**。它是文件系统 MCP Server 的权限边界。

### 设计目的

```text
问题：Client 连接 Filesystem MCP Server 后，
     Server 如何知道可以访问哪些目录？
     - 如果允许 / → 安全风险（可读取系统文件）
     - 如果限制 /tmp → Server 需要知道这个限制

解决方案：Client 在初始化后通过 Roots 机制
        告知 Server 允许访问的目录范围
```

### Roots 工作流程

```text
Client                                    Server
  │                                          │
  │  1. Initialize（Client 声明 roots 能力）   │
  │─────────────────────────────────────────►│
  │  { "capabilities": { "roots": {          │
  │      "listChanged": true                 │
  │  }}}                                     │
  │                                          │
  │  2. Server 请求 Root 列表                 │
  │◄─────────────────────────────────────────│
  │  { "method": "roots/list" }              │
  │                                          │
  │  3. Client 返回允许的 Root 列表           │
  │─────────────────────────────────────────►│
  │  { "result": { "roots": [                │
  │      {                                   │
  │        "uri": "file:///home/user/project",│
  │        "name": "Project Workspace"       │
  │      }                                   │
  │  ]}}                                     │
  │                                          │
  │  4. Server 仅在允许的 Root 范围内操作     │
  │     拒绝 file:///etc/passwd 等越权访问    │
```

### Roots 的关键特性

| 特性 | 说明 |
|------|------|
| **Server 主动请求** | Server 通过 `roots/list` 请求 Client 告知可访问范围 |
| **动态更新** | Client 可通过 `notifications/roots/list_changed` 通知 roots 变更 |
| **多个 Root** | 一个 Client 可以声明多个 Root 路径 |
| **不只文件系统** | Roots 不仅限于文件系统，可以是任何命名空间的根（如 `github://`、`mysql://`） |

### 面试题：MCP Roots 是什么？

**标准回答：**

> Roots 是 MCP Client 告知 Server **允许访问的根路径边界**的机制：
> 1. Client 在 initialize 时声明 `roots` 能力
> 2. Server 通过 `roots/list` 请求获取允许访问的根路径列表
> 3. Server 只能在 Roots 范围内操作，拒绝越权访问
> 4. Roots 可以动态更新（Client 发送 `notifications/roots/list_changed`）
>
> 这是 MCP 安全模型的关键一环：**Server 不应该假设自己可以访问整个系统，必须遵守 Client 声明的边界。**

---

## 8.8 MCP Logging（日志机制） ★★★

### 什么是 MCP Logging

MCP Logging 允许 **MCP Server 向 Client 发送结构化的日志消息**。这与 Server 进程的本地日志（stderr）不同——MCP Logging 是通过协议通道发送的，Client 可以展示给用户或写入日志系统。

### 能力声明

Server 在 initialize 时声明支持 logging：

```json
{
  "capabilities": {
    "logging": {}  // Server 支持日志功能
  }
}
```

### 日志级别设置

Client 可以设置 Server 的日志级别：

```json
// Client → Server: 设置日志级别
{
  "method": "logging/setLevel",
  "params": {
    "level": "debug"  // debug | info | notice | warning | error | critical | alert | emergency
  }
}
```

### Server 发送日志消息

```json
// Server → Client: 日志通知
{
  "method": "notifications/logging",
  "params": {
    "level": "warning",
    "logger": "database-connector",
    "data": "连接池已满（100/100），等待可用连接..."
  }
}
```

**字段说明**：
- `level`：日志级别（遵循 syslog 标准）
- `logger`：日志来源模块名称
- `data`：日志内容（可以是字符串或结构化数据）

### 与 stderr 的区别

| 维度 | MCP Logging | stderr |
|------|------------|--------|
| **传输通道** | MCP 协议通道（stdout/HTTP） | OS 标准错误流 |
| **结构化** | 统一格式（level + logger + data） | 自由格式 |
| **级别控制** | Client 可动态设置 `logging/setLevel` | 需 Server 内部实现 |
| **集成性** | Client 可展示给用户/写入日志系统 | 仅 Host 进程可见 |
| **可靠性** | 协议级保证 | 取决于进程管理 |

### 面试题：MCP Logging 和 stderr 日志有什么区别？

**标准回答：**

> MCP Logging 是**协议级别的结构化日志通道**，与 stderr 有本质区别：
> 1. **传输通道**：Logging 走 MCP 协议（stdout/HTTP），stderr 是 OS 标准错误流，与协议无关
> 2. **结构化**：Logging 有统一的 `level` + `logger` + `data` 格式，stderr 是自由文本
> 3. **级别控制**：Logging 支持 Client 通过 `logging/setLevel` 动态调整日志级别
> 4. **集成性**：Logging 消息可以被 Client 展示给用户或集成到日志系统，stderr 只能在进程输出中查看
>
> stderr 适合开发调试，MCP Logging 适合生产环境的可观测性。

---

## 8.9 MCP Elicitation（用户交互） ★★★

### 什么是 Elicitation

Elicitation 是 MCP 的一项较新能力（2025 年引入），允许 **MCP Server 在执行工具过程中向用户请求额外信息**。

### 与其他机制的区别

| 机制 | 发起方 | 接收方 | 用途 |
|------|--------|--------|------|
| **tools/call** | Client | Server | LLM 调用工具 |
| **Sampling** | Server | LLM（通过 Client） | Server 请求 LLM 推理 |
| **Elicitation** | Server | 用户（通过 Client） | Server 请求用户输入 |
| **Logging** | Server | Client（日志系统） | 推送日志消息 |

### 典型场景

```text
场景 1：模糊工具参数
  LLM: tools/call "search_files" { query: "配置" }
  Server: "找到 500 个包含'配置'的文件，你是指：
          1. .env 环境配置
          2. nginx 配置
          3. 应用配置 (config.yaml)
          请选择：" ← Elicitation 向用户请求信息

场景 2：确认敏感操作
  LLM: tools/call "delete_file" { path: "/important/data.csv" }
  Server: "即将永久删除 /important/data.csv，确认？" ← Elicitation 请求用户确认
```

### 工作流程

```text
Client                                    Server
  │                                          │
  │  1. tools/call（触发 Server 需要更多信息） │
  │─────────────────────────────────────────►│
  │                                          │
  │  2. Server 发送 elicitation 请求          │
  │◄─────────────────────────────────────────│
  │  {                                       │
  │    "method": "elicitation/create",       │
  │    "params": {                           │
  │      "message": "请选择配置类型：",        │
  │      "requestedSchema": {                │
  │        "type": "object",                 │
  │        "properties": {                   │
  │          "configType": {                 │
  │            "type": "string",             │
  │            "enum": ["env", "nginx", "app"]│
  │          }                               │
  │        }                                 │
  │      }                                   │
  │    }                                     │
  │  }                                       │
  │                                          │
  │  3. Client 展示选择界面给用户              │
  │     用户选择 "env"                         │
  │                                          │
  │  4. Client 返回用户选择                    │
  │─────────────────────────────────────────►│
  │  { "result": { "configType": "env" } }   │
  │                                          │
  │  5. Server 基于用户输入继续执行            │
  │     返回最终结果                           │
```

### 面试题：MCP Elicitation 和 Sampling 的区别？

**标准回答：**

> Elicitation 和 Sampling 都是 Server 主动发起的请求，但目标不同：
> - **Elicitation**：Server → **用户**，请求用户做选择或确认（如"请选择配置类型"、"确认删除？"）
> - **Sampling**：Server → **LLM**，请求 LLM 做推理和生成（如"请总结以下数据"）
>
> Elicitation 解决的是"工具执行中需要人类判断"的问题，Sampling 解决的是"工具执行中需要 AI 推理"的问题。

---

# 第九章 MCP Server 开发 ★★★★★

## 9.1 MCP Server 核心结构

```text
┌─────────────────────────────────┐
│         MCP Server               │
│                                   │
│  ┌─────────────────────────────┐ │
│  │       Protocol Layer        │ │
│  │  - JSON-RPC 解析/序列化      │ │
│  │  - Transport 处理            │ │
│  │  - 生命周期管理              │ │
│  └─────────────┬───────────────┘ │
│                │                 │
│  ┌─────────────┴───────────────┐ │
│  │        Handler Layer        │ │
│  │                              │ │
│  │  ┌──────────┐ ┌──────────┐  │ │
│  │  │  Tools   │ │Resources │  │ │
│  │  │  Handler │ │ Handler  │  │ │
│  │  └──────────┘ └──────────┘  │ │
│  │  ┌──────────┐               │ │
│  │  │ Prompts  │               │ │
│  │  │ Handler  │               │ │
│  │  └──────────┘               │ │
│  └─────────────┬───────────────┘ │
│                │                 │
│  ┌─────────────┴───────────────┐ │
│  │       Business Logic        │ │
│  │  - 数据库操作                │ │
│  │  - API 调用                 │ │
│  │  - 文件处理                 │ │
│  └─────────────────────────────┘ │
└─────────────────────────────────┘
```

---

## 9.2 Python MCP Server 开发 ★★★★★

### 使用 FastMCP（推荐）

FastMCP 是 MCP 官方 Python SDK 提供的高层封装，极大简化了 Server 开发。

#### 安装

```bash
pip install mcp
```

#### 最小示例

```python
from mcp.server.fastmcp import FastMCP

# 创建 MCP Server
mcp = FastMCP("My Weather Server")

# 注册 Tool
@mcp.tool()
def get_weather(city: str, unit: str = "celsius") -> str:
    """获取指定城市的当前天气信息。

    Args:
        city: 城市名称，如 'Beijing' 或 '北京'
        unit: 温度单位，celsius（摄氏度）或 fahrenheit（华氏度）
    """
    # 实际业务逻辑：调用天气 API
    return f"{city}当前温度：25°C，晴"

# 注册 Resource
@mcp.resource("file://documents/{name}")
def get_document(name: str) -> str:
    """获取文档内容"""
    # 从文件或数据库读取
    return f"Document: {name}"

# 注册 Prompt
@mcp.prompt()
def code_review(language: str = "python") -> str:
    """代码审查 Prompt 模板"""
    return f"""你是一名资深 {language} 专家。
请审查以下代码，关注：
1. 正确性
2. 安全性
3. 性能
4. 可读性"""

# 启动 Server（STDIO 模式）
if __name__ == "__main__":
    mcp.run()
```

#### 复杂 Tool 示例

```python
from typing import Annotated, Optional
from mcp.server.fastmcp import FastMCP
import httpx

mcp = FastMCP("GitHub Tools")

@mcp.tool()
async def search_github_repos(
    query: Annotated[str, "搜索关键词，如 'mcp server'"],
    language: Annotated[Optional[str], "编程语言过滤，如 'python', 'go'"] = None,
    limit: Annotated[int, "返回结果数，最大 10"] = 5,
) -> str:
    """在 GitHub 上搜索仓库。

    适用于查找开源项目、代码示例和 MCP Server 实现。
    注意：结果基于 GitHub 的搜索排名。
    """
    if limit > 10:
        limit = 10

    # 构建搜索 URL
    url = f"https://api.github.com/search/repositories"
    params = {
        "q": query,
        "per_page": limit,
        "sort": "stars",
        "order": "desc"
    }
    if language:
        params["q"] += f" language:{language}"

    async with httpx.AsyncClient() as client:
        response = await client.get(
            url,
            params=params,
            headers={
                "Accept": "application/vnd.github.v3+json",
                "User-Agent": "mcp-github-server"
            }
        )
        response.raise_for_status()
        data = response.json()

    # 格式化返回结果
    results = []
    for repo in data.get("items", []):
        results.append(
            f"📦 {repo['full_name']}\n"
            f"   ⭐ {repo['stargazers_count']} | "
            f"   {repo['description'] or '无描述'}\n"
            f"   🔗 {repo['html_url']}"
        )

    return "\n\n".join(results)
```

#### Resource 开发示例

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Database Server")

@mcp.resource("mysql://{database}/{table}")
async def get_table_data(database: str, table: str) -> str:
    """读取数据库表数据"""
    # 连接数据库，查询数据
    return f"Table {database}.{table} data..."

@mcp.resource("mysql://{database}/{table}/schema")
async def get_table_schema(database: str, table: str) -> str:
    """读取数据库表结构"""
    return f"Schema for {database}.{table}..."

# 注册资源列表回调
@mcp.resource("mysql://databases")
async def list_databases() -> str:
    """列出所有数据库"""
    return "databases: production, staging, development"
```

---

## 9.3 Golang MCP Server 开发 ★★★★★

### 使用 Go SDK

Go 语言 MCP Server 开发更贴近底层，需要手动处理 JSON-RPC 通信。

#### 基础架构

```go
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "os"

    "github.com/modelcontextprotocol/go-sdk/mcp"
)

func main() {
    // 1. 创建 Server
    server := mcp.NewServer(
        &mcp.Implementation{
            Name:    "my-go-mcp-server",
            Version: "1.0.0",
        },
        nil, // options
    )

    // 2. 注册 Tools
    registerTools(server)

    // 3. 注册 Resources
    registerResources(server)

    // 4. 启动 Server（STDIO 模式）
    if err := server.Run(context.Background(), os.Stdin, os.Stdout); err != nil {
        fmt.Fprintf(os.Stderr, "Server error: %v\n", err)
        os.Exit(1)
    }
}
```

#### 注册 Tool

```go
func registerTools(server *mcp.Server) {
    // 定义 Tool
    weatherTool := mcp.Tool{
        Name:        "get_weather",
        Description: "获取指定城市的当前天气信息",
        InputSchema: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "city": map[string]any{
                    "type":        "string",
                    "description": "城市名称",
                },
                "unit": map[string]any{
                    "type":        "string",
                    "enum":        []string{"celsius", "fahrenheit"},
                    "description": "温度单位",
                    "default":     "celsius",
                },
            },
            "required": []string{"city"},
        },
    }

    // 注册 Tool + 处理函数
    server.AddTool(weatherTool, handleGetWeather)
}

// Tool 处理函数
func handleGetWeather(ctx context.Context, args map[string]any) (*mcp.CallToolResult, error) {
    city, _ := args["city"].(string)
    unit, _ := args["unit"].(string)
    if unit == "" {
        unit = "celsius"
    }

    // 业务逻辑：调用天气 API
    result := fmt.Sprintf("%s当前温度：25°C，晴（单位：%s）", city, unit)

    return &mcp.CallToolResult{
        Content: []any{
            map[string]any{
                "type": "text",
                "text": result,
            },
        },
    }, nil
}
```

#### 注册 Resource

```go
func registerResources(server *mcp.Server) {
    // 静态资源
    server.AddResource(
        mcp.Resource{
            URI:         "file:///config/settings.json",
            Name:        "Settings",
            Description: "应用配置文件",
            MimeType:    "application/json",
        },
        handleReadSettings,
    )

    // 资源模板（动态 URI）
    server.AddResourceTemplate(
        mcp.ResourceTemplate{
            URITemplate: "file:///logs/{date}.log",
            Name:        "Log File",
            Description: "按日期读取日志文件",
        },
        handleReadLog,
    )
}

func handleReadSettings(ctx context.Context, uri string) (*mcp.ReadResourceResult, error) {
    // 读取并返回配置
    settings := `{"theme": "dark", "language": "zh-CN"}`
    return &mcp.ReadResourceResult{
        Contents: []any{
            map[string]any{
                "uri":      uri,
                "mimeType": "application/json",
                "text":     settings,
            },
        },
    }, nil
}

func handleReadLog(ctx context.Context, uri string) (*mcp.ReadResourceResult, error) {
    // 从 URI 中提取参数
    // uri = "file:///logs/2026-06-06.log"
    // 读取对应日期的日志
    return &mcp.ReadResourceResult{
        Contents: []any{
            map[string]any{
                "uri":      uri,
                "mimeType": "text/plain",
                "text":     "[2026-06-06 10:00:00] INFO Server started",
            },
        },
    }, nil
}
```

#### 完整 Tool 示例：MySQL MCP Server

```go
package main

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "os"

    _ "github.com/go-sql-driver/mysql"
    "github.com/modelcontextprotocol/go-sdk/mcp"
)

type MySQLMCPServer struct {
    db *sql.DB
}

func NewMySQLMCPServer(dsn string) (*MySQLMCPServer, error) {
    db, err := sql.Open("mysql", dsn)
    if err != nil {
        return nil, err
    }
    return &MySQLMCPServer{db: db}, nil
}

func (s *MySQLMCPServer) RegisterTools(server *mcp.Server) {
    // Tool: 执行查询
    server.AddTool(mcp.Tool{
        Name:        "query_mysql",
        Description: "执行 MySQL 查询（仅支持 SELECT）",
        InputSchema: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "sql": map[string]any{
                    "type":        "string",
                    "description": "SQL 查询语句（仅 SELECT）",
                },
                "limit": map[string]any{
                    "type":        "integer",
                    "description": "最大返回行数，默认 100",
                    "default":     100,
                },
            },
            "required": []string{"sql"},
        },
    }, s.handleQuery)

    // Tool: 列出表
    server.AddTool(mcp.Tool{
        Name:        "list_tables",
        Description: "列出数据库中的所有表",
        InputSchema: map[string]any{
            "type":       "object",
            "properties": map[string]any{},
        },
    }, s.handleListTables)

    // Tool: 描述表结构
    server.AddTool(mcp.Tool{
        Name:        "describe_table",
        Description: "获取表的结构信息",
        InputSchema: map[string]any{
            "type": "object",
            "properties": map[string]any{
                "table": map[string]any{
                    "type":        "string",
                    "description": "表名",
                },
            },
            "required": []string{"table"},
        },
    }, s.handleDescribeTable)
}

func (s *MySQLMCPServer) handleQuery(ctx context.Context, args map[string]any) (*mcp.CallToolResult, error) {
    sqlQuery := args["sql"].(string)
    limit := 100
    if l, ok := args["limit"].(float64); ok {
        limit = int(l)
    }

    // 安全：仅允许 SELECT
    // 生产环境应使用 SQL 解析器做更严格的校验

    rows, err := s.db.QueryContext(ctx, fmt.Sprintf("%s LIMIT %d", sqlQuery, limit))
    if err != nil {
        return &mcp.CallToolResult{
            IsError: true,
            Content: []any{
                map[string]any{
                    "type": "text",
                    "text": fmt.Sprintf("查询失败: %v", err),
                },
            },
        }, nil
    }
    defer rows.Close()

    // 格式化结果
    columns, _ := rows.Columns()
    var results []map[string]any

    for rows.Next() {
        values := make([]any, len(columns))
        valuePtrs := make([]any, len(columns))
        for i := range columns {
            valuePtrs[i] = &values[i]
        }
        rows.Scan(valuePtrs...)

        row := make(map[string]any)
        for i, col := range columns {
            row[col] = fmt.Sprintf("%v", values[i])
        }
        results = append(results, row)
    }

    resultJSON, _ := json.MarshalIndent(results, "", "  ")
    return &mcp.CallToolResult{
        Content: []any{
            map[string]any{
                "type": "text",
                "text": string(resultJSON),
            },
        },
    }, nil
}

func (s *MySQLMCPServer) handleListTables(ctx context.Context, args map[string]any) (*mcp.CallToolResult, error) {
    rows, err := s.db.QueryContext(ctx, "SHOW TABLES")
    if err != nil {
        return errorResult(err), nil
    }
    defer rows.Close()

    var tables []string
    for rows.Next() {
        var table string
        rows.Scan(&table)
        tables = append(tables, table)
    }

    result, _ := json.Marshal(tables)
    return textResult(string(result)), nil
}

func (s *MySQLMCPServer) handleDescribeTable(ctx context.Context, args map[string]any) (*mcp.CallToolResult, error) {
    table := args["table"].(string)
    rows, err := s.db.QueryContext(ctx, fmt.Sprintf("DESCRIBE %s", table))
    if err != nil {
        return errorResult(err), nil
    }
    defer rows.Close()

    // 格式化输出
    // ...
    return textResult("table schema"), nil
}

// 辅助函数：创建文本结果
func textResult(text string) *mcp.CallToolResult {
    return &mcp.CallToolResult{
        Content: []any{
            map[string]any{
                "type": "text",
                "text": text,
            },
        },
    }
}

// 辅助函数：创建错误结果
func errorResult(err error) *mcp.CallToolResult {
    return &mcp.CallToolResult{
        IsError: true,
        Content: []any{
            map[string]any{
                "type": "text",
                "text": fmt.Sprintf("执行失败: %v", err),
            },
        },
    }
}
```

---

## 9.4 TypeScript MCP Server 开发 ★★★★★

### 使用 TypeScript SDK（官方第一方 SDK）

TypeScript/Node.js 是 MCP 的**官方第一方 SDK**，拥有最完善的生态和最活跃的社区。

#### 安装

```bash
npm install @modelcontextprotocol/sdk
# 或
yarn add @modelcontextprotocol/sdk
```

#### 最小示例（STDIO Transport）

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

// 1. 创建 Server
const server = new McpServer({
  name: "my-weather-server",
  version: "1.0.0",
});

// 2. 注册 Tool（使用 Zod schema 定义参数）
server.registerTool(
  "get_weather",
  {
    description: "获取指定城市的当前天气信息。返回温度、湿度、天气状况。适用于查询实时天气。",
    inputSchema: {
      city: z.string().describe("城市名称，如 'Beijing' 或 '北京'"),
      unit: z.enum(["celsius", "fahrenheit"]).default("celsius").describe("温度单位"),
    },
  },
  async ({ city, unit }) => {
    // 实际业务逻辑：调用天气 API
    const result = `${city}当前温度：25°C，晴（${unit}）`;
    return {
      content: [{ type: "text", text: result }],
    };
  }
);

// 3. 注册 Resource
server.registerResource(
  "file:///config/settings",
  {
    name: "Settings",
    description: "应用配置文件",
    mimeType: "application/json",
  },
  async (uri) => {
    return {
      contents: [{
        uri: uri.href,
        mimeType: "application/json",
        text: '{"theme": "dark", "language": "zh-CN"}',
      }],
    };
  }
);

// 4. 注册 Prompt
server.registerPrompt(
  "code_review",
  {
    description: "代码审查提示词模板",
    argsSchema: {
      language: z.string().default("typescript").describe("编程语言"),
    },
  },
  async ({ language }) => {
    return {
      messages: [{
        role: "user",
        content: {
          type: "text",
          text: `你是一名资深 ${language} 专家。请审查以下代码...`,
        },
      }],
    };
  }
);

// 5. 启动 Server（STDIO）
const transport = new StdioServerTransport();
await server.connect(transport);
```

#### 复杂 Tool 示例：GitHub Search

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({
  name: "github-tools",
  version: "1.0.0",
});

server.registerTool(
  "search_github_repos",
  {
    description:
      "在 GitHub 上搜索仓库。适用于查找开源项目、代码示例和 MCP Server 实现。" +
      "注意：结果基于 GitHub 的搜索排名，最多返回 10 条。",
    inputSchema: {
      query: z.string().describe("搜索关键词，如 'mcp server'"),
      language: z.string().optional().describe("编程语言过滤，如 'python', 'go'"),
      limit: z.number().min(1).max(10).default(5).describe("返回结果数"),
    },
  },
  async ({ query, language, limit }) => {
    let q = query;
    if (language) q += ` language:${language}`;

    const response = await fetch(
      `https://api.github.com/search/repositories?q=${encodeURIComponent(q)}&per_page=${limit}&sort=stars&order=desc`,
      {
        headers: {
          Accept: "application/vnd.github.v3+json",
          "User-Agent": "mcp-github-server",
        },
      }
    );

    const data = await response.json();
    const results = data.items.map(
      (repo: any) =>
        `📦 ${repo.full_name}\n   ⭐ ${repo.stargazers_count} | ${repo.description || "无描述"}\n   🔗 ${repo.html_url}`
    );

    return {
      content: [{ type: "text", text: results.join("\n\n") }],
    };
  }
);

export default server;
```

#### Streamable HTTP Transport 启动

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const app = express();
app.use(express.json());

const server = new McpServer({
  name: "remote-mcp-server",
  version: "1.0.0",
});

// 注册 tools...
server.registerTool(/* ... */);

// 创建 Streamable HTTP Transport
app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined, // 无状态模式
  });
  await server.connect(transport);
  await transport.handleRequest(req, res);
});

app.listen(3000, () => {
  console.log("MCP Server running on http://localhost:3000/mcp");
});
```

### TypeScript SDK vs Python SDK 对比

| 维度 | TypeScript SDK | Python SDK (FastMCP) |
|------|---------------|---------------------|
| **生态地位** | 官方第一方 SDK | 官方推荐 Python SDK |
| **参数定义** | Zod schema（Type-safe） | Python type hints + Annotated |
| **类型安全** | 编译时 + 运行时双重检查 | 运行时（Pydantic 可选） |
| **社区规模** | 最大（npm 生态） | 较大（pip 生态） |
| **Streamable HTTP** | 原生支持 | 原生支持 |
| **学习曲线** | 中等（需了解 Zod） | 低（装饰器最简洁） |
| **适合场景** | Node.js 后端、全栈项目 | Python AI/ML 项目 |

---

### 面试题：如何开发一个 MCP Server？

**标准回答：**

> 开发 MCP Server 的核心步骤：
>
> 1. **选择 SDK**：TypeScript 用官方 SDK（第一方，生态最完整）、Python 用 FastMCP（高层封装，最简洁）、Go 用 go-sdk（底层 JSON-RPC）
> 2. **创建 Server 实例**：设置名称、版本
> 3. **注册 Tools**：定义 name + description + inputSchema，绑定处理函数
> 4. **注册 Resources**（可选）：定义 URI + 读取逻辑
> 5. **注册 Prompts**（可选）：定义参数化提示词模板
> 6. **启动 Server**：选择 Transport（STDIO 用于本地，Streamable HTTP 用于远程）
>
> **关键原则**：
> - description 要足够详细，LLM 才能正确选择和使用工具
> - Tool 必须做参数校验和安全检查
> - 错误信息要清晰，帮助 LLM 调整调用策略
> - 每个 Tool 职责单一，原子化操作

---

# 第十章 MCP 与 Function Calling ★★★★★

## 10.1 什么是 Function Calling

**Function Calling** 是 LLM 厂商（OpenAI、Anthropic 等）各自提供的**让模型调用外部函数**的能力。

### OpenAI Function Calling 示例

```python
import openai

response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=[{
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取城市天气",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名称"},
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
                },
                "required": ["city"]
            }
        }
    }]
)
```

### Anthropic Tool Use 示例

```python
import anthropic

response = anthropic.Anthropic().messages.create(
    model="claude-sonnet-4-6",
    messages=[{"role": "user", "content": "北京天气怎么样？"}],
    tools=[{
        "name": "get_weather",
        "description": "获取城市天气",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["city"]
        }
    }]
)
```

## 10.2 Function Calling vs MCP

### 核心区别

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| **定位** | LLM 厂商的 API 特性 | 开放协议标准 |
| **范围** | 单次 API 调用的工具定义 | 完整的工具生态体系 |
| **工具定义** | 每次请求时传入 | Server 注册，一次注册持续可用 |
| **工具管理** | 应用层管理（硬编码或配置） | 协议层管理（标准化） |
| **发现机制** | 无（开发者手动拼接） | `tools/list` 自动发现 |
| **传输层** | 随 LLM API 请求传输 | 独立传输层（STDIO/HTTP/SSE） |
| **工具实现** | 应用自己实现 | Server 独立进程，与 LLM 解耦 |
| **跨模型复用** | 每个 LLM 厂商格式不同 | 一次开发，所有 LLM 通用 |
| **生态** | 无 | 丰富的社区 Server 生态 |
| **安全控制** | 应用层自控 | 协议级权限和沙箱 |

### 关系图

```text
┌─────────────────────────────────────────────────┐
│                  MCP（协议标准层）                 │
│                                                   │
│   定义：如何发现工具、如何调用工具、如何管理生命周期  │
│                                                   │
│   ┌─────────────────────────────────────────┐    │
│   │       Function Calling（厂商实现层）      │    │
│   │                                         │    │
│   │  OpenAI Function Calling                │    │
│   │  Anthropic Tool Use                     │    │
│   │  Google Gemini Function Calling         │    │
│   │                                         │    │
│   │  各厂商按自己的方式实现工具的传递和调用   │    │
│   └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
```

**MCP 是「工具如何定义和通信」的标准，Function Calling 是「LLM 如何调用工具」的机制。两者是互补关系，不是替代关系。**

---

### 面试题：MCP 和 Function Calling 的区别？

**标准回答：**

> MCP 和 Function Calling 解决不同层面的问题：
>
> **Function Calling**：
> - 是 LLM 厂商提供的 API 级功能
> - 定义「模型如何调用函数」
> - 工具定义随每次 API 请求传递
> - 格式各厂商不同（OpenAI vs Anthropic vs Google）
> - 应用层自己管理工具的实现和生命周期
>
> **MCP**：
> - 是开放的协议标准
> - 定义「工具如何被发现、描述、调用和管理」
> - 工具由 Server 注册，Client 自动发现
> - 统一标准，跨 LLM 通用
> - 提供完整的生命周期管理（initialize → tools/list → tools/call → shutdown）
> - 有独立的传输层和社区 Server 生态
>
> **一句话总结**：Function Calling 是「LLM 怎么伸手拿工具」，MCP 是「工具有一个标准接口」。两者互补：MCP Server 提供工具，LLM 通过 Function Calling 机制来调用这些工具。

---

### 面试回答模板（MCP vs Function Calling）

**30 秒版：**

> "Function Calling 是 LLM 厂商各自的 API 特性——OpenAI 有 OpenAI 的格式，Anthropic 有 Anthropic 的格式。MCP 是跨厂商的开放协议标准。FC 解决的是'模型怎么调用函数'，MCP 解决的是'工具怎么标准化地描述、发现和管理'。两者的关系是：MCP Server 定义和托管工具，LLM 通过各自的 Function Calling 机制去调用这些工具。它们是互补关系，不是替代关系。"

**1 分钟版：**

> "理解 MCP 和 Function Calling 区别的关键是分清层次。Function Calling 是 LLM API 层的能力——每次请求时，开发者把工具定义作为参数传给 API，模型返回一个 tool_call 表示想调用哪个工具。这带来三个问题：每次请求都要传工具定义（浪费 token）、格式因厂商而异（锁定效应）、工具实现由应用层自己管理（无标准化）。
>
> MCP 在更高的协议层解决这些问题：工具由 MCP Server 独立托管，Client 通过 `tools/list` 一次发现、持续使用；工具定义格式是统一的 JSON Schema，与厂商无关；工具的生命周期（initialize → tools/list → tools/call → shutdown）由协议管理。当 LLM 通过 Function Calling 决定调用某个工具时，实际执行是通过 MCP 协议的 `tools/call` 完成的。"

**3 分钟版：**

> "Let me draw the complete picture. In a modern AI application, when a user asks '查一下北京天气', here's what happens end-to-end:
>
> **Step 1 — Tool Discovery (MCP)**: During initialization, the MCP Client calls `tools/list` and receives all available tools from the MCP Server — `get_weather`, `write_file`, etc.
>
> **Step 2 — Tool Selection (Function Calling)**: The application sends the user's message to the LLM API, along with the tool definitions. The LLM uses its built-in Function Calling capability to decide: 'I should call get_weather with city=Beijing'. It returns a tool_call in the response.
>
> **Step 3 — Tool Execution (MCP)**: The application takes the tool_call and sends it as `tools/call` through MCP to the weather Server. The Server executes the actual API call and returns the result.
>
> **Step 4 — Response Generation (Function Calling)**: The application sends the tool result back to the LLM API, which generates the final user-facing response.
>
> So MCP handles the 'how tools are described, discovered, and executed' part, while Function Calling handles the 'how the LLM decides to use tools' part. They work together: MCP standardizes the tool ecosystem, FC provides the LLM's reasoning capability to pick the right tool."

### 为什么 MCP 不能替代 Function Calling？

1. **Function Calling 是 LLM 内置能力**：模型在训练时已经学习了如何根据工具描述选择合适的工具——这是模型推理能力的一部分，不是协议层能替代的
2. **MCP 不定义"推理"**：MCP 定义的是工具的通信和管理标准，不涉及 LLM 如何决定调用哪个工具
3. **两者在不同层次工作**：MCP = 传输层 + 工具管理，FC = 推理层（模型选择工具的能力）

### MCP Sampling：反向的 Function Calling

值得特别指出的是，MCP 还定义了 **Sampling**（第 8.6 节）——这是 **Server → LLM** 的反向调用，与 Function Calling（LLM → Server）形成互补：

```text
Function Calling（LLM → Server）:
  LLM 决定调用 get_weather → tools/call → Server 执行

MCP Sampling（Server → LLM）:
  Server 需要 LLM 帮忙 → sampling/createMessage → Client 调用 LLM → 返回结果
```

这意味着 MCP 不仅支持 "LLM 调用工具"，还支持 "工具调用 LLM"，实现了双向的 AI-工具交互。

### 实战中的集成模式

```text
┌─────────────────────────────────────────────┐
│          AI 应用架构（MCP + FC 集成）          │
│                                               │
│  User: "查天气并保存"                         │
│       │                                       │
│       ▼                                       │
│  ┌─────────────┐                             │
│  │   LLM API   │ ← Function Calling          │
│  │  (推理层)    │   模型决定调用 get_weather   │
│  └──────┬──────┘                             │
│         │ tool_call: get_weather(city=Beijing) │
│         ▼                                     │
│  ┌─────────────┐                             │
│  │  MCP Client │ ← MCP 协议层                │
│  │  (执行层)    │   tools/call → Server       │
│  └──────┬──────┘                             │
│         │                                     │
│         ▼                                     │
│  ┌─────────────┐                             │
│  │  MCP Server │ ← 工具实现层                │
│  │  天气服务    │   调用天气 API              │
│  └─────────────┘                             │
└─────────────────────────────────────────────┘
```

---

# 第十一章 MCP 与 Agent ★★★★★

## 11.1 Agent 核心架构

```
┌─────────────────────────────────────────────────┐
│                    Agent                         │
│                                                  │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐  │
│  │  Memory  │    │ Planner  │    │ Executor │  │
│  │  记忆系统 │◄───│  规划器   │───►│  执行器   │  │
│  └──────────┘    └──────────┘    └──────────┘  │
│       │               │               │         │
│       │          ┌────┴────┐          │         │
│       │          │   LLM   │          │         │
│       │          └─────────┘          │         │
│       │                               │         │
│       │                    ┌──────────┴──────┐  │
│       │                    │    MCP Client   │  │
│       │                    └────────┬────────┘  │
│       │                             │           │
│       │                    ┌────────┴────────┐  │
│       │                    │   MCP Server    │  │
│       │                    │  ┌────────────┐ │  │
│       │                    │  │   Tools    │ │  │
│       │                    │  │   Resources│ │  │
│       │                    │  └────────────┘ │  │
│       │                    └─────────────────┘  │
└─────────────────────────────────────────────────┘
```

## 11.2 MCP 在 Agent 中的角色

### Agent 的核心循环

```text
┌───────────────────────────────────────────────┐
│            Agent 执行循环（简化）              │
│                                                │
│  1. 接收任务："帮我分析这个项目的代码质量"      │
│        │                                       │
│  2. Planner（规划）：                          │
│     - 需要列出文件 → list_files tool           │
│     - 需要读取代码 → read_file tool            │
│     - 需要运行检查 → run_linter tool           │
│     - 需要生成报告 → write_file tool           │
│        │                                       │
│  3. Executor（执行）：                         │
│     - Step 1: 调用 list_files                 │
│     - Step 2: 调用 read_file (× N 个文件)     │
│     - Step 3: 调用 run_linter                 │
│     - Step 4: 调用 write_file                 │
│        │                                       │
│        │  每一步都通过 MCP 协议完成：           │
│        │  tools/call → Server 执行 → 返回结果  │
│        │                                       │
│  4. Memory（记忆）：                            │
│     - 记录中间结果                             │
│     - 缓存文件列表供后续步骤使用                │
│        │                                       │
│  5. 返回最终结果给用户                         │
└───────────────────────────────────────────────┘
```

### MCP 是 Agent 的「手脚」

```text
Agent（大脑）
   │
   │ 思考 + 决策
   │
   ▼
MCP（神经系统）
   │
   │ 标准化通信
   │
   ▼
Tools（手脚）
   │
   │ 实际执行操作
   │
   ▼
外部世界（文件、数据库、API）
```

---

### 面试题：MCP 与 Agent 是什么关系？

**标准回答：**

> MCP 是 Agent 的 **「标准化工具层」**：
>
> 1. **Agent 是大脑**：负责理解任务、制定计划、做出决策（Planning + Reasoning）
> 2. **MCP 是神经系统**：负责在大脑（Agent）和手脚（Tools）之间传递标准化的信号
> 3. **Tools 是手脚**：负责实际执行操作（读文件、查数据库、调 API）
>
> **没有 MCP 时**：Agent 需要为每个工具单独实现通信逻辑，工具不可复用
> **有了 MCP 时**：Agent 只需实现 MCP Client，就能使用生态中所有的 MCP Server 提供的工具
>
> MCP 让 Agent 从「专用工具集成」升级为「通用工具生态」，就像智能手机从「内置功能」升级为「应用商店」。

---

# 第十二章 MCP 与 RAG ★★★★

## 12.1 RAG 基本原理

**RAG（Retrieval Augmented Generation，检索增强生成）**：

```text
用户问题："公司最新的休假政策是什么？"
         │
         ▼
┌────────────────────┐
│  1. Embedding      │  将问题向量化
│     (嵌入)          │
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│  2. Retrieval      │  在知识库中检索相关文档
│     (检索)          │  Top-K 相似文档
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│  3. Augmented      │  将检索结果注入 Prompt
│     (增强)          │  Prompt = 问题 + 检索到的文档
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│  4. Generation     │  LLM 基于增强后的 Prompt
│     (生成)          │  生成准确答案
└────────────────────┘
```

## 12.2 MCP vs RAG

### 本质区别

| 维度 | RAG | MCP Resource |
|------|-----|-------------|
| **数据获取方式** | 语义检索（向量相似度） | 精确访问（URI 定位） |
| **适用场景** | "与这个问题相关的文档" | "读取这个具体的文件/表" |
| **数据组织** | 向量数据库 | 结构化 URI |
| **访问模式** | 模糊匹配 → Top-K | 精确寻址 → 直接读取 |
| **实时性** | 取决于索引更新频率 | 实时（直接读取） |
| **互补性** | 发现相关数据 | 精确获取指定数据 |

### MCP 能否替代 RAG？

**不能替代，但可以互补：**

```text
                    ┌─────────────┐
                    │   用户问题    │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
     ┌──────────────┐          ┌──────────────┐
     │  RAG（发现）  │          │ MCP（获取）   │
     │              │          │              │
     │ "哪些文档    │          │ "读取        │
     │  与问题相关？"│          │  这个文件"   │
     └──────┬───────┘          └──────┬───────┘
            │                         │
            ▼                         ▼
     ┌──────────────┐          ┌──────────────┐
     │  向量数据库   │          │  MCP Server  │
     │  (知识库)     │          │  (文件/DB/API)│
     └──────────────┘          └──────────────┘
```

**最佳实践：RAG 发现 + MCP 获取**

```text
用户："帮我总结公司 Q1 的销售数据"
    │
    ▼
RAG 发现阶段：
  检索 → Top-3 相关文档：
    1. file:///reports/q1-sales-2026.xlsx
    2. file:///reports/q1-summary.md
    3. mysql://sales/q1_orders
    │
    ▼
MCP 获取阶段：
  精确读取上述 3 个资源的内容
    │
    ▼
LLM 整合分析 → 生成总结报告
```

---

### 面试题：MCP 能否替代 RAG？

**标准回答：**

> MCP 和 RAG 解决不同的问题，**不是替代关系，而是互补关系**：
>
> **RAG 擅长**：
> - "帮我找相关的文档"（模糊检索）
> - 处理大量非结构化文本
> - 基于语义相似度的内容发现
>
> **MCP 擅长**：
> - "读取这个具体的文件"（精确访问）
> - 结构化数据的实时获取
> - 工具调用和操作执行
>
> **最佳实践**：
> - 用 RAG 做「发现」：从海量数据中找到相关的上下文
> - 用 MCP 做「获取」：精确读取已定位的数据，以及执行有副作用的操作
> - 两者组合：RAG 发现 → MCP 精确获取 → LLM 生成

---

### 面试回答模板（MCP vs RAG）

**30 秒版：**

> "RAG 和 MCP 解决不同层次的问题。RAG 做的是语义检索——'哪些文档和这个问题相关？'用的是向量相似度。MCP 做的是精确访问——'读取这个具体的文件或数据库表'，用的是 URI 定位。两者互补：先用 RAG 从海量知识中发现相关文档，再用 MCP 精确读取这些文档的内容。一个负责发现，一个负责获取。"

**1 分钟版：**

> "RAG 和 MCP Resource 的本质区别在于访问模式。RAG 是模糊匹配——把问题转成向量，在向量数据库里找 Top-K 相似文档。它的优势是处理大量非结构化文本，劣势是实时性取决于索引更新频率。MCP Resource 是精确访问——通过 URI 直接定位资源，实时读取。它的优势是实时、精确，劣势是需要你知道资源的确切位置。
>
> 实际项目中，两者配合使用效果最好。比如：用户问'公司 Q1 销售数据'，RAG 先检索到相关的 3 个资源（报表文件、数据库表、会议纪要），然后 MCP 精确读取这 3 个资源的内容，LLM 基于这些内容生成总结。这个模式叫'RAG 发现 + MCP 获取'。"

**3 分钟版：**

> "Let me explain the complementary relationship between RAG and MCP with a concrete architecture.
>
> **The Knowledge Access Spectrum**: On one end, you have 'I don't know what I'm looking for' — this is RAG's domain. On the other end, you have 'I know exactly what I want' — this is MCP's domain. Most real-world AI applications need BOTH.
>
> **RAG (Retrieval-Augmented Generation)**: Works in the embedding space. Documents are chunked, embedded into vectors, and stored in a vector database (like Pinecone, Milvus, or pgvector). When a user asks a question, we embed the question, find the most similar document chunks via cosine similarity, and inject them into the LLM's prompt. This is great for 'discovery' — finding relevant information across thousands of documents.
>
> **MCP Resource**: Works in the URI space. Each resource has a unique identifier like `file:///reports/q1.xlsx` or `mysql://sales/q1_orders`. The LLM can directly request these resources via `resources/read`, getting real-time, precise data. This is great for 'access' — reading specific, known resources.
>
> **The Integration Pattern**: In an enterprise setting, you'd combine them:
> 1. User asks 'Summarize Q1 sales performance'
> 2. RAG searches the knowledge base → returns references to 3 resources
> 3. MCP reads those 3 resources with `resources/read`
> 4. LLM synthesizes the data into a summary
>
> **When to use which**:
> - RAG alone: FAQ bots, documentation search, 'find me something about X'
> - MCP alone: 'Read /path/to/config.yaml', 'Query the users table'
> - RAG + MCP: Enterprise AI assistants that need both discovery and precise data access

---

# 第十三章 MCP 安全机制 ★★★★

## 13.1 权限控制

### 多层权限体系

```text
┌─────────────────────────────────────────┐
│           MCP 安全体系                    │
│                                           │
│  ┌─────────────────────────────────────┐│
│  │  1. 用户授权层（User Consent）       ││
│  │  - 首次使用 Tool 时弹窗确认          ││
│  │  - 用户可拒绝、允许一次、始终允许    ││
│  └─────────────────────────────────────┘│
│                    │                      │
│  ┌─────────────────────────────────────┐│
│  │  2. Tool 级别权限（Tool Permissions）││
│  │  - 每个 Tool 可设置独立的权限        ││
│  │  - 只读 / 读写 / 禁止                ││
│  └─────────────────────────────────────┘│
│                    │                      │
│  ┌─────────────────────────────────────┐│
│  │  3. Resource 级别权限               ││
│  │  - URI 级别的访问控制               ││
│  │  - 文件路径白名单 / 黑名单          ││
│  └─────────────────────────────────────┘│
│                    │                      │
│  ┌─────────────────────────────────────┐│
│  │  4. 传输层安全                       ││
│  │  - STDIO：进程隔离                   ││
│  │  - HTTP：TLS 加密                   ││
│  └─────────────────────────────────────┘│
└─────────────────────────────────────────┘
```

### 权限配置文件示例

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"],
      "permissions": {
        "tools": {
          "read_file": "allow",
          "write_file": "confirm",
          "delete_file": "deny"
        },
        "resources": {
          "file:///allowed/path/**": "allow",
          "file:///**": "deny"
        }
      }
    }
  }
}
```

---

## 13.2 沙箱机制

### 进程隔离（STDIO Transport）

- MCP Server 作为独立子进程运行
- 操作系统级别的进程隔离
- Server 崩溃不影响 Host
- 内存隔离

### 容器隔离（生产环境）

```yaml
# Docker Compose 示例
services:
  mcp-github:
    image: mcp/github-server
    read_only: true       # 只读文件系统
    no_new_privileges: true  # 禁止提权
    cap_drop:
      - ALL               # 移除所有 capabilities
    networks:
      - mcp-network       # 隔离网络
```

---

## 13.3 数据隔离

### 多租户隔离策略

```text
┌─────────────────────────────────────────┐
│           MCP Gateway                    │
│                                          │
│  Tenant A ──► MCP Server A (namespace A) │
│  Tenant B ──► MCP Server B (namespace B) │
│  Tenant C ──► MCP Server C (namespace C) │
│                                          │
│  每个 Tenant 独立的：                    │
│  - 数据库连接                             │
│  - 文件路径                               │
│  - API Token                            │
│  - 配置                                  │
└─────────────────────────────────────────┘
```

---

## 13.4 OAuth 2.0 授权（MCP Authorization） ★★★★★

### 背景

MCP 在 2025 年引入了 **MCP Authorization 规范**，基于 **OAuth 2.0** 实现远程 MCP Server 的认证和授权。这是远程 MCP Server 的**标准安全机制**。

### 为什么需要 OAuth 2.0？

```text
场景：企业部署了远程 MCP Server（如 GitHub MCP Server）

问题 1：谁可以访问这个 Server？
  → 需要身份认证（Authentication）

问题 2：访问者可以做什么？
  → 需要权限授权（Authorization）

问题 3：如何安全传递凭证？
  → 不能把 API Key 明文写在配置里

解决方案：OAuth 2.0 授权框架
  → Access Token + Refresh Token 的标准流程
```

### OAuth 2.0 授权流程

```text
┌──────────┐          ┌──────────────┐          ┌──────────┐
│   MCP    │          │  Auth Server │          │   MCP    │
│  Client  │          │(授权服务器)    │          │  Server  │
└────┬─────┘          └──────┬───────┘          └────┬─────┘
     │                       │                       │
     │ 1. 发起授权请求        │                       │
     │──────────────────────►│                       │
     │                       │                       │
     │ 2. 用户登录+授权确认    │                       │
     │◄─────────────────────►│                       │
     │   (展示授权页面给用户)   │                       │
     │                       │                       │
     │ 3. 返回 Authorization Code                   │
     │◄──────────────────────│                       │
     │                       │                       │
     │ 4. 交换 Access Token   │                       │
     │──────────────────────►│                       │
     │                       │                       │
     │ 5. 返回 Access Token + Refresh Token          │
     │◄──────────────────────│                       │
     │                       │                       │
     │ 6. 携带 Access Token 访问 MCP Server          │
     │──────────────────────────────────────────────►│
     │                                               │
     │ 7. MCP Server 验证 Token                      │
     │    → 比对 Auth Server 公钥 / 自校验 JWT        │
     │                                               │
     │ 8. 正常 MCP 通信（Token 有效期内）              │
     │◄─────────────────────────────────────────────►│
     │                                               │
     │ 9. Token 过期后，用 Refresh Token 刷新         │
     │──────────────────────►│                       │
     │◄──────────────────────│                       │
```

### MCP Server 端 OAuth 配置

在 MCP Server 的元数据中声明 OAuth 支持：

```json
{
  "method": "initialize",
  "result": {
    "capabilities": {
      "tools": {},
      "resources": {}
    },
    "serverInfo": {
      "name": "github-mcp-server",
      "version": "1.0.0"
    },
    "auth": {
      "type": "oauth2",
      "authorizationUrl": "https://auth.example.com/authorize",
      "tokenUrl": "https://auth.example.com/token",
      "scopes": ["read:repos", "write:issues"],
      "clientRegistrationUrl": "https://auth.example.com/register"
    }
  }
}
```

### 安全最佳实践

| 实践 | 说明 |
|------|------|
| **Token 最小权限** | 每个 MCP Server 只请求完成功能所需的最小 scope |
| **Token 短期化** | Access Token 有效期 15-60 分钟，过期后用 Refresh Token 刷新 |
| **Token 绑定** | Token 绑定到特定 Client + Server 组合，防止滥用 |
| **PKCE 防护** | 使用 PKCE（Proof Key for Code Exchange）防止授权码拦截 |
| **Token 撤销** | 支持 Token 撤销端点，用户可随时取消授权 |
| **审计日志** | 记录所有 Token 的颁发、刷新、撤销操作 |

### 面试题：MCP 远程 Server 如何实现认证？

**标准回答：**

> MCP 远程 Server 的认证基于 OAuth 2.0 授权框架（MCP Authorization 规范）：
> 1. **授权流程**：Client 引导用户到 Auth Server 完成登录和授权确认，获取 Authorization Code
> 2. **Token 交换**：Client 用 Code 交换 Access Token + Refresh Token
> 3. **携带 Token 通信**：Client 在每次 MCP 请求中携带 Access Token（通常通过 HTTP `Authorization: Bearer <token>` 头）
> 4. **Token 验证**：MCP Server 验证 Token（JWT 自校验或向 Auth Server 校验）
> 5. **Token 刷新**：Access Token 过期后，Client 用 Refresh Token 自动刷新
>
> **关键安全原则**：最小权限 scope、Token 短期化、PKCE 防护、支持 Token 撤销。

---

### 面试题：MCP 如何保证安全？

**标准回答：**

> MCP 通过多层安全机制保证安全：
>
> 1. **OAuth 2.0 授权**（远程 Server）：基于 MCP Authorization 规范，Access Token + PKCE + scope 最小权限
> 2. **用户授权层**：首次调用工具时弹窗确认，用户可选择允许/拒绝/始终允许
> 3. **Tool 级权限**：每个 Tool 可独立设置权限（只读/读写/禁止）
> 4. **Resource 级权限**：URI 模式匹配的访问控制（白名单/黑名单）
> 5. **Roots 边界**：Client 声明允许访问的根路径，Server 不得越权
> 6. **进程隔离**：STDIO 模式下 Server 作为子进程运行，操作系统级隔离
> 7. **传输层安全**：远程通信使用 TLS 加密
> 8. **容器沙箱**：生产环境通过 Docker 等容器技术做额外隔离
> 9. **参数校验**：Server 端必须校验所有输入参数，防止注入攻击
> 10. **最小权限原则**：每个 MCP Server 只授予完成任务所需的最小权限
> 6. **容器沙箱**：生产环境通过 Docker 等容器技术做额外隔离
> 7. **参数校验**：Server 端必须校验所有输入参数，防止注入攻击
> 8. **最小权限原则**：每个 MCP Server 只授予完成任务所需的最小权限

---

# 第十四章 MCP 生态 ★★★★

## 14.1 官方/社区 MCP Server 概览

### 文件系统类

| MCP Server | 功能 | 安装 |
|-----------|------|------|
| **Filesystem** | 文件读写、目录操作 | `npx @modelcontextprotocol/server-filesystem` |
| **Google Drive** | Google Drive 文件操作 | `npx @modelcontextprotocol/server-gdrive` |
| **Everything** | 跨文件搜索（Windows） | 社区版 |

### 数据库类

| MCP Server | 功能 | 安装 |
|-----------|------|------|
| **PostgreSQL** | PostgreSQL 查询与管理 | `npx @modelcontextprotocol/server-postgres` |
| **SQLite** | SQLite 数据库操作 | `npx @modelcontextprotocol/server-sqlite` |
| **Redis** | Redis 缓存操作 | 社区版 / Python SDK |
| **MySQL** | MySQL 查询 | 社区版 / Golang SDK |
| **MongoDB** | MongoDB 操作 | 社区版 |

### 开发工具类

| MCP Server | 功能 | 安装 |
|-----------|------|------|
| **GitHub** | 仓库管理、Issue、PR | `npx @modelcontextprotocol/server-github` |
| **Git** | 本地 Git 操作 | `npx @modelcontextprotocol/server-git` |
| **GitLab** | GitLab 仓库操作 | 社区版 |
| **Docker** | 容器管理 | 社区版 |
| **Kubernetes** | K8s 集群管理 | 社区版 |

### 搜索与浏览器

| MCP Server | 功能 | 安装 |
|-----------|------|------|
| **Brave Search** | 网络搜索 | `npx @modelcontextprotocol/server-brave-search` |
| **Puppeteer** | 浏览器自动化 | `npx @modelcontextprotocol/server-puppeteer` |
| **Fetch** | HTTP 请求/网页抓取 | `npx @modelcontextprotocol/server-fetch` |
| **Tavily** | AI 搜索 | 社区版 |

### 协作工具类

| MCP Server | 功能 | 安装 |
|-----------|------|------|
| **Slack** | 消息发送、频道管理 | `npx @modelcontextprotocol/server-slack` |
| **Jira** | Issue 管理 | 社区版 |
| **Notion** | 知识库操作 | 社区版 |
| **Confluence** | 文档管理 | 社区版 |
| **Linear** | 项目管理 | 社区版 |

### 设计与创作

| MCP Server | 功能 | 安装 |
|-----------|------|------|
| **Figma** | 设计稿读取 | 社区版 |
| **Canva** | 设计创作 | 社区版 |
| **Image Generation** | 图片生成 | 社区版 |

### AI / LLM 工具

| MCP Server | 功能 | 安装 |
|-----------|------|------|
| **Memory** | 持久记忆存储 | `npx @modelcontextprotocol/server-memory` |
| **Sequential Thinking** | 链式推理 | `npx @modelcontextprotocol/server-sequential-thinking` |
| **Context7** | 库文档查询 | 社区版 |
| **EverArt** | AI 图片生成 | 社区版 |

---

### 面试题：常见 MCP Server 有哪些？

**标准回答：**

> 常见 MCP Server 分为几大类：
> 1. **文件系统**：Filesystem（本地文件）、Google Drive（云盘）
> 2. **数据库**：PostgreSQL、SQLite、Redis、MySQL
> 3. **开发工具**：GitHub、Git、GitLab、Docker、Kubernetes
> 4. **搜索与浏览器**：Brave Search、Puppeteer（浏览器控制）、Fetch（HTTP 请求）
> 5. **协作工具**：Slack、Jira、Notion、Confluence
> 6. **设计工具**：Figma
> 7. **AI 增强**：Memory（记忆存储）、Sequential Thinking（链式推理）
>
> 这些 Server 可以通过 npm（`npx`）、pip 或直接编译二进制运行。社区生态丰富且持续增长。

---

# 第十五章 MCP 高级设计 ★★★★★

## 15.1 Tool Router（工具路由）★★★★★

### 问题

当有多个 MCP Server 时，如何将 LLM 的工具调用请求路由到正确的 Server？

### 为什么需要 Tool Router

```text
在没有 Tool Router 时：
  LLM → 需要知道每个 Tool 属于哪个 Server → 耦合严重
  Server 变更（迁移/下线/替换）→ 需要重新配置 LLM

有了 Tool Router 后：
  LLM → 只关心 Tool 的名字 → tools/call "get_weather"
  Router → 查路由表 → 转发到正确的 Server
  Server 变更 → 只需更新路由表，LLM 无感知
```

### 路由策略

| 策略 | 原理 | 适用场景 |
|------|------|---------|
| **静态路由** | 预配置 Tool → Server 映射表 | 工具和 Server 关系稳定 |
| **前缀路由** | `github.*` → GitHub Server, `db.*` → DB Server | 工具按命名空间组织 |
| **动态路由** | 从服务注册中心实时获取映射 | Server 动态扩缩容 |
| **灰度路由** | 按百分比/用户分流到不同 Server 版本 | 工具升级、A/B 测试 |
| **故障转移路由** | 主 Server 不可用时自动切换到备用 Server | 高可用场景 |

### 路由架构图

```text
                    ┌────────────┐
                    │    LLM     │
                    │ Tool Call  │
                    │ get_weather│
                    └─────┬──────┘
                          │
                          ▼
                  ┌───────────────┐
                  │  Tool Router  │
                  │  (路由表)      │
                  │               │
                  │ get_weather   │
                  │   → Server A  │
                  │               │
                  │ query_mysql   │
                  │   → Server B  │
                  │               │
                  │ search_web    │
                  │   → Server C  │
                  └───┬───┬───┬───┘
                      │   │   │
               ┌──────┘   │   └──────┐
               ▼          ▼          ▼
          Server A   Server B   Server C
```

### Go 实现示例

```go
// Tool Router 完整实现
type MCPClient interface {
    CallTool(ctx context.Context, name string, args map[string]any) (*ToolResult, error)
    ListTools(ctx context.Context) ([]ToolDef, error)
}

type ToolRouter struct {
    routes   map[string]MCPClient   // tool name → client
    mu       sync.RWMutex
    fallback MCPClient               // 默认路由（可选）
}

func NewToolRouter() *ToolRouter {
    return &ToolRouter{
        routes: make(map[string]MCPClient),
    }
}

// 注册静态路由
func (r *ToolRouter) RegisterRoute(toolPattern string, client MCPClient) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.routes[toolPattern] = client
}

// 智能路由：支持前缀匹配
func (r *ToolRouter) Route(toolName string) (MCPClient, error) {
    r.mu.RLock()
    defer r.mu.RUnlock()

    // 1. 精确匹配
    if client, ok := r.routes[toolName]; ok {
        return client, nil
    }

    // 2. 前缀匹配（如 "github.*" → GitHub Server）
    for pattern, client := range r.routes {
        if strings.HasSuffix(pattern, ".*") {
            prefix := strings.TrimSuffix(pattern, ".*")
            if strings.HasPrefix(toolName, prefix) {
                return client, nil
            }
        }
    }

    // 3. Fallback
    if r.fallback != nil {
        return r.fallback, nil
    }

    return nil, fmt.Errorf("no route found for tool: %s", toolName)
}

// 从服务发现动态更新路由
func (r *ToolRouter) SyncFromRegistry(registry ServiceRegistry) {
    services, _ := registry.ListServices()
    for _, svc := range services {
        for _, tool := range svc.Tools {
            client := NewMCPClient(svc.Endpoint)
            r.RegisterRoute(tool, client)
        }
    }
}
```

### 面试题：Tool Router 的设计要点？

**标准回答：**

> Tool Router 的核心设计要点：
> 1. **路由表**：维护 Tool Name → MCP Client/Server 的映射关系
> 2. **匹配策略**：精确匹配 → 前缀匹配（namespace routing）→ 默认路由
> 3. **动态更新**：配合服务注册中心（etcd/Consul），Server 上下线时自动更新路由表
> 4. **高可用**：支持主备路由、健康检查、故障摘除
> 5. **可观测**：路由决策日志、调用链路追踪

---

## 15.2 多 MCP Server 架构

### 企业级多 Server 拓扑

```text
                         ┌───────────────┐
                         │  MCP Gateway  │
                         │  (统一入口)    │
                         └───────┬───────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │                     │                     │
           ▼                     ▼                     ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │ GitHub MCP   │    │ Database MCP │    │  Search MCP  │
   │              │    │              │    │              │
   │ - search_    │    │ - query_db   │    │ - web_search │
   │   repos      │    │ - list_      │    │ - fetch_url  │
   │ - create_    │    │   tables     │    │              │
   │   issue      │    │              │    │              │
   └──────────────┘    └──────────────┘    └──────────────┘
           │                     │                     │
           ▼                     ▼                     ▼
      GitHub API           MySQL/Redis           Search API
```

---

## 15.3 MCP Gateway（统一入口）★★★★★

### 为什么需要 Gateway？

1. **统一入口**：所有 LLM 请求通过一个端点访问所有工具
2. **负载均衡**：多个 Server 实例之间分发请求
3. **认证鉴权**：集中的 API Key / Token 管理
4. **限流控制**：防止某个 Tool 被过度调用
5. **监控日志**：统一的调用链路追踪和审计
6. **服务发现**：动态注册和发现 MCP Server
7. **协议转换**：新旧版本 MCP 协议兼容

### Gateway 架构

```text
                           ┌────────────┐
                           │    LLM     │
                           └─────┬──────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────┐
│                  MCP Gateway                        │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │            Auth / Rate Limit                 │   │
│  │         认证 & 限流层                         │   │
│  └─────────────────────┬───────────────────────┘   │
│                        │                            │
│  ┌─────────────────────┴───────────────────────┐   │
│  │              Tool Router                     │   │
│  │          工具路由 + 负载均衡                   │   │
│  └──────┬──────────┬──────────┬────────────────┘   │
│         │          │          │                     │
│  ┌──────┴──────────┴──────────┴────────────────┐   │
│  │          Monitoring / Logging               │   │
│  │        监控 + 审计 + 链路追踪                  │   │
│  └─────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────┘
         │          │          │
         ▼          ▼          ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │ Server │ │ Server │ │ Server │
    │   A    │ │   B    │ │   C    │
    └────────┘ └────────┘ └────────┘
```

---

## 15.4 MCP Proxy（代理层）★★★★

### 什么是 MCP Proxy

MCP Proxy 是位于 Client 和 Server 之间的**中间层**，拦截、修改、增强或审计所有的 MCP 通信。

```text
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  Client  │────►│  MCP Proxy   │────►│  Server  │
│          │◄────│  (中间层)     │◄────│          │
└──────────┘     └──────────────┘     └──────────┘
                       │
                       ├── 请求/响应拦截
                       ├── 参数改写/增强
                       ├── 结果缓存
                       ├── 安全审计
                       ├── 限流控制
                       └── 协议转换
```

### 应用场景详解

#### 1. 协议转换

将 MCP 协议桥接到企业内部的其他协议：

```text
MCP Client ──► MCP Proxy ──► gRPC Server（企业内部服务）
                           ──► REST API（遗留系统）
                           ──► GraphQL（数据聚合层）
```

**实现要点**：
- Proxy 接收 MCP 的 `tools/call` 请求
- 提取 `name` 和 `arguments`
- 转换为目标协议的请求格式（如 protobuf、REST）
- 将响应转换回 MCP 的 `content` 数组格式

#### 2. 安全审查

在代理层拦截和审计所有工具调用：

```go
// MCP Proxy 安全审查中间件
type AuditProxy struct {
    next   MCPClient
    logger AuditLogger
    rules  []SecurityRule
}

func (p *AuditProxy) CallTool(ctx context.Context, name string, args map[string]any) (*ToolResult, error) {
    // 1. 调用前检查
    for _, rule := range p.rules {
        if !rule.Allow(ctx, name, args) {
            return nil, fmt.Errorf("blocked by security rule: %s", rule.Name)
        }
    }

    // 2. 记录审计日志
    p.logger.Log(ctx, AuditEntry{
        Tool:      name,
        Args:      args,
        Timestamp: time.Now(),
        User:      ctx.Value("user"),
    })

    // 3. 执行调用
    result, err := p.next.CallTool(ctx, name, args)

    // 4. 记录结果（用于后续审计）
    p.logger.LogResult(ctx, result, err)

    return result, err
}
```

#### 3. 结果缓存

对确定性工具调用的结果进行缓存：

```text
第一次调用 get_weather(city="Beijing") → 调用真实 API → 结果存入缓存（TTL=5min）
第二次调用 get_weather(city="Beijing") → 缓存命中 → 直接返回

缓存策略：
├── 读操作（query_*, list_*, get_*）→ 默认缓存（可配置 TTL）
└── 写操作（create_*, update_*, delete_*）→ 不缓存 + 使失效相关缓存
```

#### 4. 请求改写 / 增强

自动补充或修改工具调用的参数：

```text
原始请求: tools/call { name: "query_db", arguments: { sql: "SELECT * FROM users" } }
                         │
                         ▼
                    ┌──────────┐
                    │  Proxy   │
                    │ 改写规则 │
                    └────┬─────┘
                         │
                         ▼
改写后: { name: "query_db", arguments: { sql: "SELECT * FROM users LIMIT 100" } }
        自动添加: LIMIT 100（防止大数据量查询）
        自动添加: 租户隔离条件（多租户场景）
        自动注入: 当前用户上下文
```

#### 5. 故障注入（混沌工程）

测试 Agent 在工具故障时的容错能力：

```text
Proxy 配置：
├── 10% 概率返回超时错误（测试重试逻辑）
├── 5% 概率返回错误结果（测试错误处理）
├── 增加 200ms 延迟（测试 LLM 的耐心）
└── 1% 概率返回空结果（测试边界处理）
```

### Proxy 链模式

多个 Proxy 可以串联形成处理链：

```text
Client → [Auth Proxy] → [Cache Proxy] → [Audit Proxy] → [Rewrite Proxy] → Server
         认证鉴权        缓存加速        审计日志         参数增强
```

### 面试题：MCP Proxy 有哪些应用场景？

**标准回答：**

> MCP Proxy 作为 Client-Server 之间的中间层，有五种核心应用场景：
> 1. **协议转换**：将 MCP 协议桥接到企业内部的 gRPC/REST/GraphQL 服务
> 2. **安全审查**：在代理层拦截所有 `tools/call`，执行安全规则检查并记录审计日志
> 3. **结果缓存**：对读操作自动缓存（TTL 可配），写操作使失效缓存，减少重复 API 调用
> 4. **请求改写**：自动补充参数（如 LIMIT、租户隔离条件、用户上下文），增强 LLM 调用的安全性
> 5. **故障注入**：模拟超时、错误、延迟，测试 Agent 的容错和重试能力
>
> 多个 Proxy 可以链式组合（Auth → Cache → Audit → Rewrite → Server），形成灵活的处理管道。

---

### 面试题：如何管理多个 MCP Server？

**标准回答：**

> 企业级多 MCP Server 管理通常采用 **Gateway + Router** 架构：
>
> 1. **MCP Gateway**：统一入口，负责认证、限流、监控、日志。所有 LLM 请求通过 Gateway 访问后端 Server
> 2. **Tool Router**：维护 Tool → Server 的路由表。LLM 请求的 `tools/call` 根据 `name` 路由到对应的 Server
> 3. **服务发现**：支持 MCP Server 的动态注册和注销（基于 etcd/Consul/Nacos）
> 4. **负载均衡**：同一 Tool 可部署多个 Server 实例，Gateway 做负载分发
> 5. **协议兼容**：支持不同版本 MCP 协议的兼容转换
> 6. **可观测性**：集中收集日志、指标、链路追踪（OpenTelemetry）
>
> 这样的架构保证了 MCP 生态的高可用、可扩展和安全性。

---

# 第十六章 MCP 面试场景题 ★★★★★

> 以下每个场景题都提供了**可直接用于面试口述的完整回答**。面试时根据时间限制，选择核心要点回答即可。

---

## 场景 1：请介绍一下什么是 MCP？

**面试官意图**：考察对 MCP 的基本理解，是否能用简洁的语言说清楚。

**标准回答（45秒）：**

> "MCP 全称是 Model Context Protocol，中文叫模型上下文协议。它是 Anthropic 在 2024 年开源的一套标准协议，主要解决一个问题：**让 AI 模型能够安全、标准化地访问外部工具和数据源**。
>
> 可以把它理解为 AI 世界的 USB 标准——就像 USB 让任何外设都能插到任何电脑上使用，MCP 让任何工具（比如数据库、GitHub、文件系统）写好一次，就能被 Claude、GPT、Gemini 等所有支持 MCP 的 AI 应用使用。
>
> 在 MCP 出现之前，每个 AI 平台都要为每个工具单独开发插件——N 个模型 × M 个工具 = N×M 种组合。MCP 把这个变成了 N+M：工具开发者只需写一个 MCP Server，所有 AI 应用都能用。"

---

## 场景 2：MCP 为什么重要？为什么最近这么火？

**面试官意图**：考察对行业趋势的理解和 MCP 的实际价值认知。

**标准回答（45秒）：**

> "MCP 的重要性可以从三个层面理解：
>
> **第一，标准化层面**：在 MCP 之前，AI 工具集成是碎片化的——OpenAI 有自己的插件体系，Anthropic 有 Tool Use，每家格式都不一样。MCP 提供了第一个跨厂商的通用标准，结束了一个一个适配的局面。
>
> **第二，生态层面**：标准协议催生了生态。现在社区有几百个现成的 MCP Server——从数据库到 GitHub 到 Slack——你不需要从零开发，拿来就用。这极大降低了 AI Agent 开发的门槛。
>
> **第三，工业趋势层面**：目前主流的 AI 开发平台——Claude Desktop、Cursor、Windsurf 都在原生支持 MCP，OpenAI 也在跟进。它正在成为 AI Agent 生态的事实标准。"

---

## 场景 3：MCP 和 Function Calling 有什么区别？能互相替代吗？

**面试官意图**：考察对两个易混淆概念的精确区分能力。

**标准回答（1分钟）：**

> "这是面试中很常见的问题。核心结论是：**两者解决不同层面的问题，是互补关系，不能互相替代。**
>
> **Function Calling** 是 LLM 厂商的 API 特性，解决的是'模型如何决定调用哪个函数'——它定义的是推理层的机制。每次 API 请求时把工具定义传给模型，模型返回一个 tool_call。
>
> **MCP** 是开放协议，解决的是'工具如何标准化地描述、发现、执行和管理'——它定义的是协议层的标准。工具在 MCP Server 中注册，Client 通过 `tools/list` 自动发现，通过 `tools/call` 统一调用。
>
> **实际运行中两者配合**：MCP Server 托管工具，LLM 通过 Function Calling 机制选择工具，然后通过 MCP 协议执行工具。打个比方：Function Calling 是大脑决定'我要拿杯子'，MCP 是神经系统把信号传到手上。一个负责决策，一个负责执行。"
>
> **关键区别速记**：
> - FC：厂商 API 特性，格式各异，随请求传递工具定义，应用层管理实现
> - MCP：开放协议，统一标准，工具独立托管，协议层管理生命周期

---

## 场景 4：Tool 和 Resource 有什么区别？什么时候用哪个？

**面试官意图**：考察对 MCP 核心概念的精确理解。

**标准回答（30秒）：**

> "Tool 和 Resource 的区别一句话：**Tool 是动作（做什么），Resource 是数据（有什么）。**
>
> - **Tool** 代表可执行的操作——比如 `query_mysql`、`create_issue`、`search_web`。它可能有副作用（写入、删除），通过 `tools/call` 调用。
> - **Resource** 代表可读取的数据——比如 `file://readme.md`、`mysql://users/table`。它是只读的，通过 `resources/read` 访问。
>
> **类比**：Tool = RPC 函数调用（执行操作），Resource = REST GET（读取数据）。
>
> **选型**：需要执行操作（查数据库、发消息、创建文件）→ 用 Tool。需要获取上下文数据（读文件、读表结构）→ 用 Resource。一个 MCP Server 可以同时提供两者。"

---

## 场景 5：MCP 的连接初始化流程是怎样的？

**面试官意图**：考察对协议生命周期的理解，以及能力协商机制。

**标准回答（45秒）：**

> "MCP 的初始化是一个三步握手过程，核心是**能力协商**：
>
> **第一步**：Client 发送 `initialize` 请求，携带三个信息——协议版本（如 `2024-11-05`）、Client 支持的能力（如 `roots`、`sampling`）、Client 自身信息（名称 + 版本）。
>
> **第二步**：Server 返回 `initialize` 响应，确认协议版本、声明 Server 支持的能力（如 `tools`、`resources`、`prompts`、`logging`）、返回 Server 信息。
>
> **第三步**：Server 发送 `notifications/initialized` 通知，表示初始化完成。此时连接就绪。
>
> **关键点是第二步的能力协商**：Client 说'我能提供这些'，Server 说'我能做这些'，双方取交集决定后续可用的功能。协商完成后，Client 通常会立即调用 `tools/list` 获取所有可用工具列表，接下来就进入正常的工具调用循环。"

---

## 场景 6：STDIO 和 Streamable HTTP 传输有什么区别？怎么选？

**面试官意图**：考察对传输层选型的工程判断能力。

**标准回答（45秒）：**

> "MCP 有四种 Transport，但实际就两个选择：
>
> **STDIO Transport**：Host 启动 Server 作为子进程，通过 stdin/stdout 通信。优点是简单、零网络配置、进程级安全隔离。缺点是只能本地通信。适合**本地 IDE 工具和个人 Agent**。
>
> **Streamable HTTP Transport**：这是 2025 年推出的推荐远程方案。单一 POST 端点，在同一个 HTTP 连接上实现双向流式通信。相比 SSE（双通道复杂、代理不友好，已不推荐新项目使用），Streamable HTTP 代理友好、支持断点续传、全平台兼容。适合**生产环境和远程服务**。
>
> **选型原则**：本地用 STDIO，远程用 Streamable HTTP。如果是简单的不需要流式推送的场景，可以降级用普通 HTTP Transport。"

---

## 场景 7：如果要你开发一个 MCP Server，你会怎么做？

**面试官意图**：考察实际开发能力和对 SDK 的熟悉程度。

**标准回答（1分钟）：**

> "开发 MCP Server 我遵循六个步骤：
>
> **第一步，选语言和 SDK**：Python 用 FastMCP（最简单，装饰器注册），Go 用 go-sdk（更底层，需要手动处理 JSON-RPC），TypeScript 用官方 SDK。
>
> **第二步，创建 Server 实例**：设置 `name` 和 `version`，这在 initialize 时会告知 Client。
>
> **第三步，注册 Tools**：这是核心工作。每个 Tool 需要 name（snake_case，动词开头）、description（越详细越好，这是 LLM 选择工具的唯一依据）、inputSchema（JSON Schema 定义参数类型、枚举、默认值、必填项）、annotations（可选，元数据标注如 readOnlyHint/destructiveHint）、handler（实际的业务逻辑函数）。
>
> **第四步，注册 Resources**（可选）：定义可读取的数据资源，用 URI 标识。
>
> **第五步，注册 Prompts**（可选）：预定义提示词模板。
>
> **第六步，启动**：选择 Transport 启动 Server。开发调试用 STDIO，部署用 Streamable HTTP。
>
> **关键原则**：description 要写详细（场景、限制、副作用），Tool 要原子化（一个 Tool 只做一件事），错误信息要清晰（帮助 LLM 调整策略），必须做参数校验和安全检查。"

---

## 场景 8：如何实现一个 MySQL MCP Server？需要注意什么？

**面试官意图**：考察具体场景的落地能力和安全意识。

**标准回答（1分钟）：**

> "实现 MySQL MCP Server 需要关注三个层面：
>
> **Tool 设计**：至少三个核心 Tool——`query_mysql`（执行 SELECT 查询）、`list_tables`（列出所有表）、`describe_table`（获取表结构）。还可以加上 `list_databases`、`explain_query` 等辅助 Tool。
>
> **安全设计（最重要）**：
> - 只允许 SELECT，拒绝 INSERT/UPDATE/DELETE/DROP——可以在 SQL 解析层校验
> - 强制 LIMIT（默认 100 行），防止 LLM 生成 `SELECT * FROM huge_table` 打爆内存
> - 如果必须支持写操作，使用**参数化查询**防止 SQL 注入
> - 设置查询超时（如 30 秒），防止慢查询
> - 使用只读数据库账户，最小权限原则
>
> **Resource 设计**：注册 `mysql://{database}/{table}` 作为只读资源，LLM 可以直接读取表数据作为上下文，不需要通过 Tool 调用。
>
> **返回格式**：返回结构化 JSON，包含 columns 和 rows，方便 LLM 理解数据。错误时返回清晰的错误信息（而不是 raw SQL error），帮助 LLM 修正后续查询。"

---

## 场景 9：MCP 在 Agent 架构中处于什么位置？如何支持 Agent 运行？

**面试官意图**：考察对 Agent 架构的系统理解。

**标准回答（1分钟）：**

> "MCP 在 Agent 架构中是**标准化的工具执行层**。我用一个三层模型来解释：
>
> **大脑层（Agent）**：负责理解任务、制定计划、做出决策。使用 LLM 作为推理引擎，通过 ReAct 循环（Thought → Action → Observation）逐步执行任务。
>
> **神经系统（MCP）**：负责在大脑和手脚之间传递标准化信号。Agent 决定调用 `get_weather` 后，通过 MCP Client 发送 `tools/call` 请求，MCP Server 执行后返回结果。所有工具调用都走统一的 JSON-RPC 协议。
>
> **手脚层（Tools）**：实际执行操作的 MCP Server——访问数据库、读写文件、调用 API。
>
> **实际运行流程**：用户说'分析这个项目'→ Agent Plan：需要 list_files、read_file、run_linter → 每一步通过 MCP tools/call 执行 → 结果返回 Agent → Agent 综合判断下一步 → 直到任务完成。
>
> **核心价值**：MCP 让 Agent 从硬编码的工具集成变成了可插拔的工具生态。一个 Agent 只需实现 MCP Client，就能使用生态中所有 MCP Server 的工具。"

---

## 场景 10：企业中有几十个 MCP Server，如何统一管理？

**面试官意图**：考察企业级架构设计能力。

**标准回答（1分钟）：**

> "企业级多 MCP Server 管理采用 **Gateway + Router** 架构，分四层：
>
> **第一层，MCP Gateway（统一入口）**：所有 Agent 请求通过 Gateway 进入。Gateway 负责：认证鉴权（API Key / OAuth）、限流控制（防止某个 Tool 被滥用）、统一的审计日志和链路追踪。
>
> **第二层，Tool Router（工具路由）**：维护 Tool Name → MCP Server 的路由表。LLM 调用 `get_weather`，Router 查找路由表转发到 Weather Server。支持精确匹配、前缀匹配（namespace routing）、和默认路由。
>
> **第三层，服务发现（动态注册）**：MCP Server 启动时向注册中心（etcd/Consul/Nacos）注册自己的 Tool 列表和端点地址。Router 从注册中心拉取路由表。Server 下线时自动摘除。
>
> **第四层，运维保障**：每个 Server 多副本部署 + 健康检查 + 自动摘除故障实例。通过 Prometheus + Grafana 监控调用量、延迟、错误率，通过 OpenTelemetry 做全链路追踪。
>
> **多租户**：通过命名空间隔离不同租户的 MCP Server 实例，每个租户有独立的数据库连接和 API Token。"

---

## 场景 11：MCP 系统如何保证安全性？

**面试官意图**：考察安全意识和纵深防御思维。

**标准回答（45秒）：**

> "MCP 安全是一个纵深防御体系，至少八层：
>
> **第一层，OAuth 2.0 认证**（远程 Server）：基于 MCP Authorization 规范，Access Token + PKCE + scope 最小权限。所有远程 MCP Server 应强制 OAuth。
>
> **第二层，用户授权**：首次调用 Tool 弹窗确认，用户可选择允许/拒绝/始终允许。这是最直接的人机交互防线。
>
> **第三层，Tool 级权限**：每个 Tool 独立配置——只读、可写需确认、禁止。比如 `read_file` 允许，`delete_file` 需确认，`execute_shell` 禁止。
>
> **第四层，Resource 级权限**：URI 级别的白名单/黑名单。`file:///workspace/**` 允许，`file:///etc/**`（系统配置）禁止。
>
> **第五层，Roots 边界**：Client 通过 Roots 机制声明 Server 允许访问的目录边界，Server 不得越权访问。
>
> **第六层，进程/容器隔离**：STDIO 模式天然进程隔离；生产环境用 Docker 容器做额外隔离——只读文件系统、禁止提权、独立网络栈。
>
> **第七层，传输安全**：远程通信强制 TLS 加密。
>
> **第八层，Server 端校验**：永远不信任 Client 传来的参数——SQL 注入防护、路径穿越检查、输入大小限制。
>
> **原则**：最小权限。每个 MCP Server 只授予完成任务所需的最小权限。"

---

## 场景 12：MCP 能不能替代 API Gateway？

**面试官意图**：考察对不同技术组件边界的清晰认知。

**标准回答（30秒）：**

> "不能。两者解决不同的问题。
>
> **API Gateway** 管理的是 REST/gRPC API——南北流量、认证、限流、路由、协议转换。面向的是传统微服务架构。
>
> **MCP** 管理的是 AI Agent 的工具调用和上下文访问——面向的是 LLM 驱动的智能体。
>
> **它们不是替代关系，但可以融合**：MCP Gateway 可以作为 API Gateway 的一个特殊模块，或者部署在 API Gateway 后面，由 API Gateway 做第一层认证和限流，MCP Gateway 专注于工具路由和 Agent 特定的逻辑。Kong、APISIX 这些 API Gateway 已经开始原生支持 MCP 协议了。"

---

## 场景 13：MCP 和 LangChain 是什么关系？

**面试官意图**：考察对 AI 开发生态中不同组件角色的理解。

**标准回答（45秒）：**

> "LangChain 和 MCP 是**不同层次**的工具，互补而非竞争。
>
> **LangChain 是 Agent 开发框架**：提供构建 AI 应用的脚手架——Chain、Agent、Memory、Callback 等。它帮你把 LLM 调用、工具使用、记忆管理串联起来。
>
> **MCP 是工具通信协议**：定义工具如何标准化地描述、发现和调用。它不关心你的 Agent 框架是什么。
>
> **两者集成方式**：LangChain Agent 通过 MCP Client 连接 MCP Server，把 MCP Tool 映射为 LangChain Tool。Agent 在 LangChain 框架内运行，但工具调用走 MCP 协议。
>
> **价值**：没有 MCP 时，LangChain 开发者需要自己实现每个工具的通信逻辑；有了 MCP，直接接入社区生态中的几百个现成工具。LangChain 负责 Agent 逻辑，MCP 负责工具生态。"

---

## 场景 14：MCP 和 RAG 有什么区别？能替代 RAG 吗？

**面试官意图**：考察对两种 AI 增强技术的理解和综合应用能力。

**标准回答（45秒）：**

> "不能替代，但可以很好地互补。
>
> **RAG 做的是'发现'**：用语义搜索在海量文档中找到相关的上下文内容，通过向量相似度匹配，返回 Top-K 结果。它的弱点是实时性差（依赖索引更新）和精确性不足。
>
> **MCP Resource 做的是'获取'**：通过 URI 精确访问具体资源，获取实时数据。它的弱点是无法做模糊发现——你必须知道资源的确切位置。
>
> **最佳组合模式**：RAG 发现 + MCP 获取。举例：用户问'Q1 销售数据'，RAG 先检索知识库，发现相关资源包括 `reports/q1.xlsx` 和 `mysql://sales/q1_orders`，然后 MCP 精确读取这些资源的内容，最后 LLM 综合生成答案。RAG 告诉你'去哪里找'，MCP 帮你'把东西拿出来'。"

---

## 场景 15：请设计一个企业级 MCP 平台架构

**面试官意图**：考察综合架构设计能力，是区分高级工程师的关键题。

**标准回答（2分钟）：**

> "企业级 MCP 平台架构我分五层来设计：
>
> **第一层，接入层 — MCP Gateway**：所有 Agent 请求的统一入口。核心功能：API Key / OAuth 认证、按 Tool 和租户的限流（如每个用户每分钟最多 60 次工具调用）、TLS 终止、请求日志。技术选型：可以用 Go 自研，也可以基于 APISIX/Kong 扩展。
>
> **第二层，路由层 — Tool Router**：维护 Tool → Server 的映射表。支持静态路由（预配置）和动态路由（从服务注册中心同步）。支持前缀匹配（`github.*` → GitHub Server）和灰度路由（A/B 测试不同版本的 Tool）。关键设计：路由表的原子更新，不能出现请求路由到一半路由表变了的情况。
>
> **第三层，代理层 — MCP Proxy Chain**（可选）：在 Router 和 Server 之间插入代理链——认证代理、缓存代理（读操作缓存 TTL）、审计代理（记录所有 tools/call）、改写代理（自动补充 LIMIT、租户隔离条件）。
>
> **第四层，服务层 — MCP Server 集群**：每个 MCP Server 独立部署为容器化服务（K8s Deployment），多副本保证高可用。支持水平扩缩（HPA 基于 CPU 和 QPS）。每个 Server 注册到服务发现中心（Consul/etcd）。
>
> **第五层，基础设施层**：
> - 服务发现（Consul/etcd）：Server 启动注册 Tool 列表和端点，下线自动摘除
> - 配置中心（Consul KV/Nacos）：Tool 权限配置、限流规则、路由表热更新
> - 可观测性：OpenTelemetry 全链路追踪 + Prometheus 指标 + Grafana 大盘 + ELK 日志
> - 多租户：通过 namespace 隔离，每个租户独立 DB 连接和 API Token
>
> **关键设计原则**：高可用（每个组件多副本）、安全性（纵深防御）、可观测性（每个 tools/call 可追踪）、可扩展（新增 Server 只需注册，Gateway 和 Router 无感知）。"

---

# 第十七章 MCP 日常开发使用指南 ★★★★★

> 本章面向日常开发场景，覆盖 Claude Desktop、VS Code/Cursor、命令行等环境中的 MCP 配置、调试和最佳实践。

---

## 17.1 Claude Desktop 中使用 MCP

### 配置文件位置

Claude Desktop 的 MCP 配置存储在 `claude_desktop_config.json`（旧版）或 `mcp.json`（新版）中：

| 平台 | 配置路径 |
|------|---------|
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` |
| **Linux** | `~/.config/Claude/claude_desktop_config.json` |

### 基础配置示例

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/username/projects"
      ]
    },
    "github": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    }
  }
}
```

### 配置字段说明

| 字段 | 必填 | 说明 |
|------|------|------|
| `command` | 是 | 启动 Server 的命令（`npx`、`python`、`uvx`、可执行文件路径） |
| `args` | 是 | 命令参数列表 |
| `env` | 否 | 环境变量（适合传递 API Key、Token 等敏感信息） |
| `disabled` | 否 | 设为 `true` 可临时禁用某个 Server |
| `timeout` | 否 | 启动超时（毫秒），默认 30000 |

### 多 Server 配置

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://user:pass@localhost:5432/mydb"
      }
    },
    "brave-search": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "BSA_xxxxxxxxxxxx"
      }
    },
    "memory": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"]
    }
  }
}
```

### Python MCP Server 配置

```json
{
  "mcpServers": {
    "my-python-server": {
      "command": "python",
      "args": ["-m", "my_mcp_server"]
    }
  }
}
```

或使用 `uvx`（推荐，自动管理虚拟环境）：

```json
{
  "mcpServers": {
    "my-python-server": {
      "command": "uvx",
      "args": ["my-mcp-server-package"]
    }
  }
}
```

### Go MCP Server 配置

```json
{
  "mcpServers": {
    "my-go-server": {
      "command": "/path/to/my-go-mcp-server"
    }
  }
}
```

---

## 17.2 VS Code / Cursor 中使用 MCP

### Cursor 配置

Cursor 原生支持 MCP，配置文件位于项目根目录的 `.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "${workspaceFolder}"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${env:GITHUB_TOKEN}"
      }
    }
  }
}
```

**Cursor 特有功能**：
- `${workspaceFolder}`：自动解析为当前工作区路径
- `${env:VARIABLE}`：从系统环境变量读取
- 支持项目级 `.cursor/mcp.json` 和用户级 `~/.cursor/mcp.json`

### VS Code + GitHub Copilot

VS Code 通过 GitHub Copilot 扩展支持 MCP（需 Copilot 订阅）：

```json
// .vscode/mcp.json（项目级）
{
  "servers": {
    "My-Tools": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@my-org/my-mcp-server"]
    }
  }
}
```

或在 VS Code 设置中配置（`settings.json`）：

```json
{
  "github.copilot.mcp.servers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/workspace"]
    }
  }
}
```

### Windsurf 配置

Windsurf 使用 `mcp.json` 配置：

```json
{
  "mcpServers": {
    "my-tools": {
      "command": "node",
      "args": ["./mcp-server/dist/index.js"]
    }
  }
}
```

---

## 17.3 MCP Inspector（调试工具）

### 安装和启动

MCP Inspector 是官方提供的可视化调试工具，用于测试和调试 MCP Server：

```bash
# 启动 Inspector（自动安装）
npx @modelcontextprotocol/inspector

# 或指定要调试的 Server
npx @modelcontextprotocol/inspector npx -y @modelcontextprotocol/server-filesystem /tmp/test
```

### Inspector 功能

```text
┌─────────────────────────────────────────────────┐
│              MCP Inspector                       │
│                                                  │
│  ┌──────────────────┐  ┌──────────────────────┐ │
│  │   左侧面板         │  │   右侧面板            │ │
│  │                  │  │                      │ │
│  │  📋 Server 信息   │  │  📨 JSON-RPC 消息    │ │
│  │  - 名称/版本      │  │  - Request/Response  │ │
│  │  - 能力列表       │  │  - 格式化 JSON       │ │
│  │                  │  │  - 消息时间线         │ │
│  │  🔧 Tools 列表    │  │                      │ │
│  │  - 点击即调用     │  │  🔍 消息详情         │ │
│  │  - 参数填写表单   │  │  - 完整 payload      │ │
│  │  - 查看返回结果   │  │  - 错误高亮          │ │
│  │                  │  │                      │ │
│  │  📁 Resources    │  │  📊 统计             │ │
│  │  📝 Prompts      │  │  - 调用次数          │ │
│  │                  │  │  - 平均延迟          │ │
│  └──────────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────┘
```

### Inspector 使用场景

| 场景 | 操作 |
|------|------|
| **测试 Tool 定义** | 查看 `tools/list` 返回的 Tool 列表，确认 name/description/inputSchema 正确 |
| **手动调用 Tool** | 在 Inspector 中填写参数并调用，查看原始 JSON-RPC 请求和响应 |
| **调试错误** | 查看 Server 返回的错误码和错误信息，排查问题 |
| **性能分析** | 查看每次 `tools/call` 的延迟，定位性能瓶颈 |
| **协议验证** | 确认 Server 的 JSON-RPC 消息格式是否符合 MCP 规范 |

### 调试工作流

```bash
# 1. 启动 Inspector 并附加到你的 MCP Server
npx @modelcontextprotocol/inspector python -m my_mcp_server

# 2. 浏览器打开 http://localhost:5173

# 3. 在 "Tools" 标签页中：
#    - 查看所有注册的 Tool
#    - 点击 Tool 名称展开参数表单
#    - 填写测试参数并点击 "Call Tool"
#    - 查看 JSON-RPC Request 和 Response

# 4. 如果出现错误：
#    - 切换到 "Messages" 标签页查看完整消息
#    - 检查 Request 的 method 和 params 是否正确
#    - 检查 Response 的 error code 定位问题
```

---

## 17.4 命令行中使用 MCP

### Claude CLI

```bash
# 启动 Claude CLI 并注册 MCP Server
claude --mcp-server filesystem -- npx -y @modelcontextprotocol/server-filesystem /workspace

# 同时注册多个 Server
claude \
  --mcp-server fs -- npx -y @modelcontextprotocol/server-filesystem /workspace \
  --mcp-server db -- npx -y @modelcontextprotocol/server-postgres
```

### Claude Code（VS Code 扩展）

Claude Code 支持通过 `.claude/mcp.json` 配置 MCP Server：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "."]
    }
  }
}
```

配置后，在 Claude Code 对话中可以直接调用 MCP 工具。

### 使用 curl 测试远程 MCP Server

对于 Streamable HTTP Transport 的 MCP Server：

```bash
# 1. 初始化连接
curl -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {"name": "curl-test", "version": "1.0.0"}
    }
  }'

# 2. 获取 Tool 列表
curl -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list"
  }'

# 3. 调用 Tool
curl -X POST http://localhost:3000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "get_weather",
      "arguments": {"city": "Beijing"}
    }
  }'
```

### 使用 STDIO Transport 测试

```bash
# 手动测试 STDIO Server（发送单条消息）
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}' | \
  python -m my_mcp_server

# 使用 printf 发送带 Content-Length 头的消息
printf 'Content-Length: 108\r\n\r\n{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' | \
  python -m my_mcp_server
```

---

## 17.5 日常开发工作流

### 开发 → 测试 → 部署流程

```text
┌─────────────────────────────────────────────────┐
│            MCP Server 开发工作流                  │
│                                                  │
│  1. 编码实现                                     │
│     └── 实现 Tool/Resource/Prompt 的处理函数     │
│                                                  │
│  2. 本地调试（Inspector）                        │
│     ├── npx @modelcontextprotocol/inspector      │
│     ├── 验证 Tool 定义是否正确                   │
│     ├── 手动调用测试每个 Tool                    │
│     └── 检查 JSON-RPC 消息格式                  │
│                                                  │
│  3. 本地集成测试（Claude Desktop）               │
│     ├── 配置到 mcp.json                         │
│     ├── 重启 Claude Desktop                     │
│     ├── 用自然语言测试 Tool 调用                 │
│     └── 观察 LLM 是否能正确选择和使用 Tool       │
│                                                  │
│  4. CI/CD 自动化测试                             │
│     ├── 单元测试（Tool 处理函数）                │
│     ├── 集成测试（JSON-RPC 协议测试）            │
│     └── Schema 验证（inputSchema 格式检查）      │
│                                                  │
│  5. 部署                                        │
│     ├── STDIO：分发二进制/脚本 + 配置文档        │
│     └── Streamable HTTP：Docker 部署 + OAuth 配置 │
│                                                  │
│  6. 监控和迭代                                   │
│     ├── 查看 MCP Logging 日志                   │
│     ├── 收集调用指标（延迟、错误率）             │
│     └── 根据 LLM 使用反馈优化 Tool description   │
└─────────────────────────────────────────────────┘
```

### 本地开发最佳实践

**1. 热重载开发**

```bash
# Python: 使用 nodemon 监听文件变化自动重启
nodemon --exec python -m my_mcp_server --ext .py

# TypeScript: 使用 tsx watch
npx tsx watch src/index.ts

# Go: 使用 air
air -- --stdio
```

**2. 分阶段测试**

```text
阶段 1：单元测试
  → 直接测试 Tool/Resource handler 函数的输入输出
  → 不依赖 MCP 协议

阶段 2：协议测试
  → 使用 Inspector 验证 JSON-RPC 消息格式
  → 检查 tools/list 返回的 Tool 定义

阶段 3：LLM 集成测试
  → 在 Claude Desktop 中用自然语言测试
  → 验证 LLM 能否正确理解和选择 Tool
  → 这是最接近真实使用场景的测试
```

**3. Debug 技巧**

```python
# Python: 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)

# 将日志输出到 stderr（不干扰 stdout 的 JSON-RPC 通信）
import sys
print(f"[DEBUG] Received args: {args}", file=sys.stderr)
```

```go
// Go: 使用 stderr 输出调试信息
fmt.Fprintf(os.Stderr, "[DEBUG] Handling tool: %s with args: %v\n", toolName, args)
```

```typescript
// TypeScript: 使用 console.error 写入 stderr
console.error(`[DEBUG] Received args:`, args);
```

> **关键规则**：调试日志必须输出到 **stderr**，不要输出到 stdout。STDIO Transport 中 stdout 用于 JSON-RPC 协议通信，任何非 JSON-RPC 输出都会导致协议解析错误。

---

## 17.6 常见问题排查

### 问题 1：MCP Server 启动失败

**现象**：Claude Desktop 或 Inspector 中看不到 MCP Server

**排查步骤**：

```bash
# 1. 手动运行 Server，确认可以正常启动
npx -y @modelcontextprotocol/server-filesystem /tmp

# 2. 检查 stderr 输出是否有错误
npx -y @modelcontextprotocol/server-filesystem /tmp 2>&1

# 3. 确认 Node.js / Python 版本满足要求
node --version  # MCP Server 通常需要 Node.js 18+
python --version  # MCP SDK 需要 Python 3.10+

# 4. 检查端口是否被占用（Streamable HTTP 模式）
lsof -i :3000
```

### 问题 2：LLM 不调用我的 Tool

**现象**：Server 正常启动，但 LLM 从不调用你定义的 Tool

**排查方向**：

| 可能原因 | 检查方法 | 修复 |
|---------|---------|------|
| **description 不够详细** | 检查 Tool 的 description 是否说明了适用场景 | 增加场景描述："适用于查询实时天气" |
| **name 不够语义化** | 检查 Tool name 是否清晰表达功能 | `do_stuff` → `search_github_repos` |
| **inputSchema 描述缺失** | 检查每个参数是否有 `description` | 为每个参数添加清晰的 description |
| **参数 required 设置有误** | 检查 required 字段是否包含了 LLM 无法推断的参数 | 为可选参数设置合理的 default 值 |
| **Tool 列表未更新** | 检查 `tools/list` 返回是否包含新 Tool | 重启 Client，触发重新 `tools/list` |

### 问题 3：STDIO 消息解析失败

**现象**：Server 日志显示 "Parse error" 或连接断开

**常见原因**：

```text
❌ 错误：使用 \n 代替 \r\n
  Content-Length: 100\n        ← 错误！
  \n                            ← 错误！
  {"jsonrpc":"2.0",...}

✅ 正确：使用 \r\n
  Content-Length: 100\r\n       ← 正确
  \r\n                           ← 正确
  {"jsonrpc":"2.0",...}
```

**修复**：

```go
// Go: 确保使用 \r\n 写入
writer.Write([]byte(fmt.Sprintf("Content-Length: %d\r\n\r\n", len(body))))
writer.Write(body)
```

```python
# Python: 使用 \r\n 分隔
sys.stdout.write(f"Content-Length: {len(body)}\r\n\r\n")
sys.stdout.write(body)
sys.stdout.flush()
```

```typescript
// TypeScript: SDK 内部正确处理，手动发送时注意
process.stdout.write(`Content-Length: ${body.length}\r\n\r\n`);
process.stdout.write(body);
```

### 问题 4：环境变量未生效

**现象**：Server 提示 API Key 未设置

**排查**：

```bash
# 1. 确认环境变量在 shell 中可用
echo $GITHUB_PERSONAL_ACCESS_TOKEN

# 2. 检查 mcp.json 中的 env 配置是否正确
# env 字段中的值不会展开 shell 变量
# 需要直接写入值或使用 ${env:VAR} 语法（Cursor 支持）

# 3. 重启 Host 应用
# Claude Desktop 修改配置后需要完全退出并重新打开
```

### 问题 5：Tool 执行超时

**现象**：LLM 调用 Tool 后长时间等待，最终报错

**排查和修复**：

```go
// 方案 1：为 Tool 执行设置 context timeout
func handleQuery(ctx context.Context, args map[string]any) (*mcp.CallToolResult, error) {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    // ... 执行查询
}

// 方案 2：对长时间操作启用进度通知
// 向 Client 发送 notifications/progress 告知进度
```

---

## 17.7 MCP Server 性能优化

### 优化清单

| 优化项 | 说明 | 优先级 |
|--------|------|--------|
| **连接池复用** | 数据库 MCP Server 使用连接池，避免每次 Tool 调用重新连接 | ⭐⭐⭐ |
| **结果缓存** | 确定性查询（如 `list_tables`）缓存结果，TTL 设置为 30-60 秒 | ⭐⭐⭐ |
| **分页限制** | 强制返回结果的行数上限（如 LIMIT 100），防止大数据量传输 | ⭐⭐⭐ |
| **懒加载** | 非必要资源延迟初始化（如大文件索引） | ⭐⭐ |
| **异步 I/O** | 使用异步 HTTP 客户端调用外部 API | ⭐⭐ |
| **预编译** | JSON Schema 验证器预先编译，避免重复解析 | ⭐ |

### 数据库 MCP Server 优化示例

```go
type MySQLMCPServer struct {
    db     *sql.DB           // 连接池
    cache  *cache.Cache      // 结果缓存（TTL 60s）
}

func (s *MySQLMCPServer) handleListTables(ctx context.Context, _ map[string]any) (*mcp.CallToolResult, error) {
    // 1. 检查缓存
    cacheKey := "list_tables"
    if cached, found := s.cache.Get(cacheKey); found {
        return textResult(cached.(string)), nil
    }

    // 2. 执行查询（设置超时）
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()

    rows, err := s.db.QueryContext(ctx, "SHOW TABLES")
    if err != nil {
        return errorResult(err), nil
    }
    defer rows.Close()

    // 3. 解析结果
    var tables []string
    for rows.Next() {
        var table string
        rows.Scan(&table)
        tables = append(tables, table)
    }

    // 4. 写入缓存
    result, _ := json.Marshal(tables)
    s.cache.Set(cacheKey, string(result), 60*time.Second)

    return textResult(string(result)), nil
}
```

---

## 17.8 MCP 在 CI/CD 中的应用

### GitHub Actions 集成

```yaml
# .github/workflows/mcp-test.yml
name: MCP Server Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run MCP protocol tests
        run: |
          # 启动 MCP Server（后台）
          node dist/index.js &
          SERVER_PID=$!
          sleep 2

          # 发送 initialize 请求
          echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"ci","version":"1.0.0"}}}' | nc localhost 3000

          # 发送 tools/list 请求
          echo '{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | nc localhost 3000

          kill $SERVER_PID

      - name: Validate Tool schemas
        run: npx @modelcontextprotocol/inspector --validate ./src/tools.ts
```

### Docker 部署模板

```dockerfile
# Dockerfile - Streamable HTTP MCP Server
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist/ ./dist/

EXPOSE 3000

# 健康检查：定期 ping MCP Server
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/mcp',{method:'POST',headers:{'Content-Type':'application/json'},body:'{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"ping\"}'})" || exit 1

CMD ["node", "dist/index.js"]
```

---

## 17.9 日常使用速查

### 快速命令参考

```bash
# ─── MCP Inspector ─────────────────────────────────
# 调试本地 MCP Server
npx @modelcontextprotocol/inspector python -m my_server
npx @modelcontextprotocol/inspector node dist/index.js

# ─── Claude Desktop ────────────────────────────────
# 配置文件位置（macOS）
open ~/Library/Application\ Support/Claude/claude_desktop_config.json

# 配置位置（Windows）
explorer %APPDATA%\Claude\claude_desktop_config.json

# ─── 测试 STDIO Server ─────────────────────────────
# 发送 initialize
printf 'Content-Length: 108\r\n\r\n{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0"}}}' | your-server

# 发送 tools/list
printf 'Content-Length: 68\r\n\r\n{"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}' | your-server

# ─── 使用 npx 快速体验 ─────────────────────────────
npx -y @modelcontextprotocol/server-filesystem /tmp
npx -y @modelcontextprotocol/server-github
npx -y @modelcontextprotocol/server-postgres
npx -y @modelcontextprotocol/server-brave-search
npx -y @modelcontextprotocol/server-memory
npx -y @modelcontextprotocol/server-sequential-thinking
```

### 环境变量速查

| MCP Server | 需要的环境变量 |
|-----------|-------------|
| **GitHub** | `GITHUB_PERSONAL_ACCESS_TOKEN` |
| **PostgreSQL** | `DATABASE_URL` |
| **Brave Search** | `BRAVE_API_KEY` |
| **Slack** | `SLACK_BOT_TOKEN` |
| **Google Drive** | `GOOGLE_DRIVE_CLIENT_ID`, `GOOGLE_DRIVE_CLIENT_SECRET` |

---

## 面试题：MCP 日常开发中的常见问题？

**标准回答：**

> MCP 日常开发中最常见的五类问题：
>
> 1. **调试困难** → 使用 MCP Inspector 可视化查看 JSON-RPC 消息，调试日志必须输出到 stderr
> 2. **LLM 不调用 Tool** → 通常是 description 不够详细或缺少参数级别的 description，需为 LLM 提供充足上下文
> 3. **STDIO 消息解析失败** → 行分隔符必须用 `\r\n`（CRLF），使用 `\n` 会导致协议解析错误
> 4. **环境变量未生效** → 确认 mcp.json 中的 env 配置正确，修改配置后需重启 Host 应用
> 5. **Tool 执行超时** → 对长时间操作设置 context timeout + 启用进度通知
>
> **开发工作流**：编码 → Inspector 调试 → Claude Desktop 集成测试 → CI/CD 自动化测试 → Docker 部署。

**关键词**：Inspector 调试、CRLF 格式、stderr 日志、LLM 集成测试、热重载开发
>
> - 与第十六章的关系：第十六章是**开放式场景题**（提供多版本回答模板，按 30s/1min/3min 分层），本章是**高频必背题**（每题一段精简回答 + 关键词速记）
> - 使用建议：先通读前十七章理解原理和日常使用 → 第十六章练习场景答辩 → 第十七章掌握开发实践 → 本章做最后冲刺背诵
> - 每题的"关键词"标签可用于面试前的快速自测

---

### Q1: 什么是 MCP？

**标准回答：**
> MCP（Model Context Protocol，模型上下文协议）是 Anthropic 于 2024 年开源的标准协议，用于连接 LLM 与外部工具、数据源和服务。它解决了 AI 工具集成的 N×M 组合爆炸问题——在 MCP 之前，每个 LLM 平台都要为每个工具单独开发插件；有了 MCP，工具只需开发一次，所有 LLM 都能使用。它基于 JSON-RPC 2.0，定义了 Host-Client-Server 三层架构，被称为"AI 世界的 USB 接口标准"。

**关键词**：模型上下文协议、AI 的 USB、Anthropic 2024、N×M → N+M、JSON-RPC 2.0

---

### Q2: MCP 解决什么问题？

**标准回答：**
> MCP 解决的是 AI 应用与外部系统集成的**碎片化问题**。没有 MCP 时，N 个 LLM 平台 × M 个工具 = N×M 个集成，每个都要单独开发、维护，开发者被锁定在特定 LLM 生态中。MCP 通过统一的协议层，将这个问题降为 N+M：工具开发者只需写一个 MCP Server，所有支持 MCP 的 LLM 应用都能即插即用。

**关键词**：碎片化集成、一次开发到处使用、N×M → N+M、厂商解耦

---

### Q3: MCP 的整体架构是怎样的？

**标准回答：**
> MCP 采用三层架构：
> - **Host**（宿主应用）：运行 AI Agent 的应用，如 Claude Desktop、Cursor。负责管理多个 Client 的生命周期，路由 LLM 的工具调用请求。
> - **Client**（客户端）：Host 内的通信组件，与一个 MCP Server 建立一对一连接，负责 JSON-RPC 消息的收发。
> - **Server**（服务端）：提供具体工具能力的独立进程，暴露 Tool、Resource、Prompt，执行实际的业务逻辑。
>
> **记忆口诀**：Host 是房子（应用），Client 是插座（通信），Server 是电器（工具）。

**关键词**：三层架构、Host-Client-Server、一对一连接、进程隔离

---

### Q4: Host、Client、Server 的区别？

**标准回答：**
> - **Host**：运行 Agent 的应用层，管理多个 Client，负责 Agent 整体逻辑和用户交互
> - **Client**：Host 内部的通信组件，与一个 Server 一对一连接，负责 JSON-RPC 消息收发和会话管理
> - **Server**：独立进程，提供具体工具/资源/提示词，执行实际业务逻辑
>
> Host 包含多个 Client，每个 Client 连接一个 Server。Host 通过 Client 与 Server 通信。

**关键词**：Host 管理、Client 通信、Server 执行、一对多关系

---

### Q5: MCP 基于什么协议？

**标准回答：**
> MCP 基于 **JSON-RPC 2.0** 协议。选择 JSON-RPC 的原因：协议极简（只有 Request、Response、Notification 三种消息类型）、语言无关（JSON 通用格式）、传输无关（可在 STDIO、HTTP、WebSocket 上运行）、支持双向通信（Request-Response + 单向 Notification）。JSON-RPC 是成熟的工业标准，不是 MCP 自己发明的。

**关键词**：JSON-RPC 2.0、Request/Response/Notification、传输无关、成熟标准

---

### Q6: JSON-RPC 在 MCP 中如何工作？

**标准回答：**
> MCP 中的所有通信都是 JSON-RPC 消息：
> - **Request**：Client 发送给 Server，包含 `id`（唯一标识）、`method`（如 `tools/call`）、`params`（参数）。Server 必须返回带相同 `id` 的 Response。
> - **Response**：包含 `id` 和 `result`（成功）或 `error`（失败，含标准错误码）。
> - **Notification**：单向消息，没有 `id` 字段，Server 不返回响应。用于进度通知、资源变更等场景。
>
> 核心区别：Request 是「我问你答」，Notification 是「我告诉你，不用回」。

**关键词**：Request(id+method+params)、Response(result/error)、Notification(无id)、双向通信

---

### Q7: MCP Tool 是什么？

**标准回答：**
> Tool 是 MCP 中最核心的概念，代表 **LLM 可以调用的一个功能单元**。每个 Tool 由以下要素定义：
> - **name**：唯一标识符，snake_case 命名，动词开头（如 `get_weather`）
> - **description**：详细的功能描述——包括适用场景、限制条件、副作用。这是 LLM 选择工具的唯一依据，必须写得足够详细
> - **inputSchema**：JSON Schema 定义参数（类型、枚举、默认值、必填项）
> - **annotations**（可选）：元数据标注——title（UI 标题）、readOnlyHint（只读提示）、destructiveHint（破坏性操作提示）等
>
> Tool 通过 `tools/list` 被发现，通过 `tools/call` 被调用，结果以 `content` 数组返回。

**关键词**：功能单元、name+description+inputSchema+annotations、tools/list 发现、tools/call 调用

---

### Q8: MCP Resource 是什么？

**标准回答：**
> Resource 是 MCP 中代表**可被 LLM 读取的上下文数据**的概念。每个 Resource 由 URI 唯一标识（如 `file://readme.md`、`mysql://users/table`）。Resource 是**只读的**——它只提供数据，不执行有副作用的操作。通过 `resources/list` 发现，通过 `resources/read` 读取。支持 URI 模板（如 `mysql://{database}/{table}`）处理动态资源。

**关键词**：只读数据、URI 标识、resources/read、URI 模板

---

### Q9: Tool 和 Resource 的核心区别？

**标准回答：**
> | Tool | Resource |
> |------|----------|
> | 动作（能做什么） | 数据（有什么） |
> | 可能有副作用（读写） | 只读 |
> | `tools/call` 调用 | `resources/read` 读取 |
> | 类比 RPC/函数调用 | 类比 REST GET/文件读取 |
>
> **一句话**：Tool 用于执行操作，Resource 用于获取上下文。

**关键词**：动作 vs 数据、副作用 vs 只读、tools/call vs resources/read

---

### Q10: MCP Prompt 是什么？

**标准回答：**
> Prompt 是 MCP Server 提供的**预定义提示词模板**。作用有三：模板复用（代码审查、测试生成等场景无需每次重写提示词）、领域知识封装（将安全审查规范、代码风格指南固化到模板中）、参数化定制（支持参数如 `language`、`focus_areas`，同一模板适用不同场景）。通过 `prompts/list` 发现，`prompts/get` 获取具体内容。

**关键词**：提示词模板、参数化、模板复用、领域知识封装

---

### Q11: MCP 的生命周期是怎样的？

**标准回答：**
> MCP 连接有五个阶段：
> 1. **启动**：Host 启动 Server 子进程（STDIO）或建立 HTTP 连接
> 2. **Initialize（握手）**：Client 和 Server 交换协议版本和能力声明（能力协商）
> 3. **Ready（就绪）**：Server 发送 `notifications/initialized`，连接就绪
> 4. **正常操作**：`tools/list`、`tools/call`、`resources/read`、`prompts/get` 等
> 5. **Shutdown（关闭）**：Server 清理资源并退出

**关键词**：启动→Initialize(协商)→Initialized(就绪)→操作→Shutdown

---

### Q12: Initialize 初始化流程具体是怎样的？

**标准回答：**
> Initialize 是 MCP 的**三步握手**：
> 1. **Client → Server**：发送 `initialize` 请求，携带协议版本、Client 能力（如 `roots`、`sampling`）、Client 信息（名称+版本）
> 2. **Server → Client**：返回 `initialize` 响应，确认协议版本，声明 Server 能力（`tools`、`resources`、`prompts`、`logging`），返回 Server 信息
> 3. **Server → Client**：发送 `notifications/initialized` 通知，连接就绪
>
> 核心是第二步的**能力协商**：双方声明自己能做什么，取交集决定后续可用功能。

**关键词**：三步握手、能力协商、protocolVersion、capabilities、initialized 通知

---

### Q13: tools/list 的流程和作用？

**标准回答：**
> `tools/list` 是**工具发现**的方法。Client 发送空参数（或分页参数）请求，Server 返回所有可用 Tool 的列表，每个 Tool 包含 name、description、inputSchema。LLM 基于这些信息理解"有哪些工具可用、每个工具做什么、怎么调用"。
>
> 通常在初始化后调用一次，结果缓存。当 Server 的工具列表发生变化时，通过 `notifications/tools/list_changed` 通知 Client 重新获取。
>
> **类比**：`tools/list` = API 文档/Swagger，`tools/call` = 实际 API 调用。

**关键词**：工具发现、name+description+inputSchema、初始化调用一次、list_changed 通知

---

### Q14: tools/call 的流程是怎样的？

**标准回答：**
> `tools/call` 是**工具执行**的方法，属于 ReAct 循环中的 Action 阶段：
> 1. LLM 决策调用某个 Tool（基于 `tools/list` 返回的 description）
> 2. Client 发送 `tools/call` 请求，携带 `name` 和 `arguments`
> 3. Server 执行实际的业务逻辑（查数据库、调 API、读写文件等）
> 4. Server 返回 `result`，包含 `content` 数组（`[{type: "text", text: "..."}]`）
> 5. 如果执行出错，返回 `isError: true` + 错误信息（帮助 LLM 调整策略）
>
> **核心**：LLM 每次推理决定调用一个 Tool → `tools/call` → 收到结果 → 再次推理 → 可能继续调用 → 直到任务完成。

**关键词**：工具执行、name+arguments、content 数组、ReAct 循环、错误即反馈

---

### Q15: STDIO Transport 的原理和适用场景？

**标准回答：**
> STDIO Transport 是 MCP 最基础的传输方式：
> - **原理**：Host 启动 Server 作为子进程，通过 stdin 发送 JSON-RPC 请求，通过 stdout 接收 JSON-RPC 响应，stderr 用于日志。消息格式为 `Content-Length` 头 + JSON body，行分隔符必须为 `\r\n`（CRLF）。
> - **优点**：实现简单、零网络配置、进程级安全隔离、无网络延迟
> - **缺点**：只能本地通信、每个 Server 独占一个进程、不支持远程和多 Host 共享
> - **适用场景**：本地 IDE 工具（Cursor + 文件系统 MCP）、个人 AI 助手、开发调试

**关键词**：子进程、stdin/stdout、Content-Length、进程隔离、仅本地

---

### Q16: Streamable HTTP 的原理和优势？

**标准回答：**
> Streamable HTTP 是 MCP 2025 年推出的**推荐远程传输方案**（替代 SSE 成为远程首选）：
> - **原理**：Client 发送 POST 请求到单一端点（如 `/mcp`），声明 `Accept: text/event-stream`。Server 在同一连接上以 SSE 格式流式返回结果，支持双向通信和服务端推送。
> - **vs SSE 的优势**：单端点（vs 双通道）、代理友好（标准 POST）、请求-响应天然匹配（同一连接）、支持断点续传（Last-Event-ID）、全平台兼容（包括移动端）、可降级为普通 HTTP。
> - **适用场景**：生产环境远程 MCP Server、多用户共享服务

**关键词**：单端点 POST、流式响应、Streamable HTTP 替代 SSE、代理友好、断点续传

---

### Q17: MCP 和 Function Calling 的区别？

**标准回答：**
> - **Function Calling**：LLM 厂商的 API 特性（OpenAI/Anthropic/Google 各有格式）。工具定义随每次 API 请求传递，应用层管理工具实现。解决"模型如何调用函数"。
> - **MCP**：开放协议标准，跨厂商统一。工具在 Server 中注册，Client 自动发现，协议层管理生命周期。解决"工具如何标准化地描述、发现和管理"。
>
> **关系**：互补，不是替代。MCP Server 托管工具，LLM 通过 Function Calling 机制决策调用哪个工具，实际执行通过 MCP 的 `tools/call`。

**关键词**：厂商 API vs 开放协议、每次传递 vs 一次发现、互补关系

---

### Q18: MCP 与 Agent 是什么关系？

**标准回答：**
> MCP 是 Agent 的**标准化工具执行层**：
> - **Agent = 大脑**：负责理解任务、制定计划、做决策（Planning + Reasoning）
> - **MCP = 神经系统**：负责在大脑和工具之间传递标准化信号
> - **Tools = 手脚**：负责实际执行操作（读写文件、查数据库、调 API）
>
> Agent 的每一步工具调用都通过 MCP 的 `tools/call` 完成。MCP 让 Agent 从"硬编码工具"升级为"动态工具生态"——一个 Agent 只需实现 MCP Client，就能使用所有 MCP Server 的工具。

**关键词**：大脑-神经-手脚、标准化工具层、ReAct 循环、动态工具生态

---

### Q19: 如何开发一个 MCP Server？

**标准回答：**
> 六个步骤：
> 1. **选 SDK**：Python 用 FastMCP（装饰器注册，最简单），Go 用 go-sdk（底层 JSON-RPC），TypeScript 用官方 SDK
> 2. **创建 Server**：设置 name 和 version
> 3. **注册 Tools**：定义 name（snake_case，动词开头）+ description（越详细越好）+ inputSchema（JSON Schema）+ handler（业务逻辑）
> 4. **注册 Resources**（可选）：定义只读数据资源和 URI
> 5. **注册 Prompts**（可选）：定义参数化提示词模板
> 6. **启动**：选 Transport（STDIO 本地 / Streamable HTTP 远程）
>
> **关键原则**：description 要详细（LLM 的唯一依据）、Tool 要原子化、必须做参数校验、错误信息要清晰。

**关键词**：选 SDK → 创建 → 注册 Tool/Resource/Prompt → 启动、FastMCP/官方TS SDK/go-sdk、annotations

---

### Q20: 企业级 MCP 平台架构如何设计？

**标准回答：**
> 五层架构：
> 1. **接入层 — MCP Gateway**：统一入口，认证鉴权（API Key/OAuth）、限流、TLS 终止
> 2. **路由层 — Tool Router**：维护 Tool → Server 路由表，支持静态/动态/灰度路由
> 3. **代理层 — Proxy Chain**（可选）：认证代理 → 缓存代理 → 审计代理 → 改写代理
> 4. **服务层 — Server 集群**：每个 Server 独立容器化部署（K8s），多副本高可用，HPA 自动扩缩
> 5. **基础设施层**：服务发现（Consul/etcd）、配置中心（Nacos）、可观测性（OpenTelemetry + Prometheus + Grafana）、多租户隔离
>
> **关键设计**：高可用、安全性（纵深防御）、可观测性、可扩展（新增 Server 只需注册）。

**关键词**：Gateway → Router → Proxy → Server 集群 → 基础设施、五层架构、OAuth 2.0、服务发现

---

# 附录：MCP 协议参考

## A.1 核心 JSON-RPC 方法速查表

| 方法 | 方向 | 类型 | 说明 |
|------|------|------|------|
| `initialize` | C→S | Request | 初始化连接 |
| `notifications/initialized` | S→C | Notification | 初始化完成通知 |
| `tools/list` | C→S | Request | 获取工具列表 |
| `tools/call` | C→S | Request | 调用工具 |
| `resources/list` | C→S | Request | 获取资源列表 |
| `resources/read` | C→S | Request | 读取资源 |
| `resources/templates/list` | C→S | Request | 获取资源模板 |
| `resources/subscribe` | C→S | Request | 订阅资源变更 |
| `prompts/list` | C→S | Request | 获取 Prompt 列表 |
| `prompts/get` | C→S | Request | 获取 Prompt 内容 |
| `ping` | C→S | Request | 心跳检测 |
| `sampling/createMessage` | S→C | Request | Server 请求 LLM 生成文本 |
| `roots/list` | S→C | Request | Server 请求 Client 的 Roots 列表 |
| `logging/setLevel` | C→S | Request | Client 设置 Server 日志级别 |
| `elicitation/create` | S→C | Request | Server 请求用户输入 |
| `notifications/progress` | S→C | Notification | 进度通知 |
| `notifications/cancelled` | S→C | Notification | 取消通知 |
| `notifications/logging` | S→C | Notification | Server 推送日志消息 |
| `notifications/resources/updated` | S→C | Notification | 资源更新 |
| `notifications/resources/list_changed` | S→C | Notification | 资源列表变更 |
| `notifications/tools/list_changed` | S→C | Notification | 工具列表变更 |
| `notifications/prompts/list_changed` | S→C | Notification | Prompt 列表变更 |
| `notifications/roots/list_changed` | C→S | Notification | Client 通知 Roots 列表变更 |

## A.2 MCP 协议版本

| 版本 | 日期 | 主要变化 |
|------|------|---------|
| `2024-11-05` | 2024-11 | 初始版本 |
| `2025-03-26` | 2025-03 | **引入 Streamable HTTP（推荐远程方案）**，SSE 标记为不推荐新项目使用 |
| `2025-06-18`（预计） | 2025-06 | 增强 Resource 模板，任务管理，Elicitation 正式引入 |

## A.3 推荐学习资源

- **MCP 官方文档**：https://modelcontextprotocol.io
- **MCP 规范**：https://spec.modelcontextprotocol.io
- **MCP GitHub**：https://github.com/modelcontextprotocol
- **官方 Server 列表**：https://github.com/modelcontextprotocol/servers
- **Python SDK**：https://github.com/modelcontextprotocol/python-sdk
- **TypeScript SDK**：https://github.com/modelcontextprotocol/typescript-sdk
- **Go SDK**：https://github.com/modelcontextprotocol/go-sdk
- **MCP Inspector（调试工具）**：`npx @modelcontextprotocol/inspector` — 可视化 MCP 消息调试器
- **MCP Registry（Server 注册中心）**：https://registry.modelcontextprotocol.io
- **Awesome MCP（社区资源汇总）**：https://github.com/punkpeye/awesome-mcp-servers
- **Claude Desktop 配置指南**：https://docs.anthropic.com/en/docs/claude-code/mcp
