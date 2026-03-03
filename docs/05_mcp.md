# 第五章 MCP (Model Context Protocol) 协议详解

> **本章目标**：深入理解 MCP 协议的架构设计、通信机制和实现细节，具备从零搭建 MCP Server/Client 的能力。

> **阅读建议**：本章内容较为深入，建议配合官方文档 https://modelcontextprotocol.io 一起阅读。如果你只是想快速使用 MCP，可以先阅读代码示例部分（第七部分），再回过头来理解理论概念。

---

## 第一部分：MCP 概述

### 1.1 为什么需要 MCP？

#### 时代背景：AI 大模型的"工具饥渴症"

2023-2024 年，随着 LLM 能力的爆发，AI Agent 如雨后春笋般涌现。然而，开发者们面临一个共同的困境：

**每个 AI 应用都要重新发明轮子**。

想象一下这个场景：你是一家公司的技术负责人，需要为三个不同的 AI 应用集成相同的工具：
- AI 客服助手：需要访问知识库、查询订单、处理退款
- AI 销售助手：需要访问 CRM、查询库存、生成报价
- AI 运营助手：需要访问数据分析平台、生成报告、监控指标

在没有 MCP 的时代，你的团队需要：
1. 为 AI 客服助手编写一套工具集成代码
2. 为 AI 销售助手**再编写一套几乎相同的**工具集成代码
3. 为 AI 运营助手**又编写一套几乎相同的**工具集成代码

这不仅造成了巨大的开发资源浪费，还带来了维护噩梦——每当工具 API 变更，三个应用都要同步更新。

#### AI Agent 集成工具的痛点

在 MCP 出现之前，每个 AI 应用都需要独立集成各种工具和数据源：

```
┌─────────────────────────────────────────────────────────┐
│                    AI 应用 A                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                │
│  │ Google   │ │ 文件系统  │ │ 数据库    │  ← 重复集成   │
│  │ Search   │ │          │ │          │                │
│  └──────────┘ └──────────┘ └──────────┘                │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    AI 应用 B                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                │
│  │ Google   │ │ 文件系统  │ │ 数据库    │  ← 再次集成   │
│  │ Search   │ │          │ │          │                │
│  └──────────┘ └──────────┘ └──────────┘                │
└─────────────────────────────────────────────────────────┘
```

**问题分析**：
1. **重复开发**：每个应用都要编写相同的集成代码
2. **维护成本高**：工具更新需要所有应用同步升级
3. **碎片化严重**：没有统一标准，集成方式五花八门
4. **安全性难以保障**：每个应用各自实现权限控制

#### 生动的类比：从"万能充电器"到"USB-C"

理解 MCP 价值的一个好类比是**手机充电器的发展历史**：

**前 MCP 时代（2010 年前）**：
- 诺基亚手机用圆孔充电器
- 摩托罗拉用扁口充电器
- 苹果用 30-pin 接口
- 索尼爱立信用梯形接口
- ...

每个手机厂商都有自己的充电接口标准，用户每换一部手机就要收集一堆全新的充电器。这造成了巨大的电子浪费和用户体验灾难。

**MCP 时代（USB-C 标准）**：
- 统一接口标准
- 一个充电器可以给任何设备充电
- 设备厂商只需实现一次 USB-C 接口
- 用户获得一致的充电体验

**MCP 就是 AI 工具集成领域的"USB-C"**：
- 工具提供者只需实现一次 MCP 接口
- 任何支持 MCP 的 AI 应用都能使用该工具
- 用户获得一致的工具使用体验

#### MCP 的解决方案

MCP 通过标准化接口解决了上述问题：

```
┌─────────────────────────────────────────────────────────┐
│                    AI 应用 A                              │
│  ┌──────────────────────────────────────────────────┐  │
│  │              MCP Client                           │  │
│  └────────────────────┬─────────────────────────────┘  │
└───────────────────────┼─────────────────────────────────┘
                        │
┌───────────────────────┼─────────────────────────────────┐
│                       ▼                                 │
│  ┌──────────────────────────────────────────────────┐  │
│  │              MCP Server                           │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │  │
│  │  │ Google   │ │ 文件系统  │ │ 数据库    │         │  │
│  │  │ Search   │ │          │ │          │         │  │
│  │  └──────────┘ └──────────┘ └──────────┘         │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**一次开发，多处使用**：
- 开发者只需编写一次 MCP Server
- 所有支持 MCP 的 AI 应用都能连接使用
- 工具升级只需更新 Server，Client 无需改动

### 1.2 MCP 的历史与发展

#### 发布时间线

| 时间 | 事件 |
|------|------|
| 2024 年 11 月 | Anthropic 正式发布 MCP 协议 |
| 2024 年 12 月 | MCP 官方 Python SDK 发布 |
| 2025 年 1 月 | Claude Desktop 集成 MCP 支持 |
| 2025 年 2 月 | 社区贡献的 MCP Server 超过 100 个 |

#### 开源社区现状

MCP 采用 MIT 开源协议，GitHub 仓库地址：
- 官方规范：https://github.com/modelcontextprotocol/specification
- Python SDK: https://github.com/modelcontextprotocol/python-sdk
- 参考服务器：https://github.com/modelcontextprotocol/servers

**热门社区项目**：
- `@modelcontextprotocol/server-filesystem` - 文件系统服务
- `@modelcontextprotocol/server-postgres` - PostgreSQL 数据库服务
- `@modelcontextprotocol/server-puppeteer` - 浏览器自动化服务
- `@modelcontextprotocol/server-slack` - Slack 集成服务

#### 主流支持平台

| 平台 | MCP 支持 | 说明 |
|------|---------|------|
| Claude Desktop | ✅ 原生支持 | 首个支持 MCP 的桌面应用 |
| Claude Code | ✅ 内置支持 | CLI 工具中的 MCP 集成 |
| VS Code 插件 | ✅ 通过插件 | Continue、Cline 等插件 |
| Zed 编辑器 | ✅ 实验性支持 | 代码编辑器的 AI 助手 |

### 1.3 MCP 核心优势

#### 1. 标准化接口

MCP 定义了统一的 API 接口：

```
tools/list       - 列出可用工具
tools/call       - 调用工具
resources/list   - 列出资源
resources/read   - 读取资源
prompts/list     - 列出提示词模板
prompts/get      - 获取提示词
```

开发者只需遵循这些接口规范，无需关心具体 AI 应用的内部实现。

#### 2. 模型无关性

MCP 不依赖于特定的 LLM 模型：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Claude    │     │    GPT-4    │     │   Llama     │
│      ↓      │     │      ↓      │     │      ↓      │
│  MCP Client │     │  MCP Client │     │  MCP Client │
│      ↓      │     │      ↓      │     │      ↓      │
└─────────────┘     └─────────────┘     └─────────────┘
         ╲               │               ╱
          ╲              │              ╱
           └─────────────┴─────────────┘
                   MCP Server
```

同一 MCP Server 可以服务于不同的 AI 模型。

#### 3. 实时上下文提供

MCP 不仅提供工具调用，还能提供实时上下文：

- **Resources**: 提供文档、配置文件、数据库记录等静态数据
- **Tools**: 执行搜索、计算、API 调用等动态操作
- **Prompts**: 提供预定义的提示模板，提高交互效率

#### 4. 安全性设计

MCP 的安全机制设计遵循**最小权限原则**和**明确授权原则**：

| 安全特性 | 说明 | 实际应用场景 |
|---------|------|-------------|
| 本地优先 | stdio 传输仅限本地进程通信，无法跨网络访问 | 防止远程攻击者通过 MCP Server 访问本地资源 |
| 明确授权 | 用户需手动配置允许的 Server，并知晓其权限范围 | Claude Desktop 会显示每个 Server 可访问的资源类型 |
| 范围限制 | 可限制 Server 的访问范围（如文件系统目录） | 文件 Server 只能访问用户指定的目录，而非整个文件系统 |
| 输入验证 | Server 端可进行严格的参数验证，防止注入攻击 | 路径参数检查是否包含 `..` 防止路径遍历攻击 |
| 透明日志 | 所有交互都可以通过日志审计 | 管理员可以查看哪些工具被调用、访问了哪些资源 |

**安全性实际案例**：

假设你配置了一个文件系统 MCP Server，允许 AI 访问 `~/documents` 目录：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/yourname/documents"],
      "env": {
        "ALLOWED_PATHS": "/Users/yourname/documents"  // 限制访问范围
      }
    }
  }
}
```

即使 AI 尝试读取 `/etc/passwd` 或其他敏感文件，MCP Server 会拒绝访问，因为该路径不在允许的范围内。

**对比传统方式的安全风险**：

| 方式 | 风险 |
|------|------|
| 直接给 AI 文件系统访问权限 | AI 可能读取任意文件，包括密码、SSH 密钥等 |
| 通过 MCP Server 限制访问 | AI 只能访问配置的目录，风险可控 |

---

---

## 第二部分：MCP 架构深度解析

### 2.1 三层架构详解

#### 与 OSI 七层模型的类比

理解 MCP 三层架构的一个好方法是类比网络领域的**OSI 七层模型**：

| OSI 模型 | MCP 架构 | 类比说明 |
|---------|---------|---------|
| 应用层 | Host 层 | 直接面向用户，提供最终功能 |
| 表示层/会话层 | Client 层 | 负责数据格式转换、会话管理 |
| 传输层/网络层 | Transport 层 | 负责可靠的消息传输 |
| 物理层 | Stdio/HTTP | 实际的物理/网络连接 |

这种分层设计的好处是**关注点分离**——每一层只需关心自己的职责，无需了解其他层的实现细节。

MCP 采用清晰的三层架构设计：

```
┌─────────────────────────────────────────────────────────┐
│                    MCP Host (AI 应用)                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │              MCP Client                          │   │
│  └───────────────────────┬─────────────────────────┘   │
└──────────────────────────┼──────────────────────────────┘
                           │ JSON-RPC 2.0
                           │ (stdio / Streamable HTTP)
                           │
┌──────────────────────────┼──────────────────────────────┐
│                          ▼                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              MCP Server                          │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐           │   │
│  │  │  Tools  │ │Resources│ │Prompts  │           │   │
│  │  └─────────┘ └─────────┘ └─────────┘           │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

**层级说明**：

| 层级 | 组件 | 职责 | 类比 |
|------|------|------|------|
| Host 层 | AI 应用 | 提供用户界面，管理 Client 连接 | 餐厅顾客 - 点餐并享用菜品 |
| Client 层 | MCP Client | 与 Server 通信，转发请求/响应 | 服务员 - 传递菜单和菜品 |
| Server 层 | MCP Server | 提供工具、资源、提示词 | 厨房 - 准备菜品 |

**关键设计原则**：
- **Client 层是可选的**：简单场景下，Host 可以直接与 Server 通信
- **一层对多层**：一个 Host 可以管理多个 Client，一个 Client 可以连接多个 Server
- **职责边界清晰**：每层只与相邻层通信，不跨层调用

### 2.2 参与者详解

#### 实际对应关系表格

为了帮助你更好地理解，以下是 MCP 参与者的实际对应关系：

| 抽象概念 | Claude 生态系统 | 其他生态系统 | 自定义实现 |
|---------|----------------|-------------|-----------|
| Host | Claude Desktop | VS Code (Cline 插件) | 你的 Python 应用 |
| Client | Claude 内置 Client | Cline 插件内置 Client | `src.mcp.MCPClient` |
| Server | 官方 Weather Server | 社区 Filesystem Server | 你编写的自定义 Server |

### 2.2 参与者详解

#### MCP Host

**定义**：运行 AI 应用的主体程序。

**常见 Host**：
- Claude Desktop (macOS/Windows) - 桌面版 Claude
- Claude Code (CLI 工具) - 命令行 AI 助手
- VS Code 插件 (Continue, Cline) - 代码编辑器插件
- 自定义 AI 应用 - 你自己开发的 AI 程序

**Host 的职责**：
1. **管理 MCP Client 实例** - 创建、初始化、销毁 Client
2. **向用户展示可用的工具/资源** - 在 UI 中显示工具列表
3. **处理用户授权确认** - 某些敏感操作需要用户确认
4. **转发 LLM 的工具调用请求** - 将 LLM 的调用意图转换为 MCP 请求

**Host 的生命周期**：
```
启动 → 读取配置 → 创建 Client → 连接 Server → 运行 → 关闭连接 → 退出
```

#### MCP Client

**定义**：内嵌在 Host 中的客户端组件。

**Client 的职责**：
1. **建立与维护 Server 的连接** - 处理底层通信细节
2. **协议初始化与能力协商** - 交换 protocolVersion、capabilities
3. **发送请求并接收响应** - 管理请求 ID、超时、重试
4. **处理连接异常** - 断线重连、错误恢复

**Client 内置能力**：
```
- Sampling (补全): 请求 LLM 生成文本（Server 可请求 AI 补全）
- Elicitation (询问): 请求用户提供信息（Server 可请求用户输入）
- Logging (日志): 发送日志消息（Server 可发送日志到 Host）
```

**Client 的关键特性**：
- **无状态**：Client 本身不维护业务状态，状态由 Server 维护
- **透明转发**：Client 不解析或修改业务数据，只负责传输
- **多路复用**：一个 Client 可以同时连接多个 Server

#### MCP Server

**定义**：提供工具、资源、提示词的服务程序。

**Server 的能力**：

| 能力 | 方法 | 用途 |
|------|------|------|
| Tools | `tools/list`, `tools/call` | 提供可执行函数 |
| Resources | `resources/list`, `resources/read` | 提供数据源 |
| Prompts | `prompts/list`, `prompts/get` | 提供提示模板 |

### 2.3 两层协议详解

MCP 协议分为两层：

```
┌─────────────────────────────────────────┐
│           Data Layer (数据层)            │
│  ┌─────────────────────────────────┐   │
│  │      JSON-RPC 2.0 Protocol      │   │
│  │  - Request/Response             │   │
│  │  - Notification                 │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│        Transport Layer (传输层)          │
│  ┌─────────────────┐ ┌───────────────┐ │
│  │  Stdio Transport│ │ HTTP Transport│ │
│  │    (本地)       │ │   (远程)      │ │
│  └─────────────────┘ └───────────────┘ │
└─────────────────────────────────────────┘
```

#### 数据层 (Data Layer)

基于 JSON-RPC 2.0 协议，定义消息格式和交互模式。

**通信模式**：
1. **Request-Response**: 请求 - 响应模式（需要响应）
2. **Notification**: 通知模式（无需响应）

#### 传输层 (Transport Layer)

负责实际的消息传输，支持两种方式：

| 传输方式 | 适用场景 | 优点 | 缺点 | 工作原理 |
|---------|---------|------|------|---------|
| Stdio | 本地 Server | 简单、低延迟、无需网络 | 仅限本地、单客户端 | 通过进程 stdin/stdout 传递 NDJSON 消息 |
| HTTP+SSE | 远程 Server | 支持远程、多客户端、易部署 | 需要认证、延迟较高 | HTTP POST 发送请求，SSE 接收响应流 |

**传输层选择决策树**：

```
是否需要远程访问？
│
├─ 是 → 选择 HTTP+SSE
│   │
│   ├─ 是否需要多租户？
│   │   ├─ 是 → 添加 OAuth 2.0 认证 + 会话隔离
│   │   └─ 否 → 使用简单的 API Key 认证
│   │
│   └─ 是否处理敏感数据？
│       ├─ 是 → 启用 HTTPS + mTLS（可选）
│       └─ 否 → HTTP 即可
│
└─ 否 → 选择 Stdio
    │
    ├─ 是否需要多客户端共享？
    │   ├─ 是 → 考虑 HTTP+SSE 或命名管道
    │   └─ 否 → 使用 Stdio（推荐）
    │
    └─ 是否需要长期运行？
        ├─ 是 → 使用守护进程模式
        └─ 否 → 按需启动进程
```

