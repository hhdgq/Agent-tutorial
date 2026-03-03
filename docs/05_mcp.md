# 第五章 MCP (Model Context Protocol) 协议详解

> **本章目标**：深入理解 MCP 协议的架构设计、通信机制和实现细节，具备从零搭建 MCP Server/Client 的能力。

---

## 第一部分：MCP 概述

### 1.1 为什么需要 MCP？

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

MCP 的安全机制：

| 安全特性 | 说明 |
|---------|------|
| 本地优先 | stdio 传输仅限本地进程通信 |
| 明确授权 | 用户需手动配置允许的 Server |
| 范围限制 | 可限制 Server 的访问范围（如文件系统目录） |
| 输入验证 | Server 端可进行严格的参数验证 |

---

## 第二部分：MCP 架构深度解析

### 2.1 三层架构详解

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

| 层级 | 组件 | 职责 |
|------|------|------|
| Host 层 | AI 应用 | 提供用户界面，管理 Client 连接 |
| Client 层 | MCP Client | 与 Server 通信，转发请求/响应 |
| Server 层 | MCP Server | 提供工具、资源、提示词 |

### 2.2 参与者详解

#### MCP Host

**定义**：运行 AI 应用的主体程序。

**常见 Host**：
- Claude Desktop (macOS/Windows)
- Claude Code (CLI 工具)
- VS Code 插件 (Continue, Cline)
- 自定义 AI 应用

**Host 的职责**：
1. 管理 MCP Client 实例
2. 向用户展示可用的工具/资源
3. 处理用户授权确认
4. 转发 LLM 的工具调用请求

#### MCP Client

**定义**：内嵌在 Host 中的客户端组件。

**Client 的职责**：
1. 建立与维护 Server 的连接
2. 协议初始化与能力协商
3. 发送请求并接收响应
4. 处理连接异常

**Client 内置能力**：
```
- Sampling (补全): 请求 LLM 生成文本
- Elicitation (询问): 请求用户提供信息
- Logging (日志): 发送日志消息
```

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

| 传输方式 | 适用场景 | 优点 | 缺点 |
|---------|---------|------|------|
| Stdio | 本地 Server | 简单、低延迟 | 仅限本地 |
| HTTP | 远程 Server | 支持远程、可扩展 | 需要认证 |

---

## 第三部分：JSON-RPC 2.0 协议基础

### 3.1 JSON-RPC 2.0 消息格式

JSON-RPC 2.0 是一种轻量级的远程过程调用协议，使用 JSON 格式进行数据交换。

#### Request 格式

```json
{
  "jsonrpc": "2.0",
  "id": "request-123",
  "method": "tools/list",
  "params": {}
}
```

**字段说明**：
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| jsonrpc | String | ✅ | 固定为 "2.0" |
| id | String/Number | ✅ | 请求 ID，用于匹配响应 |
| method | String | ✅ | 方法名称 |
| params | Object/Array | ❌ | 方法参数 |

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

### 3.2 标准错误码

| 错误码 | 含义 | 说明 |
|--------|------|------|
| -32700 | Parse error | JSON 解析失败 |
| -32600 | Invalid Request | 请求格式无效 |
| -32601 | Method not found | 方法不存在 |
| -32602 | Invalid params | 参数无效 |
| -32603 | Internal error | 服务器内部错误 |

**MCP 自定义错误码**：
| 错误码 | 含义 |
|--------|------|
| -32000 | Server error (通用服务器错误) |
| -32001 | Resource not found |
| -32002 | Prompt not found |
| -32003 | Tool execution failed |

### 3.3 MCP 特定方法

MCP 在 JSON-RPC 2.0 基础上定义了特定方法：

