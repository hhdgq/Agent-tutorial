# 第一章 AI Agent 基础概念

> **本章目标**：从零开始理解 AI Agent 的核心概念，掌握搭建完整 Agent 的能力
>
> **预计阅读时间**：45-60 分钟
>
> **实践时间**：60-90 分钟

---

## 第一部分：引言——从图灵测试到 AI Agent

## 1.1 智能的度量：图灵测试的启示

### 历史背景

1950 年，英国数学家**艾伦·图灵**（Alan Turing）发表了一篇划时代的论文《计算机器与智能》（Computing Machinery and Intelligence）。在这篇论文中，图灵提出了一个至今仍被广泛讨论的问题：**"机器能思考吗？"**（Can machines think?）

图灵意识到，要回答这个问题，首先需要定义"思考"的含义。为了避免陷入哲学争论，他设计了一个巧妙的实验——**"模仿游戏"**（Imitation Game），后来被称为**图灵测试**（Turing Test）。

### 图灵测试的设计

```
┌──────────────────────────────────────────────────────────────────┐
│                        图灵测试设置                               │
│                                                                  │
│   ┌──────────┐         ┌──────────┐         ┌──────────┐        │
│   │  人类    │         │  评判者  │         │  机器    │        │
│   │  (A)     │◄───────►│  (C)     │◄───────►│  (B)     │        │
│   └──────────┘  文本    └──────────┘  文本    └──────────┘        │
│                  对话                  对话                        │
│                                                                  │
│   评判者目标：通过文本对话判断哪一方是人类，哪一方是机器           │
│   通过标准：如果机器能让 30% 以上的评判者误判为人类，则通过测试    │
└──────────────────────────────────────────────────────────────────┘
```

### 图灵测试的核心思想

图灵测试的革命性在于它**不关心机器是否真的"思考"**，而是关注机器的**行为表现**是否与人类无法区分。这种"行为主义"方法避开了难以定义的"意识"问题，将焦点放在**可观察的输入 - 输出行为**上。

### 现代 AI 对图灵测试的超越

70 多年后的今天，AI 领域已经发生了翻天覆地的变化：

| 维度 | 图灵测试时代 | 现代 AI Agent |
|------|-------------|--------------|
| **目标** | "像人一样思考" | "像人一样行动" |
| **重点** | 欺骗评判者 | 完成实际任务 |
| **能力** | 文本对话 | 感知、决策、执行、学习 |
| **评估** | 人类误判率 | 任务完成率、效率 |

现代 AI Agent 不再追求"假装是人类"，而是追求**高效完成目标任务**。一个优秀的 Agent 应该：
- 在需要计算时，展现超越人类的计算能力
- 在需要记忆时，调用无限的存储空间
- 在需要行动时，操作各种工具和 API
- 在需要协作时，与其他 Agent 或人类高效配合

---

## 1.2 什么是 AI Agent？

### 学术定义

根据人工智能领域的权威教科书——**Russell & Norvig《人工智能：一种现代方法》**（Artificial Intelligence: A Modern Approach）中的定义：

> **Agent**（智能体）是指任何能够**感知其环境**并通过**行动**来最大化其成功概率的实体。
>
> **公式化表达**：Agent = 架构 + 程序
> - **架构**：感知和行动的硬件/软件平台
> - **程序**：将感知映射到行动的函数

### 通俗理解：生活化类比

让我们用几个生活中的例子来理解 AI Agent：

#### 类比 1：快递员

| 场景 | 传统程序 | AI Agent（快递员） |
|------|---------|------------------|
| 正常路况 | 按预定路线行驶 | 按预定路线行驶 |
| 道路施工 | ❌ 卡住，无法继续 | ✅ 观察路牌，选择绕行路线 |
| 收件人不在 | ❌ 报告错误 | ✅ 打电话联系或放快递柜 |
| 包裹易碎 | ❌ 无特殊处理 | ✅ 轻拿轻放，标注"易碎" |

**关键区别**：快递员能够**感知环境变化**（修路、收件人不在），并**动态调整策略**（绕行、联系收件人）。

#### 类比 2：私人助理

想象你有一个私人助理：

- **传统程序**：像你手机上的闹钟应用——只能在你设置的时间响铃
- **AI Agent**：像人类私人助理——不仅会设闹钟，还会：
  - 提醒你"今天有重要会议，建议提前出发"
  - 帮你"查询会议地点的交通状况"
  - 在"会议时间变更时主动通知你"

### AI Agent 的四大核心特征

根据智能体理论，一个完整的 AI Agent 应具备以下特征：

#### 1. 自主性（Autonomy）

**定义**：无需人类持续干预即可做出决策和行动。

**示例**：
```
人类：帮我安排下周的出差行程
Agent:
  1. 自动查询目的地天气
  2. 自动比较航班价格
  3. 自动预订酒店
  4. 自动生成行程单
  → 全程无需人类逐步指导
```

#### 2. 反应性（Reactivity）

**定义**：能够感知环境变化并及时做出响应。

**示例**：
```
场景：你正在开车前往机场，Agent 监控到航班延误
Agent 行动：
  1. 检测到航班状态变化
  2. 计算新的最优出发时间
  3. 通知你"航班延误 2 小时，建议晚点出发"
```

#### 3. 主动性（Pro-activeness）

**定义**：能够主动追求目标，而不仅仅是被动响应。

**示例**：
```
人类：帮我关注某股票的价格
Agent 主动行为：
  - 当价格跌破阈值时，主动通知"XX 股票已跌破你设定的预警线"
  - 当有利好消息时，主动推送"XX 公司发布财报，股价上涨 5%"
```

#### 4. 社会性（Social Ability）

**定义**：能够与其他 Agent 或人类进行交互和协作。

**示例**：
```
多 Agent 协作场景：
  旅行 Agent ←→ 航班 Agent ←→ 酒店 Agent ←→ 日历 Agent
      ↓            ↓             ↓            ↓
   协调行程     预订机票      预订酒店      安排时间
```

### 对比表格：传统程序 vs 规则系统 vs AI Agent

| 特性 | 传统程序 | 规则系统 | AI Agent |
|------|---------|---------|---------|
| **决策方式** | 硬编码逻辑 | IF-THEN 规则 | LLM 推理 + 工具调用 |
| **灵活性** | 低（修改需改代码） | 中（需添加规则） | 高（动态适应） |
| **知识范围** | 限于程序逻辑 | 限于规则库 | LLM 训练数据 + 实时工具 |
| **错误处理** | 异常捕获 | 默认规则 | 自我反思 + 纠正 |
| **学习能力** | 无 | 无（除非显式添加） | 有（从反馈中学习） |
| **适用场景** | 确定性任务 | 结构化决策 | 开放式任务 |

---

## 1.3 Agent 的演进历程

### 时间线总览

```
1950s          1980s          2000s          2020s
  │              │              │              │
  ▼              ▼              ▼              ▼
┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐
│符号  │      │连接  │      │深度  │      │大语言│
│主义  │ ───► │主义  │ ───► │学习  │ ───► │模型  │
│Agent │      │Agent │      │Agent │      │Agent │
└──────┘      └──────┘      └──────┘      └──────┘
  │              │              │              │
  │              │              │              │
基于规则       神经网络       强化学习       LLM 核心
专家系统       初步应用       AlphaGo        通用 Agent
```

### 第一阶段：符号主义 Agent（1950s-1980s）

**核心思想**：智能源于对符号的操作和推理。

**代表系统**：
- **ELIZA**（1966）：最早的心理治疗聊天机器人，使用模式匹配和规则响应
- **SHRDLU**（1968-1970）：能在积木世界中理解自然语言并操作积木
- **专家系统**（1970s-1980s）：MYCIN（医疗诊断）、XCON（计算机配置）