**快速选择指南**：

| 你的需求 | 推荐方案 |
|---------|---------|
| Claude Desktop 本地集成 | Stdio |
| CLI 工具中的 MCP 支持 | Stdio |
| 远程 API 服务 | HTTP+SSE |
| 多租户 SaaS 服务 | HTTP+SSE + OAuth 2.0 |
| 企业内部工具 | HTTP+SSE + API Key |
| 高安全性要求 | HTTP+SSE + mTLS |

---

## 第三部分：JSON-RPC 2.0 协议基础

### 3.1 JSON-RPC 2.0 消息格式

#### 为什么 MCP 选择 JSON-RPC 而不是 HTTP REST？

这是一个值得深入探讨的设计决策。让我们对比一下两种方案：

| 特性 | JSON-RPC 2.0 | HTTP REST | 为什么 MCP 需要 |
|------|-------------|-----------|----------------|
| **消息格式** | 结构化 JSON | 依赖 HTTP 方法+URL | MCP 需要统一的调用格式 |
| **双向通信** | 天然支持（同连接来回通信） | 困难（需要 WebSocket） | MCP 需要 Server→Client 的采样请求 |
| **实时性** | 低延迟，适合本地 IPC | 较高 HTTP 开销 | MCP 强调低延迟交互 |
| **批处理** | 原生支持数组批量请求 | 需要自定义实现 | MCP 未来可能扩展批量操作 |
| **错误处理** | 统一错误码格式 | 依赖 HTTP 状态码 | MCP 需要细粒度的错误分类 |
| **无状态** | 每个请求独立 | 每个请求独立 | 便于水平扩展 |
| **生态成熟度** | 成熟，广泛使用 | 非常成熟 | 降低学习成本 |

**核心原因**：MCP 需要**双向实时通信**能力。

在 AI Agent 场景中，Server 可能需要：
- 请求 LLM 补全（sampling）
- 请求用户输入（elicitation）
- 推送工具变更通知

这些需求用 HTTP REST 实现会很笨拙（需要 WebSocket 或轮询），而 JSON-RPC 天然支持双向通信。

#### Request 格式

JSON-RPC 2.0 是一种轻量级的远程过程调用协议，使用 JSON 格式进行数据交换。

```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "method": "tools/list",
  "params": {}
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 默认值 | 说明 | 示例 |
|------|------|------|--------|------|------|
| jsonrpc | String | ✅ | 无 | 固定为 "2.0"，用于协议版本识别 | `"2.0"` |
| id | String/Number | ✅ | 无 | 请求 ID，用于匹配响应。**Notification 除外** | `"req-123"` 或 `1` |
| method | String | ✅ | 无 | 方法名称，通常使用 `/` 分隔命名空间 | `"tools/list"` |
| params | Object/Array | ❌ | `{}` | 方法参数，可以是对象或数组 | `{"city": "北京"}` |

**重要约定**：
- `id` 字段在同一连接中应该唯一，用于匹配请求和响应
- `id` 可以是字符串或数字，但建议保持一致性
- 如果 `id` 缺失，表示这是一个 Notification（无需响应）
- `params` 可以省略，此时默认为空对象 `{}`

#### Response 格式

**成功响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "result": {
    "tools": [
      {
        "name": "calculate",
        "description": "计算器",
        "inputSchema": {...}
      }
    ]
  }
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| jsonrpc | String | ✅ | 固定为 "2.0" |
| id | String/Number | ✅ | 必须与请求的 id 一致 |
| result | Object | ✅ | 方法返回结果（成功时） |
| result.tools | Array | - | tools/list 返回的工具列表 |

**失败响应**：
```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "error": {
    "code": -32601,
    "message": "Method not found",
    "data": "Additional error details"
  }
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| jsonrpc | String | ✅ | 固定为 "2.0" |
| id | String/Number | ✅ | 必须与请求的 id 一致 |
| error | Object | ✅ | 错误对象（失败时） |
| error.code | Number | ✅ | 错误码，见下表 |
| error.message | String | ✅ | 人类可读的错误描述 |
| error.data | Any | ❌ | 可选的额外错误信息 |

#### Notification 格式

通知是**无需响应**的请求（没有 id 字段）：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

**特点**：
- 没有 `id` 字段
- 接收方**不返回**响应
- 用于单向通知

**Notification vs Request 对比**：

| 特性 | Request | Notification |
|------|---------|-------------|
| id 字段 | ✅ 必须有 | ❌ 必须没有 |
| 响应 | ✅ 期待响应 | ❌ 无响应 |
| 使用场景 | 需要获取结果 | 仅通知状态变化 |
| 错误处理 | 返回 error 对象 | 无法获知是否失败 |

**常见 Notification 类型**：
- `notifications/initialized` - Client 通知 Server 初始化完成
- `notifications/tools/list_changed` - Server 通知 Client 工具列表已变更
- `notifications/resources/list_changed` - Server 通知 Client 资源列表已变更
- `notifications/progress` - 发送进度更新（可选）

### 3.2 标准错误码

#### JSON-RPC 标准错误码

| 错误码 | 含义 | 说明 | 触发场景 |
|--------|------|------|---------|
| -32700 | Parse error | JSON 解析失败 | 发送了无效的 JSON |
| -32600 | Invalid Request | 请求格式无效 | 缺少必填字段或字段类型错误 |
| -32601 | Method not found | 方法不存在 | 调用了未注册的方法 |
| -32602 | Invalid params | 参数无效 | 参数不符合 schema 定义 |
| -32603 | Internal error | 服务器内部错误 | Server 代码抛出异常 |

**错误码记忆技巧**：
- `-327xx` - 解析层错误
- `-326xx` - 协议层错误
- `-320xx` - 应用层错误（MCP 自定义）

#### MCP 自定义错误码

| 错误码 | 含义 | 触发场景 |
|--------|------|---------|
| -32000 | Server error (通用服务器错误) | 未分类的服务器错误 |
| -32001 | Resource not found | 请求的 URI 不存在 |
| -32002 | Prompt not found | 请求的提示词名称不存在 |
| -32003 | Tool execution failed | 工具执行过程中抛出异常 |

**错误响应最佳实践**：

```json
// 好的错误响应 - 提供有用的调试信息
{
  "jsonrpc": "2.0",
  "id": "1",
  "error": {
    "code": -32602,
    "message": "Invalid parameters",
    "data": {
      "field": "city",
      "reason": "Missing required field",
      "expected": "string",
      "received": null
    }
  }
}

// 不好的错误响应 - 信息不足
{
  "jsonrpc": "2.0",
  "id": "1",
  "error": {
    "code": -32602,
    "message": "Bad request"
  }
}
```

### 3.3 MCP 特定方法

MCP 在 JSON-RPC 2.0 基础上定义了特定方法：

#### 方法分类总览

| 类别 | 方法 | 方向 | 说明 | 是否需要响应 |
|------|------|------|------|-------------|
| **基础** | `initialize` | C→S | 初始化连接，交换能力信息 | ✅ |
| **基础** | `notifications/initialized` | C→S | 通知初始化完成 | ❌ |
| **Tools** | `tools/list` | C→S | 获取可用工具列表 | ✅ |
| **Tools** | `tools/call` | C→S | 调用指定工具 | ✅ |
| **Tools** | `notifications/tools/list_changed` | S→C | 工具列表变更通知 | ❌ |
| **Resources** | `resources/list` | C→S | 获取资源列表 | ✅ |
| **Resources** | `resources/read` | C→S | 读取指定资源内容 | ✅ |
| **Resources** | `notifications/resources/list_changed` | S→C | 资源列表变更通知 | ❌ |
| **Prompts** | `prompts/list` | C→S | 获取提示词模板列表 | ✅ |
| **Prompts** | `prompts/get` | C→S | 获取提示词详情 | ✅ |
| **Client** | `sampling/createMessage` | S→C | 请求 LLM 生成文本 | ✅ |
| **Client** | `elicitation/create` | S→C | 请求用户提供信息 | ✅ |

**方向说明**：
- `C→S`：Client 发送给 Server（请求 - 响应模式）
- `S→C`：Server 发送给 Client（Server 主动请求）

**方法命名约定**：
- 使用 `/` 分隔命名空间，如 `tools/list`
- Notification 使用 `notifications/` 前缀
- 动词使用原形（list、call、read、get）

---

## 第四部分：MCP 生命周期管理

### 4.1 初始化流程详解

#### 为什么需要初始化？

想象两个陌生人第一次合作：
1. 他们需要先**自我介绍**（协议版本、能力）
2. **确认对方能理解自己的语言**（协议兼容性）
3. **交换联系方式**（建立通信渠道）
4. 然后才能**开始正式工作**（调用工具、读取资源）

MCP 初始化流程就是 Client 和 Server 的"初次见面"过程。

MCP 连接的建立需要经过以下步骤：

```
Client                                    Server
   │                                         │
   │──── initialize request ────────────────►│
   │   {method: "initialize",                │
   │    params: {                            │
   │      protocolVersion: "2024-11-05",     │
   │      capabilities: {...},               │
   │      clientInfo: {...}}}                │
   │                                         │
   │◄─── initialize response ────────────────│
   │   {result: {                            │
   │      protocolVersion: "2024-11-05",     │
   │      capabilities: {...},               │
   │      serverInfo: {...}}}                │
   │                                         │
   │──── notifications/initialized ─────────►│
   │   (no response expected)                │
   │                                         │
   │         ★ 连接建立，可开始通信          │
   │                                         │
   │──── tools/list ────────────────────────►│
   │◄─── {tools: [...]} ─────────────────────│
   │                                         │
   │──── tools/call ────────────────────────►│
   │◄─── {result: {...}} ────────────────────│
```

**时序图详解**（按时间顺序）：

| 步骤 | 方向 | 消息类型 | 目的 | 关键内容 |
|------|------|---------|------|---------|
| 1 | C→S | initialize request | Client 发起连接 | 声明协议版本、自身能力 |
| 2 | S→C | initialize response | Server 响应 | 确认协议版本、返回服务能力 |
| 3 | C→S | notification | Client 确认初始化完成 | 无响应，标志着握手完成 |
| 4 | C→S | tools/list | Client 获取工具列表 | 可开始业务交互 |
| 5 | C→S | tools/call | Client 调用工具 | 实际业务操作 |

**代码示例：初始化请求**

```python
# Client 发送初始化请求
await transport.send({
    "jsonrpc": "2.0",
    "id": "1",
    "method": "initialize",
    "params": {
        "protocolVersion": "2024-11-05",      # MCP 协议版本（当前最新版）
        "capabilities": {                      # Client 能力声明
            "roots": {"listChanged": False},   # 是否支持根目录变更通知
            "sampling": {}                     # 是否支持 LLM 补全请求
        },
        "clientInfo": {                        # Client 身份信息
            "name": "my-client",
            "version": "1.0.0"
        }
    }
})

# Server 响应
{
    "jsonrpc": "2.0",
    "id": "1",
    "result": {
        "protocolVersion": "2024-11-05",      # 确认使用的协议版本
        "capabilities": {                      # Server 能力声明
            "tools": {"listChanged": False},   # 是否支持工具变更通知
            "resources": {"subscribe": False, "listChanged": False}
        },
        "serverInfo": {                        # Server 身份信息
            "name": "Weather Server",
            "version": "1.0.0"
        }
    }
}

# Client 发送初始化完成通知（无需响应）
await transport.send({
    "jsonrpc": "2.0",
    "method": "notifications/initialized"
})
```

**每行代码详解**：

1. `await transport.send({...}` - 通过传输层发送 JSON 消息
2. `"id": "1"` - 请求 ID，用于匹配后续的响应
3. `"protocolVersion": "2024-11-05"` - 声明 Client 支持的协议版本
4. `"capabilities"` - Client 声明自己支持的功能（如 sampling）
5. `"clientInfo"` - Client 的身份信息，便于 Server 日志记录
6. Server 响应中的 `"result"` - 包含 Server 的能力和身份信息
7. `notifications/initialized` - 通知 Server 可以开始处理业务请求了

**运行结果说明**：

初始化成功后，Client 和 Server 进入**就绪状态**，可以开始交换业务数据（工具列表、资源列表等）。

#### 如果初始化失败会怎样？

**常见失败场景**：

| 场景 | 原因 | 错误响应 | 处理建议 |
|------|------|---------|---------|
| 协议版本不匹配 | Client 和 Server 支持不同版本 | `{"code": -32602, "message": "Incompatible protocol version"}` | 降级到双方都支持的版本 |
| Server 繁忙 | Server 已达最大连接数 | `{"code": -32000, "message": "Server too busy"}` | 稍后重试 |
| 能力不兼容 | Server 需要某能力但 Client 不支持 | `{"code": -32602, "message": "Capability not supported"}` | 提示用户升级 Client |
| 网络中断 | 传输层连接断开 | 无响应（超时） | 重连机制 |

**故障处理最佳实践**：

```python
async def connect_with_retry(client, max_retries=3):
    """带重试的连接逻辑"""
    for attempt in range(max_retries):
        try:
            await client.connect()
            return True
        except ProtocolError as e:
            if e.code == -32602:  # 协议不兼容
                print(f"协议版本不兼容：{e.message}")
                return False  # 不重试
            elif attempt < max_retries - 1:
                await asyncio.sleep(2 ** attempt)  # 指数退避
            else:
                raise
    return False
```

### 4.2 能力协商详解

#### 生活化类比：交换名片

能力协商（Capability Negotiation）就像商务会议上的**交换名片**环节：

1. **Client 递出名片**：「您好，我是 XX 公司的代表，擅长 XX 领域...」
2. **Server 递出名片**：「您好，我是 YY 公司，提供 XX 服务...」
3. **双方确认合作可能**：「好的，我们有合作基础，可以开始谈业务了」

在 MCP 中，能力协商让 Client 和 Server 了解对方「能做什么」，避免后续调用失败。

在初始化阶段，Client 和 Server 交换各自支持的能力。

#### Client Capabilities

| 能力字段 | 说明 | 典型用途 | 示例值 |
|---------|------|---------|-------|
| `roots` | 支持目录/文件根列表变更通知 | 文件管理类应用 | `{"listChanged": true}` |
| `sampling` | 支持请求 LLM 补全 | Server 需要 AI 生成内容 | `{}`（空对象表示支持） |
| `elicitation` | 支持请求用户提供信息 | Server 需要用户输入 | `{}` |

**Client 能力示例**：
```json
{
  "roots": {
    "listChanged": true   // 支持根目录变更通知
  },
  "sampling": {},          // 支持 LLM 补全请求
  "elicitation": {}        // 支持用户输入请求
}
```

#### Server Capabilities

| 能力字段 | 说明 | 典型用途 | 示例值 |
|---------|------|---------|-------|
| `tools` | 提供工具调用能力 | 执行计算、API 调用等 | `{"listChanged": false}` |
| `resources` | 提供资源读取能力 | 读取文件、数据库记录 | `{"subscribe": false, "listChanged": false}` |
| `prompts` | 提供提示词模板能力 | 代码审查、翻译模板 | `{"listChanged": false}` |
| `logging` | 支持发送日志消息 | Server 发送日志到 Host | `{}` |