| 类别 | 方法 | 方向 | 说明 |
|------|------|------|------|
| 基础 | `initialize` | C→S | 初始化连接 |
| 基础 | `notifications/initialized` | C→S | 通知初始化完成 |
| Tools | `tools/list` | C→S | 获取工具列表 |
| Tools | `tools/call` | C→S | 调用工具 |
| Tools | `notifications/tools/list_changed` | S→C | 工具列表变更通知 |
| Resources | `resources/list` | C→S | 获取资源列表 |
| Resources | `resources/read` | C→S | 读取资源 |
| Prompts | `prompts/list` | C→S | 获取提示词列表 |
| Prompts | `prompts/get` | C→S | 获取提示词详情 |
| Client | `sampling/createMessage` | S→C | 请求 LLM 补全 |
| Client | `elicitation/create` | S→C | 请求用户提供信息 |

---

## 第四部分：MCP 生命周期管理

### 4.1 初始化流程详解

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

**代码示例：初始化请求**

```python
# Client 发送初始化请求
await transport.send({
    "jsonrpc": "2.0",
    "id": "1",
    "method": "initialize",
    "params": {
        "protocolVersion": "2024-11-05",
        "capabilities": {
            "roots": {"listChanged": False},
            "sampling": {}
        },
        "clientInfo": {
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
        "protocolVersion": "2024-11-05",
        "capabilities": {
            "tools": {"listChanged": False},
            "resources": {"subscribe": False, "listChanged": False}
        },
        "serverInfo": {
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

### 4.2 能力协商详解

在初始化阶段，Client 和 Server 交换各自支持的能力。

#### Client Capabilities

| 能力 | 说明 |
|------|------|
| `roots` | 支持目录/文件根列表变更通知 |
| `sampling` | 支持请求 LLM 补全 |
| `elicitation` | 支持请求用户提供信息 |

#### Server Capabilities

| 能力 | 说明 |
|------|------|
| `tools` | 提供工具调用能力 |
| `resources` | 提供资源读取能力 |
| `prompts` | 提供提示词模板能力 |
| `logging` | 支持发送日志消息 |

**能力字段格式**：
```json
{
    "tools": {"listChanged": false},
    "resources": {"subscribe": false, "listChanged": false},
    "prompts": {"listChanged": false}
}
```

`listChanged`: 是否支持列表变更通知
`subscribe`: 是否支持资源订阅

### 4.3 会话终止流程

#### 正常关闭

Client 主动关闭连接：
```python
await client.close()
```

Server 检测到 EOF 后清理资源。

#### 异常处理

网络中断或进程崩溃时：
1. Transport 层检测到连接断开
2. 触发 `on_close` 回调
3. 清理本地状态和资源

---

## 第五部分：MCP 核心原语详解

### 5.1 Tools（工具）

**定义**：可执行的函数，AI 可调用执行操作。

#### tools/list - 获取工具列表

**请求**：
```json
{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "tools/list",
    "params": {}
}
```

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

#### tools/call - 调用工具

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

**定义**：可读取的数据源，提供上下文信息。

#### resources/list - 获取资源列表

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

#### resources/read - 读取资源

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

**URI 模式示例**：

| URI 模式 | 说明 |
|---------|------|
| `file://` | 文件系统路径 |
| `postgres://` | PostgreSQL 数据库 |
| `http://` / `https://` | 网络资源 |
| `config://` | 自定义配置资源 |

### 5.3 Prompts（提示词）

**定义**：可重用的提示模板。

#### prompts/list - 获取提示词列表

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

#### prompts/get - 获取提示词详情

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

### 5.4 Client 原语（新增）

MCP 还定义了 Client 端的原语，供 Server 调用。

#### sampling/createMessage - 请求 LLM 补全

Server 可以请求 Client 调用 LLM 生成文本：

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
        "maxTokens": 100
    }
}
```

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
        }
    }
}
```

#### elicitation/create - 请求用户提供信息

Server 可以请求 Client 向用户获取信息：

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

---

## 第六部分：Transport Layer 详解

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

#### 消息帧格式

使用 **Newline-delimited JSON** (NDJSON) 格式：
```
{"jsonrpc":"2.0","id":"1","method":"initialize",...}\n
{"jsonrpc":"2.0","id":"1","result":{...}}\n
```

每条消息以换行符 `\n` 结尾。

#### 代码示例