**架构特点**：
```
┌─────────────────────────────────────┐
│         符号主义 Agent               │
│                                     │
│  ┌─────────────────────────────┐   │
│  │      知识库 (Knowledge)      │   │
│  │  IF 症状 A AND 症状 B        │   │
│  │  THEN 疾病 C (置信度 0.8)    │   │
│  └─────────────────────────────┘   │
│              │                      │
│              ▼                      │
│  ┌─────────────────────────────┐   │
│  │    推理引擎 (Inference)      │   │
│  │    正向链 / 反向链           │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

**局限性**：
- 需要人工编写大量规则，成本高
- 无法处理模糊、不确定的情况
- 缺乏学习能力

### 第二阶段：连接主义 Agent（1990s-2010s）

**核心思想**：智能源于大量简单单元（神经元）的连接。

**关键技术**：
- **神经网络**：模拟生物神经元的计算模型
- **强化学习**：Agent 通过与环境交互学习最优策略
- **深度学习**：多层神经网络实现端到端学习

**代表成果**：
- **TD-Gammon**（1992）：使用强化学习达到世界级水平的双陆棋程序
- **AlphaGo**（2016）：击败人类世界冠军的围棋 AI
- **DQN**（2015）：能从像素输入学习玩 Atari 游戏

**架构特点**：
```
┌─────────────────────────────────────┐
│        连接主义 Agent                │
│                                     │
│  环境状态 ──► 神经网络 ──► 动作     │
│               │                     │
│               │  奖励信号           │
│               ▼                     │
│          策略更新                   │
│        (强化学习算法)               │
└─────────────────────────────────────┘
```

**局限性**：
- 需要大量训练数据
- 泛化能力有限（围棋 AI 不会下象棋）
- 难以解释决策过程（"黑箱"问题）

### 第三阶段：大语言模型 Agent（2022-至今）

**核心思想**：利用预训练的 LLM 作为通用推理引擎，结合工具调用实现通用 Agent 能力。

**关键突破**：
- **预训练 + 微调范式**：在海量文本上预训练，获得通用知识和推理能力
- **In-context Learning**：通过提示词（Prompt）引导模型完成任务
- **Function Calling**：LLM 能够结构化输出工具调用指令
- **ReAct 模式**：将推理和行动统一在一个循环中

**代表系统**：
- **AutoGPT**（2023）：能够自主完成复杂任务的开源项目
- **LangChain**：构建 LLM 应用的框架
- **Claude Desktop**：支持 MCP（Model Context Protocol）的桌面应用

**架构特点**：
```
┌─────────────────────────────────────────────┐
│          LLM-Powered Agent                   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │            LLM (大脑)                │   │
│  │   预训练知识 + 推理能力 + 语言理解    │   │
│  └─────────────────────────────────────┘   │
│           │              │                  │
│     理解意图        生成工具调用            │
│           │              │                  │
│           ▼              ▼                  │
│  ┌─────────────────┐ ┌─────────────────┐   │
│  │    Memory       │ │     Tools       │   │
│  │   (记忆系统)     │ │   (工具系统)     │   │
│  └─────────────────┘ └─────────────────┘   │
└─────────────────────────────────────────────┘
```

**优势**：
- 通用性强：同一模型可处理多种任务
- 零样本/少样本学习：无需训练即可适应新任务
- 可解释性：推理过程可通过对话追踪

---

# 第二部分：Agent 核心架构深度解析

## 2.1 完整架构图

让我们深入理解 AI Agent 的完整架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                         AI Agent                                 │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    感知层 (Perception)                     │ │
│  │  用户输入 │ 环境观察 │ 工具返回 │ API 响应 │ 文件读取      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    认知层 (Cognition)                      │ │
│  │                                                           │ │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐   │ │
│  │  │     LLM     │    │   Memory    │    │  Planning   │   │ │
│  │  │   (大脑)    │◄──►│   (记忆)    │◄──►│   (规划)    │   │ │
│  │  │             │    │             │    │             │   │ │
│  │  │ - 理解意图  │    │ - 短期记忆  │    │ - 任务分解  │   │ │
│  │  │ - 推理决策  │    │ - 长期记忆  │    │ - 策略选择  │   │ │
│  │  │ - 生成响应  │    │ - 记忆压缩  │    │ - 进度跟踪  │   │ │
│  │  └─────────────┘    └─────────────┘    └─────────────┘   │ │
│  └───────────────────────────────────────────────────────────┘ │
│                              │                                   │
│                              ▼                                   │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    执行层 (Execution)                      │ │
│  │                    Tools (工具系统)                        │ │
│  │                                                           │ │
│  │   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        │ │
│  │   │ 计算   │  │ 搜索   │  │ 文件   │  │ API    │        │ │
│  │   │ 工具   │  │ 工具   │  │ 工具   │  │ 工具   │        │ │
│  │   └────────┘  └────────┘  └────────┘  └────────┘        │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**架构解析**：

1. **感知层**：负责接收各种输入
   - 用户文本/语音输入
   - 环境状态观察
   - 工具执行结果
   - 外部 API 响应

2. **认知层**：核心处理单元
   - **LLM**：理解、推理、决策
   - **Memory**：存储和检索信息
   - **Planning**：任务分解和规划

3. **执行层**：将决策转化为行动
   - 各种专用工具
   - API 调用
   - 文件操作

---

## 2.2 LLM（大语言模型）- 大脑深度解析

### 2.2.1 LLM 如何工作

#### Transformer 架构简述

现代 LLM 都基于**Transformer**架构（Vaswani et al., 2017）。让我们用简化的方式理解其核心机制：

```
┌─────────────────────────────────────────────────────────┐
│              Transformer 架构（简化版）                    │
│                                                         │
│  输入文本："今天天气真好"                                │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────┐                                    │
│  │   Tokenization  │                                    │
│  │   (分词化)      │                                    │
│  └─────────────────┘                                    │
│       │                                                 │
│       ▼                                                 │
│  ["今天", "天气", "真", "好"]                            │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────┐                                    │
│  │   Embedding     │                                    │
│  │   (向量化)      │                                    │
│  └─────────────────┘                                    │
│       │                                                 │
│       ▼                                                 │
│  [0.23, -0.45, ...], [0.67, 0.12, ...], ...            │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────┐                                    │
│  │  Self-Attention │                                    │
│  │  (自注意力机制)  │◄── 每个词关注其他相关词             │
│  └─────────────────┘                                    │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────┐                                    │
│  │  Feed-Forward   │                                    │
│  │  (前馈网络)     │                                    │
│  └─────────────────┘                                    │
│       │                                                 │
│       ▼                                                 │
│  输出概率分布：P(下一个 token | 上下文)                   │
│       │                                                 │
│       ▼                                                 │
│  采样/贪婪选择 → "啊"                                    │
└─────────────────────────────────────────────────────────┘
```

**关键概念**：

1. **Tokenization（分词化）**
   - 将文本切分为模型可处理的单元（tokens）
   - 中文：通常按词或字切分
   - 英文：按词根、词缀切分
   - 示例："Hello world!" → ["Hello", " world", "!"]

2. **Embedding（向量化）**
   - 将每个 token 映射为高维向量（如 4096 维）
   - 语义相似的词向量距离更近
   - "国王" - "男人" + "女人" ≈ "女王"

3. **Self-Attention（自注意力机制）**
   - 让模型在处理每个词时，能够"关注"其他相关词
   - 解决长距离依赖问题
   - 示例："动物过**马路**，因为**它**想..." → "它"关注"动物"

4. **自回归生成**
   - 每次预测下一个 token
   - 将生成的 token 加入上下文，继续预测
   - 直到生成结束标记或达到长度限制

#### 预训练 + 微调范式

```
┌─────────────────────────────────────────────────────────┐
│              LLM 训练流程                                │
│                                                         │
│  第一阶段：预训练（Pre-training）                        │
│  ═══════════════════════════                           │
│  数据：互联网规模的文本（TB 级别）                        │
│  任务：预测下一个 token（自监督学习）                    │
│  结果：获得通用语言理解和世界知识                        │
│                                                         │
│  第二阶段：指令微调（Instruction Tuning）                │
│  ════════════════════════════                          │
│  数据：人工编写的高质量指令 - 响应对                     │
│  任务：理解并遵循指令                                   │
│  结果：能够按用户要求完成任务                            │
│                                                         │
│  第三阶段：人类对齐（RLHF/RLAIF）                       │
│  ═══════════════════════════                           │
│  数据：人类对多个响应的偏好评分                          │
│  任务：学习人类偏好（有帮助、无害、诚实）                │
│  结果：输出符合人类价值观                                │
└─────────────────────────────────────────────────────────┘
```

### 2.2.2 主流 LLM 对比

| 模型系列 | 提供商 | 上下文窗口 | 特点 | 适用场景 | 相对成本 |
|---------|--------|-----------|------|---------|---------|
| **GPT-4/4o** | OpenAI | 128K | 综合能力最强，多模态能力强 | 通用任务、复杂推理 | 高 |
| **GPT-4-Turbo** | OpenAI | 128K | 性价比平衡 | 日常应用 | 中 |
| **Claude 3.5/3.7** | Anthropic | 200K | 长文本理解好，安全性高 | 文档分析、代码审查 | 高 |
| **Claude 3 Haiku** | Anthropic | 200K | 速度快，成本低 | 简单任务、实时响应 | 低 |
| **Qwen-2.5/3** | 阿里 | 32K-128K | 中文能力强，本地化好 | 中文应用、国内部署 | 中低 |
| **Qwen-Turbo** | 阿里 | 32K | 速度最快 | 高并发场景 | 低 |
| **Gemini 1.5/2.0** | Google | 1M+ | 超长上下文，多模态 | 视频/长文档分析 | 中高 |
| **Llama 3** | Meta | 8K-128K | 开源，可自部署 | 私有化部署、定制 | 低（自建）|

**选择建议**：

| 需求 | 推荐模型 | 理由 |
|------|---------|------|
| 追求最佳效果 | GPT-4o / Claude 3.7 | 综合能力最强 |
| 中文应用 | Qwen 系列 | 中文训练数据多 |
| 长文档处理 | Claude 3.5 / Gemini 1.5 | 上下文窗口大 |
| 成本敏感 | Qwen-Turbo / Haiku | 单价低 |
| 数据隐私 | Llama 3（自部署）| 本地运行 |

### 2.2.3 LLM 客户端实现详解

让我们实现一个完整的 LLM 客户端，包含所有必要的功能：

```python
"""
完整的 LLM 客户端实现

功能包含：
1. API 调用封装
2. 错误处理与重试
3. 流式响应支持
4. Token 计数
5. 成本估算
"""