**Server 能力示例**：
```json
{
  "tools": {
    "listChanged": false   // 不支持工具变更通知
  },
  "resources": {
    "subscribe": false,    // 不支持资源订阅
    "listChanged": false   // 不支持资源变更通知
  },
  "prompts": {
    "listChanged": false   // 不支持提示词变更通知
  }
}
```

**能力字段详解**：

| 子字段 | 含义 | 启用后的效果 |
|-------|------|-------------|
| `listChanged` | 是否支持列表变更通知 | 启用后，Server 会在列表变化时主动通知 Client |
| `subscribe` | 是否支持资源订阅 | 启用后，Client 可以订阅特定资源的变化 |

**能力协商的实际意义**：

假设 Client 不支持 `sampling` 能力：
- Server 在调用 `sampling/createMessage` 之前会检查能力声明
- 如果 Client 未声明支持，Server 不应发起该请求
- 避免「对牛弹琴」式的无效调用

### 4.3 会话终止流程

#### 正常关闭（优雅退出）

**场景**：用户主动关闭应用、Server 需要重启更新

Client 主动关闭连接：
```python
await client.close()
```

**关闭流程**：

```
Client                                    Server
   │                                         │
   │──── close() ───────────────────────────►│
   │                                         │
   │  停止接收新消息                          │
   │  等待进行中的请求完成                     │
   │                                         │
   │                        清理资源          │
   │                        关闭连接          │
   │                                         │
   │  检测到 EOF                              │
   │  触发 on_close 回调                      │
   │                                         │
```

**Server 检测到 EOF 后清理资源**：
1. 关闭所有打开的文件句柄
2. 释放数据库连接
3. 清理会话状态
4. 记录关闭日志

#### 异常处理（意外断开）

**场景**：网络中断、进程崩溃、系统关机

网络中断或进程崩溃时：
1. Transport 层检测到连接断开
2. 触发 `on_close` 回调
3. 清理本地状态和资源

**常见异常场景**：

| 场景 | 检测方式 | 处理建议 |
|------|---------|---------|
| 网络中断 | TCP 连接断开/读取超时 | 实现自动重连机制 |
| Server 进程崩溃 | stdin 返回 EOF | 重启 Server 进程 |
| Client 进程崩溃 | 写入失败（Broken pipe） | 清理资源，等待重连 |
| 系统资源不足 | 内存分配失败 | 降级处理或退出 |

**异常处理最佳实践**：

```python
class ResilientMCPClient:
    def __init__(self, max_reconnect_attempts=3):
        self.max_reconnect_attempts = max_reconnect_attempts
        self.reconnect_delay = 1.0  # 秒

    async def connect_with_cleanup(self):
        """带清理和重试的连接逻辑"""
        for attempt in range(self.max_reconnect_attempts):
            try:
                await self.connect()
                return True
            except ConnectionError as e:
                logger.warning(f"连接失败 (尝试 {attempt + 1}/{self.max_reconnect_attempts}): {e}")
                await self.cleanup()  # 清理残留资源

                if attempt < self.max_reconnect_attempts - 1:
                    await asyncio.sleep(self.reconnect_delay * (2 ** attempt))  # 指数退避
                else:
                    logger.error("达到最大重连次数，放弃连接")
                    return False
        return False

    async def cleanup(self):
        """清理本地状态"""
        if self.transport:
            await self.transport.close()
        self.transport = None
        self.server_info = None
```

**类比：电话通话中断**：
- 正常挂断：双方道别后挂断电话
- 异常断开：信号不好突然断线，双方尝试回拨

#### 异常处理

网络中断或进程崩溃时：
1. Transport 层检测到连接断开
2. 触发 `on_close` 回调
3. 清理本地状态和资源

---

## 第五部分：MCP 核心原语详解

MCP 定义了三种核心原语（Primitive），它们是 MCP Server 提供给 Client 的基本能力：

```
┌─────────────────────────────────────────────────────────┐
│                   MCP 核心原语                           │
├─────────────────┬─────────────────┬─────────────────────┤
│     Tools       │    Resources    │      Prompts        │
│   (可执行函数)   │   (可读取数据)   │   (提示词模板)      │
├─────────────────┼─────────────────┼─────────────────────┤
│ 主动操作世界     │ 被动获取信息     │ 规范化交互模式      │
│ 如：搜索、计算    │ 如：文件、数据库  │ 如：代码审查模板    │
└─────────────────┴─────────────────┴─────────────────────┘
```

### 5.1 Tools（工具）

#### 为什么需要 Tools 原语？

**Tools 是 AI 的「手和脚」**——让 AI 能够执行操作，而不仅仅是生成文本。

想象一下：
- 没有工具的 AI = 只能纸上谈兵的军师
- 有工具的 AI = 可以调兵遣将的统帅

Tools 原语让 AI 能够：
- **获取实时信息**：查询天气、股票、新闻
- **执行实际操作**：发送邮件、创建文件、调用 API
- **扩展能力边界**：任何可编程的功能都可以封装为 Tool

#### 与 OpenAI Function Calling 的对比

MCP Tools 的设计灵感来自 OpenAI Function Calling，但有重要区别：

| 特性 | MCP Tools | OpenAI Function Calling |
|------|-----------|------------------------|
| **部署方式** | 独立进程，跨应用共享 | 绑定到单个 API 调用 |
| **通信协议** | JSON-RPC 2.0 | OpenAI 私有协议 |
| **发现机制** | `tools/list` 动态发现 | 静态传入 functions 参数 |
| **变更通知** | 支持 `listChanged` 通知 | 不支持 |
| **生态兼容性** | 多厂商支持 | 仅限 OpenAI 生态 |

**设计哲学差异**：
- OpenAI：集中式控制，函数定义随请求传入
- MCP：去中心化，Server 自主管理工具注册

#### tools/list - 获取工具列表

**方法说明**：Client 通过 `tools/list` 获取 Server 提供的所有可用工具。

**请求**：
```json
{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "tools/list",
    "params": {}
}
```

**请求详解**：
- `method`: `"tools/list"` - 命名空间 + 操作名
- `params`: 空对象，因为此方法不需要参数

**响应**：
```json
{
    "jsonrpc": "2.0",
    "id": "1",
    "result": {
        "tools": [
            {
                "name": "get_weather",
                "description": "获取指定城市的天气信息",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "city": {
                            "type": "string",
                            "description": "城市名称"
                        }
                    },
                    "required": ["city"]
                }
            }
        ]
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.tools` | Array | 工具列表数组 |
| `tools[].name` | String | 工具唯一标识符 |
| `tools[].description` | String | 工具描述，帮助 LLM 理解何时使用 |
| `tools[].inputSchema` | Object | JSON Schema，定义参数的格式和约束 |

**inputSchema 详解**：

`inputSchema` 遵循 **JSON Schema** 规范，定义工具参数的格式：

```json
{
  "type": "object",           // 参数是对象
  "properties": {             // 参数列表
    "city": {
      "type": "string",       // 参数类型
      "description": "..."    // 参数描述
    }
  },
  "required": ["city"]        // 必填参数
}
```

**支持的参数类型**：
- `string` - 字符串
- `number` - 数字（整数或浮点数）
- `integer` - 整数
- `boolean` - 布尔值
- `array` - 数组
- `object` - 对象
- `enum` - 枚举（限定取值）

#### tools/call - 调用工具

**方法说明**：Client 通过 `tools/call` 调用 Server 上注册的工具。

**请求**：
```json
{
    "jsonrpc": "2.0",
    "id": "2",
    "method": "tools/call",
    "params": {
        "name": "get_weather",
        "arguments": {
            "city": "北京"
        }
    }
}
```

**请求字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `method` | String | 固定为 `"tools/call"` |
| `params.name` | String | 要调用的工具名称（必须存在于 tools/list 返回的列表中） |
| `params.arguments` | Object | 工具的参数，必须符合该工具的 `inputSchema` 定义 |

**响应**：
```json
{
    "jsonrpc": "2.0",
    "id": "2",
    "result": {
        "content": [
            {
                "type": "text",
                "text": "北京天气：晴，25°C，湿度 30%，西北风 3 级"
            }
        ],
        "isError": false
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.content` | Array | 返回内容数组（可以包含多个内容项） |
| `content[].type` | String | 内容类型：`text`（文本）、`image`（图片）、`resource`（资源引用） |
| `content[].text` | String | 文本内容（当 type 为 text 时） |
| `result.isError` | Boolean | 是否为错误响应（`true` 表示工具执行失败） |

**错误响应示例**：

```json
{
    "jsonrpc": "2.0",
    "id": "2",
    "result": {
        "content": [
            {
                "type": "text",
                "text": "错误：未知城市「abc」，请输入有效的城市名称"
            }
        ],
        "isError": true
    }
}
```

**使用注意事项**：

1. **参数验证**：Server 应该验证 `arguments` 是否符合 `inputSchema`
2. **超时处理**：工具调用可能耗时，建议设置超时（如 30 秒）
3. **错误隔离**：工具执行失败不应影响整个 MCP 连接
4. **幂等性**：工具调用应该是幂等的（重复调用产生相同结果）

**完整交互流程示例**：

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │    Server   │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │ tools/list                       │
       │─────────────────────────────────►│
       │                                  │
       │ tools: [                         │
       │   {name: "get_weather", ...}     │
       │ ]                                │
       │◄─────────────────────────────────│
       │                                  │
       │ tools/call (get_weather)         │
       │─────────────────────────────────►│
       │                                  │
       │ 执行天气查询                     │
       │                                  │
       │ result: "北京天气：晴，25°C"      │
       │◄─────────────────────────────────│
       │                                  │