**Server 端**：
```python
async def read_loop():
    while True:
        # 读取一行（一条 JSON 消息）
        line = await loop.run_in_executor(None, sys.stdin.readline)
        if not line:
            break

        # 解析 JSON
        data = json.loads(line.strip())
        request = JSONRPCRequest.from_dict(data)

        # 处理请求
        response = await self.handle_request(request)

        # 写回响应
        print(json.dumps(response.to_dict()), flush=True)
```

**Client 端**：
```python
# 启动子进程
process = await asyncio.create_subprocess_exec(
    "python",
    "mcp_server.py",
    stdin=asyncio.subprocess.PIPE,
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.PIPE,
)

# 发送消息
message = json.dumps({"jsonrpc": "2.0", "id": "1", "method": "tools/list"})
process.stdin.write((message + "\n").encode())
await process.stdin.drain()

# 读取响应
response = await process.stdout.readline()
data = json.loads(response.decode())
```

#### 优缺点分析

| 优点 | 缺点 |
|------|------|
| 简单可靠，无需额外依赖 | 仅限本地通信 |
| 低延迟，直接进程间通信 | 单客户端，无法共享 |
| 天然隔离，每个 Client 独立 Server | 进程管理复杂 |

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

#### 消息格式

**请求** (Client → Server)：
```http
POST /mcp HTTP/1.1
Host: example.com
Content-Type: application/json

{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "tools/list"
}
```

**响应** (Server → Client via SSE)：
```http
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"jsonrpc":"2.0","id":"1","result":{...}}

```

#### 认证方式

| 认证方式 | 说明 |
|---------|------|
| OAuth 2.0 | 标准 OAuth 2.0 授权流程 |
| Bearer Token | 简单的 Token 认证 |
| API Key | 通过 Header 传递 API Key |

**示例**：
```http
Authorization: Bearer your_access_token
```
或
```http
X-API-Key: your_api_key
```

#### 优缺点分析