import time
import logging
from typing import List, Dict, Any, Optional, Generator
from dataclasses import dataclass
from enum import Enum

# 假设使用 OpenAI 兼容的 SDK
from openai import OpenAI, APIError, RateLimitError, Timeout

logger = logging.getLogger(__name__)


class LLMProvider(Enum):
    """支持的 LLM 提供商"""
    BAILIAN = "bailian"      # 阿里百炼
    OPENAI = "openai"        # OpenAI
    ANTHROPIC = "anthropic"  # Anthropic
    MOCK = "mock"           # 模拟（测试用）


@dataclass
class LLMConfig:
    """LLM 配置"""
    provider: LLMProvider
    api_key: str
    model: str
    base_url: Optional[str] = None
    temperature: float = 0.7
    max_tokens: int = 4096
    timeout: int = 60
    max_retries: int = 3
    retry_delay: float = 1.0


@dataclass
class LLMResponse:
    """LLM 响应"""
    content: str
    role: str = "assistant"
    usage: Optional[Dict[str, int]] = None
    finish_reason: Optional[str] = None
    latency_ms: int = 0
    cost_usd: float = 0.0


class LLMClient:
    """
    LLM 客户端

    提供统一的接口访问不同的 LLM 提供商
    """

    # 定价信息（每 1K tokens，美元）
    PRICING = {
        "gpt-4o": {"input": 0.005, "output": 0.015},
        "gpt-4-turbo": {"input": 0.01, "output": 0.03},
        "claude-3-7-sonnet": {"input": 0.003, "output": 0.015},
        "qwen-plus": {"input": 0.0005, "output": 0.0015},
        "qwen-turbo": {"input": 0.0001, "output": 0.0003},
    }

    def __init__(self, config: LLMConfig):
        self.config = config
        self.client = self._create_client()
        self.total_tokens = {"input": 0, "output": 0}
        self.total_cost = 0.0

    def _create_client(self) -> OpenAI:
        """创建底层客户端"""
        if self.config.provider == LLMProvider.MOCK:
            return None

        base_url = self.config.base_url
        if self.config.provider == LLMProvider.BAILIAN:
            base_url = base_url or "https://dashscope.aliyuncs.com/compatible-mode/v1"

        return OpenAI(
            api_key=self.config.api_key,
            base_url=base_url,
            timeout=self.config.timeout,
            max_retries=self.config.max_retries,
        )

    def _calculate_cost(self, usage: Dict[str, int]) -> float:
        """计算调用成本"""
        model = self.config.model
        if model not in self.PRICING:
            return 0.0

        pricing = self.PRICING[model]
        input_cost = (usage.get("prompt_tokens", 0) / 1000) * pricing["input"]
        output_cost = (usage.get("completion_tokens", 0) / 1000) * pricing["output"]
        return input_cost + output_cost

    def _retry_with_backoff(self, func, *args, **kwargs):
        """带指数退避的重试"""
        last_error = None
        for attempt in range(self.config.max_retries + 1):
            try:
                return func(*args, **kwargs)
            except RateLimitError as e:
                last_error = e
                if attempt < self.config.max_retries:
                    delay = self.config.retry_delay * (2 ** attempt)
                    logger.warning(f"速率限制，等待 {delay} 秒后重试...")
                    time.sleep(delay)
            except (APIError, Timeout) as e:
                last_error = e
                if attempt < self.config.max_retries:
                    delay = self.config.retry_delay * (2 ** attempt)
                    logger.warning(f"API 错误，等待 {delay} 秒后重试...")
                    time.sleep(delay)

        raise last_error

    def chat(
        self,
        messages: List[Dict[str, str]],
        tools: Optional[List[Dict[str, Any]]] = None,
        stream: bool = False,
    ) -> LLMResponse | Generator[str, None, None]:
        """
        发送聊天请求

        Args:
            messages: 消息历史，格式为 [{"role": "user", "content": "..."}]
            tools: 可选的工具定义列表（OpenAI function calling 格式）
            stream: 是否使用流式响应

        Returns:
            LLMResponse 或 流式生成器
        """
        start_time = time.time()

        def _make_request():
            kwargs = {
                "model": self.config.model,
                "messages": messages,
                "temperature": self.config.temperature,
                "max_tokens": self.config.max_tokens,
            }

            if tools:
                kwargs["tools"] = tools
                kwargs["tool_choice"] = "auto"

            if stream:
                return self.client.chat.completions.create(**kwargs, stream=True)
            else:
                return self.client.chat.completions.create(**kwargs)

        if stream:
            def generator():
                response = self._retry_with_backoff(_make_request)
                for chunk in response:
                    if chunk.choices[0].delta.content:
                        yield chunk.choices[0].delta.content

            return generator()

        else:
            response = self._retry_with_backoff(_make_request)

            # 更新统计
            usage = response.usage.model_dump() if response.usage else {}
            self.total_tokens["input"] += usage.get("prompt_tokens", 0)
            self.total_tokens["output"] += usage.get("completion_tokens", 0)

            cost = self._calculate_cost(usage)
            self.total_cost += cost

            latency = int((time.time() - start_time) * 1000)

            return LLMResponse(
                content=response.choices[0].message.content or "",
                role=response.choices[0].message.role,
                usage=usage,
                finish_reason=response.choices[0].finish_reason,
                latency_ms=latency,
                cost_usd=cost,
            )

    def get_statistics(self) -> Dict[str, Any]:
        """获取使用统计"""
        return {
            "total_input_tokens": self.total_tokens["input"],
            "total_output_tokens": self.total_tokens["output"],
            "total_tokens": sum(self.total_tokens.values()),
            "total_cost_usd": self.total_cost,
        }
```

**使用示例**：

```python
# 初始化客户端
config = LLMConfig(
    provider=LLMProvider.BAILIAN,
    api_key="your_api_key",
    model="qwen-plus",
    temperature=0.7,
)
client = LLMClient(config)

# 简单对话
messages = [
    {"role": "system", "content": "你是一个有帮助的助手。"},
    {"role": "user", "content": "你好，请介绍一下自己。"},
]

response = client.chat(messages)
print(f"响应：{response.content}")
print(f"Token 使用：{response.usage}")
print(f"成本：${response.cost_usd:.4f}")
print(f"延迟：{response.latency_ms}ms")

# 流式响应
print("\n流式响应：")
for chunk in client.chat(messages, stream=True):
    print(chunk, end="", flush=True)
print()

# 获取统计
stats = client.get_statistics()
print(f"\n统计：{stats}")
```

**示例输出**：

```
响应：你好！我是一个 AI 助手，由阿里巴巴开发...
Token 使用：{'prompt_tokens': 25, 'completion_tokens': 89, 'total_tokens': 114}
成本：$0.0001
延迟：342ms

流式响应：
你好！我是一个 AI 助手，由阿里巴巴开发...

统计：{'total_input_tokens': 25, 'total_output_tokens': 89, 'total_tokens': 114, 'total_cost_usd': 0.0001335}
```

### 2.2.4 Prompt Engineering 基础

**Prompt（提示词）**是指导 LLM 行为的关键。优秀的 Prompt 设计能显著提升 Agent 的表现。

#### Zero-shot Prompting（零样本提示）

直接提问，不提供示例：

```
请总结以下文章的主要内容：

[文章内容]
```

**适用场景**：简单任务、通用知识

#### Few-shot Prompting（少样本提示）

提供几个示例，引导模型理解任务格式：

```
请将以下句子翻译成英文：

中文：今天天气真好
英文：The weather is really nice today.

中文：我喜欢吃苹果
英文：I like to eat apples.

中文：他正在学习编程
英文：_______
```

**适用场景**：格式固定的任务、特殊领域

#### System Prompt 设计技巧

System Prompt 是设定 Agent 人设和行为准则的关键。

**好的 System Prompt 示例**：

```
你是一个专业的数据分析助手。

你的职责：
1. 帮助用户分析数据、生成统计报告
2. 解释分析结果，提供可操作的见解
3. 使用清晰、简洁的语言

可用工具：
- calculate：执行数学计算
- search：搜索信息
- read_file：读取数据文件

注意事项：
- 在进行计算前，先确认数据格式
- 对于不确定的信息，说明不确定性
- 优先使用图表展示复杂数据