```

### 5.2 Resources（资源）

#### 为什么需要 Resources 原语？

**Resources 是 AI 的「图书馆」**——提供静态数据源，让 AI 获取上下文信息。

**Resources vs Tools 的区别**：

| 特性 | Tools | Resources |
|------|-------|-----------|
| **操作类型** | 动态执行（写操作） | 静态读取（读操作） |
| **返回结果** | 执行结果 | 原始数据 |
| **类比** | 函数调用 | 文件读取 |
| **典型用途** | 搜索、计算、API 调用 | 配置文件、文档、数据库记录 |

**生活化类比**：
- **Tools** = 餐厅的厨师（现做菜品）
- **Resources** = 餐厅的菜单和酒窖（已有库存）

#### resources/list - 获取资源列表

**方法说明**：Client 通过 `resources/list` 获取 Server 提供的所有可读取资源。

**请求**：
```json
{
    "jsonrpc": "2.0",
    "id": "3",
    "method": "resources/list",
    "params": {}
}
```

**响应**：
```json
{
    "jsonrpc": "2.0",
    "id": "3",
    "result": {
        "resources": [
            {
                "uri": "file:///config/app.json",
                "name": "应用程序配置",
                "description": "应用程序的 JSON 配置文件",
                "mimeType": "application/json"
            },
            {
                "uri": "file:///docs/readme.txt",
                "name": "说明文档",
                "description": "项目说明文档",
                "mimeType": "text/plain"
            }
        ]
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.resources` | Array | 资源列表数组 |
| `resources[].uri` | String | 资源的唯一标识符（URI 格式） |
| `resources[].name` | String | 人类可读的资源名称 |
| `resources[].description` | String | 资源描述，帮助理解资源用途 |
| `resources[].mimeType` | String | 资源的 MIME 类型，帮助解析 |

#### resources/read - 读取资源

**方法说明**：Client 通过 `resources/read` 读取指定 URI 的资源内容。

**请求**：
```json
{
    "jsonrpc": "2.0",
    "id": "4",
    "method": "resources/read",
    "params": {
        "uri": "file:///config/app.json"
    }
}
```

**响应**：
```json
{
    "jsonrpc": "2.0",
    "id": "4",
    "result": {
        "contents": [
            {
                "uri": "file:///config/app.json",
                "mimeType": "application/json",
                "text": "{\n  \"app_name\": \"MCP Tutorial\",\n  \"version\": \"1.0.0\"\n}"
            }
        ]
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.contents` | Array | 返回内容数组（可能包含多个内容项） |
| `contents[].uri` | String | 资源的 URI（与请求一致） |
| `contents[].mimeType` | String | 资源的 MIME 类型 |
| `contents[].text` | String | 资源的文本内容 |
| `contents[].blob` | String | 资源的二进制内容（Base64 编码，可选） |

**text vs blob 的区别**：

| 字段 | 适用场景 | 编码方式 |
|------|---------|---------|
| `text` | 文本文件（JSON、TXT、MD 等） | UTF-8 字符串 |
| `blob` | 二进制文件（图片、音频等） | Base64 编码 |

**使用示例**：

```json
// 读取图片资源
{
  "uri": "file:///images/logo.png",
  "mimeType": "image/png",
  "blob": "iVBORw0KGgoAAAANSUhEUgAA..."  // Base64 编码的图片数据
}
```

**URI 模式详解**：

URI（Uniform Resource Identifier）是资源的唯一标识符。MCP 支持多种 URI 模式：

| URI 模式 | 说明 | 示例 | 应用场景 |
|---------|------|------|---------|
| `file://` | 文件系统路径 | `file:///etc/hosts` | 读取本地文件 |
| `postgres://` | PostgreSQL 数据库 | `postgres://localhost/mydb/users` | 查询数据库记录 |
| `http://` / `https://` | 网络资源 | `https://api.example.com/data` | 访问 Web API |
| `config://` | 自定义配置资源 | `config://app/settings` | 应用配置 |
| `memory://` | 内存中的资源 | `memory://session/context` | 会话上下文 |

**URI 最佳实践**：

1. **使用绝对路径**：避免相对路径导致的歧义
2. **URI 编码**：特殊字符需要 URL 编码（如空格→`%20`）
3. **命名空间隔离**：使用 URI 前缀区分不同资源类型

**URI 安全性检查**：

```python
from pathlib import Path
from urllib.parse import urlparse

def validate_file_uri(uri: str, allowed_base: str) -> bool:
    """验证文件 URI 是否在允许的范围内"""
    if not uri.startswith("file://"):
        return False

    # 解析 URI
    parsed = urlparse(uri)
    file_path = Path(parsed.path)

    # 确保在允许的目录内
    allowed = Path(allowed_base).resolve()
    resolved = file_path.resolve()

    try:
        resolved.relative_to(allowed)  # 检查是否在 allowed 的子目录
        return True
    except ValueError:
        return False

# 使用示例
is_valid = validate_file_uri(
    "file:///data/config/app.json",
    "/data"  # 只允许访问 /data 目录下的文件
)
```

### 5.3 Prompts（提示词）

#### 为什么需要 Prompts 原语？

**Prompts 是 AI 的「剧本」**——提供预定义的交互模板，规范化 AI 行为。

**实际应用场景**：

| 场景 | 问题 | Prompts 解决方案 |
|------|------|-----------------|
| 代码审查 | 每次审查标准不一致 | 统一的审查清单模板 |
| 翻译工作 | 翻译风格不稳定 | 标准化的翻译指令 |
| 数据分析 | 分析方法碎片化 | 结构化的分析框架 |

**生活化类比**：
- **没有 Prompts** = 即兴演讲（质量不稳定）
- **有 Prompts** = 按稿演讲（质量可控）

#### prompts/list - 获取提示词列表

**方法说明**：Client 通过 `prompts/list` 获取 Server 提供的所有提示词模板。

**请求**：
```json
{
    "jsonrpc": "2.0",
    "id": "5",
    "method": "prompts/list",
    "params": {}
}
```

**响应**：
```json
{
    "jsonrpc": "2.0",
    "id": "5",
    "result": {
        "prompts": [
            {
                "name": "code_review",
                "description": "代码审查提示模板",
                "arguments": [
                    {
                        "name": "code",
                        "description": "要审查的代码",
                        "required": true
                    }
                ]
            },
            {
                "name": "translate",
                "description": "翻译提示模板",
                "arguments": [
                    {
                        "name": "text",
                        "description": "要翻译的文本",
                        "required": true
                    },
                    {
                        "name": "target_lang",
                        "description": "目标语言",
                        "required": true
                    }
                ]
            }
        ]
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.prompts` | Array | 提示词模板列表 |
| `prompts[].name` | String | 模板唯一标识符 |
| `prompts[].description` | String | 模板用途描述 |
| `prompts[].arguments` | Array | 模板参数列表（可选） |
| `arguments[].name` | String | 参数名称 |
| `arguments[].description` | String | 参数描述 |
| `arguments[].required` | Boolean | 是否为必填参数 |

#### prompts/get - 获取提示词详情

**方法说明**：Client 通过 `prompts/get` 获取指定提示词模板的完整内容。

**请求**：
```json
{
    "jsonrpc": "2.0",
    "id": "6",
    "method": "prompts/get",
    "params": {
        "name": "code_review"
    }
}
```

**响应**：
```json
{
    "jsonrpc": "2.0",
    "id": "6",
    "result": {
        "description": "代码审查提示模板",
        "messages": [
            {
                "role": "system",
                "content": {
                    "type": "text",
                    "text": "你是一个专业的代码审查员。请检查代码的质量、潜在 bug、性能问题和安全风险。"
                }
            },
            {
                "role": "user",
                "content": {
                    "type": "text",
                    "text": "请审查以下代码：\n\n{code}"
                }
            }
        ]
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.description` | String | 模板描述 |
| `result.messages` | Array | 消息列表（构成完整对话） |
| `messages[].role` | String | 角色：`system`、`user`、`assistant` |
| `messages[].content` | Object | 消息内容 |
| `content.type` | String | 内容类型：`text`（文本） |
| `content.text` | String | 文本内容（支持参数占位符） |

**消息角色说明**：

| 角色 | 用途 | 典型内容 |
|------|------|---------|
| `system` | 设定 AI 的行为准则 | 「你是一个专业的代码审查员...」 |
| `user` | 用户输入的内容 | 「请审查以下代码...」 |
| `assistant` | AI 的示例回复（可选） | 「好的，我来审查这段代码...」 |

**参数占位符详解**：

`{code}` 是参数占位符，在使用时会被实际值替换：

```python
# 伪代码示例
prompt = await client.get_prompt("code_review", arguments={"code": "def foo():..."})
# {code} 被替换为实际的代码内容
messages = prompt.messages  # 此时代码已注入到消息中
```

**实际应用场景**：

1. **代码审查工作流**：
   - 开发者在 IDE 中选择一段代码
   - 触发「代码审查」Prompt
   - AI 按预定义的审查清单进行检查
   - 返回结构化的审查报告

2. **翻译工作流**：
   - 用户提供待翻译文本和目标语言
   - 使用「专业翻译」Prompt
   - AI 按预定义的翻译规范执行
   - 返回翻译结果和术语表

### 5.4 Client 原语（Server 可调用的 Client 能力）

#### 为什么需要 Client 原语？

MCP 的设计是**双向的**——不仅 Client 可以调用 Server，Server 也可以请求 Client 的能力。

**双向通信的意义**：

```
┌─────────────────────────────────────────────────────────┐
│                  MCP 双向通信                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Client ──────────────► Server                         │
│   - tools/list          - tools 定义                    │
│   - tools/call          - resources 读取                │
│   - resources/read      - prompts 获取                  │
│   - prompts/get                                         │
│                                                         │
│   Server ──────────────► Client                         │
│   - sampling/createMessage (请求 LLM 补全)              │
│   - elicitation/create     (请求用户输入)               │
│   - logging                  (发送日志)                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**生活化类比**：
- **Server 调用 Client** = 餐厅厨师请服务员帮忙
  - 「帮我问问顾客要不要加辣」（elicitation）
  - 「帮我尝尝这道菜的味道」（sampling）

#### sampling/createMessage - 请求 LLM 补全

**方法说明**：Server 通过 `sampling/createMessage` 请求 Client 调用 LLM 生成文本。

**典型应用场景**：
- Server 需要生成一段解释性文本
- Server 需要对用户输入进行分类
- Server 需要提取结构化数据

**请求** (Server → Client)：
```json
{
    "jsonrpc": "2.0",
    "id": "7",
    "method": "sampling/createMessage",
    "params": {
        "messages": [
            {
                "role": "user",
                "content": {
                    "type": "text",
                    "text": "请用一句话总结今天的天气"
                }
            }
        ],
        "maxTokens": 100,
        "temperature": 0.7,
        "systemPrompt": "你是一个简洁的天气播报员"
    }
}
```

**请求字段详解**：

| 字段路径 | 类型 | 必填 | 说明 |
|---------|------|------|------|
| `params.messages` | Array | ✅ | 对话消息列表 |
| `params.maxTokens` | Number | ❌ | 最大生成 token 数 |
| `params.temperature` | Number | ❌ | 温度（0-1，越高越随机） |
| `params.systemPrompt` | String | ❌ | 系统提示词 |

**响应** (Client → Server)：
```json
{
    "jsonrpc": "2.0",
    "id": "7",
    "result": {
        "role": "assistant",
        "content": {
            "type": "text",
            "text": "今天各地天气晴朗，气温适宜，北京 25°C，上海 28°C，适合户外活动。"
        },
        "model": "claude-sonnet-4-20250514",
        "stopReason": "endTurn"
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.role` | String | 固定为 `"assistant"` |
| `result.content` | Object | 生成的内容 |
| `result.model` | String | 使用的模型名称 |
| `result.stopReason` | String | 停止原因：`endTurn`（自然结束）、`maxTokens`（达到上限） |

#### elicitation/create - 请求用户提供信息

**方法说明**：Server 通过 `elicitation/create` 请求 Client 向用户获取信息。

**典型应用场景**：
- Server 需要用户确认敏感操作
- Server 需要用户在多个选项中选择
- Server 需要用户补充缺失的参数

**与 sampling 的区别**：
- `sampling` = Server 请求 AI 生成内容
- `elicitation` = Server 请求**用户**提供输入

**请求** (Server → Client)：
```json
{
    "jsonrpc": "2.0",
    "id": "8",
    "method": "elicitation/create",
    "params": {
        "message": "请选择要查询的城市",
        "requestedSchema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "enum": ["北京", "上海", "广州", "深圳"]
                }
            },
            "required": ["city"]
        }
    }
}
```

**请求字段详解**：

| 字段路径 | 类型 | 必填 | 说明 |
|---------|------|------|------|
| `params.message` | String | ✅ | 向用户显示的消息 |
| `params.requestedSchema` | Object | ✅ | 期望的用户输入格式（JSON Schema） |

**响应** (Client → Server)：
```json
{
    "jsonrpc": "2.0",
    "id": "8",
    "result": {
        "action": "accept",
        "content": {
            "city": "北京"
        }
    }
}
```

**响应字段详解**：

| 字段路径 | 类型 | 说明 |
|---------|------|------|
| `result.action` | String | 用户操作：`accept`（接受）、`decline`（拒绝） |
| `result.content` | Object | 用户输入的内容（符合 requestedSchema） |

**用户拒绝的响应示例**：

```json
{
    "jsonrpc": "2.0",
    "id": "8",
    "result": {
        "action": "decline"
    }
}
```

**使用注意事项**：

1. **用户友好**：message 应该清晰说明需要什么信息
2. **_schema 验证**：Client 应该验证用户输入是否符合 requestedSchema
3. **超时处理**：用户可能长时间不响应，需要设置超时
4. **隐私保护**：敏感信息（如密码）不应通过 elicitation 获取

---

## 第六部分：Transport Layer 详解

MCP 的传输层（Transport Layer）负责消息的可靠传输，支持两种传输方式：

```
┌─────────────────────────────────────────────────────────┐
│              MCP Transport Layer                         │
├──────────────────────────┬──────────────────────────────┤
│     Stdio Transport      │    Streamable HTTP           │
│     (本地进程通信)        │    (远程网络通信)             │
├──────────────────────────┼──────────────────────────────┤
│  优点：                  │  优点：                      │
│  - 简单可靠              │  - 支持远程                  │
│  - 低延迟                │  - 多客户端共享              │
│  - 天然隔离              │  - 易于部署                  │
├──────────────────────────┼──────────────────────────────┤
│  缺点：                  │  缺点：                      │
│  - 仅限本地              │  - 需要认证                  │
│  - 单客户端              │  - 延迟较高                  │
│  - 进程管理复杂          │  - 实现复杂                  │
└──────────────────────────┴──────────────────────────────┘
```

### 6.1 Stdio Transport

**适用场景**：本地 MCP Server

#### 工作原理

Stdio Transport 使用进程的 stdin/stdout 进行通信：

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │    Server   │
│             │    stdout/stdin    │             │
│  read()     │◄──────────────────►│  write()    │
│  write()    │                    │  read()     │
└─────────────┘                    └─────────────┘
```

**通信流程详解**：

1. **Client 启动 Server 进程**：通过 `subprocess` 模块创建子进程
2. **建立管道**：Client 的 stdout 连接到 Server 的 stdin
3. **双向通信**：
   - Client → Server：通过 Server 的 stdin 写入
   - Server → Client：通过 Server 的 stdout 读取
4. **日志分离**：Server 的日志输出到 stderr，避免污染通信流

#### 消息帧格式

使用 **Newline-delimited JSON** (NDJSON) 格式：

```
{"jsonrpc":"2.0","id":"1","method":"initialize",...}\n
{"jsonrpc":"2.0","id":"1","result":{...}}\n
```

每条消息以换行符 `\n` 结尾。

**NDJSON 格式详解**：

NDJSON（Newline-delimited JSON）是一种简单的流式 JSON 格式：

| 特性 | 说明 | 优势 |
|------|------|------|
| **每行一条消息** | 每条 JSON 消息占一行 | 易于解析，无需 buffering |
| **换行符分隔** | 使用 `\n` 分隔消息 | 与 Unix 工具兼容 |
| **无数组包裹** | 消息独立，不在数组中 | 支持流式处理 |

**为什么选择 NDJSON 而不是 JSON 数组？**

```
// JSON 数组 - 需要等待整个数组加载
[
  {"jsonrpc": "2.0", "id": "1", ...},
  {"jsonrpc": "2.0", "id": "2", ...},
  ...  // 必须等待所有消息到达才能开始解析
]

// NDJSON - 可以边接收边处理
{"jsonrpc": "2.0", "id": "1", ...}\n   ← 立即解析
{"jsonrpc": "2.0", "id": "2", ...}\n   ← 立即解析
```

**NDJSON 解析示例**：

```python
import json

async def read_ndjson_stream(reader):
    """从 stdio 读取 NDJSON 消息流"""
    while True:
        # 读取一行（直到换行符）
        line = await reader.readline()
        if not line:
            break  # EOF，连接结束

        # 解析 JSON
        try:
            message = json.loads(line.decode('utf-8'))
            yield message
        except json.JSONDecodeError as e:
            print(f"JSON 解析错误：{e}")
            continue  # 跳过无效消息
```

#### 代码示例

**Server 端**：
```python
async def read_loop():
    while True:
        # 读取一行（一条 JSON 消息）
        line = await loop.run_in_executor(None, sys.stdin.readline)
        if not line:
            break  # EOF，Client 关闭了连接

        # 解析 JSON
        data = json.loads(line.strip())
        request = JSONRPCRequest.from_dict(data)

        # 处理请求
        response = await self.handle_request(request)

        # 写回响应（注意 flush=True，确保立即发送）
        print(json.dumps(response.to_dict()), flush=True)
```

**代码详解**：

| 代码行 | 作用 | 关键点 |
|-------|------|--------|
| `sys.stdin.readline` | 从标准输入读取一行 | 阻塞操作，需用 `run_in_executor` |
| `if not line: break` | 检测 EOF | Client 关闭时 stdin 返回空 |
| `json.loads(line.strip())` | 解析 JSON | `strip()` 去除换行符 |
| `print(..., flush=True)` | 输出响应 | `flush=True` 确保立即发送，不缓冲 |

**Client 端**：
```python
# 启动子进程
process = await asyncio.create_subprocess_exec(
    "python",
    "mcp_server.py",
    stdin=asyncio.subprocess.PIPE,
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.PIPE,  # 日志输出
)

# 发送消息
message = json.dumps({"jsonrpc": "2.0", "id": "1", "method": "tools/list"})
process.stdin.write((message + "\n").encode())  # 别忘了换行符！
await process.stdin.drain()  # 确保发送完成

# 读取响应
response = await process.stdout.readline()
data = json.loads(response.decode())
```

**代码详解**：

| 代码行 | 作用 | 关键点 |
|-------|------|--------|
| `create_subprocess_exec` | 创建子进程 | 使用 PIPE 建立通信管道 |
| `stderr=asyncio.subprocess.PIPE` | 捕获日志 | Server 日志输出到 stderr |
| `write((message + "\n").encode())` | 发送消息 | 必须添加换行符，否则 Server 不会解析 |
| `await drain()` | 确保发送 | 等待数据写入内核缓冲区 |

#### 优缺点分析

| 优点 | 说明 | 应用场景 |
|------|------|---------|
| 简单可靠，无需额外依赖 | 只需标准输入输出，无需网络库 | 桌面应用、CLI 工具 |
| 低延迟，直接进程间通信 | 微秒级延迟，适合频繁交互 | 实时性要求高的场景 |
| 天然隔离，每个 Client 独立 Server | 进程级别的资源隔离 | 安全性要求高的场景 |
| 无需认证 | 进程间通信，外部无法访问 | 本地工具集成 |

| 缺点 | 说明 | 规避方案 |
|------|------|---------|
| 仅限本地通信 | 无法跨网络访问 | 需要远程时使用 HTTP |
| 单客户端，无法共享 | 每个 Client 启动独立 Server | 使用 HTTP 传输 |
| 进程管理复杂 | 需要处理进程启动、退出、异常 | 使用进程池或守护进程 |
| 资源占用较高 | 每个连接一个进程 | 限制最大连接数 |

#### 选择建议

**使用 Stdio 的场景**：
- Claude Desktop 本地集成
- CLI 工具中的 MCP 支持
- 需要访问本地文件/资源

**不使用 Stdio 的场景**：
- 需要远程访问
- 需要多客户端共享
- Server 需要长期运行

### 6.2 Streamable HTTP Transport

**适用场景**：远程 MCP Server

#### 工作原理

使用 HTTP POST + SSE (Server-Sent Events) 进行通信：

```
┌─────────────┐                    ┌─────────────┐
│   Client    │                    │    Server   │
│             │   HTTP POST        │             │
│  send()     │───────────────────►│  receive()  │
│             │                    │             │
│  receive()  │◄───────────────────│  send()     │
│             │   SSE Stream       │             │
└─────────────┘                    └─────────────┘
```

**通信流程详解**：

1. **Client 发送请求**：通过 HTTP POST 发送 JSON-RPC 请求
2. **Server 接收并处理**：解析请求，执行业务逻辑
3. **Server 发送响应**：通过 SSE 流式返回响应
4. **保持连接**：SSE 连接保持打开，支持服务器推送

**SSE（Server-Sent Events）详解**：

SSE 是一种服务器向客户端推送事件的技术：

| 特性 | 说明 |
|------|------|
| **单向推送** | 只能 Server→Client，Client→Server 需用 HTTP POST |
| **文本格式** | 基于文本的协议，易于调试 |
| **自动重连** | 浏览器/客户端自动处理重连 |
| **二进制支持** | 支持 base64 编码传输二进制数据 |

**与传统 HTTP 请求的对比**：

```
传统 HTTP 请求-响应模式：
Client ─────► Server ─────► Client
   请求         处理         响应
              (Client 等待)

SSE 流式响应模式：
Client ─────► Server ──┬──► Client
   请求                 │   响应 1
                       ├──► Client
                       │   响应 2
                       └──► Client
                           响应 3 (推送)
```

#### 消息格式

**请求** (Client → Server)：
```http
POST /mcp HTTP/1.1
Host: example.com
Content-Type: application/json
Accept: text/event-stream

{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "tools/list"
}
```

**请求头详解**：

|  Header | 值 | 说明 |
|--------|-----|------|
| `Content-Type` | `application/json` | 请求体是 JSON 格式 |
| `Accept` | `text/event-stream` | 期望 SSE 格式响应 |
| `Authorization` | `Bearer xxx` | 认证 Token（可选） |

**响应** (Server → Client via SSE)：
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"jsonrpc":"2.0","id":"1","result":{...}}

```

**响应头详解**：

| Header | 值 | 说明 |
|--------|-----|------|
| `Content-Type` | `text/event-stream` | SSE 格式 |
| `Cache-Control` | `no-cache` | 不缓存，实时传输 |
| `Connection` | `keep-alive` | 保持连接 |

**SSE 消息格式**：

```
data: {"jsonrpc":"2.0","id":"1","result":{...}}\n\n
data: {"jsonrpc":"2.0","id":"2","result":{...}}\n\n
```

- `data:` 前缀表示 SSE 数据行
- 两个换行符 `\n\n` 表示消息结束

#### 认证方式

远程 MCP Server 需要认证来保护资源：

| 认证方式 | 说明 | 安全等级 | 适用场景 |
|---------|------|---------|---------|
| OAuth 2.0 | 标准 OAuth 2.0 授权流程 | 高 | 多租户 SaaS 服务 |
| Bearer Token | 简单的 Token 认证 | 中 | 内部服务、个人项目 |
| API Key | 通过 Header 传递 API Key | 中 | 服务间认证 |
| mTLS | 双向 TLS 证书认证 | 极高 | 企业级安全要求 |

**示例**：

Bearer Token 认证：
```http
Authorization: Bearer your_access_token
```

API Key 认证：
```http
X-API-Key: your_api_key
```

OAuth 2.0 完整流程：
```
1. Client 请求授权 → Authorization Server
2. 用户确认授权 → 返回 authorization_code
3. Client 换取 Token → 返回 access_token
4. Client 使用 Token 访问 MCP Server
```

#### 优缺点分析

| 优点 | 说明 | 应用场景 |
|------|------|---------|
| 支持远程通信 | 可跨网络访问 | 云服务、分布式部署 |
| 支持多客户端 | 多个 Client 共享 Server | 团队协作场景 |
| 易于部署和扩展 | 标准 HTTP 服务，可用负载均衡 | 生产环境部署 |
| 生态丰富 | 可利用成熟的 HTTP 生态 | 认证、监控、日志 |

| 缺点 | 说明 | 规避方案 |
|------|------|---------|
| 需要处理认证 | 远程访问需要安全措施 | 使用 OAuth 2.0 或 API Key |
| 延迟相对较高 | 网络传输 + HTTP 开销 | 使用连接池、HTTP/2 |
| 需要 HTTP 服务器 | 增加部署复杂度 | 使用轻量框架如 FastAPI |
| 状态管理复杂 | 需要考虑会话状态 | 使用无状态设计或外部存储 |

### 6.3 传输层对比

| 特性 | Stdio | HTTP |
|------|-------|------|
| 通信范围 | 本地 | 远程 |
| 延迟 | 极低 (μs 级) | 中等 (ms 级) |
| 认证 | 简单 (进程隔离) | OAuth 2.0 / Token |
| 扩展性 | 单客户端 | 多客户端 |
| 部署复杂度 | 低 | 中 |
| 适用场景 | 桌面应用、CLI | 云服务、Web 应用 |

---

## 第七部分：完整代码示例

### 7.1 简单 MCP Server 实现

创建 `examples/mcp_weather_server.py`：

```python
#!/usr/bin/env python3
"""
完整 MCP Server 示例 - 天气查询服务

功能说明：
- 实现一个天气查询工具（模拟数据）
- 使用 stdio 传输方式
- 可与任何 MCP Client（如 Claude Desktop）集成

运行方式：
    python examples/mcp_weather_server.py

测试方式（使用 MCP Inspector）：
    npx @modelcontextprotocol/inspector python examples/mcp_weather_server.py
"""

import sys
import json
from src.mcp import MCPServer, ServerConfig
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class WeatherTool(BaseTool):
    """天气查询工具（模拟数据）

    这个工具模拟天气查询 API，返回预定义的天气数据。
    在实际应用中，这里可以调用真实的天气 API。
    """

    @property
    def name(self):
        """工具名称：在 MCP 中的唯一标识符"""
        return "get_weather"

    @property
    def description(self):
        """工具描述：帮助 LLM 理解何时使用此工具

        LLM 会根据这个描述决定是否调用此工具。
        描述应该清晰说明工具的用途和适用场景。
        """
        return "获取指定城市的天气信息"

    @property
    def parameters(self):
        """工具参数：定义工具的输入格式（JSON Schema）

        这个 schema 会告诉 LLM：
        1. 需要提供什么参数
        2. 参数的类型和约束
        3. 哪些参数是必填的
        """
        return {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如'北京'、'上海'"
                }
            },
            "required": ["city"]  # city 是必填参数
        }

    def execute(self, **kwargs):
        """执行工具：实现具体的业务逻辑

        参数：
            **kwargs: 工具调用时传入的参数，这里是 city

        返回：
            ToolResult: 包含执行状态和输出结果

        注意：
        - 应该进行参数验证
        - 应该处理异常情况
        - 返回结果应该是 LLM 可读的文本格式
        """
        city = kwargs.get("city", "")

        # 模拟天气数据（实际应用中调用真实 API）
        mock_data = {
            "北京": "晴，25°C，湿度 30%",
            "上海": "多云，28°C，湿度 60%",
            "广州": "雨，30°C，湿度 80%",
            "深圳": "晴，32°C，湿度 70%",
        }

        # 查询天气
        result = mock_data.get(city, "未知城市，暂无数据")

        # 返回结果
        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=f"{city}天气：{result}"
        )