| 优点 | 缺点 |
|------|------|
| 支持远程通信 | 需要处理认证 |
| 支持多客户端 | 延迟相对较高 |
| 易于部署和扩展 | 需要 HTTP 服务器 |

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
"""

import sys
import json
from src.mcp import MCPServer, ServerConfig
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class WeatherTool(BaseTool):
    """天气查询工具（模拟数据）"""

    @property
    def name(self): return "get_weather"

    @property
    def description(self): return "获取指定城市的天气信息"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"}
            },
            "required": ["city"]
        }

    def execute(self, **kwargs):
        city = kwargs.get("city", "")
        mock_data = {
            "北京": "晴，25°C",
            "上海": "多云，28°C",
            "广州": "雨，30°C",
        }
        result = mock_data.get(city, "未知城市")
        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=f"{city}天气：{result}"
        )


if __name__ == "__main__":
    config = ServerConfig(name="Weather Server", version="1.0.0")
    server = MCPServer(config=config)
    server.register_tool(WeatherTool())
    print("Starting MCP Weather Server...", file=sys.stderr)
    server.run_stdio()
```

**运行并测试**：
```bash
# 启动 Server（等待客户端连接）
python examples/mcp_weather_server.py
```

**参考输出**（stderr 日志）：
```
2025-03-02 10:00:00 - INFO - Initialized MCP Server: Weather Server
2025-03-02 10:00:01 - INFO - Starting MCP server with stdio transport
2025-03-02 10:00:02 - DEBUG - Received: {"jsonrpc":"2.0","id":"1","method":"initialize",...}
2025-03-02 10:00:02 - DEBUG - Sent: {"jsonrpc":"2.0","id":"1","result":{"protocolVersion":"2024-11-05",...}
```

### 7.2 MCP Client 连接示例

创建 `examples/mcp_client_test.py`：

```python
#!/usr/bin/env python3
"""
MCP Client 连接示例
"""

import asyncio
import sys
sys.path.insert(0, "..")

from src.mcp import MCPClient


async def main():
    client = MCPClient()

    # 连接到 Weather Server
    await client.connect_stdio(
        command="python",
        args=["mcp_weather_server.py"],
    )

    # 获取服务器信息
    print(f"Connected to: {client.server_info.name}")
    print(f"Server version: {client.server_info.version}")

    # 列出工具
    tools = await client.list_tools()
    print(f"Available tools: {[t.name for t in tools]}")

    # 调用工具
    result = await client.call_tool("get_weather", {"city": "北京"})
    print(f"Result: {result}")

    # 关闭连接
    await client.close()


if __name__ == "__main__":
    asyncio.run(main())
```

**参考输出**：
```
Connected to: Weather Server
Server version: 1.0.0
Available tools: ['get_weather']
Result: 北京天气：晴，25°C
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

### 8.1 配置文件位置

| 操作系统 | 配置文件路径 |
|---------|-------------|
| **macOS** | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json` |
| **Linux** | `~/.config/Claude/claude_desktop_config.json` |

### 8.2 配置格式详解

```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/path/to/weather_server.py"],
      "env": {
        "API_KEY": "your_api_key"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "./data"]
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"]
    }
  }
}
```

**配置字段说明**：

| 字段 | 说明 |
|------|------|
| `command` | 启动命令（如 `python`, `npx`） |
| `args` | 命令参数列表 |
| `env` | 环境变量（可选） |

### 8.3 调试技巧

#### 查看 Claude Desktop 日志

**macOS**：
```bash
tail -F ~/Library/Logs/Claude/*.log
```

**Windows**：
```powershell
Get-Content "$env:APPDATA\Claude\logs\*.log" -Wait -Tail 50
```

#### 使用 MCP Inspector

安装 MCP Inspector：
```bash
npx -y @modelcontextprotocol/inspector
```

运行 Inspector：
```bash
npx -y @modelcontextprotocol/inspector python weather_server.py
```

Inspector 提供：
- 可视化查看可用工具
- 手动测试工具调用
- 查看原始 JSON-RPC 消息

#### 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|---------|---------|
| Server 不显示 | 路径错误 | 使用绝对路径 |
| 工具调用失败 | 参数不匹配 | 检查 inputSchema |
| 连接超时 | Server 启动慢 | 增加超时时间 |
| 权限错误 | 文件不可执行 | `chmod +x server.py` |

---

## 第九部分：高级主题

### 9.1 无状态 MCP Server

#### 使用 Streamable HTTP 的无状态模式

无状态 Server 不维护会话状态，每次请求独立处理：

```python
class StatelessMCPServer:
    def __init__(self):
        self.tools = ToolRegistry()

    async def handle_request(self, request: dict) -> dict:
        # 每次请求都是独立的
        method = request.get("method")
        params = request.get("params", {})

        if method == "tools/list":
            return self._list_tools()
        elif method == "tools/call":
            return self._call_tool(params)
        else:
            return {"error": {"code": -32601, "message": "Method not found"}}
```

#### 会话管理

对于需要状态的 Server，可以使用会话 ID：

```python
import uuid

class SessionManager:
    def __init__(self):
        self.sessions = {}

    def create_session(self) -> str:
        session_id = str(uuid.uuid4())
        self.sessions[session_id] = {}
        return session_id

    def get_session(self, session_id: str) -> dict:
        return self.sessions.get(session_id)
```

#### 适用场景

| 场景 | 推荐模式 |
|------|---------|
| 简单工具服务 | 无状态 |
| 需要上下文对话 | 有状态 |
| 高并发场景 | 无状态 + 外部存储 |

### 9.2 安全性考虑

#### 输入验证

```python
def execute(self, **kwargs):
    city = kwargs.get("city", "")

    # 防止注入攻击
    if not city or not city.isalnum():
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error="Invalid city name"
        )

    # 防止路径遍历攻击
    path = kwargs.get("path", "")
    if ".." in path or path.startswith("/"):
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error="Invalid path"
        )

    return self._execute_tool(city)
```

#### 路径遍历防护

```python
from pathlib import Path

class FilesystemTool(BaseTool):
    def __init__(self, base_dir: str):
        self.base_dir = Path(base_dir).resolve()

    def execute(self, **kwargs):
        path = kwargs.get("path", "")
        full_path = (self.base_dir / path).resolve()

        # 安全检查：确保在 base_dir 内
        if not str(full_path).startswith(str(self.base_dir)):
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="Access denied: Path traversal detected"
            )

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=full_path.read_text()
        )
```

#### OAuth 2.0 集成

```python
from authlib.integrations.httpx_client import OAuth2Client

class OAuthMCPClient:
    def __init__(self, client_id: str, client_secret: str, token_url: str):
        self.client = OAuth2Client(
            client_id=client_id,
            client_secret=client_secret,
            token_endpoint=token_url,
        )

    async def get_token(self, username: str, password: str):
        token = await self.client.fetch_token(
            grant_type="password",
            username=username,
            password=password,
        )
        return token["access_token"]

    async def request(self, method: str, url: str, **kwargs):
        token = await self.get_token(...)
        headers = kwargs.get("headers", {})
        headers["Authorization"] = f"Bearer {token}"
        # 发送请求...
```

#### 权限控制

```python
class Permission:
    READ = "read"
    WRITE = "write"
    EXECUTE = "execute"

class AccessControl:
    def __init__(self):
        self.permissions = {}

    def grant(self, user: str, resource: str, permission: Permission):
        key = f"{user}:{resource}"
        self.permissions[key] = permission

    def check(self, user: str, resource: str, required: Permission) -> bool:
        key = f"{user}:{resource}"
        return self.permissions.get(key) == required
```

### 9.3 性能优化

#### 连接池

对于 HTTP Transport：

```python
import aiohttp

class HTTPConnectionPool:
    def __init__(self, max_size: int = 10):
        self.max_size = max_size
        self.pool = aiohttp.ClientSession(
            connector=aiohttp.TCPConnector(limit=max_size)
        )

    async def request(self, url: str, **kwargs):
        async with self.pool.post(url, **kwargs) as response:
            return await response.json()

    async def close(self):
        await self.pool.close()
```

#### 缓存策略

```python
from functools import lru_cache
import time

class CachedMCPServer:
    def __init__(self, cache_ttl: int = 60):
        self.cache_ttl = cache_ttl
        self._cache = {}

    def _get_cache_key(self, method: str, params: dict) -> str:
        return f"{method}:{hash(str(params))}"

    async def handle_request(self, request: dict) -> dict:
        method = request.get("method")
        params = request.get("params", {})

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

```python
import asyncio

class ConcurrentMCPServer:
    def __init__(self, max_concurrency: int = 10):
        self.semaphore = asyncio.Semaphore(max_concurrency)

    async def handle_request(self, request: dict) -> dict:
        async with self.semaphore:
            return await self._execute(request)

    async def _execute(self, request: dict) -> dict:
        # 执行请求...
        pass
```

---

## 第十部分：实践练习与故障排查

### 10.1 实践练习

#### 练习 1：创建简单的计算器 MCP Server

**要求**：
- 实现加减乘除四个工具
- 支持错误处理（除零、溢出等）

**参考代码**：
```python
class CalculatorTool(BaseTool):
    @property
    def name(self): return "calculate"

    @property
    def description(self): return "执行数学计算"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，如 2 + 3"
                }
            },
            "required": ["expression"]
        }

    def execute(self, **kwargs):
        expression = kwargs.get("expression", "")
        try:
            # 安全计算（限制可用操作符）
            allowed_chars = set("0123456789+-*/.() ")
            if not all(c in allowed_chars for c in expression):
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    error="Invalid characters in expression"
                )
            result = eval(expression)
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output=str(result)
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error=f"Calculation error: {str(e)}"
            )
```

#### 练习 2：创建文件读取 MCP Server

**要求**：
- 限制访问目录范围
- 支持读取文本和 JSON 文件
- 防止路径遍历攻击

#### 练习 3：将现有工具封装为 MCP Server

**要求**：
- 选择项目中的现有工具
- 创建 MCP Server 包装器
- 配置到 Claude Desktop 测试

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