回答格式：
1. 先给出核心结论
2. 然后展示分析过程
3. 最后提供建议
```

**设计原则**：

| 原则 | 说明 | 示例 |
|------|------|------|
| **角色定义** | 明确 Agent 的身份 | "你是一个专业的数据分析助手" |
| **职责边界** | 说明做什么、不做什么 | "你只提供数据分析，不提供投资建议" |
| **工具说明** | 列出可用工具及用途 | "可用工具：calculate（计算）、search（搜索）" |
| **行为准则** | 指导如何决策 | "对于不确定的信息，说明不确定性" |
| **输出格式** | 规范响应格式 | "先给出核心结论，然后展示分析过程" |

---

## 2.3 Tools（工具）- 手脚深度解析

### 2.3.1 为什么需要工具

**LLM 的固有限制**：

1. **知识截止时间**
   - LLM 的知识来自训练数据，有明确的时间截止点
   - 无法知道训练后发生的事件
   - 示例：GPT-4 训练截止于 2023 年，不知道 2024 年的新闻

2. **无法执行操作**
   - LLM 只能生成文本，不能实际操作外部系统
   - 无法发送文件、调用 API、执行代码
   - 示例：LLM 可以说"天气很好"，但不能实际查询天气

3. **计算能力弱**
   - LLM 擅长语言理解和生成，但不擅长精确计算
   - 大数运算、复杂数学容易出错
   - 示例：LLM 可能算错 12345 × 67890

**工具如何扩展 Agent 能力**：

```
┌─────────────────────────────────────────────────────────┐
│                    工具扩展能力                          │
│                                                         │
│  LLM 核心能力          工具扩展能力                     │
│  ┌──────────┐         ┌──────────────┐                 │
│  │ 语言理解  │         │  实时信息     │  ← 搜索工具     │
│  │ 推理决策  │    +    │  精确计算     │  ← 计算工具     │
│  │ 文本生成  │         │  文件操作     │  ← 文件工具     │
│  │          │         │  API 调用     │  ← API 工具      │
│  └──────────┘         └──────────────┘                 │
│                                                         │
│  结果：LLM + 工具 = 全能 Agent                           │
└─────────────────────────────────────────────────────────┘
```

### 2.3.2 Function Calling 工作机制

Function Calling（函数调用）是 LLM 与工具交互的标准方式。

**完整流程**：

```
┌──────────────────────────────────────────────────────────────┐
│ Step 1: 用户请求                                              │
│ ════════════════                                            │
│ 用户："北京明天的天气怎么样？"                                │
│                                                              │
│ 消息历史：[{"role": "user", "content": "北京明天的天气怎么样？"}] │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 2: LLM 分析并决定调用工具                                │
│ ═══════════════════════════                                  │
│ LLM 思考（隐式）：                                            │
│   1. 用户想知道天气信息                                       │
│   2. 我没有实时天气数据                                       │
│   3. 我需要调用天气查询工具                                   │
│                                                              │
│ LLM 输出（工具调用格式）：                                    │
│ {                                                            │
│   "role": "assistant",                                       │
│   "tool_calls": [{                                           │
│     "id": "call_123",                                        │
│     "type": "function",                                      │
│     "function": {                                            │
│       "name": "get_weather",                                 │
│       "arguments": {"city": "北京", "date": "tomorrow"}       │
│     }                                                        │
│   }]                                                         │
│ }                                                            │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 3: Agent 执行工具                                        │
│ ═════════════════                                            │
│ Agent 解析工具调用，执行实际函数：                            │
│                                                              │
│ def get_weather(city: str, date: str) -> dict:               │
│     # 调用天气 API                                           │
│     return {                                                 │
│         "city": city,                                        │
│         "date": date,                                        │
│         "temperature": 25,                                   │
│         "condition": "晴"                                    │
│     }                                                        │
│                                                              │
│ 工具返回：                                                   │
│ {"city": "北京", "date": "明天", "temperature": 25, "condition": "晴"} │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 4: 将结果返回给 LLM                                      │
│ ═══════════════════════                                      │
│ 更新消息历史：                                                │
│ [                                                          │
│   {"role": "user", "content": "北京明天的天气怎么样？"},        │
│   {"role": "assistant", "tool_calls": [...]},               │
│   {                                                         │
│     "role": "tool",                                          │
│     "tool_call_id": "call_123",                             │
│     "content": '{"city": "北京", "temperature": 25, ...}'    │
│   }                                                         │
│ ]                                                            │
│                                                              │
│ 再次调用 LLM：                                                │
│ LLM 基于工具结果生成最终响应                                  │
└──────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│ Step 5: LLM 生成最终响应                                      │
│ ════════════════                                            │
│ LLM 输出：                                                   │
│ "北京明天天气晴朗，最高温度 25°C。适合外出活动。"              │
│                                                              │
│ 这是最终返回给用户的回答                                      │
└──────────────────────────────────────────────────────────────┘
```

### 2.3.3 完整工具实现示例

让我们实现 5 个完整的工具，每个工具包含设计思路、代码和使用示例。

#### 工具 1：计算器工具（CalculatorTool）

**设计思路**：
- 支持基本算术运算（加减乘除）
- 支持表达式解析（如 "15 + 7 * 3"）
- 安全性：使用 `ast` 而非 `eval` 避免代码注入

```python
"""
计算器工具实现

支持：
- 基本运算：+、-、*、/
- 括号优先级
- 浮点数计算
"""

import ast
import operator
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


# 支持的运算符
OPERATORS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv,
    ast.Pow: operator.pow,
    ast.USub: operator.neg,
}


class CalculatorTool(BaseTool):
    """计算器工具"""

    @property
    def name(self) -> str:
        return "calculator"

    @property
    def description(self) -> str:
        return "执行数学计算。支持加减乘除、括号、幂运算。例如：'15 + 7 * 3', '(10 - 2) / 4'"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "要计算的数学表达式，例如：'15 + 7 * 3'"
                }
            },
            "required": ["expression"]
        }

    def execute(self, expression: str, **kwargs) -> ToolResult:
        try:
            # 解析表达式
            tree = ast.parse(expression, mode='eval')
            result = self._eval_node(tree.body)

            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output=f"计算结果：{expression} = {result}"
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"计算错误：{str(e)}"
            )

    def _eval_node(self, node):
        """递归计算 AST 节点"""
        if isinstance(node, ast.Num):  # Python 3.7 及以下
            return node.n
        elif isinstance(node, ast.Constant):  # Python 3.8+
            return node.value
        elif isinstance(node, ast.BinOp):
            left = self._eval_node(node.left)
            right = self._eval_node(node.right)
            op_type = type(node.op)
            if op_type not in OPERATORS:
                raise ValueError(f"不支持的运算符：{op_type}")
            return OPERATORS[op_type](left, right)
        elif isinstance(node, ast.UnaryOp):
            operand = self._eval_node(node.operand)
            op_type = type(node.op)
            if op_type not in OPERATORS:
                raise ValueError(f"不支持的一元运算符：{op_type}")
            return OPERATORS[op_type](operand)
        else:
            raise ValueError(f"不支持的表达式类型：{type(node)}")


# 使用示例
if __name__ == "__main__":
    tool = CalculatorTool()

    # 测试用例
    test_cases = [
        "15 * 7",           # 基本乘法
        "100 - 37 + 15",    # 混合运算
        "(10 - 2) / 4",     # 括号优先级
        "2 ** 10",          # 幂运算
        "3.14 * 2 * 2",     # 浮点数
    ]

    for expr in test_cases:
        result = tool.execute(expression=expr)
        print(f"{expr} = {result.output}")
```

**参考输出**：

```
15 * 7 = 计算结果：15 * 7 = 105
100 - 37 + 15 = 计算结果：100 - 37 + 15 = 78
(10 - 2) / 4 = 计算结果：(10 - 2) / 4 = 2.0
2 ** 10 = 计算结果：2 ** 10 = 1024
3.14 * 2 * 2 = 计算结果：3.14 * 2 * 2 = 12.56
```

#### 工具 2：搜索工具（SearchTool）

**设计思路**：
- 模拟搜索引擎行为（实际使用时可接入真实 API）
- 支持关键词搜索
- 返回摘要结果

```python
"""
搜索工具实现

功能：
- 模拟搜索引擎查询
- 返回相关结果摘要
- 可扩展为接入真实搜索 API（Google、Bing 等）
"""