if __name__ == "__main__":
    # 创建服务器配置
    # name: Server 名称，会显示在 Client 端
    # version: Server 版本号，便于版本管理
    config = ServerConfig(name="Weather Server", version="1.0.0")

    # 创建 MCP Server 实例
    server = MCPServer(config=config)

    # 注册工具
    # 注册后的工具可以通过 tools/list 和 tools/call 访问
    server.register_tool(WeatherTool())

    # 启动服务器（使用 stdio 传输）
    # 日志输出到 stderr，避免污染 stdout 通信流
    print("Starting MCP Weather Server...", file=sys.stderr)
    server.run_stdio()
```

**运行并测试**：

```bash
# 方式 1：直接启动 Server（等待客户端连接）
python examples/mcp_weather_server.py

# 方式 2：使用 MCP Inspector 测试（推荐）
npx @modelcontextprotocol/inspector python examples/mcp_weather_server.py

# 方式 3：配置到 Claude Desktop 使用
# 在 claude_desktop_config.json 中添加：
# {
#   "mcpServers": {
#     "weather": {
#       "command": "python",
#       "args": ["/absolute/path/to/mcp_weather_server.py"]
#     }
#   }
# }
```

**参考输出**（stderr 日志）：
```
2025-03-02 10:00:00 - INFO - Initialized MCP Server: Weather Server
2025-03-02 10:00:01 - INFO - Starting MCP server with stdio transport
2025-03-02 10:00:02 - DEBUG - Received: {"jsonrpc":"2.0","id":"1","method":"initialize",...}
2025-03-02 10:00:02 - DEBUG - Sent: {"jsonrpc":"2.0","id":"1","result":{"protocolVersion":"2024-11-05",...}
```

**代码是如何运行的**（分步说明）：

```
步骤 1: 启动 Server 进程
   └─> python mcp_weather_server.py
       └─> 创建 MCPServer 实例
       └─> 注册 WeatherTool
       └─> 启动 stdio 监听

步骤 2: Client 连接（如 Claude Desktop）
   └─> 创建子进程连接到 Server
   └─> 发送 initialize 请求

步骤 3: 能力协商
   └─> Server 响应 capabilities
   └─> Client 发送 initialized 通知

步骤 4: 工具调用
   └─> Client 请求 tools/list
   └─> Server 返回 [get_weather]
   └─> Client 请求 tools/call(get_weather, city="北京")
   └─> Server 执行 WeatherTool.execute()
   └─> 返回"北京天气：晴，25°C"

步骤 5: 关闭连接
   └─> Client 调用 close()
   └─> Server 检测到 EOF，清理资源
   └─> Server 进程退出
```

**常见问题**：

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| Server 不启动 | Python 路径错误 | 使用 `which python` 确认路径 |
| 工具不显示 | 未正确注册 | 检查 `register_tool()` 调用 |
| 调用失败 | 参数不匹配 | 确保参数符合 `inputSchema` |
| 日志看不到 | 输出到 stdout | 日志应该输出到 `stderr` |

### 7.2 MCP Client 连接示例

创建 `examples/mcp_client_test.py`：

```python
#!/usr/bin/env python3
"""
MCP Client 连接示例

功能说明：
- 演示如何使用 MCPClient 连接 Server
- 展示完整的初始化和工具调用流程
- 可用作测试自定义 MCP Server 的客户端脚本

前置条件：
- 确保 mcp_weather_server.py 在当前目录
- 或者修改 connect_stdio 中的路径

运行方式：
    python examples/mcp_client_test.py
"""

import asyncio
import sys
sys.path.insert(0, "..")  # 导入 src 模块

from src.mcp import MCPClient


async def main():
    """主函数：演示 MCP Client 的完整使用流程"""

    # 步骤 1：创建 Client 实例
    client = MCPClient()

    try:
        # 步骤 2：连接到 Server
        # connect_stdio 会启动一个子进程并建立 stdio 连接
        # command: 启动命令
        # args: 命令参数列表
        await client.connect_stdio(
            command="python",
            args=["mcp_weather_server.py"],
        )

        # 步骤 3：获取服务器信息
        # server_info 在初始化时自动获取
        print(f"Connected to: {client.server_info.name}")
        print(f"Server version: {client.server_info.version}")
        print(f"MCP Protocol Version: {client.protocol_version}")

        # 步骤 4：列出可用工具
        # 这相当于发送 tools/list 请求
        tools = await client.list_tools()
        print(f"\nAvailable tools:")
        for tool in tools:
            print(f"  - {tool.name}: {tool.description}")

        # 步骤 5：调用工具
        # 这相当于发送 tools/call 请求
        # name: 工具名称
        # arguments: 工具参数（必须符合 inputSchema）
        print("\nCalling get_weather tool...")
        result = await client.call_tool("get_weather", {"city": "北京"})

        # 步骤 6：处理结果
        # result.content 是一个列表，包含返回的内容项
        print(f"Result: {result.content[0].text}")

    except Exception as e:
        # 错误处理
        print(f"Error: {e}")
        raise
    finally:
        # 步骤 7：关闭连接
        # 这会关闭 stdio 管道并终止 Server 进程
        await client.close()
        print("\nConnection closed.")


if __name__ == "__main__":
    # 运行异步主函数
    asyncio.run(main())
```

**参考输出**：
```
Connected to: Weather Server
Server version: 1.0.0
MCP Protocol Version: 2024-11-05

Available tools:
  - get_weather: 获取指定城市的天气信息

Calling get_weather tool...
Result: 北京天气：晴，25°C，湿度 30%

Connection closed.
```

**运行流程详解**：

```
┌─────────────────────────────────────────────────────────┐
│ 步骤 1: 创建 Client                                      │
│   client = MCPClient()                                   │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 2: 连接到 Server                                    │
│   - 启动子进程：python mcp_weather_server.py            │
│   - 建立 stdio 管道                                      │
│   - 发送 initialize 请求                                 │
│   - 等待 Server 响应                                      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 3: 获取服务器信息                                   │
│   - server_info.name: "Weather Server"                  │
│   - server_info.version: "1.0.0"                        │
│   - protocol_version: "2024-11-05"                      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 4: 列出工具                                         │
│   - 发送：{"method": "tools/list"}                      │
│   - 接收：{"tools": [{"name": "get_weather", ...}]}    │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 5: 调用工具                                         │
│   - 发送：{"method": "tools/call",                      │
│            "params": {"name": "get_weather",            │
│                       "arguments": {"city": "北京"}}}   │
│   - 接收：{"result": {"content": [...]}}                │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 6: 关闭连接                                         │
│   - 关闭 stdio 管道                                      │
│   - Server 检测到 EOF，退出进程                          │
└─────────────────────────────────────────────────────────┘
```

**调试技巧**：

```python
# 1. 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)

# 2. 捕获并打印原始 JSON-RPC 消息
class DebugMCPClient(MCPClient):
    async def send(self, message):
        print(f"SEND: {json.dumps(message, indent=2)}")
        await super().send(message)

    async def receive(self):
        message = await super().receive()
        print(f"RECV: {json.dumps(message, indent=2)}")
        return message

# 3. 设置超时
try:
    await asyncio.wait_for(
        client.call_tool("get_weather", {"city": "北京"}),
        timeout=10.0
    )
except asyncio.TimeoutError:
    print("工具调用超时！")
```

### 7.3 Resources 和 Prompts 示例

创建 `examples/mcp_resources_server.py`：

```python
#!/usr/bin/env python3
"""
MCP Resources 示例 - 文件读取服务
"""

import sys
import json
from src.mcp import MCPServer, ServerConfig
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class FileReadTool(BaseTool):
    """文件读取工具"""

    @property
    def name(self): return "read_file"

    @property
    def description(self): return "读取指定路径的文件内容"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "文件路径"}
            },
            "required": ["path"]
        }

    def execute(self, **kwargs):
        path = kwargs.get("path", "")
        try:
            with open(path, "r", encoding="utf-8") as f:
                content = f.read()
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output=f"文件内容:\n{content}"
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error=f"读取失败：{str(e)}"
            )


if __name__ == "__main__":
    config = ServerConfig(name="File Server", version="1.0.0")
    server = MCPServer(config=config)

    # 注册工具
    server.register_tool(FileReadTool())

    # 注册资源
    server.register_resource(
        uri="file:///config/app.json",
        resource={
            "name": "应用程序配置",
            "description": "应用程序的 JSON 配置文件",
            "mimeType": "application/json",
            "content": json.dumps({
                "app_name": "MCP Tutorial",
                "version": "1.0.0"
            }, indent=2)
        }
    )

    # 注册提示词模板
    server.register_prompt(
        name="code_review",
        prompt={
            "description": "代码审查提示模板",
            "arguments": [
                {"name": "code", "description": "要审查的代码", "required": True}
            ],
            "messages": [
                {
                    "role": "user",
                    "content": {
                        "type": "text",
                        "text": "请审查以下代码：\n\n{code}"
                    }
                }
            ]
        }
    )

    print("Starting MCP File Server...", file=sys.stderr)
    server.run_stdio()
```

### 7.4 通知机制示例

#### 工具列表变更通知

当 Server 的工具列表发生变化时，可以发送通知：

**Server 发送通知**：
```python
await transport.send({
    "jsonrpc": "2.0",
    "method": "notifications/tools/list_changed"
})
```

**Client 接收并刷新**：
```python
# 收到通知后重新获取工具列表
tools = await client.list_tools()
print(f"工具列表已更新：{[t.name for t in tools]}")
```

#### 资源变更通知

```python
# Server 发送资源变更通知
await transport.send({
    "jsonrpc": "2.0",
    "method": "notifications/resources/list_changed"
})
```

---

## 第八部分：与 Claude Desktop 集成

Claude Desktop 是 Anthropic 推出的桌面应用，内置 MCP 支持，是测试和部署 MCP Server 的理想平台。

### 8.1 配置文件位置

MCP Server 的配置需要添加到 Claude Desktop 的配置文件中。

| 操作系统 | 配置文件路径 | 快速打开命令 |
|---------|-------------|-------------|
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` | `open ~/Library/Application\ Support/Claude/claude_desktop_config.json` |
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` | `notepad %APPDATA%\Claude\claude_desktop_config.json` |
| **Linux** | `~/.config/Claude/claude_desktop_config.json` | `nano ~/.config/Claude/claude_desktop_config.json` |

**查找配置文件的方法**：

1. **macOS**: 打开 Finder → 按 `Cmd+Shift+G` → 输入 `~/Library/Application Support/Claude/`
2. **Windows**: 按 `Win+R` → 输入 `%APPDATA%\Claude\` → 回车
3. **Linux**: 直接在终端打开路径

**注意**：如果配置文件不存在，可以手动创建。

### 8.2 配置格式详解

#### 完整配置示例

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/absolute/path/to/weather_server.py"],
      "env": {
        "API_KEY": "your_api_key",
        "DEBUG": "true"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/yourname/documents"],
      "disabled": false
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    },
    "custom": {
      "command": "/usr/local/bin/python3.11",
      "args": ["-m", "my_package.mcp_server"],
      "cwd": "/path/to/working/directory",
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

#### 配置字段详解

**顶层字段**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `mcpServers` | Object | ✅ | 包含所有 MCP Server 配置 |

**每个 Server 的配置字段**：

| 字段 | 类型 | 必填 | 默认值 | 说明 | 示例 |
|------|------|------|--------|------|------|
| `command` | String | ✅ | 无 | 启动命令 | `"python"`, `"npx"`, `"node"` |
| `args` | Array | ✅ | 无 | 命令参数列表 | `["server.py"]`, `["-y", "package"]` |
| `env` | Object | ❌ | `{}` | 环境变量 | `{"API_KEY": "xxx"}` |
| `cwd` | String | ❌ | 当前目录 | 工作目录 | `"/path/to/workdir"` |
| `disabled` | Boolean | ❌ | `false` | 是否禁用此 Server | `true` 临时禁用 |

**字段详解和最佳实践**：

1. **command（启动命令）**：
   - `python` - Python 脚本
   - `npx` - Node.js 包（无需安装）
   - `node` - Node.js 脚本
   - 绝对路径 - 直接可执行文件

   **最佳实践**：
   ```json
   // 推荐：使用绝对路径，避免路径问题
   {"command": "/usr/bin/python3"}

   // 不推荐：依赖 PATH 环境变量
   {"command": "python"}
   ```

2. **args（参数列表）**：
   - 每个参数是数组的一个元素
   - 路径建议使用绝对路径

   **最佳实践**：
   ```json
   // 推荐：参数分开，路径明确
   {"args": ["/absolute/path/to/server.py", "--port", "8080"]}

   // 不推荐：参数连在一起，难以调试
   {"args": ["/absolute/path/to/server.py --port 8080"]}
   ```

3. **env（环境变量）**：
   - 用于传递 API Key、配置等敏感信息
   - 不要将敏感信息硬编码在 Server 代码中

   **最佳实践**：
   ```json
   // 推荐：通过 env 传递敏感信息
   {
     "command": "python",
     "args": ["server.py"],
     "env": {
       "OPENAI_API_KEY": "sk-...",
       "DATABASE_URL": "postgresql://..."
     }
   }
   ```

4. **disabled（禁用字段）**：
   - 临时禁用 Server 而不删除配置
   - 方便调试和问题排查

   **使用场景**：
   ```json
   // 某个 Server 出现问题，临时禁用
   {
     "problematic-server": {
       "command": "python",
       "args": ["server.py"],
       "disabled": true  // 临时禁用
     }
   }
   ```

### 8.3 调试技巧

#### 完整的调试流程

```
┌─────────────────────────────────────────────────────────┐
│ 步骤 1: 检查配置文件语法                                 │
│   - 使用 JSON 验证工具检查语法                            │
│   - 确保没有注释（JSON 不支持注释）                       │
│   - 检查路径是否正确（使用绝对路径）                      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 2: 手动测试 Server 脚本                              │
│   - 在终端运行 Server 脚本                               │
│   - 确认没有导入错误或依赖问题                           │
│   - 检查是否需要虚拟环境                                  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 3: 使用 MCP Inspector 测试                          │
│   - 独立测试 Server，不依赖 Claude Desktop                │
│   - 验证工具列表和工具调用                               │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 4: 重启 Claude Desktop                              │
│   - 完全退出 Claude Desktop（不只是关闭窗口）             │
│   - 重新启动以加载新配置                                  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ 步骤 5: 查看日志                                        │
│   - 检查 Claude Desktop 日志                             │
│   - 查看 Server 的 stderr 输出                            │
└─────────────────────────────────────────────────────────┘
```

#### 查看 Claude Desktop 日志

**macOS**：
```bash
# 实时查看日志
tail -F ~/Library/Logs/Claude/*.log

# 查看最近的错误日志
grep -i "error" ~/Library/Logs/Claude/*.log | tail -50
```

**Windows**：
```powershell
# 实时查看日志
Get-Content "$env:APPDATA\Claude\logs\*.log" -Wait -Tail 50

# 搜索错误
Select-String -Path "$env:APPDATA\Claude\logs\*.log" -Pattern "error" -Tail 50
```

**Linux**：
```bash
# 实时查看日志
tail -f ~/.config/Claude/logs/*.log

# 查看最近的错误
grep -i "error" ~/.config/Claude/logs/*.log | tail -50
```

#### 使用 MCP Inspector

MCP Inspector 是官方的 MCP 调试工具，提供可视化界面。

**安装 MCP Inspector**：
```bash
# 使用 npx（无需安装）
npx @modelcontextprotocol/inspector

# 或者全局安装
npm install -g @modelcontextprotocol/inspector
```

**运行 Inspector**：
```bash
# 基本用法
npx @modelcontextprotocol/inspector python weather_server.py

# 带环境变量
npx @modelcontextprotocol/inspector \
  -e API_KEY=xxx \
  -e DEBUG=true \
  python weather_server.py

# 指定工作目录
npx @modelcontextprotocol/inspector \
  -d /path/to/workdir \
  python weather_server.py
```

**Inspector 界面功能**：

| 功能 | 说明 | 使用场景 |
|------|------|---------|
| Server Info | 显示服务器信息和能力 | 确认 Server 正常启动 |
| Tools | 列出可用工具并测试调用 | 验证工具定义和参数 |
| Resources | 列出资源并测试读取 | 验证资源访问 |
| Prompts | 列出提示词并测试获取 | 验证提示词模板 |
| Console | 显示原始 JSON-RPC 消息 | 调试协议级问题 |

#### 启用详细日志

在 Server 中启用详细日志输出：

```python
#!/usr/bin/env python3
import sys
import logging

# 配置日志输出到 stderr（不影响 stdio 通信）
logging.basicConfig(
    level=logging.DEBUG,  # 或 logging.INFO
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    stream=sys.stderr  # 重要：输出到 stderr，不是 stdout！
)

logger = logging.getLogger("weather_server")
logger.info("Starting MCP Weather Server...")

# 在工具执行中记录日志
def execute(self, **kwargs):
    city = kwargs.get("city", "")
    logger.debug(f"get_weather called with city={city}")
    # ...
```

**日志级别说明**：

| 级别 | 说明 | 使用场景 |
|------|------|---------|
| DEBUG | 详细调试信息 | 开发阶段 |
| INFO | 一般信息 | 生产环境 |
| WARNING | 警告信息 | 潜在问题 |
| ERROR | 错误信息 | 必须关注 |
| CRITICAL | 严重错误 | 系统崩溃 |

#### 常见问题排查

| 问题 | 可能原因 | 解决方案 | 诊断命令 |
|------|---------|---------|---------|
| Server 不显示 | 路径错误 | 使用绝对路径 | `which python` 或 `where python` |
| Server 不显示 | 配置文件语法错误 | 使用 JSON 验证工具 | `python -m json.tool claude_desktop_config.json` |
| Server 不显示 | 依赖缺失 | 在 Server 目录安装依赖 | `pip install -r requirements.txt` |
| 工具调用失败 | 参数不匹配 | 检查 `inputSchema` 定义 | 使用 Inspector 测试 |
| 工具调用失败 | 工具执行抛出异常 | 查看 stderr 日志 | 启用 `logging.DEBUG` |
| 连接超时 | Server 启动慢 | 增加超时或优化启动 | 检查 Server 初始化代码 |
| 权限错误 | 文件不可执行 | `chmod +x server.py` | `ls -la server.py` |
| 日志看不到 | 输出到 stdout | 日志应输出到 `stderr` | 检查 `stream=sys.stderr` |
| 环境变量未生效 | 配置格式错误 | 检查 env 嵌套层级 | 在 Server 中打印 `os.environ` |

**问题排查清单**：

```
□ 1. 配置文件是否为有效的 JSON？
□ 2. command 和 args 路径是否正确（使用绝对路径）？
□ 3. Server 脚本能否在终端独立运行？
□ 4. 依赖包是否已安装？
□ 5. 环境变量是否正确配置？
□ 6. 日志是否输出到 stderr？
□ 7. Claude Desktop 是否完全重启？
□ 8. 是否使用 Inspector 进行过独立测试？
```

---

## 第九部分：高级主题

### 9.1 无状态 MCP Server

#### 什么是有状态 vs 无状态？

**有状态 Server**：
- 维护会话状态
- 记住之前的交互
- 适合需要上下文的场景（如多轮对话）

**无状态 Server**：
- 每次请求独立处理
- 不维护会话状态
- 适合简单工具服务

**类比**：
- **有状态** = 私人医生（记得你的病史）
- **无状态** = 急诊医生（只看当前症状）

#### 无状态 Server 实现

使用 Streamable HTTP 的无状态模式：

```python
class StatelessMCPServer:
    """无状态 MCP Server 示例

    特点：
    - 每次请求独立处理
    - 不维护客户端连接状态
    - 适合部署在 Serverless 平台
    """

    def __init__(self):
        self.tools = ToolRegistry()
        # 注册工具...

    async def handle_request(self, request: dict) -> dict:
        """处理单个请求

        参数：
            request: JSON-RPC 请求对象

        返回：
            JSON-RPC 响应对象
        """
        # 每次请求都是独立的，没有会话状态
        method = request.get("method")
        params = request.get("params", {})

        if method == "tools/list":
            return self._list_tools()
        elif method == "tools/call":
            return self._call_tool(params)
        else:
            return {"error": {"code": -32601, "message": "Method not found"}}
```

#### 会话管理（有状态场景）

对于需要状态的 Server，可以使用会话 ID：

```python
import uuid
from typing import Dict, Any

class SessionManager:
    """会话管理器

    用于在多个请求之间维护状态。
    适用于需要多轮对话、购物车等场景。
    """

    def __init__(self, ttl: int = 3600):
        """初始化会话管理器

        参数：
            ttl: 会话过期时间（秒），默认 1 小时
        """
        self.sessions: Dict[str, Dict[str, Any]] = {}
        self.ttl = ttl

    def create_session(self) -> str:
        """创建新会话"""
        session_id = str(uuid.uuid4())
        self.sessions[session_id] = {
            "created_at": time.time(),
            "data": {}  # 会话数据
        }
        return session_id

    def get_session(self, session_id: str) -> Optional[Dict[str, Any]]:
        """获取会话数据"""
        session = self.sessions.get(session_id)
        if session is None:
            return None

        # 检查是否过期
        if time.time() - session["created_at"] > self.ttl:
            del self.sessions[session_id]
            return None

        return session["data"]

    def update_session(self, session_id: str, data: Dict[str, Any]):
        """更新会话数据"""
        if session_id in self.sessions:
            self.sessions[session_id]["data"] = data
            self.sessions[session_id]["created_at"] = time.time()

    def delete_session(self, session_id: str):
        """删除会话"""
        if session_id in self.sessions:
            del self.sessions[session_id]
```

**会话使用示例**：

```python
class ShoppingCartServer:
    """带购物车的 MCP Server（有状态）"""

    def __init__(self):
        self.session_manager = SessionManager()

    async def handle_request(self, request: dict) -> dict:
        method = request.get("method")
        params = request.get("params", {})

        # 从请求头或参数中获取 session_id
        session_id = params.get("session_id")

        if method == "cart/add":
            session = self.session_manager.get_session(session_id)
            if session is None:
                session_id = self.session_manager.create_session()
                session = {"cart": []}

            # 添加到购物车
            session["cart"].append(params.get("item"))
            self.session_manager.update_session(session_id, session)

            return {
                "result": {
                    "session_id": session_id,
                    "cart": session["cart"]
                }
            }
```

#### 适用场景对比

| 场景 | 推荐模式 | 原因 |
|------|---------|------|
| 简单工具服务（计算器、天气） | 无状态 | 无需上下文，每次调用独立 |
| 多轮对话（客服机器人） | 有状态 | 需要记住对话历史 |
| 购物车/订单处理 | 有状态 | 需要累积用户选择 |
| 高并发 API 服务 | 无状态 + 外部存储 | 水平扩展，状态存 Redis |
| Serverless 部署 | 无状态 | 函数执行完即销毁 |

### 9.2 安全性考虑

MCP Server 通常具有访问文件系统、数据库、API 等敏感操作的能力，因此安全性至关重要。

#### 安全威胁模型

```
┌─────────────────────────────────────────────────────────┐
│                  MCP 安全威胁                            │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 注入攻击                                            │
│     - SQL 注入：通过工具参数注入恶意 SQL                   │
│     - 命令注入：执行系统命令                              │
│     - 路径遍历：访问未授权文件                            │
│                                                         │
│  2. 越权访问                                            │
│     - 水平越权：访问其他用户的数据                        │
│     - 垂直越权：执行未授权的操作                          │
│                                                         │
│  3. 信息泄露                                            │
│     - 敏感数据返回给 LLM                                 │
│     - 日志中泄露密钥                                     │
│                                                         │
│  4. 拒绝服务                                            │
│     - 大量请求耗尽资源                                   │
│     - 慢速请求占用连接                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 输入验证

**实际攻击案例**：

假设有一个文件读取工具，如果没有 proper 的输入验证：

```python
# ❌ 危险的代码 - 不要这样写！
def execute(self, **kwargs):
    path = kwargs.get("path", "")
    # 直接读取文件，没有任何验证！
    with open(path, "r") as f:
        return f.read()
```

**攻击者可以利用**：

```
# 读取敏感文件
path = "/etc/passwd"

# 路径遍历
path = "../../../etc/passwd"

# 读取密钥
path = "~/.ssh/id_rsa"
```

**正确的防护代码**：

```python
def execute(self, **kwargs):
    path = kwargs.get("path", "")

    # 1. 基础验证：检查空值
    if not path:
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error="路径不能为空"
        )

    # 2. 字符合法性检查
    if not re.match(r'^[a-zA-Z0-9_./-]+$', path):
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error="路径包含非法字符"
        )

    # 3. 防止路径遍历
    if ".." in path:
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error="不允许使用相对路径"
        )

    # 4. 限制访问范围
    allowed_dirs = ["/data/public", "/data/user_docs"]
    full_path = os.path.abspath(path)
    if not any(full_path.startswith(d) for d in allowed_dirs):
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error="访问超出允许范围"
        )

    # 5. 安全读取
    try:
        with open(full_path, "r", encoding="utf-8") as f:
            content = f.read()
        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=content[:10000]  # 限制返回大小
        )
    except Exception as e:
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error=f"读取失败：{str(e)}"
        )
```

#### 路径遍历防护

```python
from pathlib import Path
import os

class FilesystemTool(BaseTool):
    """安全的文件系统工具

    安全特性：
    1. 限制访问目录范围
    2. 解析符号链接
    3. 防止路径遍历攻击
    """

    def __init__(self, base_dir: str):
        """初始化

        参数：
            base_dir: 允许访问的根目录
        """
        # 解析并标准化基础目录
        self.base_dir = Path(base_dir).resolve()

    def execute(self, **kwargs):
        path = kwargs.get("path", "")
        full_path = (self.base_dir / path).resolve()

        # 安全检查：确保在 base_dir 内
        try:
            # 使用 relative_to 检查是否在允许范围内
            full_path.relative_to(self.base_dir)
        except ValueError:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="Access denied: Path traversal detected"
            )

        # 额外检查：确保不是符号链接指向外部
        if full_path.is_symlink():
            real_path = full_path.resolve()
            try:
                real_path.relative_to(self.base_dir)
            except ValueError:
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    error="Access denied: Symlink points outside"
                )

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=full_path.read_text()
        )
```

#### OAuth 2.0 集成

对于远程 MCP Server，推荐使用 OAuth 2.0 进行认证：

```python
from authlib.integrations.httpx_client import OAuth2Client
import httpx

class OAuthMCPClient:
    """OAuth 2.0 认证的 MCP Client

    使用场景：
    - 远程 MCP Server 访问
    - 多租户 SaaS 服务
    - 需要细粒度权限控制
    """

    def __init__(
        self,
        client_id: str,
        client_secret: str,
        token_url: str,
        scope: str = "mcp:read mcp:write"
    ):
        self.client = OAuth2Client(
            client_id=client_id,
            client_secret=client_secret,
            token_endpoint=token_url,
            scope=scope,
        )
        self._token = None

    async def get_token(self) -> str:
        """获取访问 Token

        支持多种授权模式：
        1. Client Credentials（服务端应用）
        2. Authorization Code（Web 应用）
        3. Password（遗留系统，不推荐）
        """
        if self._token and not self._token.is_expired():
            return self._token["access_token"]

        # 使用 Client Credentials 模式
        self._token = await self.client.fetch_token(
            grant_type="client_credentials",
        )
        return self._token["access_token"]

    async def request(self, method: str, url: str, **kwargs):
        """发送带认证的请求"""
        token = await self.get_token()
        headers = kwargs.get("headers", {})
        headers["Authorization"] = f"Bearer {token}"

        async with httpx.AsyncClient() as client:
            response = await client.request(
                method, url, headers=headers, **kwargs
            )
            return response.json()
```

**OAuth 2.0 配置示例**（Server 端）：

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    """验证 Token 并获取当前用户"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("sub")
        if user_id is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        return user_id
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/mcp")
async def mcp_endpoint(user: str = Depends(get_current_user)):
    """带认证的 MCP 端点"""
    # user 已验证，可以处理请求
    ...
```

#### 权限控制

```python
from enum import Enum
from typing import Dict, Set

class Permission(Enum):
    """权限枚举"""
    READ = "read"
    WRITE = "write"
    EXECUTE = "execute"
    ADMIN = "admin"

class AccessControl:
    """基于角色的访问控制（RBAC）

    功能：
    - 用户 - 角色 - 权限三层模型
    - 支持资源级别的权限控制
    - 支持权限继承
    """

    def __init__(self):
        # 用户 -> 角色
        self.user_roles: Dict[str, Set[str]] = {}
        # 角色 -> 权限
        self.role_permissions: Dict[str, Set[Permission]] = {}
        # 资源 -> (用户 -> 权限)
        self.resource_permissions: Dict[str, Dict[str, Set[Permission]]] = {}

    def assign_role(self, user: str, role: str):
        """给用户分配角色"""
        if user not in self.user_roles:
            self.user_roles[user] = set()
        self.user_roles[user].add(role)

    def grant_permission(self, role: str, permission: Permission):
        """给角色授予权限"""
        if role not in self.role_permissions:
            self.role_permissions[role] = set()
        self.role_permissions[role].add(permission)

    def check(self, user: str, resource: str, required: Permission) -> bool:
        """检查用户是否有资源的指定权限"""
        # 检查资源级别的权限
        if resource in self.resource_permissions:
            if user in self.resource_permissions[resource]:
                if required in self.resource_permissions[resource][user]:
                    return True

        # 检查角色级别的权限
        if user in self.user_roles:
            for role in self.user_roles[user]:
                if role in self.role_permissions:
                    if required in self.role_permissions[role]:
                        return True

        return False

# 使用示例
acl = AccessControl()

# 定义角色
acl.assign_role("alice", "admin")
acl.assign_role("bob", "viewer")

# 授予权限
acl.grant_permission("admin", Permission.READ)
acl.grant_permission("admin", Permission.WRITE)
acl.grant_permission("admin", Permission.EXECUTE)
acl.grant_permission("viewer", Permission.READ)

# 检查权限
if acl.check("alice", "database", Permission.WRITE):
    # 允许写入
    ...
else:
    # 拒绝访问
    return ToolResult(
        status=ToolResultStatus.ERROR,
        error="Access denied: insufficient permissions"
    )
```

#### 安全最佳实践清单

```
□ 1. 所有输入都进行验证和过滤
□ 2. 使用参数化查询防止 SQL 注入
□ 3. 限制文件访问范围（白名单）
□ 4. 敏感信息（密钥、密码）不输出到日志
□ 5. 远程访问使用 OAuth 2.0 或 API Key 认证
□ 6. 实现请求限流防止 DoS
□ 7. 工具执行结果进行脱敏处理
□ 8. 定期审计日志发现异常行为
```

### 9.3 性能优化

#### 性能优化概览

MCP Server 的性能优化可以从以下几个层面入手：

```
┌─────────────────────────────────────────────────────────┐
│                  MCP 性能优化层级                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 传输层优化                                          │
│     - 连接池：复用 TCP 连接                                │
│     - HTTP/2：多路复用，减少延迟                         │
│     - 压缩：减少传输数据量                               │
│                                                         │
│  2. 应用层优化                                          │
│     - 缓存：减少重复计算                                 │
│     - 并发：提高吞吐量                                   │
│     - 限流：防止过载                                     │
│                                                         │
│  3. 数据层优化                                          │
│     - 分页：减少单次返回数据量                           │
│     - 索引：加快数据库查询                               │
│     - 批量：减少往返次数                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 连接池（Connection Pool）

对于 HTTP Transport，使用连接池可以显著提高性能：

**为什么需要连接池？**

```
没有连接池：                      使用连接池：
请求 1 → 建立连接 → 发送 → 接收 → 关闭  请求 1 ─┬→ 发送 → 接收 ─┐
请求 2 → 建立连接 → 发送 → 接收 → 关闭  请求 2 ─┤              │
请求 3 → 建立连接 → 发送 → 接收 → 关闭  请求 3 ─┴→ 发送 → 接收 ─┘
(每次 3 次 RTT)                        (复用连接，1 次 RTT)
```

**连接池实现**：

```python
import aiohttp
from typing import Optional

class HTTPConnectionPool:
    """HTTP 连接池

    功能：
    - 复用 TCP 连接，减少握手延迟
    - 控制最大连接数，防止资源耗尽
    - 自动清理空闲连接

    性能提升：
    - 首次请求：~100ms（包含握手）
    - 后续请求：~10ms（复用连接）
    """

    def __init__(
        self,
        max_size: int = 10,
        timeout: int = 30,
        keepalive: int = 60
    ):
        """初始化连接池

        参数：
            max_size: 最大连接数
            timeout: 请求超时时间（秒）
            keepalive: 连接保持时间（秒）
        """
        # TCP 连接器配置
        connector = aiohttp.TCPConnector(
            limit=max_size,              # 最大连接数
            limit_per_host=max_size,     # 每主机最大连接数
            ttl_dns_cache=300,          # DNS 缓存时间
            keepalive_timeout=keepalive, # 空闲连接超时
        )

        # 创建会话
        self.pool = aiohttp.ClientSession(
            connector=connector,
            timeout=aiohttp.ClientTimeout(total=timeout),
        )

    async def request(
        self,
        method: str,
        url: str,
        **kwargs
    ) -> dict:
        """发送 HTTP 请求

        参数：
            method: HTTP 方法
            url: 请求 URL
            **kwargs: 其他参数传给 session.post

        返回：
            JSON 响应
        """
        async with self.pool.request(method, url, **kwargs) as response:
            return await response.json()

    async def close(self):
        """关闭连接池"""
        await self.pool.close()

# 使用示例
pool = HTTPConnectionPool(max_size=20)

# 多次请求复用连接
for i in range(100):
    result = await pool.request(
        "POST",
        "https://api.example.com/mcp",
        json={"method": "tools/list"}
    )
```

**基准测试对比**：

| 场景 | 无连接池 | 连接池（10） | 提升 |
|------|---------|-------------|------|
| 100 次请求 | ~10s | ~2s | 5x |
| 1000 次请求 | ~100s | ~15s | 6.7x |
| 并发 10 请求 | ~3s | ~0.5s | 6x |

#### 缓存策略

缓存可以显著减少重复计算和外部 API 调用：

**缓存装饰器实现**：

```python
import hashlib
import time
from functools import wraps
from typing import Any, Dict

class CacheEntry:
    """缓存条目"""

    def __init__(self, value: Any, ttl: int):
        self.value = value
        self.expires_at = time.time() + ttl

    def is_expired(self) -> bool:
        return time.time() > self.expires_at


def cache_result(ttl: int = 60, max_size: int = 100):
    """缓存装饰器

    参数：
        ttl: 缓存生存时间（秒）
        max_size: 最大缓存条目数

    使用示例：
        @cache_result(ttl=300)
        def get_weather(city: str) -> str:
            # 调用天气 API...
    """
    cache: Dict[str, CacheEntry] = {}

    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 生成缓存键
            key_data = f"{func.__name__}:{args}:{sorted(kwargs.items())}"
            key = hashlib.md5(key_data.encode()).hexdigest()

            # 检查缓存
            if key in cache:
                entry = cache[key]
                if not entry.is_expired():
                    return entry.value
                else:
                    del cache[key]

            # 执行函数
            result = await func(*args, **kwargs)

            # 更新缓存（LRU 淘汰）
            if len(cache) >= max_size:
                # 删除最旧的条目
                oldest_key = min(cache.keys(),
                               key=lambda k: cache[k].expires_at)
                del cache[oldest_key]

            cache[key] = CacheEntry(result, ttl)
            return result

        return wrapper
    return decorator


# 使用示例
class WeatherServer:
    def __init__(self):
        self.cache = {}

    @cache_result(ttl=300)  # 缓存 5 分钟
    async def get_weather(self, city: str) -> str:
        """获取天气（带缓存）

        性能对比：
        - 无缓存：调用 API ~500ms
        - 有缓存：内存查询 ~1ms
        """
        # 调用天气 API...
        return f"{city}天气：晴，25°C"
```

**手动缓存管理**：

```python
class CachedMCPServer:
    """带缓存的 MCP Server

    特性：
    - 支持自定义缓存键
    - 支持缓存失效通知
    - 支持缓存统计
    """

    def __init__(self, cache_ttl: int = 60):
        self.cache_ttl = cache_ttl
        self._cache: Dict[str, Dict[str, Any]] = {}
        self._stats = {
            "hits": 0,
            "misses": 0,
            "evictions": 0
        }

    def _get_cache_key(self, method: str, params: dict) -> str:
        """生成缓存键"""
        # 使用 JSON 序列化确保一致性
        import json
        param_str = json.dumps(params, sort_keys=True)
        return f"{method}:{hashlib.md5(param_str.encode()).hexdigest()}"

    async def handle_request(self, request: dict) -> dict:
        method = request.get("method")
        params = request.get("params", {})

        # 只对特定方法使用缓存
        if method in ["tools/call", "resources/read"]:
            cache_key = self._get_cache_key(method, params)
            cached = self._cache.get(cache_key)

            if cached and time.time() - cached["time"] < self.cache_ttl:
                self._stats["hits"] += 1
                return cached["response"]

            self._stats["misses"] += 1

        # 执行请求
        response = await self._execute(method, params)

        # 更新缓存
        if method in ["tools/call", "resources/read"]:
            self._cache[cache_key] = {
                "time": time.time(),
                "response": response
            }
            # 清理过期缓存
            self._cleanup_cache()

        return response

    def _cleanup_cache(self):
        """清理过期缓存"""
        now = time.time()
        expired_keys = [
            k for k, v in self._cache.items()
            if now - v["time"] >= self.cache_ttl
        ]
        for key in expired_keys:
            del self._cache[key]
            self._stats["evictions"] += 1

    def get_cache_stats(self) -> dict:
        """获取缓存统计"""
        total = self._stats["hits"] + self._stats["misses"]
        hit_rate = self._stats["hits"] / total if total > 0 else 0
        return {
            **self._stats,
            "hit_rate": f"{hit_rate:.2%}",
            "cache_size": len(self._cache)
        }
```

**缓存最佳实践**：

| 数据类型 | 推荐 TTL | 说明 |
|---------|---------|------|
| 天气数据 | 5-15 分钟 | 变化较快 |
| 新闻/文章 | 30-60 分钟 | 更新频率中等 |
| 配置数据 | 5-10 分钟 | 较少变化 |
| 静态资源 | 1-24 小时 | 几乎不变 |
| 计算结果 | 根据需求 | 取决于计算成本 |

        # 检查缓存
        cache_key = self._get_cache_key(method, params)
        cached = self._cache.get(cache_key)

        if cached and time.time() - cached["time"] < self.cache_ttl:
            return cached["response"]

        # 执行请求
        response = await self._execute(method, params)

        # 更新缓存
        self._cache[cache_key] = {
            "time": time.time(),
            "response": response
        }

        return response
```

#### 并发处理

使用信号量（Semaphore）限制并发请求数，防止服务器过载：

```python
import asyncio
from asyncio import Semaphore

class ConcurrentMCPServer:
    """带并发控制的 MCP Server

    特性：
    - 使用信号量限制并发数
    - 防止资源耗尽
    - 公平调度（FIFO）

    性能优化：
    - 并发数太小：资源利用率低
    - 并发数太大：上下文切换开销大
    - 推荐值：CPU 核心数 * 2~4
    """

    def __init__(
        self,
        max_concurrency: int = 10,
        request_timeout: int = 30
    ):
        """初始化

        参数：
            max_concurrency: 最大并发请求数
            request_timeout: 请求超时时间（秒）
        """
        # 信号量：控制同时执行的协程数
        self.semaphore = Semaphore(max_concurrency)
        self.request_timeout = request_timeout

    async def handle_request(self, request: dict) -> dict:
        """处理请求（带并发控制）

        流程：
        1. 获取信号量（如果没有可用配额则等待）
        2. 执行请求（带超时）
        3. 释放信号量
        """
        async with self.semaphore:
            try:
                # 使用 wait_for 添加超时
                return await asyncio.wait_for(
                    self._execute(request),
                    timeout=self.request_timeout
                )
            except asyncio.TimeoutError:
                return {
                    "error": {
                        "code": -32000,
                        "message": "Request timeout"
                    }
                }

    async def _execute(self, request: dict) -> dict:
        """执行具体请求"""
        method = request.get("method")
        # ...处理逻辑
```

**并发性能对比**：

| 并发数 | 吞吐量（req/s） | P99 延迟 | CPU 使用率 |
|-------|---------------|---------|-----------|
| 1（串行） | 10 | 100ms | 10% |
| 10 | 80 | 120ms | 60% |
| 50 | 150 | 200ms | 90% |
| 100 | 140 | 350ms | 100% |

**最佳实践**：
- 对于 I/O 密集型任务（如 API 调用），使用较高并发数（50-100）
- 对于 CPU 密集型任务（如计算），使用较低并发数（2-4）
- 始终设置超时，防止慢请求占用资源

---

## 第十部分：实践练习与故障排查

### 10.1 实践练习

#### 练习 1：创建简单的计算器 MCP Server

**难度**：⭐⭐☆☆☆

**目标**：实现一个支持四则运算的 MCP Server。

**要求**：
- 实现加减乘除四个工具（或一个统一的 calculate 工具）
- 支持错误处理（除零、溢出、非法字符等）
- 使用 stdio 传输方式
- 配置到 Claude Desktop 测试

**步骤指导**：

```
步骤 1: 创建项目结构
mkdir mcp_calculator
cd mcp_calculator
touch calculator_server.py

步骤 2: 实现 CalculatorTool 类
- 定义 name, description, parameters
- 实现 execute 方法，包含输入验证
- 处理除零、非法字符等异常

步骤 3: 创建 MCPServer 并注册工具
- 创建 ServerConfig
- 创建 MCPServer
- 注册 CalculatorTool

步骤 4: 测试 Server
- 使用 MCP Inspector 测试
- 或直接配置到 Claude Desktop

步骤 5: 验证功能
- 测试正常计算：2 + 3 * 4
- 测试错误处理：1 / 0, "hello + world"
```

**参考代码**：
```python
#!/usr/bin/env python3
"""
计算器 MCP Server - 练习 1 参考答案
"""

import sys
import re
import operator
from src.mcp import MCPServer, ServerConfig
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class CalculatorTool(BaseTool):
    """安全的计算器工具

    支持的操作：
    - 加法 (+), 减法 (-), 乘法 (*), 除法 (/)
    - 括号优先级
    - 负数

    安全防护：
    - 只允许数字和基本运算符
    - 禁止函数调用
    - 限制表达式长度
    """

    @property
    def name(self):
        return "calculate"

    @property
    def description(self):
        return "执行数学计算，支持加减乘除和括号"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，如 '2 + 3 * 4' 或 '(10 - 5) / 2'",
                    "maxLength": 100  # 限制表达式长度
                }
            },
            "required": ["expression"]
        }

    def execute(self, **kwargs):
        """执行计算

        使用安全的表达式解析，避免 eval 的安全风险
        """
        expression = kwargs.get("expression", "").strip()

        # 1. 长度检查
        if len(expression) > 100:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="表达式过长（最大 100 字符）"
            )

        # 2. 字符合法性检查（只允许数字、运算符、括号、空格、小数点）
        allowed_pattern = r'^[0-9+\-*/().\s]+$'
        if not re.match(allowed_pattern, expression):
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="表达式包含非法字符（只允许数字、+-*/() 和空格）"
            )

        # 3. 括号匹配检查
        if expression.count('(') != expression.count(')'):
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="括号不匹配"
            )

        # 4. 安全检查：确保没有连续的操作符
        if re.search(r'[+\-*/]{2,}', expression.replace('**', '')):
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="非法的操作符组合"
            )

        # 5. 安全计算
        try:
            # 使用 eval 但已经过了严格的输入验证
            result = eval(expression)

            # 检查结果类型
            if isinstance(result, complex):
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    error="不支持复数结果"
                )

            # 格式化结果
            if isinstance(result, float):
                # 限制小数位数
                result_str = f"{result:.10g}"
            else:
                result_str = str(result)

            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output=f"{expression} = {result_str}"
            )

        except ZeroDivisionError:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="错误：除数不能为零"
            )
        except OverflowError:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="错误：计算结果溢出"
            )
        except SyntaxError:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="错误：表达式语法错误"
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error=f"计算错误：{str(e)}"
            )