from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class SearchTool(BaseTool):
    """搜索工具"""

    # 模拟搜索结果数据库
    MOCK_RESULTS = {
        "天气": [
            {"title": "中国天气网", "snippet": "提供全国各城市天气预报、气象预警..."},
            {"title": "天气网", "snippet": "实时天气查询，未来 7 天天气预报..."},
        ],
        "新闻": [
            {"title": "新华网", "snippet": "权威新闻媒体，提供最新国内外新闻..."},
            {"title": "人民网", "snippet": "重点新闻网站，报道国内外大事..."},
        ],
        "股票": [
            {"title": "东方财富网", "snippet": "股票行情、财经资讯、投资分析..."},
            {"title": "新浪财经", "snippet": "全球金融市场行情，实时股票数据..."},
        ],
    }

    @property
    def name(self) -> str:
        return "search"

    @property
    def description(self) -> str:
        return "搜索信息。输入查询关键词，返回相关结果摘要。例如：'北京天气'、'AI 新闻'"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索关键词"
                },
                "num_results": {
                    "type": "integer",
                    "description": "返回结果数量",
                    "default": 3
                }
            },
            "required": ["query"]
        }

    def execute(self, query: str, num_results: int = 3, **kwargs) -> ToolResult:
        try:
            # 简单的关键词匹配（实际应用中应调用搜索 API）
            results = []
            for keyword, mock_data in self.MOCK_RESULTS.items():
                if keyword in query:
                    results.extend(mock_data)

            # 如果没有匹配，返回通用结果
            if not results:
                results = [
                    {"title": f"搜索结果：{query}", "snippet": f"关于'{query}'的相关信息..."},
                ]

            # 截取指定数量的结果
            results = results[:num_results]

            # 格式化输出
            output = f"搜索'{query}'的结果：\n\n"
            for i, result in enumerate(results, 1):
                output += f"{i}. {result['title']}\n"
                output += f"   {result['snippet']}\n\n"

            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output=output
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"搜索错误：{str(e)}"
            )


# 使用示例
if __name__ == "__main__":
    tool = SearchTool()

    queries = ["北京天气", "AI 新闻", "股票行情"]
    for query in queries:
        result = tool.execute(query=query, num_results=2)
        print(f"搜索：{query}")
        print(result.output)
```

**参考输出**：

```
搜索：北京天气
搜索'北京天气'的结果：

1. 中国天气网
   提供全国各城市天气预报、气象预警...
2. 天气网
   实时天气查询，未来 7 天天气预报...

搜索：AI 新闻
搜索'AI 新闻'的结果：

1. 搜索结果：AI 新闻
   关于'AI 新闻'的相关信息...
```

#### 工具 3：文件读写工具（FileSystemTool）

**设计思路**：
- 安全的文件操作（限制访问目录）
- 支持读写文本文件
- 防止路径遍历攻击

```python
"""
文件读写工具实现

功能：
- 读取文本文件
- 写入文本文件
- 安全检查：限制访问目录，防止路径遍历
"""

import os
from pathlib import Path
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class FileSystemTool(BaseTool):
    """文件读写工具"""

    def __init__(self, allowed_root: str = "."):
        """
        初始化文件工具

        Args:
            allowed_root: 允许访问的根目录
        """
        self.allowed_root = Path(allowed_root).resolve()

    @property
    def name(self) -> str:
        return "file"

    @property
    def description(self) -> str:
        return "读写文件。支持操作文本文件。例如：读取'./data.txt'，写入'./output.txt'"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "action": {
                    "type": "string",
                    "enum": ["read", "write"],
                    "description": "操作类型：read（读取）或 write（写入）"
                },
                "path": {
                    "type": "string",
                    "description": "文件路径"
                },
                "content": {
                    "type": "string",
                    "description": "写入的内容（仅 write 操作需要）"
                }
            },
            "required": ["action", "path"]
        }

    def _safe_path(self, path: str) -> Path:
        """
        验证并返回安全的路径

        防止路径遍历攻击（如 ../../../etc/passwd）
        """
        # 解析路径
        target = Path(path).resolve()

        # 检查是否在允许的根目录下
        try:
            target.relative_to(self.allowed_root)
        except ValueError:
            raise ValueError(f"不允许访问该路径：{path}")

        return target

    def execute(
        self,
        action: str,
        path: str,
        content: str = None,
        **kwargs
    ) -> ToolResult:
        try:
            # 验证路径
            safe_path = self._safe_path(path)

            if action == "read":
                if not safe_path.exists():
                    return ToolResult(
                        status=ToolResultStatus.ERROR,
                        output=f"文件不存在：{path}"
                    )

                with open(safe_path, 'r', encoding='utf-8') as f:
                    file_content = f.read()

                # 限制返回内容长度
                max_len = 5000
                if len(file_content) > max_len:
                    file_content = file_content[:max_len] + "\n...（内容被截断）"

                return ToolResult(
                    status=ToolResultStatus.SUCCESS,
                    output=f"文件内容（{len(file_content)} 字符）:\n\n{file_content}"
                )

            elif action == "write":
                if content is None:
                    return ToolResult(
                        status=ToolResultStatus.ERROR,
                        output="写入操作需要提供 content 参数"
                    )

                # 创建父目录（如果不存在）
                safe_path.parent.mkdir(parents=True, exist_ok=True)

                with open(safe_path, 'w', encoding='utf-8') as f:
                    f.write(content)

                return ToolResult(
                    status=ToolResultStatus.SUCCESS,
                    output=f"成功写入文件：{path}\n写入 {len(content)} 字符"
                )

            else:
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    output=f"不支持的操作类型：{action}"
                )

        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"文件操作错误：{str(e)}"
            )


# 使用示例
if __name__ == "__main__":
    tool = FileSystemTool(allowed_root="./data")

    # 写入文件
    result = tool.execute(
        action="write",
        path="./data/test.txt",
        content="Hello, World!\n这是测试内容。"
    )
    print(result.output)

    # 读取文件
    result = tool.execute(
        action="read",
        path="./data/test.txt"
    )
    print(result.output)
```

**参考输出**：

```
成功写入文件：./data/test.txt
写入 31 字符

文件内容（31 字符）:

Hello, World!
这是测试内容。
```

#### 工具 4：日期时间工具（DateTimeTool）

**设计思路**：
- 获取当前日期时间
- 日期格式化
- 时区支持

```python
"""
日期时间工具实现

功能：
- 获取当前日期时间
- 格式化日期
- 计算日期差
"""

from datetime import datetime
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class DateTimeTool(BaseTool):
    """日期时间工具"""

    @property
    def name(self) -> str:
        return "datetime"

    @property
    def description(self) -> str:
        return "获取当前日期时间或进行日期计算。例如：'获取当前日期'、'格式化日期为 YYYY-MM-DD'"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "action": {
                    "type": "string",
                    "enum": ["now", "format", "diff"],
                    "description": "操作类型：now（当前时间）、format（格式化）、diff（日期差）"
                },
                "date": {
                    "type": "string",
                    "description": "日期字符串（format 和 diff 操作需要）"
                },
                "format": {
                    "type": "string",
                    "description": "日期格式，例如：'%Y-%m-%d %H:%M:%S'"
                }
            },
            "required": ["action"]
        }

    def execute(
        self,
        action: str,
        date: str = None,
        format: str = None,
        **kwargs
    ) -> ToolResult:
        try:
            if action == "now":
                now = datetime.now()
                return ToolResult(
                    status=ToolResultStatus.SUCCESS,
                    output=f"当前日期时间：{now.strftime('%Y-%m-%d %H:%M:%S')}\n"
                           f"星期：{now.strftime('%A')}"
                )

            elif action == "format":
                if date is None:
                    return ToolResult(
                        status=ToolResultStatus.ERROR,
                        output="format 操作需要提供 date 参数"
                    )

                fmt = format or "%Y-%m-%d %H:%M:%S"
                dt = datetime.fromisoformat(date)
                return ToolResult(
                    status=ToolResultStatus.SUCCESS,
                    output=f"格式化结果：{dt.strftime(fmt)}"
                )

            elif action == "diff":
                if date is None:
                    return ToolResult(
                        status=ToolResultStatus.ERROR,
                        output="diff 操作需要提供 date 参数"
                    )

                target = datetime.fromisoformat(date)
                now = datetime.now()
                diff = target - now

                return ToolResult(
                    status=ToolResultStatus.SUCCESS,
                    output=f"距离{date}还有：{diff.days}天 {diff.seconds // 3600}小时"
                )

            else:
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    output=f"不支持的操作类型：{action}"
                )

        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"日期时间错误：{str(e)}"
            )


# 使用示例
if __name__ == "__main__":
    tool = DateTimeTool()

    # 获取当前时间
    result = tool.execute(action="now")
    print(result.output)

    # 格式化日期
    result = tool.execute(action="format", date="2024-12-31")
    print(result.output)

    # 计算日期差
    result = tool.execute(action="diff", date="2024-12-31")
    print(result.output)
```

**参考输出**：

```
当前日期时间：2024-03-15 14:30:45
星期：Friday

格式化结果：2024-12-31 00:00:00

距离 2024-12-31 还有：291 天 9 小时
```

#### 工具 5：URL 抓取工具（URLFetcherTool）

**设计思路**：
- 获取网页内容
- 提取正文（去除广告、导航等）
- 超时和错误处理

```python
"""
URL 抓取工具实现

功能：
- 获取网页 HTML 内容
- 提取正文文本
- 超时和错误处理
"""

import requests
from bs4 import BeautifulSoup
from src.tools.base import BaseTool, ToolResult, ToolResultStatus


class URLFetcherTool(BaseTool):
    """URL 抓取工具"""

    def __init__(self, timeout: int = 10):
        self.timeout = timeout

    @property
    def name(self) -> str:
        return "fetch_url"

    @property
    def description(self) -> str:
        return "获取网页内容。输入 URL，返回网页正文。例如：'https://example.com'"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "url": {
                    "type": "string",
                    "description": "要抓取的 URL"
                },
                "max_length": {
                    "type": "integer",
                    "description": "返回内容的最大长度",
                    "default": 5000
                }
            },
            "required": ["url"]
        }

    def execute(self, url: str, max_length: int = 5000, **kwargs) -> ToolResult:
        try:
            # 验证 URL
            if not url.startswith(("http://", "https://")):
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    output="URL 必须以 http:// 或 https:// 开头"
                )

            # 发送请求
            headers = {
                "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
            }
            response = requests.get(url, headers=headers, timeout=self.timeout)
            response.raise_for_status()

            # 解析 HTML
            soup = BeautifulSoup(response.text, 'html.parser')

            # 移除不需要的元素
            for element in soup(["script", "style", "nav", "footer", "header"]):
                element.decompose()

            # 提取正文
            text = soup.get_text(separator='\n', strip=True)

            # 截断过长内容
            if len(text) > max_length:
                text = text[:max_length] + "\n...（内容被截断）"

            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output=f"网页内容（{len(text)} 字符）:\n\n{text}"
            )

        except requests.Timeout:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"请求超时（{self.timeout}秒）"
            )
        except requests.RequestException as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"请求错误：{str(e)}"
            )
        except Exception as e:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"解析错误：{str(e)}"
            )


# 使用示例
if __name__ == "__main__":
    tool = URLFetcherTool()

    result = tool.execute(url="https://www.example.com")
    print(result.output)
```

**参考输出**：

```
网页内容（1256 字符）:

Example Domain

This domain is for use in illustrative examples in documents.
You may use this domain in literature without prior permission or a fee.

Domain usage examples:
- Example documents
- Testing web applications
- Educational materials
```

### 2.3.4 工具注册表设计

**工具注册表**（ToolRegistry）负责管理所有可用工具，并提供统一的调用接口。

```python
"""
工具注册表实现

功能：
- 工具注册与发现
- 工具描述标准化（JSON Schema）
- OpenAI Function Calling 格式转换
"""

from typing import Dict, List, Any, Optional
from src.tools.base import BaseTool, ToolResult


class ToolRegistry:
    """工具注册表"""

    def __init__(self):
        self._tools: Dict[str, BaseTool] = {}

    def register(self, tool: BaseTool) -> None:
        """注册一个工具"""
        self._tools[tool.name] = tool

    def unregister(self, name: str) -> None:
        """注销一个工具"""
        if name in self._tools:
            del self._tools[name]

    def get_tool(self, name: str) -> Optional[BaseTool]:
        """获取工具实例"""
        return self._tools.get(name)

    def execute(self, name: str, **kwargs) -> ToolResult:
        """执行工具"""
        tool = self.get_tool(name)
        if tool is None:
            from src.tools.base import ToolResult, ToolResultStatus
            return ToolResult(
                status=ToolResultStatus.ERROR,
                output=f"未知工具：{name}"
            )
        return tool.execute(**kwargs)

    def list_tools(self) -> List[Dict[str, Any]]:
        """列出所有工具的描述"""
        return [
            {
                "name": tool.name,
                "description": tool.description,
                "parameters": tool.parameters,
            }
            for tool in self._tools.values()
        ]

    def to_openai_tools(self) -> List[Dict[str, Any]]:
        """
        转换为 OpenAI Function Calling 格式

        返回格式：
        [
            {
                "type": "function",
                "function": {
                    "name": "calculator",
                    "description": "...",
                    "parameters": {...}
                }
            },
            ...
        ]
        """
        return [
            {
                "type": "function",
                "function": {
                    "name": tool.name,
                    "description": tool.description,
                    "parameters": tool.parameters,
                }
            }
            for tool in self._tools.values()
        ]

    def to_openai_tool_choice(self, auto: bool = True) -> Dict[str, Any]:
        """
        生成 tool_choice 参数

        Args:
            auto: True（自动选择）或 False（强制不使用工具）
        """
        if auto and self._tools:
            return "auto"
        return "none"


# 使用示例
if __name__ == "__main__":
    from src.tools.calculator import CalculatorTool
    from src.tools.search import SearchTool

    registry = ToolRegistry()
    registry.register(CalculatorTool())
    registry.register(SearchTool())

    # 执行工具
    result = registry.execute("calculator", expression="15 * 7")
    print(result.output)

    # 获取 OpenAI 格式工具定义
    tools = registry.to_openai_tools()
    print(f"注册的工具数量：{len(tools)}")
```

---

## 2.4 Memory（记忆）- 记忆系统深度解析

### 2.4.1 人类记忆系统的启发

人类记忆系统给 AI Agent 的记忆设计提供了重要启发：

```
┌─────────────────────────────────────────────────────────┐
│              人类记忆系统 vs AI 记忆系统                  │
│                                                         │
│  人类记忆                      AI 记忆                   │
│  ──────────                   ────────                  │
│                                                         │
│  感觉记忆                    输入缓存                    │
│  (Sensory Memory)           (Input Buffer)              │
│  ↓ 注意                       ↓ 筛选                     │
│  短期记忆                      短期记忆                   │
│  (Short-term Memory)        (Conversation History)      │
│  ↓ 编码                       ↓ 压缩/检索                 │
│  长期记忆                      长期记忆                   │
│  (Long-term Memory)         (Vector Database)           │
│                                                         │
│  工作记忆：短期处理 + 长期提取                            │
│  (Working Memory)           (Agent Context)             │
└─────────────────────────────────────────────────────────┘
```

**关键概念映射**：

| 人类记忆 | AI 记忆 | 实现方式 |
|---------|--------|---------|
| 感觉记忆 | 输入缓存 | 原始输入暂存 |
| 短期记忆 | 对话历史 | 滑动窗口消息列表 |
| 工作记忆 | Agent 上下文 | 当前处理的上下文 |
| 长期记忆 | 向量数据库 | ChromaDB/Faiss |

### 2.4.2 短期记忆实现

**短期记忆**负责存储最近的对话历史，是 LLM 进行推理的基础。

**核心设计**：

```python
"""
短期记忆实现

功能：
- 滑动窗口机制
- Token 限制管理
- 系统消息保留
"""

from typing import List, Dict, Any
from dataclasses import dataclass, field


@dataclass
class Message:
    """消息"""
    role: str
    content: str


class ShortTermMemory:
    """
    短期记忆

    使用滑动窗口管理对话历史
    """

    def __init__(
        self,
        max_messages: int = 50,
        max_tokens: int = 8000,
        system_prompt: str = None,
    ):
        """
        Args:
            max_messages: 最大消息数量
            max_tokens: 最大 token 数（估算值，按 1 字符≈0.5 token）
            system_prompt: 系统提示词（始终保留）
        """
        self.max_messages = max_messages
        self.max_tokens = max_tokens
        self.system_prompt = system_prompt
        self._messages: List[Message] = []

    def add(self, role: str, content: str) -> None:
        """添加消息"""
        self._messages.append(Message(role=role, content=content))
        self._trim_if_needed()

    def _trim_if_needed(self) -> None:
        """修剪消息以保持限制"""
        # 按消息数量修剪
        while len(self._messages) > self.max_messages:
            # 保留最新的消息
            self._messages.pop(0)

        # 按 token 数修剪（简化估算）
        total_chars = sum(len(m.content) for m in self._messages)
        estimated_tokens = total_chars // 2

        while estimated_tokens > self.max_tokens and len(self._messages) > 2:
            self._messages.pop(0)
            total_chars = sum(len(m.content) for m in self._messages)
            estimated_tokens = total_chars // 2

    def get_messages(self) -> List[Dict[str, str]]:
        """获取 OpenAI 格式的消息列表"""
        messages = []

        # 首先添加系统消息
        if self.system_prompt:
            messages.append({"role": "system", "content": self.system_prompt})

        # 添加对话历史
        for msg in self._messages:
            messages.append({"role": msg.role, "content": msg.content})

        return messages

    def get_recent(self, n: int) -> List[Dict[str, str]]:
        """获取最近 N 条消息"""
        recent = self._messages[-n:]
        return [{"role": m.role, "content": m.content} for m in recent]

    def clear(self) -> None:
        """清空记忆（保留系统提示）"""
        self._messages = []

    def get_statistics(self) -> Dict[str, Any]:
        """获取记忆统计"""
        total_chars = sum(len(m.content) for m in self._messages)
        return {
            "message_count": len(self._messages),
            "total_characters": total_chars,
            "estimated_tokens": total_chars // 2,
            "max_messages": self.max_messages,
            "max_tokens": self.max_tokens,
        }