if __name__ == "__main__":
    # 创建服务器
    config = ServerConfig(name="Calculator Server", version="1.0.0")
    server = MCPServer(config=config)

    # 注册工具
    server.register_tool(CalculatorTool())

    # 启动
    print("Starting MCP Calculator Server...", file=sys.stderr)
    server.run_stdio()
```

**测试用例**：

| 输入 | 期望输出 |
|------|---------|
| `2 + 3` | `2 + 3 = 5` |
| `10 - 5 * 2` | `10 - 5 * 2 = 0` |
| `(10 - 5) * 2` | `(10 - 5) * 2 = 10` |
| `10 / 3` | `10 / 3 = 3.333333333` |
| `1 / 0` | 错误：除数不能为零 |
| `2 + "hello"` | 错误：表达式包含非法字符 |
| `2 ** 10` | `2 ** 10 = 1024` |
| `((2 + 3) * 4)` | `((2 + 3) * 4) = 20` |

#### 练习 2：创建文件读取 MCP Server

**难度**：⭐⭐⭐☆☆

**目标**：实现一个安全的文件读取 MCP Server。

**要求**：
- 限制访问目录范围（如只能读取 `/data` 目录下的文件）
- 支持读取文本和 JSON 文件
- 防止路径遍历攻击（`..`、符号链接等）
- 实现文件列表工具

**步骤指导**：

```
步骤 1: 设计安全模型
- 确定允许访问的根目录
- 设计路径验证逻辑
- 考虑符号链接的处理