# 使用示例
if __name__ == "__main__":
    memory = ShortTermMemory(
        max_messages=10,
        max_tokens=4000,
        system_prompt="你是一个有帮助的助手。"
    )

    # 添加消息
    memory.add("user", "你好")
    memory.add("assistant", "你好！有什么可以帮你的？")
    memory.add("user", "今天天气怎么样？")
    memory.add("assistant", "我很抱歉，我无法获取实时天气信息。你可以查看天气应用或网站。")

    # 获取消息
    messages = memory.get_messages()
    print(f"消息数量：{len(messages)}")

    # 获取统计
    stats = memory.get_statistics()
    print(f"统计：{stats}")
```

**参考输出**：

```
消息数量：5  # system + 4 条对话
统计：{
    'message_count': 4,
    'total_characters': 120,
    'estimated_tokens': 60,
    'max_messages': 10,
    'max_tokens': 4000
}
```

---

### 2.4.3 长期记忆实现

**长期记忆**用于存储大量历史信息，通过向量检索实现"回忆"功能。

#### 向量数据库基础

**核心概念**：

1. **Embedding（嵌入）**
   - 将文本映射为固定长度的向量
   - 语义相似的文本向量距离更近
   - 常用模型：text-embedding-ada-002、bge-large-zh

2. **相似度搜索**
   - 计算查询向量与数据库中向量的相似度
   - 常用度量：余弦相似度、欧氏距离
   - 返回最相似的 Top-K 结果

3. **ANN（Approximate Nearest Neighbors）算法**

| 算法 | 全称 | 特点 | 适用场景 |
|------|------|------|---------|
| **LSH** | Locality Sensitive Hashing | 哈希-based，简单快速 | 低精度要求 |
| **ANNOY** | Approximate Nearest Neighbors Oh Yeah | 树-based，内存友好 | 中小规模 |
| **HNSW** | Hierarchical Navigable Small World | 图-based，精度高 | 大规模、高精度 |
| **FAISS** | Facebook AI Similarity Search | 多种算法，高度优化 | 超大规模 |

#### 长期记忆架构

```
┌─────────────────────────────────────────────────────────┐
│                  长期记忆系统                             │
│  新记忆 ──► Embedding ──► 向量 ──► 存入向量数据库       │
│  回忆请求 ──► Embedding ──► 向量 ──► 相似度搜索         │
│                                          │              │
│                                          ▼              │
│                                    Top-K 相似记忆        │
└─────────────────────────────────────────────────────────┘
```

### 2.4.4 记忆压缩技术

**记忆压缩策略**：

| 策略 | 说明 | 优点 | 缺点 |
|------|------|------|------|
| **滑动窗口** | 只保留最近 N 条消息 | 简单直接 | 可能丢失重要信息 |
| **摘要压缩** | 使用 LLM 将多轮对话压缩为摘要 | 保留核心信息 | 可能丢失细节 |
| **重要性评分** | 为每条记忆评分，优先保留高重要性 | 智能筛选 | 需要额外模型 |
| **混合策略** | 结合以上多种策略 | 平衡效果和复杂度 | 实现复杂 |

---

## 2.5 Planning（规划）- 规划器深度解析

### 2.5.1 任务分解（Task Decomposition）

#### Chain of Thought（思维链）

**核心思想**：让模型"一步一步思考"，显式展示推理过程。

**Few-shot CoT 示例**：

```
Q: 罗杰有 5 个网球。他又买了两筒网球。每筒有 3 个网球。他现在有多少个网球？

A: 让我们一步一步思考：
1. 罗杰开始有 5 个网球
2. 他买了 2 筒网球
3. 每筒有 3 个网球，所以 2×3 = 6 个网球
4. 5 + 6 = 11
5. 罗杰现在有 11 个网球
```

#### Tree of Thoughts（思维树）

**核心思想**：探索多条推理路径，选择最优解。

```
                    初始问题
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    思路 A          思路 B          思路 C
        │              │              │
    ┌───┴───┐      ┌───┴───┐      ┌───┴───┐
    ▼       ▼      ▼       ▼      ▼       ▼
  A1      A2    B1      B2    C1      C2
    │       │      │       │      │       │
    ▼       ▼      ▼       ▼      ▼       ▼
  评估     评估    评估     评估    评估     评估
                   │
                   ▼
              选择最优路径
```

### 2.5.2 自我反思（Self-Reflection）

**Reflexion 框架**：执行任务 → 评估结果 → 反思不足 → 改进策略

---

# 第三部分：推理方法对比

## 3.1 Zero-shot Reasoning（零样本推理）

直接提问，不提供示例。适用于简单事实性问题。

## 3.2 Chain of Thought (CoT)（思维链）

引导模型展示推理步骤，"Let's think step by step"。

**适用场景**：数学推理、逻辑题

## 3.3 Tree of Thoughts (ToT)（思维树）

探索多条推理路径，选择最优解。

**适用场景**：需要创造性解决的问题

## 3.4 ReAct（推理 + 行动）

将推理和行动统一在一个循环中。

### Thought-Action-Observation 循环

```
Thought ──► Action ──► Observation ─────┐
    ▲                                   │
    └───────────────────────────────────┘
```

### 完整 ReAct 示例

**用户问题**："《三体》的作者是谁？他获得过哪些奖项？"

```
Iteration 1:
Thought: 用户想知道《三体》的作者及其奖项。
Action: search(query="三体 作者")
Observation: 《三体》是刘慈欣创作的科幻小说。

Iteration 2:
Thought: 作者是刘慈欣，现在查询他的奖项。
Action: search(query="刘慈欣 奖项")
Observation: 2015 年雨果奖、2016 年星云奖...

Iteration 3:
Response: 《三体》的作者是刘慈欣。他获得的主要奖项包括...
```

## 3.5 方法选择指南

| 任务类型 | 推荐方法 | 原因 |
|---------|---------|------|
| **简单问答** | Zero-shot | 快速直接 |
| **数学计算** | CoT | 分步计算减少错误 |
| **复杂规划** | ToT | 多路径探索 |
| **需要工具** | ReAct | 整合推理与行动 |

---

# 第四部分：Agent 工作循环详解

## 4.1 完整执行流程图

```
用户输入
   │
   ▼
┌─────────────────┐
│ 1. 感知层        │ 接收用户输入，添加到记忆
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 2. LLM 推理      │ 分析意图，决定是否调用工具
└────────┬────────┘
         │
    ┌────┴────┐
    │ 有工具调用？│
    └────┬────┘
     Yes │ No
         │     │
         ▼     ▼
    ┌─────────┐ ┌─────────────┐
    │ 3. 执行工具 │ │ 4. 生成回答 │
    └────┬────┘ └──────┬──────┘
         │             │
         ▼             │
┌─────────────────┐    │
│ 5. 处理结果      │    │
│ 添加到记忆       │    │
└────────┬────────┘    │
         │             │
         └──────┬──────┘
                │
                ▼
         继续推理/完成
```

## 4.2 状态管理

**AgentState 枚举**：

```python
class AgentState(Enum):
    IDLE = auto()       # 空闲
    THINKING = auto()   # 思考中
    EXECUTING = auto()  # 执行中
    WAITING = auto()    # 等待中
    COMPLETE = auto()   # 完成
    ERROR = auto()      # 错误
```

### 状态转换图

```
         ┌─────────┐
         │  IDLE   │
         └────┬────┘
              │ 接收输入
              ▼
         ┌─────────┐
    ┌────│ THINKING│◄─────┐
    │    └────┬────┘      │
    │         │ 工具调用   │
    │         ▼           │
    │    ┌─────────┐      │
    │    │EXECUTING│      │
    │    └────┬────┘      │
    │         │ 完成      │
    │         ▼           │
    │    ┌─────────┐      │
    └────│ WAITING │──────┘
         └────┬────┘
              │ 最终回答
         ┌────┴────┐
         ▼         ▼
    ┌─────────┐ ┌─────────┐
    │COMPLETE │ │  ERROR  │
    └─────────┘ └─────────┘
```

## 4.3 错误处理

### 最大迭代次数限制

```python
class AgentConfig:
    def __init__(
        self,
        max_iterations: int = 10,  # 最大迭代次数
        max_tool_calls: int = 50,
        timeout_seconds: int = 300,
    ):
        self.max_iterations = max_iterations
        self.max_tool_calls = max_tool_calls
        self.timeout_seconds = timeout_seconds
```

## 4.4 可观察性设计

### 执行追踪（Trace）

```python
@dataclass
class TraceEntry:
    timestamp: str
    iteration: int
    event_type: str  # "thought", "action", "observation", "response"
    data: Any


class AgentTracer:
    def __init__(self, enabled: bool = True):
        self.enabled = enabled
        self.entries: List[TraceEntry] = []

    def log_thought(self, thought: str):
        self.entries.append(TraceEntry(..., "thought", {"thought": thought}))

    def log_action(self, action: str, arguments: dict):
        self.entries.append(TraceEntry(..., "action", {"action": action}))

    def save(self, filename: str):
        with open(filename, 'w') as f:
            json.dump([e.__dict__ for e in self.entries], f, indent=2)
```

---

# 第五部分：从零搭建完整 Agent

## 5.1 项目结构设计

```
my_agent/
├── agent/
│   ├── base.py        # 抽象基类
│   └── react.py       # ReAct 实现
├── llm/
│   └── client.py      # LLM 客户端
├── tools/
│   ├── base.py        # 工具基类
│   ├── calculator.py
│   └── search.py
├── memory/
│   └── short_term.py
└── main.py
```

## 5.2 核心类实现

### BaseAgent 抽象类

```python
"""agent/base.py"""

from abc import ABC, abstractmethod
from enum import Enum


class AgentState(Enum):
    IDLE = "idle"
    THINKING = "thinking"
    EXECUTING = "executing"
    WAITING = "waiting"
    COMPLETE = "complete"
    ERROR = "error"


class AgentConfig:
    def __init__(self, name: str = "Agent", max_iterations: int = 10):
        self.name = name
        self.max_iterations = max_iterations


class BaseAgent(ABC):
    def __init__(self, config: AgentConfig):
        self.config = config
        self.state = AgentState.IDLE

    @abstractmethod
    def run(self, user_input: str) -> str:
        pass

    def reset(self):
        self.state = AgentState.IDLE
```

### ReActAgent 实现

```python
"""agent/react.py"""

from .base import BaseAgent, AgentConfig, AgentState


class ReActAgent(BaseAgent):
    def __init__(self, config, llm_client, tools, memory):
        super().__init__(config)
        self.llm = llm_client
        self.tools = tools
        self.memory = memory

    def run(self, user_input: str) -> str:
        self.memory.add("user", user_input)

        for _ in range(self.config.max_iterations):
            result = self.step()
            if result:
                self.memory.add("assistant", result)
                return result
        return "已达到最大迭代次数。"

    def step(self):
        self.state = AgentState.THINKING
        messages = self.memory.get_messages()

        response = self.llm.chat(
            messages=messages,
            tools=self.tools.to_openai_tools(),
        )

        if response.tool_calls:
            self.state = AgentState.EXECUTING
            for tool_call in response.tool_calls:
                result = self.tools.execute(
                    tool_call.function.name,
                    **tool_call.function.arguments
                )
                self.memory.add("tool", f"工具返回：{result.output}")
            self.state = AgentState.WAITING
            return None
        else:
            self.state = AgentState.COMPLETE
            return response.content
```

### ToolRegistry

```python
"""tools/base.py"""

from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum


class ToolResultStatus(Enum):
    SUCCESS = "success"
    ERROR = "error"


@dataclass
class ToolResult:
    status: ToolResultStatus
    output: str


class BaseTool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: pass

    @property
    @abstractmethod
    def description(self) -> str: pass

    @property
    @abstractmethod
    def parameters(self) -> dict: pass

    @abstractmethod
    def execute(self, **kwargs) -> ToolResult: pass


class ToolRegistry:
    def __init__(self):
        self._tools: Dict[str, BaseTool] = {}

    def register(self, tool: BaseTool):
        self._tools[tool.name] = tool

    def execute(self, name: str, **kwargs) -> ToolResult:
        tool = self._tools.get(name)
        if tool is None:
            return ToolResult(ToolResultStatus.ERROR, f"未知工具：{name}")
        return tool.execute(**kwargs)

    def to_openai_tools(self) -> List[Dict]:
        return [
            {"type": "function", "function": {
                "name": t.name,
                "description": t.description,
                "parameters": t.parameters,
            }}
            for t in self._tools.values()
        ]
```

### ShortTermMemory

```python
"""memory/short_term.py"""

from dataclasses import dataclass
from typing import List, Dict


@dataclass
class Message:
    role: str
    content: str


class ShortTermMemory:
    def __init__(self, max_messages: int = 50, system_prompt: str = None):
        self.max_messages = max_messages
        self.system_prompt = system_prompt
        self._messages: List[Message] = []

    def add(self, role: str, content: str):
        self._messages.append(Message(role, content))
        while len(self._messages) > self.max_messages:
            self._messages.pop(0)

    def get_messages(self) -> List[Dict[str, str]]:
        messages = []
        if self.system_prompt:
            messages.append({"role": "system", "content": self.system_prompt})
        for msg in self._messages:
            messages.append({"role": msg.role, "content": msg.content})
        return messages
```

## 5.3 整合与测试

### main.py

```python
from agent.react import ReActAgent, AgentConfig
from llm.client import LLMClient, LLMConfig, LLMProvider
from tools.base import ToolRegistry
from tools.calculator import CalculatorTool
from memory.short_term import ShortTermMemory


def create_agent():
    llm_config = LLMConfig(
        provider=LLMProvider.BAILIAN,
        api_key="your_api_key",
        model="qwen-plus"
    )
    agent_config = AgentConfig(name="MyAgent", max_iterations=10)

    llm_client = LLMClient(llm_config)
    tools = ToolRegistry()
    tools.register(CalculatorTool())
    memory = ShortTermMemory(
        max_messages=50,
        system_prompt="你是有帮助的 AI 助手。"
    )

    return ReActAgent(
        config=agent_config,
        llm_client=llm_client,
        tools=tools,
        memory=memory
    )


def main():
    agent = create_agent()
    print("Agent 已就绪！输入'quit'退出。")

    while True:
        user_input = input("\n你：").strip()
        if user_input.lower() == 'quit':
            break
        if not user_input:
            continue

        response = agent.run(user_input)
        print(f"Agent: {response}")


if __name__ == "__main__":
    main()
```

---

# 第六部分：实践练习

## 6.1 基础练习

### 练习 1：运行第一个 Agent

```bash
cd agent-tutorial
python examples/01_simple_agent.py
```

### 练习 2：添加自定义工具

创建天气查询工具：

```python
class WeatherTool(BaseTool):
    @property
    def name(self): return "weather"

    @property
    def description(self): return "查询城市天气"

    @property
    def parameters(self):
        return {"type": "object", "properties": {
            "city": {"type": "string", "description": "城市名称"}
        }, "required": ["city"]}

    def execute(self, city: str, **kwargs):
        return ToolResult(ToolResultStatus.SUCCESS, f"{city}: 晴，25°C")
```

### 练习 3：修改系统提示词

尝试不同角色：严肃助手、活泼伙伴、简洁工程师

## 6.2 进阶练习

### 练习 4：实现记忆压缩

### 练习 5：多工具协作

## 6.3 挑战练习

### 构建日报 Agent

```python
class DailyReportAgent:
    def __init__(self, llm_client):
        self.llm = llm_client
        self.work_records = []

    def record_work(self, content: str):
        self.work_records.append(content)

    def generate_report(self) -> str:
        prompt = f"根据以下记录生成日报：\n"
        prompt += "\n".join(f"- {r}" for r in self.work_records)
        return self.llm.chat(
            messages=[{"role": "user", "content": prompt}]
        ).content
```

---

# 第七部分：常见问题与最佳实践

## 7.1 常见问题

### Q1: Agent 不调用工具怎么办？

**解决方案**：
1. 优化工具描述
2. 在 System Prompt 中明确说明
3. 降低温度参数（如 0.3）

### Q2: Agent 陷入无限循环如何处理？

**解决方案**：
1. 设置最大迭代次数限制
2. 检测重复模式并打破循环

### Q3: 如何提高响应速度？

1. 减少迭代次数
2. 使用更快的模型
3. 并行工具调用

### Q4: 如何降低 API 成本？

1. 减少 token 使用
2. 缓存常见响应
3. 批量处理

## 7.2 最佳实践

### System Prompt 设计原则

1. 角色清晰
2. 能力说明
3. 行为准则
4. 输出格式
5. 限制说明

### 工具描述编写技巧

**公式**：[功能] + [使用时机] + [参数说明] + [示例]

---

# 总结

## 核心知识点

1. **AI Agent 定义与特征**：自主性、反应性、主动性、社会性
2. **核心架构**：LLM、Tools、Memory、Planning
3. **推理方法**：Zero-shot、CoT、ToT、ReAct
4. **实践技能**：从零搭建完整 Agent

## 下一步学习

- [第二章：ReAct 模式详解](/docs/02_react.md)
- [第三章：工具系统高级用法](/docs/03_tools.md)
- [第四章：记忆系统](/docs/04_memory.md)

## 参考资料

### 核心论文
1. [ReAct](https://arxiv.org/abs/2210.03629) (ICLR 2023)
2. [Chain of Thought](https://arxiv.org/abs/2103.03874) (NeurIPS 2022)
3. [Tree of Thoughts](https://arxiv.org/abs/2305.10601)

---

*完成本章后，你应该能够：*
- [ ] 理解 AI Agent 的基本概念和架构
- [ ] 区分不同的推理方法
- [ ] 创建自定义工具
- [ ] 搭建简单 Agent
- [ ] 调试和优化 Agent 行为