步骤 2: 实现 FileReadTool
- 参数：path（文件路径）
- 验证路径是否在允许范围内
- 读取文件内容
- 处理异常（文件不存在、权限不足等）

步骤 3: 实现 FileListTool
- 参数：directory（目录路径，可选）
- 列出目录下的文件
- 过滤隐藏文件和子目录

步骤 4: 注册资源（可选）
- 注册配置文件为 Resource
- 实现 resources/read 方法

步骤 5: 测试
- 正常读取：path = "config.json"
- 路径遍历：path = "../../../etc/passwd"（应拒绝）
- 符号链接：创建指向外部的符号链接（应拒绝）
```

**参考代码框架**：

```python
from pathlib import Path
import json

class FileReadTool(BaseTool):
    """安全的文件读取工具"""

    def __init__(self, base_dir: str):
        """初始化

        参数：
            base_dir: 允许访问的根目录
        """
        self.base_dir = Path(base_dir).resolve()

    @property
    def name(self):
        return "read_file"

    @property
    def description(self):
        return f"读取指定文件内容（限制在 {self.base_dir} 目录内）"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "相对于基目录的文件路径"
                }
            },
            "required": ["path"]
        }

    def execute(self, **kwargs):
        path = kwargs.get("path", "")

        # 1. 基础验证
        if not path:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="路径不能为空"
            )

        # 2. 防止路径遍历
        if ".." in path or path.startswith("/"):
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="非法路径：不允许使用 '..' 或绝对路径"
            )

        # 3. 解析完整路径
        full_path = (self.base_dir / path).resolve()

        # 4. 安全检查：确保在 base_dir 内
        try:
            full_path.relative_to(self.base_dir)
        except ValueError:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="访问拒绝：路径超出允许范围"
            )

        # 5. 检查文件是否存在
        if not full_path.exists():
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error=f"文件不存在：{path}"
            )

        # 6. 检查是否是文件（不是目录）
        if not full_path.is_file():
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="指定路径不是文件"
            )

        # 7. 检查符号链接
        if full_path.is_symlink():
            real_path = full_path.resolve()
            try:
                real_path.relative_to(self.base_dir)
            except ValueError:
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    error="访问拒绝：符号链接指向外部"
                )

        # 8. 读取文件
        try:
            # 检查文件类型
            suffix = full_path.suffix.lower()

            if suffix in ['.txt', '.md', '.json', '.py', '.js', '.ts']:
                content = full_path.read_text(encoding='utf-8')
                # 限制返回大小（前 10KB）
                if len(content) > 10000:
                    content = content[:10000] + "\n...（内容过长，已截断）"
                return ToolResult(
                    status=ToolResultStatus.SUCCESS,
                    output=content
                )
            else:
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    error=f"不支持的文件类型：{suffix}"
                )
        except PermissionError:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="权限不足：无法读取该文件"
            )
        except UnicodeDecodeError:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="文件编码错误：无法解码为 UTF-8"
            )
```

**测试用例**：

| 测试场景 | 输入 | 期望结果 |
|---------|------|---------|
| 正常读取 | `path = "config.json"` | 返回文件内容 |
| 路径遍历 | `path = "../../../etc/passwd"` | 拒绝访问 |
| 绝对路径 | `path = "/etc/passwd"` | 拒绝访问 |
| 符号链接外部 | `path = "link_to_etc"` | 拒绝访问 |
| 文件不存在 | `path = "not_exists.txt"` | 返回错误 |
| 大文件 | `path = "large.txt"` (1MB) | 返回前 10KB |

#### 练习 3：将现有工具封装为 MCP Server

**难度**：⭐⭐⭐⭐☆

**目标**：将项目中的现有工具封装为 MCP Server。

**要求**：
- 选择项目中的现有工具（如搜索工具、数据库工具等）
- 创建 MCP Server 包装器
- 配置到 Claude Desktop 测试
- 实现完整的错误处理

**步骤指导**：

```
步骤 1: 选择要封装的工具
- 查看项目中现有的工具类
- 选择一个功能完整、有实际用途的工具
- 确认工具的输入输出格式

步骤 2: 创建工具适配器
- 继承 BaseTool 类
- 实现 name, description, parameters 属性
- 在 execute 方法中调用原有工具

步骤 3: 处理依赖
- 列出工具需要的依赖包
- 创建 requirements.txt
- 编写安装说明

步骤 4: 创建 Server 入口
- 创建 main 函数
- 配置 MCPServer
- 注册所有工具

步骤 5: 测试和调试
- 使用 MCP Inspector 测试
- 配置到 Claude Desktop 测试
- 记录并修复问题
```

**示例：封装一个搜索工具**

```python
# 假设原有搜索工具
class SearchEngine:
    """原有的搜索引擎"""

    def search(self, query: str, limit: int = 10) -> list:
        # ...搜索逻辑
        pass


# MCP 适配器
class SearchTool(BaseTool):
    """搜索工具的 MCP 适配器"""

    def __init__(self, search_engine: SearchEngine):
        self.search_engine = search_engine

    @property
    def name(self):
        return "search"

    @property
    def description(self):
        return "搜索文档或数据库"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索关键词"
                },
                "limit": {
                    "type": "integer",
                    "description": "返回结果数量上限",
                    "default": 10,
                    "maximum": 100
                }
            },
            "required": ["query"]
        }

    def execute(self, **kwargs):
        query = kwargs.get("query", "")
        limit = kwargs.get("limit", 10)

        if not query:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="搜索关键词不能为空"
            )

        if limit < 1 or limit > 100:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="limit 必须在 1-100 之间"
            )

        try:
            results = self.search_engine.search(query, limit=limit)
            formatted = "\n\n".join([
                f"**{r['title']}**\n{r['snippet']}\n链接：{r['url']}"
                for r in results
            ])
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output=f"找到 {len(results)} 条结果：\n\n{formatted}"
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error=f"搜索失败：{str(e)}"
            )
```

**检查清单**：

```
□ 工具名称是否清晰描述了功能？
□ 描述是否足够详细，让 LLM 知道何时使用？
□ parameters 是否符合 JSON Schema 规范？
□ 是否对所有输入参数进行了验证？
□ 是否有完善的错误处理？
□ 错误信息是否对用户友好？
□ 是否限制了返回结果的大小？
□ 日志是否输出到 stderr？
```

### 10.2 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| 连接失败 | 路径错误 | 使用绝对路径检查 server 脚本 |
| 工具不显示 | 未正确注册 | 检查 `register_tool` 调用 |
| 调用失败 | 参数不匹配 | 检查 `inputSchema` 定义 |
| Server 无响应 | 未处理 stdout | 确保 `flush=True` |
| 日志看不到 | 输出到 stdout | 日志应输出到 stderr |

### 10.3 调试工具

#### MCP Inspector

```bash
# 安装
npm install -g @modelcontextprotocol/inspector

# 运行
npx @modelcontextprotocol/inspector python weather_server.py
```

#### 日志级别设置

```python
import logging

# 启用详细日志
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    stream=sys.stderr  # 输出到 stderr，不影响 stdio 通信
)
```

#### 网络抓包（HTTP transport）

```bash
# 使用 Wireshark 或 tcpdump
tcpdump -i lo0 -s 0 -w mcp.pcap port 8080
```

---

## 第十一部分：小测验与总结

### 小测验

1. **MCP 的三层架构是什么？**
   - 答：Host 层（AI 应用）、Client 层（MCP 客户端）、Server 层（MCP 服务器）

2. **JSON-RPC Notification 的特点是什么？**
   - 答：没有 id 字段，接收方不返回响应

3. **如何配置 Claude Desktop 连接 MCP Server？**
   - 答：在 `claude_desktop_config.json` 中添加 server 配置，指定 command 和 args

4. **Stdio Transport 和 HTTP Transport 的区别是什么？**
   - 答：Stdio 用于本地通信，HTTP 用于远程通信

5. **MCP 的三种核心原语是什么？**
   - 答：Tools（工具）、Resources（资源）、Prompts（提示词）

### 总结

#### MCP 核心价值回顾

1. **标准化**：统一的工具集成接口
2. **模块化**：Server 和 Client 解耦
3. **可扩展**：支持多种传输方式
4. **安全性**：本地优先，明确授权

#### 下一步学习方向

1. **深入学习 MCP 规范**：https://modelcontextprotocol.io/specification
2. **探索社区 Server**：https://github.com/modelcontextprotocol/servers
3. **实践项目**：为自己的应用创建 MCP Server
4. **继续学习**：[Skills 系统](/docs/06_skills.md) - 为 Agent 添加专业化能力

---

**参考资料**：

1. [MCP Official Documentation](https://modelcontextprotocol.io/docs/learn/architecture)
2. [MCP Specification](https://modelcontextprotocol.io/specification/latest)
3. [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/)
4. [MCP Reference Servers](https://github.com/modelcontextprotocol/servers)