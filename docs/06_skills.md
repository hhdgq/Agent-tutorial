# 第六章 Skills 系统详解

> **本章导读**：Skills（技能）是 Agent 的专业化能力模块。如果说 Tools 是 Agent 的"手和脚"，那么 Skills 就是 Agent 的"专业知识"和"职业素养"。本章将深入讲解 Skills 系统的设计原理、架构实现，并通过多个实战案例带你掌握创建自定义 Skills 的完整方法。

**本章目标**：
- 理解 Skills 系统的核心概念和设计原理
- 掌握 Skill 和 SkillManager 的源码实现
- 能够创建自定义 Skills 解决实际问题
- 了解业界主流的 Skills/Plugins 实现方案
- 掌握前沿的 Agent 能力提升技巧

**预计阅读时间**：45-60 分钟

---

## 目录

1. [Skills 基础概念](#1-skills-基础概念)
2. [Skill 核心架构深度解析](#2-skill-核心架构深度解析)
3. [SkillManager 技能管理器](#3-skillmanager-技能管理器)
4. [创建自定义 Skills 实战](#4-创建自定义-skills-实战)
5. [Skill 与 Agent 集成](#5-skill-与-agent-集成)
6. [高级话题](#6-高级话题)
7. [前沿技能设计](#7-前沿技能设计)
8. [实践练习与挑战](#8-实践练习与挑战)
9. [常见问题与最佳实践](#9-常见问题与最佳实践)

---

# 1. Skills 基础概念

## 1.1 什么是 Skills？

### 从人类技能学习角度类比

想象一下人类学习技能的过程：

```
┌─────────────────────────────────────────────────────────────┐
│                    人类技能学习类比                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  学习驾驶                                                    │
│  ├── 理论知识：交通规则、车辆原理（System Prompt）           │
│  ├── 操作工具：方向盘、油门、刹车（Tools）                  │
│  ├── 实践经验：倒车入库、侧方停车（State）                  │
│  └── 激活场景：需要开车时（Trigger）                        │
│                                                             │
│  学习编程                                                    │
│  ├── 理论知识：语法、设计模式（System Prompt）              │
│  ├── 操作工具：IDE、调试器、版本控制（Tools）               │
│  ├── 实践经验：代码风格、最佳实践（State）                  │
│  └── 激活场景：需要写代码时（Trigger）                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Skills 在 AI Agent 中的作用与此类似：
- **系统提示词** = 专业知识和行为规范
- **工具集** = 完成工作所需的能力
- **状态管理** = 上下文和配置信息
- **触发器** = 自动激活的条件

### Skills vs Tools vs Capabilities 概念辨析

| 概念 | 定义 | 特点 | 示例 |
|------|------|------|------|
| **Tool（工具）** | 单一功能的函数封装 | 无状态、直接执行、粒度细 | 计算器、文件读取 |
| **Skill（技能）** | 专业化能力模块 | 有状态、可配置、包含多个工具 | 代码审查、翻译助手 |
| **Capability（能力）** | 更高层次的综合能力 | 可能包含多个 Skills、更抽象 | 多模态理解、创造性思维 |

```
┌─────────────────────────────────────────────────────────────┐
│                    能力层次结构                              │
│                                                             │
│                      ┌─────────┐                            │
│                      │Capability│                           │
│                      │ (能力)  │                            │
│                      └────┬────┘                            │
│                           │                                 │
│           ┌───────────────┼───────────────┐                 │
│           │               │               │                 │
│      ┌────┴────┐    ┌────┴────┐    ┌────┴────┐             │
│      │ Skill 1  │    │ Skill 2  │    │ Skill 3  │            │
│      │ (技能)  │    │ (技能)  │    │ (技能)  │            │
│      └────┬────┘    └────┬────┘    └────┬────┘             │
│           │               │               │                 │
│      ┌────┴────┐    ┌────┴────┐    ┌────┴────┐             │
│      │ Tool   │    │ Tool   │    │ Tool   │                 │
│      │ (工具) │    │ (工具) │    │ (工具) │                 │
│      └─────────┘    └─────────┘    └─────────┘             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Skills 的核心特征

1. **有状态（Stateful）**
   - Skill 可以维护内部状态和配置
   - 可以在多次调用之间保持上下文

2. **可配置（Configurable）**
   - 可以通过参数定制行为
   - 可以动态加载/卸载

3. **组合性（Composable）**
   - 多个 Skills 可以协同工作
   - 可以形成技能链（Skill Chain）

4. **领域专精（Domain-specific）**
   - 每个 Skill 专注一个特定领域
   - 提供该领域的专业知识和工具

---

## 1.2 为什么需要 Skills 系统？

### 单一 Agent 的局限性

想象一个没有专业分工的世界：每个人都必须自己种粮食、做衣服、盖房子...效率极低。AI Agent 同理：

```
┌─────────────────────────────────────────────────────────────┐
│                    单一 Agent 的局限性                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  问题 1：上下文限制                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  通用 Agent 的上下文就像一个"万金油"，什么都懂一点，  │   │
│  │  但什么都不精。当遇到专业问题时，表现往往不理想。    │   │
│  │                                                      │   │
│  │  例如：让通用 Agent 做代码审查，它可能会：            │   │
│  │  - 忽略特定语言的最佳实践                            │   │
│  │  - 无法识别领域特定的问题模式                        │   │
│  │  - 给出泛泛而谈的建议                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  问题 2：专业领域知识不足                                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  通用训练数据无法覆盖所有专业领域：                  │   │
│  │  - 医疗诊断需要医学知识                              │   │
│  │  - 法律分析需要法律知识                              │   │
│  │  - 代码审查需要编程经验                              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  问题 3：任务复杂性                                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  复杂任务需要多步骤协作：                            │   │
│  │  "分析这份销售数据并生成报告"                        │   │
│  │  ├── 数据加载和清洗                                  │   │
│  │  ├── 统计分析和趋势识别                              │   │
│  │  ├── 可视化图表生成                                  │   │
│  │  └── 报告撰写和格式化                                │   │
│  │                                                      │   │
│  │  单一 Agent 难以有效协调这么多步骤。                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 专业化分工的必要性

类似人类社会的职业分工，Skills 系统让 Agent 能够：

```
┌─────────────────────────────────────────────────────────────┐
│                    Skills 分工体系                           │
│                                                             │
│                                                             │
│   用户请求："审查这段 Python 代码的性能问题"                  │
│                      │                                      │
│                      ▼                                      │
│              ┌───────────────┐                              │
│              │  Skill Router │                              │
│              └───────┬───────┘                              │
│                      │                                      │
│         ┌────────────┼────────────┐                         │
│         │            │            │                         │
│         ▼            ▼            ▼                         │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│   │代码审查  │ │性能分析  │ │Python 专家 │                   │
│   │Skill     │ │Skill     │ │Skill     │                   │
│   │          │ │          │ │          │                   │
│   │- 风格检查│ │- 复杂度  │ │- 语法   │                   │
│   │- 安全扫描│ │- 内存   │ │- 最佳实践│                   │
│   │- 问题定位│ │- CPU    │ │- 库推荐  │                   │
│   └──────────┘ └──────────┘ └──────────┘                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 技能组合带来的涌现能力

1+1>2 的效应：当多个 Skills 协同工作时，可以产生单个 Skill 无法实现的能力。

**示例：多语言代码审查**

```
翻译 Skill + 代码审查 Skill = 跨语言代码审查能力

用户（中文）："请审查这段代码"
    │
    ▼
┌─────────────────┐
│  翻译 Skill      │  将中文请求翻译为英文
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  代码审查 Skill  │  用英文进行专业审查
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  翻译 Skill      │  将审查结果翻译回中文
└────────┬────────┘
         │
         ▼
用户收到中文的审查报告
```

---

## 1.3 Skills 在 Agent 架构中的位置

### 完整架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Agent 整体架构                           │
│                                                                 │
│  ┌───────────┐                                                  │
│  │  用户请求  │                                                  │
│  └─────┬─────┘                                                  │
│        │                                                        │
│        ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Agent Core                            │   │
│  │                                                          │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │   │
│  │  │  Thought    │───▶│   Action    │───▶│  Observation│  │   │
│  │  │  (思考)     │    │  (行动)     │    │   (观察)    │  │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘  │   │
│  │         ▲                                      │         │   │
│  │         └──────────────────┬───────────────────┘         │   │
│  └────────────────────────────┼─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Skill System                          │   │
│  │                                                          │   │
│  │  ┌─────────────────┐    ┌─────────────────────────┐     │   │
│  │  │  Skill Router   │───▶│    Active Skill Stack   │     │   │
│  │  │  (技能路由)     │    │    (激活技能栈)          │     │   │
│  │  └─────────────────┘    └─────────────────────────┘     │   │
│  │           │                      │                       │   │
│  │           ▼                      ▼                       │   │
│  │  ┌─────────────────┐    ┌─────────────────────────┐     │   │
│  │  │ Skill Registry  │    │   Combined System       │     │   │
│  │  │ (技能注册表)    │    │     Prompt              │     │   │
│  │  └─────────────────┘    └─────────────────────────┘     │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Tool Layer                           │   │
│  │                                                          │   │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  │   │
│  │   │ CalcTool │  │ Search   │  │ File     │  │  ...   │  │   │
│  │   └──────────┘  └──────────┘  └──────────┘  └────────┘  │   │
│  │                                                          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 数据流详解

```
步骤 1: 用户请求输入
  │
  ▼
步骤 2: Skill Router 根据关键词检测相关 Skills
  │
  ├── 匹配 triggers
  ├── 检查优先级
  └── 返回候选 Skills 列表
  │
  ▼
步骤 3: 激活 Skills 并生成组合提示词
  │
  ├── 调用 on_activate 钩子
  ├── 收集 system_prompt
  └── 拼接为完整提示词
  │
  ▼
步骤 4: Agent 执行 ReAct 循环
  │
  ├── Thought: 基于当前状态和提示词思考
  ├── Action: 选择并调用工具
  ├── Observation: 获取工具执行结果
  └── 循环直到问题解决
  │
  ▼
步骤 5: 清理和停用 Skills
  │
  ├── 调用 on_deactivate 钩子
  └── 返回最终结果
```

---

## 1.4 业界 Skills/Plugins 系统对比

### 主流方案对比表

| 方案 | 提供方 | 名称 | 核心机制 | 配置方式 | 适用场景 |
|------|--------|------|---------|---------|---------|
| **Anthropic** | Anthropic | Claude Desktop Skills | MCP 协议 | JSON 配置文件 | 桌面应用集成 |
| **OpenAI** | OpenAI | GPTs | Actions + Knowledge | 可视化界面 + OpenAPI | 非技术用户 |
| **LangChain** | LangChain | Tools + Agents | Tool 接口 + Executor | Python 代码 | 开发者 |
| **LlamaIndex** | LlamaIndex | Tools + QueryEngine | ToolSpec 模式 | Python 代码 | RAG 应用 |
| **Semantic Kernel** | Microsoft | Plugins | 函数签名解析 | C#/Python 代码 | 企业应用 |
| **本项目** | - | Skills 系统 | Skill 基类 + Manager | Python 代码 + 自动激活 | 教学/自定义 |

### 各方案详细分析

#### Anthropic Claude Desktop (MCP)

```
┌─────────────────────────────────────────────────────────────┐
│                    MCP 架构                                 │
│                                                             │
│  ┌─────────────┐         ┌─────────────┐                   │
│  │   Claude    │◀───────▶│     MCP     │                   │
│  │   Desktop   │  JSON   │    Host     │                   │
│  │             │  RPC    │             │                   │
│  └─────────────┘         └──────┬──────┘                   │
│                                 │                           │
│                    ┌────────────┼────────────┐              │
│                    │            │            │              │
│                    ▼            ▼            ▼              │
│              ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│              │  GitHub  │ │  SQLite  │ │  自定义  │        │
│              │  Server  │ │  Server  │ │  Server  │        │
│              └──────────┘ └──────────┘ └──────────┘        │
│                                                             │
│  配置文件示例 (.json)：                                      │
│  {                                                         │
│    "mcpServers": {                                         │
│      "github": {                                           │
│        "command": "npx",                                   │
│        "args": ["-y", "@modelcontextprotocol/server-github"]│
│      }                                                     │
│    }                                                       │
│  }                                                         │
└─────────────────────────────────────────────────────────────┘
```

**优点**：
- 标准化协议，兼容性好
- 支持本地服务集成
- 安全沙箱机制

**缺点**：
- 配置相对复杂
- 需要额外的 Server 进程

#### OpenAI GPTs

```
┌─────────────────────────────────────────────────────────────┐
│                    GPTs 配置界面                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Name: 数据分析助手                                  │   │
│  │  Description: 帮助分析数据并生成可视化报告            │   │
│  │                                                      │   │
│  │  Instructions:                                       │   │
│  │  你是一位数据分析专家，擅长...                        │   │
│  │                                                      │   │
│  │  Capabilities:                                       │   │
│  │  ☑ DALL-E 3 图像生成                                  │   │
│  │  ☑ Web 浏览                                          │   │
│  │  ☑ 代码解释器                                        │   │
│  │                                                      │   │
│  │  Actions (OpenAPI):                                  │   │
│  │  ┌─────────────────────────────────────────────┐     │   │
│  │  │  import { Action } from '@langchain/core'   │     │   │
│  │  │  export const dataAnalysis: Action = {...}  │     │   │
│  │  └─────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**优点**：
- 可视化配置，门槛低
- 内置丰富能力
- 易于分享和发布

**缺点**：
- 封闭生态系统
- 自定义能力有限
- 依赖 OpenAI 平台

#### LangChain Tools

```python
# LangChain 工具定义方式
from langchain.tools import tool

@tool
def calculate(expression: str) -> str:
    """Calculate mathematical expressions."""
    return str(eval(expression))

# Agent 使用
from langchain.agents import initialize_agent, Tool
from langchain.llms import OpenAI

tools = [
    Tool(
        name="Calculator",
        func=calculate.run,
        description="Useful for calculations"
    )
]

agent = initialize_agent(tools, OpenAI(), agent="zero-shot-react-description")
```

**优点**：
- 高度灵活
- 丰富的内置工具
- 强大的 Agent 框架

**缺点**：
- 学习曲线陡峭
- 代码量大
- 状态管理复杂

---

# 2. Skill 核心架构深度解析

## 2.1 Skill 抽象类详解

### 完整源码（带详细注释）

让我们逐行分析 `src/skills/base.py` 中的 Skill 抽象类：

```python
"""
Skill Base Classes

Defines the foundation for the Skills system:
- SkillContext: Context in which a skill operates
- Skill: A specialized capability with prompts and tools
"""

from typing import Any, Callable, Dict, List, Optional
from dataclasses import dataclass, field
from abc import ABC, abstractmethod
import logging

logger = logging.getLogger(__name__)


@dataclass
class SkillContext:
    """
    Context provided to a skill during execution.

    Attributes:
        query: The user's query
        agent: Reference to the agent
        memory: Reference to agent's memory
        tools: Available tools
        metadata: Additional context data
    """

    query: str
    agent: Optional[Any] = None
    memory: Optional[Any] = None
    tools: Optional[Any] = None
    metadata: Dict[str, Any] = field(default_factory=dict)


class Skill(ABC):
    """
    A Skill is a specialized capability that enhances an agent.

    Unlike a simple tool, a Skill can:
    - Provide a system prompt (注入专业提示词)
    - Register multiple related tools (管理多个工具)
    - Have lifecycle hooks (activate, deactivate) (生命周期管理)
    - Maintain internal state (维护内部状态)

    Example:
        >>> class CodeReviewSkill(Skill):
        ...     @property
        ...     def name(self) -> str:
        ...         return "code_review"
        ...
        ...     @property
        ...     def description(self) -> str:
        ...         return "Review code for quality and issues"
        ...
        ...     @property
        ...     def system_prompt(self) -> str:
        ...         return "You are a code reviewer..."
        ...
        ...     def get_tools(self) -> List:
        ...         return [CodeAnalyzerTool()]
    """

    @property
    @abstractmethod
    def name(self) -> str:
        """
        Unique identifier for the skill.

        设计原则：
        1. 唯一性：不能与其他 Skill 重名
        2. 简洁性：使用下划线分隔的小写单词
        3. 语义化：名称应反映技能功能

        好的命名示例：
        - code_review      ✓
        - translation      ✓
        - data_analysis    ✓

        不好的命名示例：
        - skill1           ✗ (无语义)
        - mySkill          ✗ (驼峰式，不符合 Python 规范)
        - CodeReviewSkill  ✗ (与类名重复)
        """
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """
        Human-readable description of the skill.

        如何写好技能描述：
        1. 简洁明了：一句话概括核心功能
        2. 包含场景：说明适用场景
        3. 突出价值：说明能解决什么问题

        示例：
        - "代码审查专家，发现潜在问题并提供改进建议"
        - "多语言翻译助手，支持中英日韩等语言互译"
        - "数据分析助手，生成统计报告和可视化建议"
        """
        pass

    @property
    def system_prompt(self) -> Optional[str]:
        """
        System prompt to inject when skill is active.

        这是 Skills 区别于 Tools 的核心特性之一。

        提示词工程技巧：
        1. 角色定义：明确 Skill 的专业身份
        2. 任务描述：说明具体要做什么
        3. 步骤指导：提供清晰的操作步骤
        4. 格式规范：定义输出格式
        5. 约束条件：说明注意事项

        示例模板：
        ```
        你是 [角色]。

        你的任务是 [任务描述]。

        执行步骤：
        1. [步骤 1]
        2. [步骤 2]
        3. [步骤 3]

        输出格式：
        [格式说明]

        注意事项：
        [约束条件]
        ```
        """
        return None

    @property
    def triggers(self) -> List[str]:
        """
        Keywords or patterns that trigger this skill.

        触发器设计原则：
        1. 覆盖常用表达：包括同义词、相关短语
        2. 考虑多语言：支持中英文触发
        3. 避免过度触发：不要使用太常见的词

        示例（代码审查 Skill）：
        ```python
        @property
        def triggers(self) -> List[str]:
            return [
                # 中文触发词
                "代码审查", "代码检查", "审查代码",
                "代码质量", "代码问题",

                # 英文触发词
                "code review", "review code", "check code",
                "code quality",

                # 相关术语
                "bug", "lint", "refactor"
            ]
        ```
        """
        return []

    def get_tools(self) -> List:
        """
        Get tools provided by this skill.

        技能提供的工具列表。这些工具会被自动注册到 Agent。

        工具设计原则：
        1. 相关性：工具应与技能功能紧密相关
        2. 完整性：覆盖技能所需的所有能力
        3. 独立性：每个工具职责单一

        示例（代码审查 Skill 的工具）：
        - CodeAnalyzerTool: 分析代码结构和语法
        - ComplexityCheckerTool: 检查代码复杂度
        - StyleLinterTool: 检查代码风格
        """
        return []

    def on_activate(self, context: SkillContext) -> None:
        """
        Called when the skill is activated.

        生命周期钩子：技能激活时调用。

        典型用途：
        - 初始化内部状态
        - 加载配置文件
        - 记录激活日志
        - 准备所需资源

        Args:
            context: Skill 执行上下文，包含查询、Agent 引用等
        """
        logger.debug(f"Skill {self.name} activated")

    def on_deactivate(self) -> None:
        """
        Called when the skill is deactivated.

        生命周期钩子：技能停用时调用。

        典型用途：
        - 清理临时资源
        - 保存状态
        - 记录停用日志
        - 释放连接

        最佳实践：
        - 确保不会泄露资源
        - 处理可能的异常情况
        """
        logger.debug(f"Skill {self.name} deactivated")

    def should_activate(self, query: str) -> bool:
        """
        Determine if this skill should activate for a query.

        默认的关键词匹配实现。可以重写此方法实现更复杂的激活逻辑。

        Args:
            query: User query

        Returns:
            True if skill should activate
        """
        query_lower = query.lower()
        for trigger in self.triggers:
            if trigger.lower() in query_lower:
                return True
        return False

    def __repr__(self) -> str:
        return f"Skill(name={self.name!r})"
```

### 各属性/方法详细解析

#### `name` 属性

```python
@property
@abstractmethod
def name(self) -> str:
    """Unique identifier for the skill."""
    pass
```

**设计理由**：
- 使用 `@property` 而非普通属性：确保只读，防止外部修改
- 使用 `@abstractmethod`：强制子类实现，确保每个 Skill 都有唯一标识
- 返回 `str` 类型：简单、可序列化、易于比较

**命名规范**：
```python
# 推荐 ✓
name = "code_review"
name = "data_analysis"
name = "translation"

# 不推荐 ✗
name = "CodeReview"      # 不符合 Python 命名规范
name = "my-skill"        # 包含特殊字符
name = "skill_123"       # 无语义
```

#### `description` 属性

```python
@property
@abstractmethod
def description(self) -> str:
    """Human-readable description of the skill."""
    pass
```

**写好描述的技巧**：

| 技巧 | 不好的例子 | 好的例子 |
|------|-----------|---------|
| 具体明确 | "一个工具" | "代码质量审查工具" |
| 说明用途 | "可以做很多事" | "发现 bug 和安全漏洞" |
| 包含场景 | "很好用" | "适用于 PR 审查和重构" |

#### `system_prompt` 属性

这是 Skills 系统的核心特性，用于注入领域知识。

**提示词结构模板**：

```python
@property
def system_prompt(self) -> str:
    return """
你是 [专业角色]。

【核心职责】
- 职责 1
- 职责 2

【执行流程】
1. 第一步：...
2. 第二步：...
3. 第三步：...

【输出规范】
- 格式要求
- 内容要求

【注意事项】
- 注意点 1
- 注意点 2
"""
```

#### `triggers` 属性

**触发器算法实现**：

```python
def should_activate(self, query: str) -> bool:
    """
    基于关键词匹配的激活检测算法。

    实现细节：
    1. 大小写不敏感：都转换为小写后比较
    2. 子串匹配：触发词出现在查询任何位置都激活
    3. 短路求值：找到第一个匹配就返回 True

    时间复杂度：O(n * m)，n 为触发词数量，m 为查询长度
    """
    query_lower = query.lower()
    for trigger in self.triggers:
        if trigger.lower() in query_lower:
            return True
    return False
```

**高级激活策略**（重写示例）：

```python
def should_activate(self, query: str) -> bool:
    """使用更复杂的匹配逻辑"""

    # 1. 精确匹配优先
    if query.strip() in self.triggers:
        return True

    # 2. 正则匹配（例如匹配特定格式）
    import re
    if re.search(r'\b(code|代码)\s*(review|审查)\b', query, re.I):
        return True

    # 3. 多语言支持
    for trigger in self.triggers:
        if trigger in query:  # 不转小写，支持中文
            return True

    return False
```

#### 生命周期钩子

**`on_activate` 最佳实践**：

```python
def on_activate(self, context: SkillContext) -> None:
    """激活时的初始化操作"""

    # 1. 记录日志
    logger.info(f"激活技能：{self.name}")

    # 2. 验证上下文
    if not context.query:
        logger.warning("空查询激活了技能")

    # 3. 初始化状态
    self._execution_count = 0
    self._start_time = time.time()

    # 4. 加载配置
    self._load_config()

    # 5. 准备资源
    self._connection = create_database_connection()
```

**`on_deactivate` 最佳实践**：

```python
def on_deactivate(self) -> None:
    """停用时的清理操作"""

    # 1. 记录统计信息
    duration = time.time() - self._start_time
    logger.info(f"技能执行：{self._execution_count}次，耗时{duration:.2f}秒")

    # 2. 释放资源
    if hasattr(self, '_connection'):
        self._connection.close()

    # 3. 保存状态
    self._save_state()
```

---

## 2.2 SkillContext 上下文类深度解析

### 完整定义

```python
@dataclass
class SkillContext:
    """
    Context provided to a skill during execution.

    SkillContext 是 Skill 执行时的"环境"，提供了 Skill 与外界交互的所有必要信息。

    为什么要需要 Context？
    --------------------
    1. 解耦：Skill 不需要知道 Agent 的具体实现
    2. 灵活：可以随时替换或修改 Context
    3. 可测试：测试时可以传入 Mock Context

    设计原理：
    ---------
    使用 @dataclass 的原因：
    - 自动生成 __init__、__repr__ 等方法
    - 代码简洁，易于阅读
    - 支持默认值
    - 类型提示友好
    """

    query: str                    # 用户查询（必填）
    agent: Optional[Any] = None   # Agent 引用（可选）
    memory: Optional[Any] = None  # 记忆引用（可选）
    tools: Optional[Any] = None   # 工具注册表（可选）
    metadata: Dict[str, Any] = field(default_factory=dict)  # 元数据
```

### 字段详解

#### `query: str`

```python
"""用户查询 - 唯一必填字段"""

# 使用场景
def on_activate(self, context: SkillContext):
    # 根据查询内容做不同的初始化
    if "python" in context.query.lower():
        self.language = "python"
    elif "javascript" in context.query.lower():
        self.language = "javascript"

    # 提取查询中的关键信息
    self.code_snippet = self._extract_code(context.query)
```

#### `agent: Optional[Any] = None`

```python
"""Agent 引用 - 用于需要访问 Agent 状态的复杂场景"""

# 使用场景
def on_activate(self, context: SkillContext):
    # 访问 Agent 的配置
    if context.agent:
        self.max_iterations = context.agent.config.max_iterations

    # 修改 Agent 状态
    if context.agent:
        context.agent.set_state(AgentState.SKILL_EXECUTING)
```

#### `memory: Optional[Any] = None`

```python
"""记忆引用 - 用于需要历史上下文的场景"""

# 使用场景
def on_activate(self, context: SkillContext):
    if context.memory:
        # 获取历史对话
        history = context.memory.get_messages(limit=5)

        # 检查是否有相关记忆
        relevant = context.memory.search("代码审查")
```

#### `tools: Optional[Any] = None`

```python
"""工具注册表引用 - 用于需要调用其他工具的场景"""

# 使用场景
def execute(self, context: SkillContext):
    if context.tools:
        # 调用其他工具
        search_result = context.tools.get("search").execute(query="...")
```

#### `metadata: Dict[str, Any]`

```python
"""元数据 - 传递自定义配置和状态"""

# 使用场景 1：传递配置
context = SkillContext(
    query="审查代码",
    metadata={
        "language": "python",
        "strict_mode": True,
        "output_format": "json"
    }
)

# 使用场景 2：状态共享
context.metadata["execution_id"] = str(uuid.uuid4())
context.metadata["start_time"] = time.time()

# 在 Skill 中读取
def on_activate(self, context: SkillContext):
    self.language = context.metadata.get("language", "auto")
    self.strict_mode = context.metadata.get("strict_mode", False)
```

### Context 使用最佳实践

```python
# 好的实践 ✓
context = SkillContext(query="hello")  # 最小化使用
context = SkillContext(query="...", metadata={"key": "value"})  # 按需添加

# 不好的实践 ✗
context = SkillContext(
    query="...",
    agent=agent,
    memory=memory,
    tools=tools,
    config=config,
    extra1=...,  # 不要添加不需要的字段
    extra2=...,
)
```

---

## 2.3 FunctionSkill：函数快速封装

### 完整实现

```python
class FunctionSkill(Skill):
    """
    A simple skill created from functions.

    FunctionSkill 的核心思想：装饰器模式
    - 将普通函数"装饰"为 Skill
    - 隐藏复杂的类继承细节
    - 适合快速创建简单 Skill

    Example:
        >>> skill = FunctionSkill(
        ...     name="greeting",
        ...     description="Handle greetings",
        ...     system_prompt="Be friendly and helpful.",
        ...     triggers=["hello", "hi"],
        ... )
    """

    def __init__(
        self,
        name: str,
        description: str,
        system_prompt: Optional[str] = None,
        triggers: Optional[List[str]] = None,
        tools: Optional[List] = None,
        on_activate: Optional[Callable] = None,
        on_deactivate: Optional[Callable] = None,
    ):
        """
        Initialize function skill.

        参数说明：
        - name: 技能名称（必填）
        - description: 技能描述（必填）
        - system_prompt: 系统提示词（可选）
        - triggers: 触发关键词（可选）
        - tools: 提供的工具（可选）
        - on_activate: 激活回调函数（可选）
        - on_deactivate: 停用回调函数（可选）
        """
        self._name = name
        self._description = description
        self._system_prompt = system_prompt
        self._triggers = triggers or []
        self._tools = tools or []
        self._on_activate = on_activate
        self._on_deactivate = on_deactivate

    @property
    def name(self) -> str:
        return self._name

    @property
    def description(self) -> str:
        return self._description

    @property
    def system_prompt(self) -> Optional[str]:
        return self._system_prompt

    @property
    def triggers(self) -> List[str]:
        return self._triggers

    def get_tools(self) -> List:
        return self._tools

    def on_activate(self, context: SkillContext) -> None:
        """调用用户提供的回调函数"""
        if self._on_activate:
            self._on_activate(context)
        super().on_activate(context)

    def on_deactivate(self) -> None:
        """调用用户提供的回调函数"""
        if self._on_deactivate:
            self._on_deactivate()
        super().on_deactivate()
```

### 使用示例

```python
# 示例 1：简单的问候技能
greeting_skill = FunctionSkill(
    name="greeting",
    description="处理问候请求",
    system_prompt="你是一个友好的助手，用热情的方式回应用户的问候。",
    triggers=["你好", "hello", "hi", "早上好", "下午好"],
)

# 示例 2：带回调的技能
def my_activate_handler(context: SkillContext):
    print(f"天气技能已激活，查询：{context.query}")

weather_skill = FunctionSkill(
    name="weather",
    description="查询天气信息",
    system_prompt="你是天气查询助手，提供准确的天气信息。",
    triggers=["天气", "weather", "气温", "下雨"],
    on_activate=my_activate_handler,
)

# 示例 3：带工具的 FunctionSkill
@tool(name="calculate", description="执行数学计算")
def calculate(expression: str) -> str:
    return str(eval(expression))

math_skill = FunctionSkill(
    name="math",
    description="数学计算助手",
    system_prompt="你是数学计算专家，准确执行各种计算。",
    triggers=["计算", "calculate", "math"],
    tools=[calculate],
)
```

### 适用场景和限制

| 场景 | 推荐度 | 说明 |
|------|--------|------|
| 简单技能（无状态） | ⭐⭐⭐⭐⭐ | 非常适合 |
| 快速原型 | ⭐⭐⭐⭐⭐ | 几行代码搞定 |
| 复杂技能（有状态） | ⭐⭐ | 建议继承 Skill 类 |
| 需要多工具 | ⭐⭐⭐ | 可以但代码较长 |
| 需要生命周期管理 | ⭐⭐⭐ | 支持但有限 |

---

## 2.4 内置 Skills 源码逐行分析

### CalculatorSkill

```python
class CalculatorSkill(Skill):
    """Skill for mathematical calculations."""

    @property
    def name(self) -> str:
        return "calculator"
        # 技能名称：使用小写 + 下划线
        # 这个名字会被 SkillManager 用来注册和查找

    @property
    def description(self) -> str:
        return "Perform mathematical calculations and solve math problems"
        # 技能描述：清晰说明功能是"数学计算"和"解题"
        # 这个描述会展示给用户，帮助他们了解何时使用

    @property
    def system_prompt(self) -> str:
        return (
            "You are a mathematical assistant. "
            "Use the calculate tool to perform accurate calculations. "
            "Always verify your results and show your work."
        )
        # 系统提示词设计分析：
        # 1. 角色定义："mathematical assistant"
        # 2. 工具使用："use the calculate tool"
        # 3. 行为准则："verify results"、"show your work"
        # 这样的提示词能让 LLM 更好地扮演角色

    @property
    def triggers(self) -> List[str]:
        return [
            "calculate", "compute", "math", "add", "subtract",
            "multiply", "divide", "equals", "sum", "total",
        ]
        # 触发词设计：
        # - 覆盖常用数学操作
        # - 包含同义词（calculate/compute）
        # - 包含具体运算（add/subtract/multiply/divide）
        # - 包含相关概念（equals/sum/total）

    def get_tools(self):
        from ..tools import CalculatorTool
        return [CalculatorTool()]
        # 延迟导入：避免循环引用
        # 只提供一个工具：CalculatorTool
        # 这符合单一职责原则
```

**参考输出**：

```python
# 使用示例
from src.skills import CalculatorSkill

skill = CalculatorSkill()
print(f"名称：{skill.name}")
print(f"描述：{skill.description}")
print(f"触发词：{skill.triggers}")
print(f"工具：{skill.get_tools()}")
```

```
参考输出：
名称：calculator
描述：Perform mathematical calculations and solve math problems
触发词：['calculate', 'compute', 'math', 'add', 'subtract', 'multiply', 'divide', 'equals', 'sum', 'total']
工具：[Tool(name='calculator')]
```

### SearchSkill

```python
class SearchSkill(Skill):
    """Skill for searching information."""

    @property
    def name(self) -> str:
        return "search"

    @property
    def description(self) -> str:
        return "Search for information on the web"

    @property
    def system_prompt(self) -> str:
        return (
            "You are a search assistant. "
            "Use the search tool to find relevant information. "
            "Summarize and cite your sources."
        )
        # 提示词要点：
        # 1. 强调"search assistant"角色
        # 2. 说明使用 search tool
        # 3. 要求"summarize"和"cite sources"（重要！）

    @property
    def triggers(self) -> List[str]:
        return [
            "search", "find", "look up", "what is", "who is",
            "when did", "where is", "how to",
        ]
        # 触发词包括：
        # - 直接搜索词：search, find, look up
        # - 疑问词：what is, who is, when did, where is, how to
        # 覆盖了大部分信息查询场景

    def get_tools(self):
        from ..tools import SearchTool
        return [SearchTool()]
```

### FilesystemSkill

```python
class FilesystemSkill(Skill):
    """Skill for file system operations."""

    @property
    def name(self) -> str:
        return "filesystem"

    @property
    def description(self) -> str:
        return "Read, write, and manage files"

    @property
    def system_prompt(self) -> str:
        return (
            "You are a file management assistant. "
            "Use the filesystem tool to read, write, and manage files. "
            "Always confirm before making destructive operations."
        )
        # 提示词特别强调安全：
        # "Always confirm before making destructive operations"
        # 这是为了防止意外删除或覆盖重要文件

    @property
    def triggers(self) -> List[str]:
        return [
            "file", "read", "write", "directory", "folder",
            "save", "load", "create file", "delete file",
        ]
        # 触发词覆盖所有文件操作场景

    def get_tools(self):
        from ..tools import FilesystemTool
        return [FilesystemTool()]
```

---

# 3. SkillManager 技能管理器

## 3.1 SkillManager 的核心职责

SkillManager 是 Skills 系统的"中枢神经"，负责管理所有技能的注册、激活、停用和协调。

```
┌─────────────────────────────────────────────────────────────┐
│                    SkillManager 职责图                       │
│                                                             │
│                      ┌───────────────┐                      │
│                      │ SkillManager  │                      │
│                      └───────┬───────┘                      │
│                              │                              │
│         ┌────────────────────┼────────────────────┐         │
│         │                    │                    │         │
│         ▼                    ▼                    ▼         │
│   ┌───────────┐        ┌───────────┐       ┌───────────┐   │
│   │ 注册管理   │        │ 激活管理   │       │ 提示词管理 │   │
│   │           │        │           │       │           │   │
│   │ • register│        │ • detect  │       │ • combine │   │
│   │ • unregister      │ • activate│       │ • inject  │   │
│   │ • list    │        │ • deactivate       │ • format│   │
│   │ • get     │        │ • status  │       │           │   │
│   └───────────┘        └───────────┘       └───────────┘   │
│                                                             │
│         ┌────────────────────┐                              │
│         │ 工具管理            │                              │
│         │ • get_tools_for_skill                            │
│         │ • aggregate tools                                │
│         └────────────────────┘                              │
└─────────────────────────────────────────────────────────────┘
```

### 核心职责详解

| 职责 | 说明 | 关键方法 |
|------|------|---------|
| **注册管理** | 管理技能的注册和注销 | `register_skill()`, `unregister_skill()` |
| **查询检索** | 根据名称获取技能 | `get_skill()` |
| **技能检测** | 根据查询自动检测相关技能 | `detect_skills()` |
| **激活管理** | 控制技能的激活和停用 | `activate_skill()`, `deactivate_skill()` |
| **提示词生成** | 组合多个技能的提示词 | `get_combined_prompt()` |
| **工具聚合** | 收集所有技能提供的工具 | `get_tools_for_skills()` |

---

## 3.2 完整源码解析（带详细注释）

```python
"""
Skill Manager

Manages skill registration, activation, and lifecycle.
"""

from typing import Any, Dict, List, Optional, Set
import logging

from .base import Skill, SkillContext

logger = logging.getLogger(__name__)


class SkillManager:
    """
    Manager for skill registration, activation, and lifecycle.

    The SkillManager handles:
    - Registering skills（技能注册）
    - Detecting which skills to activate（技能检测）
    - Injecting system prompts（提示词注入）
    - Providing skill tools to the agent（工具提供）

    Example:
        >>> manager = SkillManager()
        >>> manager.register_skill(CalculatorSkill())
        >>> manager.register_skill(SearchSkill())
        >>>
        >>> # Detect which skills should activate for a query
        >>> active = manager.detect_skills("Calculate 15 * 7")
        >>> print(active)  # [CalculatorSkill]
        >>>
        >>> # Get combined system prompt from active skills
        >>> prompt = manager.get_combined_prompt(active)
    """

    def __init__(self):
        """Initialize the skill manager.

        初始化三个核心数据结构：
        1. _skills: 技能注册表（字典）
        2. _active_skills: 激活的技能集合
        3. _priority_order: 技能优先级顺序列表
        """
        self._skills: Dict[str, Skill] = {}          # 技能注册表
        self._active_skills: Set[str] = set()        # 激活的技能集合
        self._priority_order: List[str] = []         # 优先级顺序

        logger.info("Initialized SkillManager")
        # 日志记录初始化事件，便于调试和追踪

    def register_skill(
        self,
        skill: Skill,
        priority: int = 0,
    ) -> None:
        """
        Register a skill.

        注册技能是将技能添加到管理器的过程。注册后的技能可以被检测、激活和使用。

        Args:
            skill: Skill to register（要注册的技能实例）
            priority: Priority for activation, higher = more important（优先级，越高越重要）

        优先级机制说明：
        ---------------
        - 优先级用于决定技能检测时的返回顺序
        - 高优先级的技能会排在前面，更容易被 Agent 使用
        - 优先级为 0 表示普通优先级
        - 优先级可以是负数，表示低优先级

        示例：
        >>> manager = SkillManager()
        >>> manager.register_skill(CalculatorSkill(), priority=1)  # 高优先级
        >>> manager.register_skill(SearchSkill(), priority=0)      # 普通优先级
        >>> manager.register_skill(WeatherSkill(), priority=-1)    # 低优先级
        """
        if skill.name in self._skills:
            logger.warning(f"Overwriting existing skill: {skill.name}")
            # 警告：覆盖已有技能
            # 这可能是无意的，所以记录警告日志

        self._skills[skill.name] = skill
        # 将技能添加到注册表

        # Maintain priority order（维护优先级顺序）
        self._priority_order.append(skill.name)
        # 先添加到列表末尾

        self._priority_order.sort(
            key=lambda name: getattr(self._skills[name], "_priority", 0),
            reverse=True,  # 降序排列：高优先级在前
        )
        # 按优先级重新排序

        setattr(skill, "_priority", priority)
        # 将优先级属性设置到技能实例上（便于后续访问）

        logger.info(f"Registered skill: {skill.name}")
        # 记录注册日志

    def unregister_skill(self, name: str) -> bool:
        """
        Unregister a skill.

        Args:
            name: Skill name to remove

        Returns:
            True if removed, False if not found

        注意：注销技能会同时：
        1. 从注册表删除
        2. 从优先级列表删除
        3. 从激活集合删除（如果已激活）
        """
        if name in self._skills:
            del self._skills[name]              # 从注册表删除
            self._priority_order.remove(name)   # 从优先级列表删除
            self._active_skills.discard(name)   # 从激活集合删除
            logger.info(f"Unregistered skill: {name}")
            return True
        return False

    def get_skill(self, name: str) -> Optional[Skill]:
        """
        Get a skill by name.

        Args:
            name: Skill name

        Returns:
            Skill instance or None if not found

        示例：
        >>> skill = manager.get_skill("calculator")
        >>> if skill:
        ...     print(f"Found: {skill.name}")
        """
        return self._skills.get(name)

    def list_skills(self) -> List[str]:
        """List all registered skill names."""
        return list(self._skills.keys())

    def detect_skills(self, query: str) -> List[Skill]:
        """
        Detect which skills should activate for a query.

        检测算法：
        --------
        1. 遍历所有注册的技能（按优先级顺序）
        2. 对每个技能调用 should_activate(query)
        3. 如果返回 True，加入结果列表
        4. 返回所有匹配的技能

        Args:
            query: User query string

        Returns:
            List of skills that should activate, ordered by priority

        示例：
        >>> query = "请计算 15 * 7 并搜索结果"
        >>> skills = manager.detect_skills(query)
        >>> print([s.name for s in skills])
        ['calculator', 'search']
        """
        detected = []

        for name in self._priority_order:
            # 按优先级顺序遍历
            skill = self._skills[name]
            if skill.should_activate(query):
                # 调用技能的 should_activate 方法
                detected.append(skill)

        logger.debug(f"Detected skills for query: {[s.name for s in detected]}")
        return detected

    def activate_skill(
        self,
        name: str,
        context: Optional[SkillContext] = None,
    ) -> bool:
        """
        Manually activate a skill.

        手动激活技能会：
        1. 检查技能是否存在
        2. 检查是否已经激活
        3. 添加到激活集合
        4. 调用技能的 on_activate 钩子

        Args:
            name: Skill name to activate
            context: Activation context (optional, defaults to empty context)

        Returns:
            True if activated, False if not found or already active

        示例：
        >>> context = SkillContext(query="hello", agent=agent)
        >>> manager.activate_skill("greeting", context)
        """
        skill = self._skills.get(name)
        if not skill:
            logger.warning(f"Skill not found: {name}")
            return False

        if name in self._active_skills:
            logger.debug(f"Skill already active: {name}")
            return True

        self._active_skills.add(name)
        # 添加到激活集合

        skill.on_activate(context or SkillContext(query=""))
        # 调用技能的激活钩子

        logger.info(f"Activated skill: {name}")
        return True

    def deactivate_skill(self, name: str) -> bool:
        """
        Deactivate a skill.

        Args:
            name: Skill name to deactivate

        Returns:
            True if deactivated, False if not active or not found
        """
        skill = self._skills.get(name)
        if not skill or name not in self._active_skills:
            return False

        self._active_skills.remove(name)
        skill.on_deactivate()
        logger.info(f"Deactivated skill: {name}")
        return True

    def get_active_skills(self) -> List[Skill]:
        """Get currently active skills."""
        return [self._skills[name] for name in self._active_skills if name in self._skills]

    def get_tools_for_skills(self, skills: List[Skill]) -> List:
        """
        Get all tools from specified skills.

        Args:
            skills: List of skills to get tools from

        Returns:
            Combined list of all tools from all skills

        用途：
        - 将多个技能的工具聚合在一起
        - 用于注册到 Agent 的工具注册表

        示例：
        >>> skills = manager.detect_skills("计算并搜索")
        >>> tools = manager.get_tools_for_skills(skills)
        >>> for tool in tools:
        ...     registry.register(tool)
        """
        tools = []
        for skill in skills:
            tools.extend(skill.get_tools())
        return tools

    def get_combined_prompt(self, skills: Optional[List[Skill]] = None) -> str:
        """
        Get combined system prompt from skills.

        Args:
            skills: Skills to include (defaults to active skills)

        Returns:
            Combined system prompt string

        组合规则：
        --------
        1. 每个技能的提示词用空行分隔
        2. 每个提示词前添加技能名称标记
        3. 如果没有技能或技能没有提示词，返回空字符串

        输出格式：
        ```
        [skill_name_1]
        prompt_1

        [skill_name_2]
        prompt_2
        ```

        示例：
        >>> prompt = manager.get_combined_prompt()
        >>> print(prompt)
        [calculator]
        You are a mathematical assistant...

        [search]
        You are a search assistant...
        """
        if skills is None:
            skills = self.get_active_skills()
        # 默认使用激活的技能

        prompts = []
        for skill in skills:
            if skill.system_prompt:
                # 只包含有提示词的技能
                prompts.append(f"[{skill.name}]\n{skill.system_prompt}")

        return "\n\n".join(prompts)
        # 用双换行符连接所有提示词

    def clear_active(self) -> None:
        """Deactivate all active skills.

        调用所有激活技能的 on_deactivate 钩子，然后清空激活集合。

        使用场景：
        - 完成一次对话后清理状态
        - 重置 SkillManager
        """
        for name in list(self._active_skills):
            self.deactivate_skill(name)

    def __contains__(self, name: str) -> bool:
        """Check if a skill is registered."""
        return name in self._skills

    def __len__(self) -> int:
        """Return number of registered skills."""
        return len(self._skills)

    def __iter__(self):
        """Iterate over registered skills."""
        return iter(self._skills.values())
```

---

## 3.3 注册机制详解

### 完整注册流程

```
┌─────────────────────────────────────────────────────────────┐
│                    技能注册流程图                            │
│                                                             │
│   创建技能实例          调用 register_skill                 │
│        │                      │                             │
│        │                      ▼                             │
│        │           ┌──────────────────┐                     │
│        │           │ 检查是否重名      │                     │
│        │           └────────┬─────────┘                     │
│        │                    │                              │
│        │           ┌───────┴───────┐                       │
│        │           │  是           │  否                    │
│        │           │   ▼           │                       │
│        │      │ 记录警告日志 │                           │
│        │           │              │                       │
│        │           └───────┬───────┘                       │
│        │                   │                               │
│        │                   ▼                               │
│        │          ┌─────────────────┐                      │
│        │          │ 添加到注册表    │                      │
│        │          └────────┬────────┘                      │
│        │                   │                               │
│        │                   ▼                               │
│        │          ┌─────────────────┐                      │
│        │          │ 维护优先级顺序  │                      │
│        │          └────────┬────────┘                      │
│        │                   │                               │
│        │                   ▼                               │
│        │          ┌─────────────────┐                      │
│        │          │ 记录注册日志    │                      │
│        │          └─────────────────┘                      │
│        │                                                    │
└─────────────────────────────────────────────────────────────┘
```

### 注册示例代码 + 参考输出

```python
from src.skills import SkillManager, CalculatorSkill, SearchSkill

# 创建管理器
manager = SkillManager()

# 注册技能（带优先级）
manager.register_skill(CalculatorSkill(), priority=1)
manager.register_skill(SearchSkill(), priority=0)

# 列出所有技能
print("注册的技能:", manager.list_skills())

# 检查是否注册
print("calculator 已注册:", "calculator" in manager)

# 获取技能
skill = manager.get_skill("calculator")
print("获取技能:", skill.name if skill else None)

# 获取技能数量
print("技能数量:", len(manager))
```

**参考输出**：
```
注册的技能：['calculator', 'search']
calculator 已注册：True
获取技能：calculator
技能数量：2
```

---

## 3.4 技能检测算法

### should_activate 方法详解

```python
# Skill 基类中的默认实现
def should_activate(self, query: str) -> bool:
    """
    基于关键词匹配的激活检测算法。

    算法步骤：
    1. 将查询转换为小写（大小写不敏感）
    2. 遍历所有触发词
    3. 将触发词也转换为小写
    4. 检查触发词是否在查询中
    5. 找到第一个匹配就返回 True

    时间复杂度：O(n * m)
    - n = 触发词数量
    - m = 查询长度
    """
    query_lower = query.lower()
    for trigger in self.triggers:
        if trigger.lower() in query_lower:
            return True
    return False
```

### 优先级排序机制

```python
# SkillManager.detect_skills() 中的实现
def detect_skills(self, query: str) -> List[Skill]:
    detected = []

    # 关键点：按优先级顺序遍历
    # 高优先级的技能会先被检测到
    for name in self._priority_order:
        skill = self._skills[name]
        if skill.should_activate(query):
            detected.append(skill)

    return detected
```

### 检测示例 + 参考输出

```python
from src.skills import SkillManager, CalculatorSkill, SearchSkill, FilesystemSkill

manager = SkillManager()
manager.register_skill(CalculatorSkill(), priority=1)
manager.register_skill(SearchSkill(), priority=0)
manager.register_skill(FilesystemSkill(), priority=-1)

# 测试不同查询
queries = [
    "计算 15 * 7",
    "搜索 Python 教程",
    "读取文件内容",
    "计算并搜索相关信息",
    "今天天气不错",  # 无匹配
]

for query in queries:
    skills = manager.detect_skills(query)
    print(f"查询：'{query}'")
    print(f"匹配技能：{[s.name for s in skills]}")
    print()
```

**参考输出**：
```
查询：'计算 15 * 7'
匹配技能：['calculator']

查询：'搜索 Python 教程'
匹配技能：['search']

查询：'读取文件内容'
匹配技能：['filesystem']

查询：'计算并搜索相关信息'
匹配技能：['calculator', 'search']

查询：'今天天气不错'
匹配技能：[]
```

---

## 3.5 组合提示词生成

### 提示词拼接顺序的影响

```python
# get_combined_prompt 实现
def get_combined_prompt(self, skills: Optional[List[Skill]] = None) -> str:
    if skills is None:
        skills = self.get_active_skills()

    prompts = []
    for skill in skills:
        if skill.system_prompt:
            prompts.append(f"[{skill.name}]\n{skill.system_prompt}")

    return "\n\n".join(prompts)
```

### 提示词冲突解决策略

当多个技能的提示词有潜在冲突时，可以采用以下策略：

| 策略 | 说明 | 适用场景 |
|------|------|---------|
| **优先级覆盖** | 高优先级技能的提示词在后（覆盖前面） | 技能有明显优先级差异 |
| **模块化隔离** | 每个技能提示词独立成段，用标记分隔 | 技能功能相对独立 |
| **合并去重** | 分析提示词内容，合并重复部分 | 提示词较长时 |
| **动态选择** | 根据查询选择最相关的一个技能提示词 | 技能互斥时 |

### 组合提示词示例 + 参考输出

```python
from src.skills import SkillManager, CalculatorSkill, SearchSkill

manager = SkillManager()
manager.register_skill(CalculatorSkill())
manager.register_skill(SearchSkill())

# 激活技能
manager.activate_skill("calculator")
manager.activate_skill("search")

# 获取组合提示词
prompt = manager.get_combined_prompt()
print("组合提示词：")
print("=" * 50)
print(prompt)
print("=" * 50)
```

**参考输出**：
```
组合提示词：
==================================================
[calculator]
You are a mathematical assistant. Use the calculate tool to perform accurate calculations. Always verify your results and show your work.

[search]
You are a search assistant. Use the search tool to find relevant information. Summarize and cite your sources.
==================================================
```

---

## 3.6 SkillManager 使用场景

### 场景 1：独立使用 SkillManager

```python
# 不集成到 Agent，独立管理技能
from src.skills import SkillManager, CalculatorSkill, SearchSkill

def main():
    manager = SkillManager()
    manager.register_skill(CalculatorSkill())
    manager.register_skill(SearchSkill())

    while True:
        query = input("用户：")
        if query == "quit":
            break

        # 检测技能
        skills = manager.detect_skills(query)

        if skills:
            print(f"检测到技能：{[s.name for s in skills]}")

            # 激活技能
            for skill in skills:
                manager.activate_skill(skill.name)

            # 获取提示词
            prompt = manager.get_combined_prompt()
            print(f"组合提示词：{prompt[:100]}...")

            # 清理
            manager.clear_active()
        else:
            print("没有匹配的技能")

if __name__ == "__main__":
    main()
```

### 场景 2：与 Agent 集成

详见第 5 部分 [Skill 与 Agent 集成](#5-skill-与-agent-集成)。

---

# 4. 创建自定义 Skills 实战

本章将通过 4 个完整的实战案例，带你掌握创建自定义 Skills 的完整方法。每个案例都包含：
- 需求分析
- 工具设计
- 系统提示词设计
- 完整代码实现
- 参考输出示例

---

## 4.1 实战 1：代码审查 Skill（完整实现）

### 需求分析

代码审查（Code Review）是软件开发中的重要环节，目的是：
1. **发现 bug**：识别潜在的逻辑错误和边界条件问题
2. **安全检查**：发现安全漏洞和风险代码
3. **风格检查**：确保代码遵循团队规范
4. **性能建议**：识别性能瓶颈和优化空间
5. **改进建议**：提供具体的重构建议

### 工具设计

```
┌─────────────────────────────────────────────────────────────┐
│              CodeReviewSkill 工具集                          │
│                                                             │
│   ┌─────────────────┐                                       │
│   │ CodeAnalyzerTool│  分析代码结构和语法                    │
│   │                 │  - 解析代码                           │
│   │                 │  - 识别语法问题                       │
│   │                 │  - 检查 TODO/FIXME 标记                 │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │StyleLinterTool  │  检查代码风格                          │
│   │                 │  - PEP8 规范检查（Python）              │
│   │                 │  - 命名规范检查                       │
│   │                 │  - 格式检查                           │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │SecurityScannerTool│ 安全检查                            │
│   │                 │  - 敏感信息检测                       │
│   │                 │  - 危险函数识别                       │
│   │                 │  - SQL 注入风险检查                    │
│   └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### 完整代码实现

```python
"""
代码审查 Skill 完整实现

用法：
    from src.skills import CodeReviewSkill
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Any, Dict
import re


# ==================== 工具实现 ====================

class CodeAnalyzerTool(BaseTool):
    """代码分析工具：分析代码结构和语法"""

    @property
    def name(self) -> str:
        return "analyze_code"

    @property
    def description(self) -> str:
        return "Analyze code structure, identify syntax issues, and detect code smells"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "code": {
                    "type": "string",
                    "description": "The source code to analyze"
                },
                "language": {
                    "type": "string",
                    "description": "Programming language (python, javascript, etc.)"
                }
            },
            "required": ["code"]
        }

    def execute(self, **kwargs) -> ToolResult:
        code = kwargs.get("code", "")
        language = kwargs.get("language", "python")

        issues = []
        lines = code.split("\n")

        # 检查 1：检测 TODO/FIXME 标记
        for i, line in enumerate(lines, 1):
            if "TODO" in line.upper():
                issues.append(f"Line {i}: Found TODO marker")
            if "FIXME" in line.upper():
                issues.append(f"Line {i}: Found FIXME marker")

        # 检查 2：检测过长的函数（超过 50 行）
        func_lines = 0
        in_function = False
        for i, line in enumerate(lines, 1):
            if re.match(r'^\s*(def|function)\s+\w+', line):
                if in_function and func_lines > 50:
                    issues.append(f"Function starting at line {i - func_lines} is too long ({func_lines} lines)")
                in_function = True
                func_lines = 0
            elif in_function:
                func_lines += 1

        # 检查 3：检测空函数
        if re.search(r'def\s+\w+\s*\(\s*\)\s*:\s*\n\s*pass', code):
            issues.append("Found empty function (only contains 'pass')")

        # 检查 4：检测魔法数字
        magic_numbers = re.findall(r'(?<!\w)(\d{2,})(?!\w)', code)
        if magic_numbers:
            issues.append(f"Found magic numbers: {', '.join(set(magic_numbers))}. Consider using named constants.")

        # 检查 5：检测重复的 except 捕获
        if re.search(r'except\s*:|except\s+Exception\s*:', code):
            issues.append("Bare 'except' or 'except Exception' caught. Consider catching specific exceptions.")

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "issues": issues,
                "line_count": len(lines),
                "language": language,
            }
        )


class StyleLinterTool(BaseTool):
    """代码风格检查工具"""

    @property
    def name(self) -> str:
        return "lint_style"

    @property
    def description(self) -> str:
        return "Check code style and formatting issues"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "code": {
                    "type": "string",
                    "description": "The source code to lint"
                },
                "language": {
                    "type": "string",
                    "description": "Programming language"
                }
            },
            "required": ["code"]
        }

    def execute(self, **kwargs) -> ToolResult:
        code = kwargs.get("code", "")
        language = kwargs.get("language", "python")

        issues = []
        lines = code.split("\n")

        if language == "python":
            # Python 风格检查

            # 检查 1：缩进混合
            has_tabs = any('\t' in line for line in lines)
            has_spaces = any('    ' in line for line in lines)
            if has_tabs and has_spaces:
                issues.append("Mixed indentation: using both tabs and spaces")

            # 检查 2：行尾空格
            for i, line in enumerate(lines, 1):
                if line.endswith(' ') or line.endswith('\t'):
                    issues.append(f"Line {i}: Trailing whitespace")
                    break  # 只报告一次

            # 检查 3：行长度超过 100
            for i, line in enumerate(lines, 1):
                if len(line) > 100:
                    issues.append(f"Line {i}: Line too long ({len(line)} chars, max 100)")
                    break

            # 检查 4：import 顺序
            import_lines = [i for i, line in enumerate(lines) if line.startswith(('import ', 'from '))]
            if import_lines:
                # 检查 import 是否在代码中间
                if import_lines[-1] < len(lines) - 1:
                    last_import = import_lines[-1]
                    if any(lines[i].strip() and not lines[i].startswith('#') for i in range(last_import + 1, len(lines))):
                        issues.append("Imports should be at the top of the file")

            # 检查 5：类名是否大写开头
            class_matches = re.findall(r'class\s+(\w+)', code)
            for class_name in class_matches:
                if not class_name[0].isupper():
                    issues.append(f"Class name '{class_name}' should start with uppercase")

            # 检查 6：函数名是否小写 + 下划线
            func_matches = re.findall(r'def\s+(\w+)', code)
            for func_name in func_matches:
                if func_name != func_name.lower() or '__' in func_name:
                    if not func_name.startswith('__'):
                        issues.append(f"Function name '{func_name}' should use lowercase with underscores")

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "issues": issues,
                "language": language,
            }
        )


class SecurityScannerTool(BaseTool):
    """代码安全扫描工具"""

    @property
    def name(self) -> str:
        return "scan_security"

    @property
    def description(self) -> str:
        return "Scan code for security vulnerabilities and risky patterns"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "code": {
                    "type": "string",
                    "description": "The source code to scan"
                },
                "language": {
                    "type": "string",
                    "description": "Programming language"
                }
            },
            "required": ["code"]
        }

    def execute(self, **kwargs) -> ToolResult:
        code = kwargs.get("code", "")
        language = kwargs.get("language", "python")

        issues = []

        # 检查 1：硬编码密码/密钥
        secret_patterns = [
            (r'password\s*=\s*["\'][^"\']+["\']', "Hardcoded password"),
            (r'secret\s*=\s*["\'][^"\']+["\']', "Hardcoded secret"),
            (r'api_key\s*=\s*["\'][^"\']+["\']', "Hardcoded API key"),
            (r'token\s*=\s*["\'][^"\']+["\']', "Hardcoded token"),
        ]
        for pattern, message in secret_patterns:
            if re.search(pattern, code, re.IGNORECASE):
                issues.append(f"SECURITY: {message} detected")

        # 检查 2：eval/exec 使用
        if re.search(r'\beval\s*\(', code):
            issues.append("SECURITY: Use of eval() is dangerous, consider alternatives")
        if re.search(r'\bexec\s*\(', code):
            issues.append("SECURITY: Use of exec() is dangerous, consider alternatives")

        # 检查 3：SQL 拼接
        sql_patterns = [
            r'execute\s*\(\s*f["\'].*SELECT.*\{',
            r'execute\s*\(\s*["\'].*SELECT.*\+',
            r'execute\s*\(\s*["\'].*INSERT.*\+',
        ]
        for pattern in sql_patterns:
            if re.search(pattern, code, re.IGNORECASE):
                issues.append("SECURITY: Possible SQL injection risk, use parameterized queries")
                break

        # 检查 4：pickle 反序列化
        if re.search(r'pickle\.loads?\s*\(', code):
            issues.append("SECURITY: pickle.loads() can execute arbitrary code, use with caution")

        # 检查 5：临时文件风险
        if re.search(r'tempfile\.mktemp\s*\(', code):
            issues.append("SECURITY: tempfile.mktemp() is vulnerable to race conditions, use mkstemp()")

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "issues": issues,
                "severity": "HIGH" if any("SECURITY" in i for i in issues) else "LOW",
            }
        )


# ==================== Skill 实现 ====================

class CodeReviewSkill(Skill):
    """代码审查技能"""

    @property
    def name(self) -> str:
        return "code_review"

    @property
    def description(self) -> str:
        return "审查代码质量、发现潜在问题、提供改进建议"

    @property
    def system_prompt(self) -> str:
        return """
你是一位专业的代码审查专家，拥有丰富的编程经验和严谨的代码质量标准。

【核心职责】
1. **代码质量检查**：确保代码清晰、可读、易维护
2. **潜在问题识别**：发现 bug、安全漏洞、性能问题
3. **最佳实践**：确保遵循语言规范和最佳实践
4. **改进建议**：提供具体的、可执行的改进建议

【审查流程】
1. 首先使用 analyze_code 分析代码结构和语法问题
2. 然后使用 lint_style 检查代码风格
3. 最后使用 scan_security 进行安全扫描
4. 综合所有结果，生成审查报告

【输出格式】
请按以下格式输出审查结果：

## 代码审查报告

### 基本信息
- 语言：[编程语言]
- 行数：[代码行数]

### 问题列表

#### 严重问题
- [ ] 问题描述（位置 + 原因 + 修复建议）

#### 一般问题
- [ ] 问题描述（位置 + 原因 + 修复建议）

#### 建议改进
- [ ] 改进建议

### 总体评价
[对代码质量的整体评价]

【注意事项】
- 指出具体行号和问题原因
- 提供可执行的修复代码示例
- 语气建设性，避免批评性语言
- 优先报告安全问题
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 中文触发词
            "代码审查", "代码检查", "审查代码",
            "代码质量", "代码问题", "代码改进",
            "review", "check code", "lint",
            # 相关术语
            "bug", "refactor", "optimize",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            CodeAnalyzerTool(),
            StyleLinterTool(),
            SecurityScannerTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[CodeReview] 代码审查技能已激活")
        print(f"[CodeReview] 正在分析：{context.query[:50]}...")

    def on_deactivate(self) -> None:
        print(f"[CodeReview] 代码审查技能已停用")
```

### 参考输出示例

```python
# 测试代码审查 Skill
from src.skills import CodeReviewSkill

skill = CodeReviewSkill()
print(f"技能名称：{skill.name}")
print(f"技能描述：{skill.description}")
print(f"触发词：{skill.triggers}")
print(f"提供的工具：{[t.name for t in skill.get_tools()]}")
print()
print("系统提示词（前 200 字符）：")
print(skill.system_prompt[:200] + "...")
```

**参考输出**：
```
技能名称：code_review
技能描述：审查代码质量、发现潜在问题、提供改进建议
触发词：['代码审查', '代码检查', '审查代码', '代码质量', '代码问题', '代码改进', 'review', 'check code', 'lint', 'bug', 'refactor', 'optimize']
提供的工具：['analyze_code', 'lint_style', 'scan_security']

系统提示词（前 200 字符）：
你是一位专业的代码审查专家，拥有丰富的编程经验和严谨的代码质量标准。

【核心职责】
1. **代码质量检查**：确保代码清晰、可读、易维护
2. **潜在问题识别**：发现 bug、安全漏洞、性能问题
...
```

### 工具执行示例

```python
# 测试工具执行
test_code = """
def calculate(a, b):
    # TODO: 添加错误处理
    password = "secret123"
    result = eval(input("Enter expression: "))
    return result

def another_function(  ):
    pass
"""

analyzer = CodeAnalyzerTool()
result = analyzer.execute(code=test_code, language="python")
print("代码分析结果：")
print(result.to_string())
```

**参考输出**：
```
代码分析结果：
{
  "issues": [
    "Line 3: Found TODO marker",
    "Found empty function (only contains 'pass')"
  ],
  "line_count": 9,
  "language": "python"
}
```

---

## 4.2 实战 2：翻译助手 Skill（完整实现）

### 需求分析

翻译助手是一个实用的多语言翻译技能，需要：
1. **多语言支持**：支持中英日韩等常见语言互译
2. **术语管理**：保持专业术语的一致性
3. **上下文保持**：在多轮对话中保持翻译一致性
4. **格式规范**：按照指定格式输出翻译结果

### 工具设计

```
┌─────────────────────────────────────────────────────────────┐
│              TranslationSkill 工具集                         │
│                                                             │
│   ┌─────────────────┐                                       │
│   │ DictionaryTool  │  词典查询工具                          │
│   │                 │  - 单词释义                           │
│   │                 │  - 词性标注                           │
│   │                 │  - 例句展示                           │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │TerminologyTool  │  术语管理工具                          │
│   │                 │  - 专业术语查询                       │
│   │                 │  - 术语对照表                         │
│   │                 │  - 领域识别                           │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │TranslationMemoryTool│ 翻译记忆工具                       │
│   │                 │  - 历史翻译记录                       │
│   │                 │  - 相似句匹配                         │
│   │                 │  - 一致性检查                         │
│   └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### 完整代码实现

```python
"""
翻译助手 Skill 完整实现

用法：
    from src.skills import TranslationSkill
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Optional
import re


# ==================== 工具实现 ====================

class DictionaryTool(BaseTool):
    """词典查询工具"""

    @property
    def name(self) -> str:
        return "lookup_dictionary"

    @property
    def description(self) -> str:
        return "Look up word definitions, pronunciations, and example sentences"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "word": {
                    "type": "string",
                    "description": "The word to look up"
                },
                "source_lang": {
                    "type": "string",
                    "description": "Source language (en, zh, ja, ko)"
                },
                "target_lang": {
                    "type": "string",
                    "description": "Target language (en, zh, ja, ko)"
                }
            },
            "required": ["word"]
        }

    def execute(self, **kwargs) -> ToolResult:
        word = kwargs.get("word", "")
        source_lang = kwargs.get("source_lang", "auto")
        target_lang = kwargs.get("target_lang", "en")

        # 模拟词典查询（实际应用中可接入真实词典 API）
        # 这里提供简单的模拟数据用于演示

        mock_dictionary = {
            "hello": {
                "en": "你好",
                "zh": "hello",
                "pos": "interjection",
                "definition": "Used as a greeting or to attract attention",
                "example": "Hello, how are you?"
            },
            "你好": {
                "en": "hello",
                "zh": "你好",
                "pos": "greeting",
                "definition": "常用的问候语",
                "example": "你好，很高兴认识你"
            },
            "translate": {
                "en": "翻译",
                "zh": "translate",
                "pos": "verb",
                "definition": "To express in another language",
                "example": "Can you translate this document?"
            },
        }

        if word in mock_dictionary:
            entry = mock_dictionary[word]
            result = {
                "word": word,
                "translation": entry.get(target_lang if target_lang != "en" else "en", entry.get("en")),
                "pos": entry.get("pos", "unknown"),
                "definition": entry.get("definition", ""),
                "example": entry.get("example", ""),
            }
        else:
            result = {
                "word": word,
                "translation": f"[需要查询：{word}]",
                "pos": "unknown",
                "definition": "Word not found in dictionary",
                "example": "",
            }

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=result
        )


class TerminologyTool(BaseTool):
    """术语管理工具"""

    @property
    def name(self) -> str:
        return "lookup_terminology"

    @property
    def description(self) -> str:
        return "Look up professional terminology and domain-specific translations"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "term": {
                    "type": "string",
                    "description": "The terminology to look up"
                },
                "domain": {
                    "type": "string",
                    "description": "Domain/field (tech, medical, legal, etc.)"
                }
            },
            "required": ["term"]
        }

    def execute(self, **kwargs) -> ToolResult:
        term = kwargs.get("term", "")
        domain = kwargs.get("domain", "general")

        # 模拟术语库（实际应用中可接入真实术语库）
        mock_terminology = {
            "API": {
                "zh": "应用程序接口",
                "domain": "tech",
                "note": "Application Programming Interface"
            },
            "机器学习": {
                "en": "Machine Learning",
                "domain": "tech",
                "note": "ML"
            },
            "神经网络": {
                "en": "Neural Network",
                "domain": "tech",
                "note": "ANN (Artificial Neural Network)"
            },
            "contract": {
                "zh": "合同/契约",
                "domain": "legal",
                "note": "法律文件"
            },
        }

        if term in mock_terminology:
            entry = mock_terminology[term]
            if domain != "general" and entry.get("domain") != domain:
                return ToolResult(
                    status=ToolResultStatus.SUCCESS,
                    output={
                        "term": term,
                        "translation": "[可能不匹配当前领域]",
                        "note": f"Defined for {entry.get('domain')} domain"
                    }
                )
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output={
                    "term": term,
                    "translation": entry.get("zh", entry.get("en", "")),
                    "domain": entry.get("domain", "general"),
                    "note": entry.get("note", ""),
                }
            )

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "term": term,
                "translation": f"[需要查询术语：{term}]",
                "domain": domain,
            }
        )


class TranslationMemoryTool(BaseTool):
    """翻译记忆工具"""

    @property
    def name(self) -> str:
        return "search_translation_memory"

    @property
    def description(self) -> str:
        return "Search for similar translations in translation memory"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "text": {
                    "type": "string",
                    "description": "The text to search for"
                },
                "threshold": {
                    "type": "number",
                    "description": "Similarity threshold (0-1)"
                }
            },
            "required": ["text"]
        }

    def execute(self, **kwargs) -> ToolResult:
        text = kwargs.get("text", "")
        threshold = kwargs.get("threshold", 0.8)

        # 模拟翻译记忆库
        mock_memory = [
            {"source": "Hello, how are you?", "target": "你好，你好吗？", "similarity": 0.95},
            {"source": "Welcome to our company", "target": "欢迎来到我们公司", "similarity": 0.9},
            {"source": "Thank you for your help", "target": "谢谢你的帮助", "similarity": 0.85},
            {"source": "Please contact us", "target": "请联系我们", "similarity": 0.8},
        ]

        # 简单模拟相似度匹配（实际应用应使用向量相似度）
        matches = []
        for entry in mock_memory:
            # 简单关键词匹配模拟
            words = set(text.lower().split())
            source_words = set(entry["source"].lower().split())
            overlap = len(words & source_words) / max(len(words), len(source_words))
            if overlap >= threshold:
                matches.append({
                    "source": entry["source"],
                    "target": entry["target"],
                    "similarity": round(overlap, 2)
                })

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "matches": matches,
                "count": len(matches),
            }
        )


# ==================== Skill 实现 ====================

class TranslationSkill(Skill):
    """翻译助手技能"""

    @property
    def name(self) -> str:
        return "translation"

    @property
    def description(self) -> str:
        return "多语言翻译助手，支持中英日韩等语言互译"

    @property
    def system_prompt(self) -> str:
        return """
你是一位专业的翻译专家，精通多国语言，包括中文、英文、日文、韩文等。

【核心职责】
1. **准确翻译**：保持原文意思和语气的准确翻译
2. **地道表达**：使用目标语言的自然表达方式
3. **术语一致**：保持专业术语的一致性
4. **文化适应**：考虑文化差异，进行本地化调整

【翻译原则】
1. 保持原文的语气和风格（正式/非正式、书面/口语）
2. 使用地道的表达方式，避免直译
3. 对于专业术语，提供原文对照
4. 如有歧义，提供多种翻译选项并说明
5. 对于文化特定的表达，添加必要的注释

【输出格式】
请按照以下格式输出翻译结果：

---
**原文**：[原文]
**译文**：[译文]

**注释**：[如有必要的解释、文化背景或替代翻译]
---

【可用工具】
- lookup_dictionary: 查询单词释义和例句
- lookup_terminology: 查询专业术语
- search_translation_memory: 查找相似翻译

【注意事项】
- 不要逐字翻译，要传达整体意思
- 对于习语、谚语，寻找对应的习惯表达
- 对于无法翻译的概念，保留原文并解释
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 中文触发词
            "翻译", "中译英", "英译中", "翻译成",
            "日语", "韩语", "中文", "英文",
            # 英文触发词
            "translate", "translation",
            "chinese to english", "english to chinese",
            # 日语触发词
            "翻訳", "日本語",
            # 韩语触发词
            "번역", "한국어",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            DictionaryTool(),
            TerminologyTool(),
            TranslationMemoryTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[Translation] 翻译技能已激活")
        # 尝试检测语言对
        query = context.query
        if any(c in query for c in "中英"):
            print(f"[Translation] 检测到中文相关请求")
        elif any(c in query for c in "英日韓"):
            print(f"[Translation] 检测到大语言相关请求")

    def on_deactivate(self) -> None:
        print(f"[Translation] 翻译技能已停用")
```

### 参考输出示例

```python
# 测试翻译 Skill
from src.skills import TranslationSkill

skill = TranslationSkill()
print(f"技能名称：{skill.name}")
print(f"技能描述：{skill.description}")
print(f"触发词数量：{len(skill.triggers)}")
print(f"触发词示例：{skill.triggers[:5]}")
print(f"提供的工具：{[t.name for t in skill.get_tools()]}")
```

**参考输出**：
```
技能名称：translation
技能描述：多语言翻译助手，支持中英日韩等语言互译
触发词数量：15
触发词示例：['翻译', '中译英', '英译中', '翻译成', '日语']
提供的工具：['lookup_dictionary', 'lookup_terminology', 'search_translation_memory']
```

### 工具执行示例

```python
# 测试词典工具
dict_tool = DictionaryTool()
result = dict_tool.execute(word="hello", source_lang="en", target_lang="zh")
print("词典查询结果：")
print(result.to_string())

# 测试术语工具
term_tool = TerminologyTool()
result = term_tool.execute(term="API", domain="tech")
print("\n术语查询结果：")
print(result.to_string())
```

**参考输出**：
```
词典查询结果：
{
  "word": "hello",
  "translation": "你好",
  "pos": "interjection",
  "definition": "Used as a greeting or to attract attention",
  "example": "Hello, how are you?"
}

术语查询结果：
{
  "term": "API",
  "translation": "应用程序接口",
  "domain": "tech",
  "note": "Application Programming Interface"
}
```

---

## 4.3 实战 3：数据分析 Skill（完整实现）

### 需求分析

数据分析 Skill 旨在帮助用户进行数据探索和统计分析：
1. **数据加载**：支持多种数据格式（CSV、Excel、JSON 等）
2. **统计分析**：描述性统计、相关性分析等
3. **可视化建议**：根据数据特点推荐合适的图表类型
4. **报告生成**：生成结构化的数据分析报告

### 工具设计

```
┌─────────────────────────────────────────────────────────────┐
│             DataAnalysisSkill 工具集                         │
│                                                             │
│   ┌─────────────────┐                                       │
│   │ DataLoaderTool  │  数据加载工具                          │
│   │                 │  - CSV/Excel/JSON 读取                 │
│   │                 │  - 数据格式检测                       │
│   │                 │  - 数据预览                           │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │ StatsTool       │  统计分析工具                          │
│   │                 │  - 描述性统计                         │
│   │                 │  - 相关性分析                         │
│   │                 │  - 分布分析                           │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │ChartSuggestionTool│ 图表推荐工具                         │
│   │                 │  - 根据数据类型推荐图表               │
│   │                 │  - 根据分析目的推荐图表               │
│   │                 │  - 图表配置建议                       │
│   └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### 完整代码实现

```python
"""
数据分析 Skill 完整实现

用法：
    from src.skills import DataAnalysisSkill
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Any, Optional
import json


# ==================== 工具实现 ====================

class DataLoaderTool(BaseTool):
    """数据加载工具"""

    @property
    def name(self) -> str:
        return "load_data"

    @property
    def description(self) -> str:
        return "Load and preview data from various formats (CSV, Excel, JSON)"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "file_path": {
                    "type": "string",
                    "description": "Path to the data file"
                },
                "format": {
                    "type": "string",
                    "description": "File format: csv, excel, json",
                    "enum": ["csv", "excel", "json"]
                },
                "preview_rows": {
                    "type": "integer",
                    "description": "Number of rows to preview (default: 5)"
                }
            },
            "required": ["file_path"]
        }

    def execute(self, **kwargs) -> ToolResult:
        file_path = kwargs.get("file_path", "")
        format_type = kwargs.get("format", "csv")
        preview_rows = kwargs.get("preview_rows", 5)

        # 模拟数据加载（实际应用中使用 pandas 等库）
        # 这里提供模拟数据用于演示

        mock_data = {
            "columns": ["date", "product", "sales", "revenue"],
            "data_types": {
                "date": "datetime",
                "product": "string",
                "sales": "integer",
                "revenue": "float"
            },
            "preview": [
                {"date": "2024-01-01", "product": "Product A", "sales": 100, "revenue": 1500.50},
                {"date": "2024-01-02", "product": "Product B", "sales": 150, "revenue": 2250.75},
                {"date": "2024-01-03", "product": "Product A", "sales": 120, "revenue": 1800.60},
                {"date": "2024-01-04", "product": "Product C", "sales": 80, "revenue": 1200.00},
                {"date": "2024-01-05", "product": "Product B", "sales": 200, "revenue": 3000.00},
            ],
            "shape": {"rows": 1000, "cols": 4},
            "missing_values": {
                "date": 0,
                "product": 0,
                "sales": 5,
                "revenue": 2
            }
        }

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "file_path": file_path,
                "format": format_type,
                **mock_data
            }
        )


class StatsTool(BaseTool):
    """统计分析工具"""

    @property
    def name(self) -> str:
        return "calculate_stats"

    @property
    def description(self) -> str:
        return "Calculate descriptive statistics for numerical columns"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "column": {
                    "type": "string",
                    "description": "Column name to analyze"
                },
                "statistics": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Statistics to calculate: mean, median, std, min, max, etc."
                }
            },
            "required": ["column"]
        }

    def execute(self, **kwargs) -> ToolResult:
        column = kwargs.get("column", "")
        statistics = kwargs.get("statistics", ["mean", "median", "std", "min", "max"])

        # 模拟统计结果
        mock_stats = {
            "sales": {
                "count": 1000,
                "mean": 145.6,
                "median": 140.0,
                "std": 45.2,
                "min": 50,
                "max": 350,
                "q1": 110,
                "q3": 180
            },
            "revenue": {
                "count": 998,  # 2 missing values
                "mean": 2150.75,
                "median": 2100.00,
                "std": 650.30,
                "min": 750.00,
                "max": 5250.00,
                "q1": 1650.00,
                "q3": 2700.00
            }
        }

        if column in mock_stats:
            stats = mock_stats[column]
            # 只返回请求的统计量
            result = {k: v for k, v in stats.items() if k in statistics or k == "count"}
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output={
                    "column": column,
                    "statistics": result
                }
            )

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "column": column,
                "statistics": {"error": f"Column '{column}' not found or not numerical"}
            }
        )


class ChartSuggestionTool(BaseTool):
    """图表推荐工具"""

    @property
    def name(self) -> str:
        return "suggest_chart"

    @property
    def description(self) -> str:
        return "Suggest appropriate chart types based on data characteristics"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "data_types": {
                    "type": "object",
                    "description": "Dictionary of column names to data types"
                },
                "analysis_goal": {
                    "type": "string",
                    "description": "What do you want to show? (trend, comparison, distribution, relationship)"
                }
            },
            "required": ["data_types"]
        }

    def execute(self, **kwargs) -> ToolResult:
        data_types = kwargs.get("data_types", {})
        analysis_goal = kwargs.get("analysis_goal", "exploration")

        suggestions = []

        # 根据数据类型推荐
        has_datetime = any("datetime" in str(v) for v in data_types.values())
        has_numerical = any(t in ["integer", "float", "numerical"] for t in data_types.values())
        has_categorical = any(t in ["string", "categorical"] for t in data_types.values())

        if has_datetime and has_numerical:
            suggestions.append({
                "chart_type": "line_chart",
                "description": "折线图 - 展示随时间变化的趋势",
                "use_case": "适合展示时间序列数据",
                "config": {"x_axis": "datetime column", "y_axis": "numerical column"}
            })

        if has_categorical and has_numerical:
            suggestions.append({
                "chart_type": "bar_chart",
                "description": "柱状图 - 比较不同类别的数值",
                "use_case": "适合分类比较",
                "config": {"x_axis": "categorical column", "y_axis": "numerical column"}
            })

            suggestions.append({
                "chart_type": "box_plot",
                "description": "箱线图 - 展示数据分布和异常值",
                "use_case": "适合分析分布特征",
                "config": {"x_axis": "categorical column", "y_axis": "numerical column"}
            })

        if has_numerical:
            suggestions.append({
                "chart_type": "histogram",
                "description": "直方图 - 展示数值分布",
                "use_case": "适合分析单个数值变量的分布",
                "config": {"x_axis": "numerical column", "bins": "auto"}
            })

            suggestions.append({
                "chart_type": "scatter_plot",
                "description": "散点图 - 展示两个数值变量的关系",
                "use_case": "适合分析变量相关性",
                "config": {"x_axis": "numerical column 1", "y_axis": "numerical column 2"}
            })

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "suggestions": suggestions,
                "analysis_goal": analysis_goal,
            }
        )


# ==================== Skill 实现 ====================

class DataAnalysisSkill(Skill):
    """数据分析技能"""

    @property
    def name(self) -> str:
        return "data_analysis"

    @property
    def description(self) -> str:
        return "分析数据、生成统计报告、提供可视化建议"

    @property
    def system_prompt(self) -> str:
        return """
你是一位专业的数据分析师，擅长从数据中提取有价值的洞察。

【核心职责】
1. **数据探索**：理解数据结构、识别数据质量問題
2. **统计分析**：计算描述性统计、识别模式
3. **可视化建议**：根据数据特点推荐合适的图表
4. **报告生成**：生成清晰、结构化的分析报告

【分析流程】
1. 首先使用 load_data 加载和预览数据
2. 检查数据质量（缺失值、异常值）
3. 使用 calculate_stats 计算关键统计量
4. 使用 suggest_chart 获取可视化建议
5. 综合所有信息，生成分析报告

【输出格式】
请按以下格式输出分析报告：

## 数据分析报告

### 1. 数据概览
- 数据来源：[文件/表名]
- 数据规模：[行数 × 列数]
- 时间范围：[如有]

### 2. 数据质量
- 缺失值情况
- 异常值检测

### 3. 描述性统计
[关键数值变量的统计摘要]

### 4. 主要发现
- 发现 1
- 发现 2
- ...

### 5. 可视化建议
[推荐的图表类型和理由]

### 6. 后续建议
[进一步分析的建议]

【注意事项】
- 用清晰、简洁的语言表达
- 突出重要的洞察和发现
- 提供具体的数据支持
- 避免过度解读
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 中文触发词
            "数据分析", "分析数据", "数据统计",
            "数据报告", "数据探索",
            # 英文触发词
            "analyze data", "data analysis", "statistics",
            "explore data", "data report",
            # 相关术语
            "csv", "excel", "dataset", "图表", "可视化",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            DataLoaderTool(),
            StatsTool(),
            ChartSuggestionTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[DataAnalysis] 数据分析技能已激活")

    def on_deactivate(self) -> None:
        print(f"[DataAnalysis] 数据分析技能已停用")
```

### 参考输出示例

```python
# 测试数据分析 Skill
from src.skills import DataAnalysisSkill

skill = DataAnalysisSkill()
print(f"技能名称：{skill.name}")
print(f"提供的工具：{[t.name for t in skill.get_tools()]}")
print(f"触发词：{skill.triggers[:8]}")
```

**参考输出**：
```
技能名称：data_analysis
提供的工具：['load_data', 'calculate_stats', 'suggest_chart']
触发词：['数据分析', '分析数据', '数据统计', '数据报告', '数据探索', 'analyze data', 'data analysis', 'statistics']
```

---

## 4.4 实战 4：Web 搜索 Skill（高级完整实现）

### 需求分析

Web 搜索 Skill 是一个实用的高级技能，需要：
1. **搜索引擎调用**：接入真实搜索 API（如 Bing、Google）
2. **结果聚合**：整合多个来源的搜索结果
3. **摘要生成**：对搜索结果进行摘要和排序
4. **引用管理**：正确标注信息来源

### 工具设计

```
┌─────────────────────────────────────────────────────────────┐
│              WebSearchSkill 工具集                           │
│                                                             │
│   ┌─────────────────┐                                       │
│   │SearchEngineTool │  搜索引擎调用                          │
│   │                 │  - API 调用                            │
│   │                 │  - 结果获取                           │
│   │                 │  - 分页处理                           │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │URLFetcherTool   │  网页内容获取                          │
│   │                 │  - HTML 抓取                           │
│   │                 │  - 内容提取                           │
│   │                 │  - 文本清理                           │
│   └─────────────────┘                                       │
│                                                             │
│   ┌─────────────────┐                                       │
│   │ContentSummarizerTool│ 内容摘要                           │
│   │                 │  - 关键信息提取                       │
│   │                 │  - 摘要生成                           │
│   │                 │  - 相关性评分                         │
│   └─────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

### 完整代码实现

```python
"""
Web 搜索 Skill 完整实现

注意：实际使用需要配置搜索 API 密钥
- Bing Search API: https://www.microsoft.com/en-us/bing/apis/bing-web-search-api
- Google Custom Search API: https://developers.google.com/custom-search/v1/introduction
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Optional
import os


# ==================== 工具实现 ====================

class SearchEngineTool(BaseTool):
    """搜索引擎调用工具"""

    @property
    def name(self) -> str:
        return "search_web"

    @property
    def description(self) -> str:
        return "Search the web for information using search engines"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query"
                },
                "num_results": {
                    "type": "integer",
                    "description": "Number of results to return (default: 10)"
                },
                "language": {
                    "type": "string",
                    "description": "Result language (zh-CN, en-US, etc.)"
                }
            },
            "required": ["query"]
        }

    def execute(self, **kwargs) -> ToolResult:
        query = kwargs.get("query", "")
        num_results = kwargs.get("num_results", 10)
        language = kwargs.get("language", "zh-CN")

        # 检查 API 密钥
        api_key = os.environ.get("BING_SEARCH_API_KEY")

        if not api_key:
            # 无 API 密钥时返回模拟结果
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output={
                    "query": query,
                    "results": self._get_mock_results(query, num_results),
                    "note": "Using mock results (no API key configured)"
                }
            )

        # 实际调用 Bing Search API（示例代码）
        # import requests
        # headers = {"Ocp-Apim-Subscription-Key": api_key}
        # response = requests.get(
        #     "https://api.bing.microsoft.com/v7.0/search",
        #     params={"q": query, "count": num_results, "mkt": language},
        #     headers=headers
        # )
        # return ToolResult(status=ToolResultStatus.SUCCESS, output=response.json())

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "query": query,
                "results": self._get_mock_results(query, num_results),
            }
        )

    def _get_mock_results(self, query: str, num_results: int) -> List[Dict]:
        """生成模拟搜索结果"""
        return [
            {
                "title": f"关于「{query}」的搜索结果 {i+1}",
                "url": f"https://example.com/result-{i+1}",
                "snippet": f"这是关于「{query}」的相关介绍和信息...",
                "source": "示例网站",
            }
            for i in range(min(num_results, 5))  # 模拟返回 5 条结果
        ]


class URLFetcherTool(BaseTool):
    """网页内容获取工具"""

    @property
    def name(self) -> str:
        return "fetch_url"

    @property
    def description(self) -> str:
        return "Fetch and extract content from a URL"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "url": {
                    "type": "string",
                    "description": "URL to fetch"
                },
                "extract_type": {
                    "type": "string",
                    "description": "Type of content to extract: text, links, images",
                    "enum": ["text", "links", "images"]
                }
            },
            "required": ["url"]
        }

    def execute(self, **kwargs) -> ToolResult:
        url = kwargs.get("url", "")
        extract_type = kwargs.get("extract_type", "text")

        # 模拟网页抓取
        mock_content = {
            "url": url,
            "title": "示例网页标题",
            "content": "这是从网页中提取的主要内容。包含关于该主题的详细信息...",
            "word_count": 1500,
            "extract_type": extract_type,
        }

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=mock_content
        )


class ContentSummarizerTool(BaseTool):
    """内容摘要工具"""

    @property
    def name(self) -> str:
        return "summarize_content"

    @property
    def description(self) -> str:
        return "Generate a concise summary from content"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "content": {
                    "type": "string",
                    "description": "Content to summarize"
                },
                "max_length": {
                    "type": "integer",
                    "description": "Maximum summary length (default: 200)"
                },
                "focus": {
                    "type": "string",
                    "description": "Focus area for summarization"
                }
            },
            "required": ["content"]
        }

    def execute(self, **kwargs) -> ToolResult:
        content = kwargs.get("content", "")
        max_length = kwargs.get("max_length", 200)
        focus = kwargs.get("focus", "")

        # 简单摘要（实际应用应使用文本摘要模型）
        sentences = content.split("。")
        summary_sentences = sentences[:3]
        summary = "。".join(summary_sentences) + "。"

        if len(summary) > max_length:
            summary = summary[:max_length] + "..."

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "summary": summary,
                "original_length": len(content),
                "summary_length": len(summary),
                "focus": focus,
            }
        )


# ==================== Skill 实现 ====================

class WebSearchSkill(Skill):
    """Web 搜索技能"""

    @property
    def name(self) -> str:
        return "web_search"

    @property
    def description(self) -> str:
        return "搜索互联网信息，获取最新资讯和知识"

    @property
    def system_prompt(self) -> str:
        return """
你是一位专业的信息检索专家，擅长从互联网上查找和整合信息。

【核心职责】
1. **精准搜索**：理解用户需求，构建有效的搜索查询
2. **结果筛选**：从搜索结果中筛选高质量信息
3. **信息整合**：整合多个来源的信息
4. **引用标注**：正确标注信息来源

【搜索流程】
1. 理解用户需求，确定搜索目标
2. 使用 search_web 进行搜索
3. 如有需要，使用 fetch_url 获取详细网页内容
4. 使用 summarize_content 生成摘要
5. 整合信息，生成最终回答

【输出格式】
请按照以下格式输出搜索结果：

## 搜索结果

**查询**：[搜索关键词]

### 主要发现
[整合后的主要信息]

### 参考来源
1. [来源 1 标题](URL) - 来源说明
2. [来源 2 标题](URL) - 来源说明
3. ...

### 相关信息
[其他可能有用的信息]

【注意事项】
- 优先使用权威来源
- 注意信息的时效性
- 标注信息的不确定性
- 提醒用户核实重要信息
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 中文触发词
            "搜索", "查找", "查询", "网上找",
            "最新资讯", "最新消息", "新闻动态",
            # 英文触发词
            "search", "find information", "look up",
            "latest news", "recent",
            # 疑问词（暗示需要搜索）
            "什么是", "谁是", "哪里可以", "如何",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            SearchEngineTool(),
            URLFetcherTool(),
            ContentSummarizerTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[WebSearch] Web 搜索技能已激活")

    def on_deactivate(self) -> None:
        print(f"[WebSearch] Web 搜索技能已停用")
```

---

至此，4 个完整的实战案例已经完成。接下来是第 5 部分 Skill 与 Agent 集成、第 6 部分高级话题、第 7 部分前沿技能设计等内容：

## 5. Skill 与 Agent 集成

### 5.1 集成模式

将 Skills 系统与 Agent 集成有三种主要模式：

```
┌─────────────────────────────────────────────────────────────┐
│                 Skill-Agent 集成模式                        │
│                                                             │
│  模式 1：外部 SkillManager + 手动注入提示词                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │ SkillManager│───▶│ Agent       │    │ 用户        │     │
│  │             │    │             │◀───│             │     │
│  │ - detect    │    │ - run       │    │             │     │
│  │ - activate  │    │             │    │             │     │
│  │ - prompt    │    │             │    │             │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  模式 2：继承 SkilledReActAgent 内置管理                    │
│  ┌─────────────────────────────────┐                       │
│  │      SkilledReActAgent          │                       │
│  │  ┌───────────┐  ┌───────────┐  │                       │
│  │  │   Skill   │  │   Agent   │  │                       │
│  │  │  Manager  │  │  Engine   │  │                       │
│  │  └───────────┘  └───────────┘  │                       │
│  └─────────────────────────────────┘                       │
│                                                             │
│  模式 3：动态技能加载（插件式）                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  Plugin Dir │───▶│ SkillLoader │───▶│    Agent    │     │
│  │  /skills/   │    │             │    │             │     │
│  │  skill.py   │    │             │    │             │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 模式 1：外部 SkillManager + 手动注入

这是最简单直接的集成方式，适合初学者和简单场景。

```python
"""
模式 1：外部 SkillManager 集成示例

特点：
- 代码简单，易于理解
- SkillManager 和 Agent 解耦
- 适合一次性对话场景
"""

from src.agent import ReActAgent, AgentConfig
from src.skills import SkillManager, CalculatorSkill, SearchSkill
from src.tools import ToolRegistry
from src.llm import MockClient  # 或使用 BailianClient


def create_skilled_agent():
    """创建带技能的 Agent"""
    # 1. 创建 SkillManager 并注册技能
    skill_manager = SkillManager()
    skill_manager.register_skill(CalculatorSkill())
    skill_manager.register_skill(SearchSkill())

    # 2. 创建工具注册表
    tools = ToolRegistry()

    # 3. 将所有技能的工具注册到 Agent
    for skill in skill_manager:
        for tool in skill.get_tools():
            tools.register(tool)

    # 4. 创建 Agent
    config = AgentConfig(name="SkilledAgent")
    agent = ReActAgent(
        llm_client=MockClient(),  # 测试时使用 MockClient
        config=config,
        tools=tools,
    )

    return agent, skill_manager


def run_with_skills(agent, skill_manager, query):
    """
    使用技能运行 Agent

    流程：
    1. 检测相关技能
    2. 激活技能并获取提示词
    3. 将提示词注入 Agent
    4. 运行 Agent
    5. 清理技能状态
    """
    # 步骤 1：检测相关技能
    detected_skills = skill_manager.detect_skills(query)

    if detected_skills:
        print(f"检测到技能：{[s.name for s in detected_skills]}")

        # 步骤 2：激活技能
        for skill in detected_skills:
            skill_manager.activate_skill(skill.name)

        # 步骤 3：获取组合提示词并注入
        skill_prompt = skill_manager.get_combined_prompt()
        if skill_prompt:
            # 将技能提示词添加到系统消息
            agent.memory.add_message("system", skill_prompt)
            print(f"已注入技能提示词（{len(skill_prompt)} 字符）")

    # 步骤 4：运行 Agent
    result = agent.run(query)

    # 步骤 5：清理技能状态
    skill_manager.clear_active()

    return result


# 使用示例
if __name__ == "__main__":
    agent, manager = create_skilled_agent()

    # 测试 1：数学计算
    print("\n=== 测试 1：数学计算 ===")
    result = run_with_skills(agent, manager, "计算 15 * 7 + 23")
    print(f"结果：{result}")

    # 测试 2：搜索请求
    print("\n=== 测试 2：搜索请求 ===")
    result = run_with_skills(agent, manager, "搜索 Python 教程")
    print(f"结果：{result}")
```

### 5.3 模式 2：内置 SkillManager 的 Agent

创建一个新的 Agent 类，将 SkillManager 内置其中。

```python
"""
模式 2：SkilledReActAgent - 内置技能管理的 Agent

特点：
- 技能管理内置于 Agent
- API 更简洁
- 适合需要频繁使用技能的场景
"""

from typing import Optional, List
from src.agent import ReActAgent, AgentConfig
from src.skills import SkillManager, SkillContext
from src.tools import ToolRegistry


class SkilledReActAgent(ReActAgent):
    """
    内置技能管理的 ReAct Agent

    扩展了 ReActAgent，添加了 SkillManager 功能：
    - 自动技能检测和激活
    - 技能提示词自动注入
    - 技能工具自动注册
    """

    def __init__(
        self,
        llm_client,
        config: Optional[AgentConfig] = None,
        tools: Optional[ToolRegistry] = None,
    ):
        """
        初始化 SkilledReActAgent

        Args:
            llm_client: LLM 客户端
            config: Agent 配置
            tools: 工具注册表（可选，会合并技能的工具）
        """
        super().__init__(llm_client=llm_client, config=config, tools=tools or ToolRegistry())

        # 内置 SkillManager
        self.skill_manager = SkillManager()

        # 用于临时存储技能提示词
        self._skill_system_prompt: Optional[str] = None

    def register_skill(self, skill) -> None:
        """
        注册一个技能

        注册技能后会：
        1. 将技能添加到 SkillManager
        2. 将技能的工具添加到 Agent 的工具注册表
        """
        self.skill_manager.register_skill(skill)

        # 注册技能提供的工具
        for tool in skill.get_tools():
            self.tools.register(tool)

        print(f"已注册技能：{skill.name}")

    def run(self, query: str, use_skills: bool = True, **kwargs):
        """
        运行 Agent（带技能支持）

        Args:
            query: 用户查询
            use_skills: 是否使用技能系统（默认 True）
            **kwargs: 传递给父类的其他参数

        Returns:
            Agent 运行结果
        """
        if use_skills:
            # 步骤 1：检测相关技能
            detected = self.skill_manager.detect_skills(query)

            if detected:
                print(f"[Skill] 检测到技能：{[s.name for s in detected]}")

                # 步骤 2：激活技能
                context = SkillContext(query=query, agent=self)
                for skill in detected:
                    self.skill_manager.activate_skill(skill.name, context)

                # 步骤 3：获取并注入技能提示词
                skill_prompt = self.skill_manager.get_combined_prompt()
                if skill_prompt:
                    self._skill_system_prompt = skill_prompt
                    # 临时添加到系统消息
                    self.memory.add_message("system", skill_prompt)
                    print(f"[Skill] 已注入提示词（{len(skill_prompt)} 字符）")

        # 步骤 4：运行 Agent
        result = super().run(query, **kwargs)

        # 步骤 5：清理技能状态
        if use_skills:
            self.skill_manager.clear_active()
            # 移除之前添加的技能提示词（如果有）
            if self._skill_system_prompt:
                self._remove_skill_prompt()
                self._skill_system_prompt = None

        return result

    def _remove_skill_prompt(self):
        """移除技能提示词"""
        # 简单实现：遍历消息，移除包含技能标记的系统消息
        messages = self.memory.get_messages()
        skill_markers = ["[calculator]", "[search]", "[code_review]", "[translation]"]

        for msg in messages:
            if msg.get("role") == "system":
                content = msg.get("content", "")
                if any(marker in content for marker in skill_markers):
                    # 这里可以选择移除或保留
                    pass  # 为简单起见，保留消息

    def get_active_skills(self) -> List[str]:
        """获取当前激活的技能名称列表"""
        return [skill.name for skill in self.skill_manager.get_active_skills()]

    def list_registered_skills(self) -> List[str]:
        """获取所有已注册的技能名称"""
        return self.skill_manager.list_skills()
```

### 5.4 使用示例

```python
# SkilledReActAgent 使用示例

from src.llm import MockClient
from src.skills import CalculatorSkill, SearchSkill

# 创建 Agent
agent = SkilledReActAgent(llm_client=MockClient())

# 注册技能
agent.register_skill(CalculatorSkill())
agent.register_skill(SearchSkill())

# 查看已注册的技能
print(f"已注册技能：{agent.list_registered_skills()}")

# 运行查询（自动使用技能）
result = agent.run("计算 123 + 456")
print(f"结果：{result}")

# 运行查询（不使用技能）
result = agent.run("你好", use_skills=False)
print(f"结果：{result}")
```

**参考输出**：
```
已注册技能：['calculator', 'search']
[Skill] 检测到技能：['calculator']
[Skill] 已注入提示词（156 字符）
=== Agent 思考过程 ===
Thought: 用户需要进行数学计算，我应该使用计算器工具。
Action: calculate_tool(expression="123 + 456")
Observation: 579
Thought: 我已经得到了计算结果。
Final Answer: 123 + 456 = 579
结果：123 + 456 = 579
```

---

## 5.5 调试技巧

### 日志配置

```python
import logging

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# 单独控制 Skill 相关日志
logging.getLogger('src.skills').setLevel(logging.DEBUG)
logging.getLogger('src.agent').setLevel(logging.INFO)
```

### 技能激活追踪

```python
# 添加技能激活的日志输出
class DebugSkillManager(SkillManager):
    def detect_skills(self, query: str) -> List[Skill]:
        detected = super().detect_skills(query)
        print(f"[DEBUG] 查询：'{query}'")
        print(f"[DEBUG] 检测到技能：{[s.name for s in detected]}")
        return detected

    def activate_skill(self, name: str, context: Optional[SkillContext] = None) -> bool:
        result = super().activate_skill(name, context)
        if result:
            print(f"[DEBUG] 技能已激活：{name}")
        return result
```

### 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|---------|---------|
| 技能不激活 | 触发词不匹配 | 检查 triggers 列表，确保包含查询中的关键词 |
| 工具未注册 | get_tools 返回空列表 | 检查技能的 get_tools 方法实现 |
| 提示词未注入 | get_combined_prompt 返回空 | 检查 skill.system_prompt 是否有值 |
| 多个技能冲突 | 提示词相互矛盾 | 调整提示词或修改技能优先级 |

---

# 6. 高级话题

## 6.1 技能冲突与优先级

### 当多个技能响应同一查询时的处理

```python
# 示例：查询"搜索如何计算房贷利率"
# 可能同时触发 CalculatorSkill 和 SearchSkill

query = "搜索如何计算房贷利率"
detected = manager.detect_skills(query)
print(f"检测到的技能：{[s.name for s in detected]}")
# 输出：['calculator', 'search']
```

### 优先级设计原则

```python
# 注册时指定优先级
manager.register_skill(CalculatorSkill(), priority=1)   # 高优先级
manager.register_skill(SearchSkill(), priority=0)       # 普通优先级
manager.register_skill(WeatherSkill(), priority=-1)     # 低优先级

# 优先级生效：高优先级的技能排在前面
detected = manager.detect_skills("计算并搜索")
# detected 顺序：[CalculatorSkill, SearchSkill]
```

### 互斥技能的处理

```python
class SkillManager:
    # 添加互斥组配置
    def __init__(self):
        super().__init__()
        self._mutually_exclusive_groups = [
            {"translation", "code_review"},  # 这两个技能不能同时激活
        ]

    def activate_skill(self, name: str, context: Optional[SkillContext] = None) -> bool:
        # 检查互斥
        for group in self._mutually_exclusive_groups:
            if name in group:
                # 停用同组的其他技能
                for other_name in group:
                    if other_name != name and other_name in self._active_skills:
                        self.deactivate_skill(other_name)

        return super().activate_skill(name, context)
```

---

## 6.2 技能组合与编排

### 顺序组合：先翻译后审查

```python
def translate_then_review(agent, skill_manager, code: str, target_lang: str = "en"):
    """
    先将代码注释翻译为目标语言，然后进行代码审查
    """
    # 步骤 1：激活翻译技能
    skill_manager.activate_skill("translation")

    # 步骤 2：翻译代码注释
    translate_query = f"将以下代码中的注释翻译为{target_lang}：\n{code}"
    # ... 执行翻译

    # 步骤 3：停用翻译，激活审查技能
    skill_manager.clear_active()
    skill_manager.activate_skill("code_review")

    # 步骤 4：审查代码
    # ... 执行审查
```

### 并行组合：同时调用多个技能

```python
def parallel_analysis(agent, skill_manager, query: str):
    """
    同时使用多个技能分析同一个查询
    """
    # 激活所有相关技能
    skills = skill_manager.detect_skills(query)

    for skill in skills:
        skill_manager.activate_skill(skill.name)

    # 获取组合提示词
    combined_prompt = skill_manager.get_combined_prompt(skills)

    # 使用组合提示词运行 Agent
    # ...
```

### 条件组合：根据上下文选择技能

```python
def context_aware_execution(agent, skill_manager, query: str, context: dict):
    """
    根据上下文选择最合适的技能
    """
    skills = skill_manager.detect_skills(query)

    # 根据上下文过滤
    if context.get("mode") == "quick":
        # 快速模式：只使用第一个（最高优先级）技能
        skills = skills[:1]
    elif context.get("mode") == "thorough":
        # 全面模式：使用所有相关技能
        pass

    # 激活选中的技能
    for skill in skills:
        skill_manager.activate_skill(skill.name)
```

---

## 6.3 动态技能加载（插件系统）

### 从文件系统自动发现技能

```python
"""
动态技能加载器

从指定目录自动发现和加载技能插件
"""

import importlib.util
import os
from pathlib import Path
from typing import List
from src.skills import Skill, SkillManager


class PluginLoader:
    """插件加载器"""

    def __init__(self, plugin_dir: str):
        self.plugin_dir = Path(plugin_dir)

    def discover_skills(self) -> List[type]:
        """
        从插件目录发现所有 Skill 类
        """
        skill_classes = []

        for file_path in self.plugin_dir.glob("*.py"):
            if file_path.name.startswith("_"):
                continue  # 跳过特殊文件

            # 动态导入模块
            spec = importlib.util.spec_from_file_location(
                file_path.stem, file_path
            )
            module = importlib.util.module_from_spec(spec)
            spec.loader.exec_module(module)

            # 查找模块中的 Skill 子类
            for name in dir(module):
                obj = getattr(module, name)
                if (
                    isinstance(obj, type) and
                    issubclass(obj, Skill) and
                    obj is not Skill
                ):
                    skill_classes.append(obj)

        return skill_classes

    def load_all_skills(self, manager: SkillManager) -> int:
        """
        加载所有发现的技能到管理器

        Returns:
            加载的技能数量
        """
        skill_classes = self.discover_skills()

        for skill_class in skill_classes:
            skill = skill_class()
            manager.register_skill(skill)

        return len(skill_classes)


# 使用示例
if __name__ == "__main__":
    # 假设 skills/plugins/ 目录下有自定义技能文件
    loader = PluginLoader("skills/plugins")

    manager = SkillManager()
    count = loader.load_all_skills(manager)

    print(f"已加载 {count} 个技能插件")
    print(f"技能列表：{manager.list_skills()}")
```

### 插件目录结构示例

```
skills/
├── plugins/
│   ├── __init__.py
│   ├── weather_skill.py      # 天气技能
│   ├── news_skill.py         # 新闻技能
│   └── crypto_skill.py       # 加密货币技能
└── loader.py                # 加载器
```

### 热加载/热卸载技能

```python
class HotReloadSkillManager(SkillManager):
    """支持热加载的 SkillManager"""

    def reload_plugin(self, skill_name: str, plugin_path: str) -> bool:
        """
        热重载技能插件
        """
        # 1. 卸载旧技能
        self.unregister_skill(skill_name)

        # 2. 清除模块缓存
        if skill_name in sys.modules:
            del sys.modules[skill_name]

        # 3. 重新加载
        spec = importlib.util.spec_from_file_location(skill_name, plugin_path)
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)

        # 4. 注册新技能
        for name in dir(module):
            obj = getattr(module, name)
            if isinstance(obj, type) and issubclass(obj, Skill) and obj is not Skill:
                self.register_skill(obj())
                return True

        return False
```

---

## 6.4 技能评估与优化

### 如何评估技能的表现？

```python
from dataclasses import dataclass
from typing import List
from datetime import datetime

@dataclass
class SkillMetrics:
    """技能性能指标"""
    name: str
    activation_count: int = 0          # 激活次数
    success_count: int = 0             # 成功次数
    failure_count: int = 0             # 失败次数
    total_execution_time: float = 0.0  # 总执行时间

    @property
    def success_rate(self) -> float:
        """成功率"""
        total = self.success_count + self.failure_count
        if total == 0:
            return 0.0
        return self.success_count / total

    @property
    def avg_execution_time(self) -> float:
        """平均执行时间"""
        if self.activation_count == 0:
            return 0.0
        return self.total_execution_time / self.activation_count


class SkillMonitor:
    """技能监控器"""

    def __init__(self):
        self._metrics: dict[str, SkillMetrics] = {}

    def record_activation(self, skill_name: str):
        """记录技能激活"""
        if skill_name not in self._metrics:
            self._metrics[skill_name] = SkillMetrics(name=skill_name)
        self._metrics[skill_name].activation_count += 1

    def record_success(self, skill_name: str, execution_time: float):
        """记录成功执行"""
        if skill_name not in self._metrics:
            self._metrics[skill_name] = SkillMetrics(name=skill_name)
        metrics = self._metrics[skill_name]
        metrics.success_count += 1
        metrics.total_execution_time += execution_time

    def record_failure(self, skill_name: str, execution_time: float):
        """记录失败执行"""
        if skill_name not in self._metrics:
            self._metrics[skill_name] = SkillMetrics(name=skill_name)
        metrics = self._metrics[skill_name]
        metrics.failure_count += 1
        metrics.total_execution_time += execution_time

    def get_report(self) -> str:
        """生成监控报告"""
        report = "## 技能性能报告\n\n"
        report += "| 技能 | 激活次数 | 成功率 | 平均耗时 |\n"
        report += "|------|---------|-------|----------|\n"

        for name, metrics in self._metrics.items():
            report += f"| {name} | {metrics.activation_count} | "
            report += f"{metrics.success_rate:.1%} | "
            report += f"{metrics.avg_execution_time:.2f}s |\n"

        return report
```

### A/B 测试不同提示词

```python
class ABTestSkill(Skill):
    """
    支持 A/B 测试的技能

    可以测试不同提示词版本的效果
    """

    def __init__(self, name: str, description: str, prompt_versions: List[str]):
        self._name = name
        self._description = description
        self._prompt_versions = prompt_versions
        self._version_stats = {i: {"used": 0, "success": 0} for i in range(len(prompt_versions))}
        self._current_version = 0

    @property
    def system_prompt(self) -> str:
        # 轮流使用不同版本的提示词
        version = self._current_version % len(self._prompt_versions)
        self._current_version += 1
        self._version_stats[version]["used"] += 1
        return self._prompt_versions[version]

    def record_feedback(self, version: int, is_success: bool):
        """记录用户反馈"""
        if is_success:
            self._version_stats[version]["success"] += 1

    def get_best_version(self) -> int:
        """获取表现最好的提示词版本"""
        best_version = 0
        best_rate = 0

        for version, stats in self._version_stats.items():
            if stats["used"] > 0:
                rate = stats["success"] / stats["used"]
                if rate > best_rate:
                    best_rate = rate
                    best_version = version

        return best_version
```

---

## 6.5 安全考虑

### 技能权限控制

```python
from enum import Enum
from typing import Set

class Permission(Enum):
    """权限类型"""
    READ = "read"              # 只读权限
    WRITE = "write"            # 写入权限
    EXECUTE = "execute"        # 执行权限
    NETWORK = "network"        # 网络访问权限
    SENSITIVE = "sensitive"    # 敏感操作权限


class SkillPermissions:
    """技能权限管理"""

    def __init__(self):
        self._skill_permissions: dict[str, Set[Permission]] = {}
        self._granted_permissions: Set[Permission] = set()

    def define_permission(self, skill_name: str, permissions: Set[Permission]):
        """定义技能所需权限"""
        self._skill_permissions[skill_name] = permissions

    def grant_permission(self, permission: Permission):
        """授予权限"""
        self._granted_permissions.add(permission)

    def can_execute(self, skill_name: str) -> bool:
        """检查技能是否可以执行"""
        required = self._skill_permissions.get(skill_name, set())
        return required.issubset(self._granted_permissions)

    def require_confirmation(self, skill_name: str) -> bool:
        """检查技能是否需要用户确认"""
        sensitive_skills = {"delete_file", "execute_code", "transfer_money"}
        return skill_name in sensitive_skills
```

### 敏感操作确认

```python
class SecureSkillManager(SkillManager):
    """安全的 SkillManager，支持敏感操作确认"""

    SENSITIVE_SKILLS = {"filesystem", "web_search", "code_executor"}

    def activate_skill(self, name: str, context: Optional[SkillContext] = None,
                       skip_confirmation: bool = False) -> bool:
        """
        激活技能（带安全确认）
        """
        # 检查是否需要确认
        if name in self.SENSITIVE_SKILLS and not skip_confirmation:
            # 这里应该暂停并等待用户确认
            print(f"[安全] 技能 '{name}' 是敏感操作，需要用户确认")
            # confirm = input("是否继续？(y/n): ")
            # if confirm.lower() != 'y':
            #     return False

        return super().activate_skill(name, context)
```

### 资源使用限制

```python
class ResourceLimit:
    """资源限制配置"""

    def __init__(
        self,
        max_executions_per_minute: int = 60,
        max_total_executions: int = 1000,
        max_memory_mb: int = 512,
    ):
        self.max_executions_per_minute = max_executions_per_minute
        self.max_total_executions = max_total_executions
        self.max_memory_mb = max_memory_mb


class LimitedSkillManager(SkillManager):
    """带资源限制的 SkillManager"""

    def __init__(self, limit: ResourceLimit = None):
        super().__init__()
        self.limit = limit or ResourceLimit()
        self._execution_times: list[datetime] = []
        self._total_executions = 0

    def activate_skill(self, name: str, context: Optional[SkillContext] = None) -> bool:
        # 检查总执行次数限制
        if self._total_executions >= self.limit.max_total_executions:
            print(f"[限制] 已达到最大执行次数：{self.limit.max_total_executions}")
            return False

        # 检查每分钟执行次数限制
        now = datetime.now()
        self._execution_times = [t for t in self._execution_times
                                  if (now - t).total_seconds() < 60]

        if len(self._execution_times) >= self.limit.max_executions_per_minute:
            print(f"[限制] 已达到每分钟最大执行次数：{self.limit.max_executions_per_minute}")
            return False

        self._execution_times.append(now)
        self._total_executions += 1

        return super().activate_skill(name, context)
```

---

# 7. 前沿技能设计

本章介绍 5 种能够显著提升 Agent 整体水平的前沿技能设计。这些技能超越了传统的工具调用，涉及 Agent 的元认知、自我反思、规划等高级能力。

## 7.1 反思技能（Reflection Skill）

### 设计目的

让 Agent 能够自我检查和改进输出质量。类似人类的"三思而后行"，反思技能使 Agent 能够：
- 在行动前评估计划的合理性
- 在执行后检查结果的正确性
- 识别潜在错误并提供修正建议

### 实现方法

添加自我评估步骤到 ReAct 循环中：

```
标准 ReAct 循环：
Thought → Action → Observation → Thought → ... → Answer

带反思的 ReAct 循环：
Thought → Action → Observation → [Reflection] → Thought → ... → [Final Reflection] → Answer
```

### 工具设计

```python
"""
反思技能（Reflection Skill）完整实现
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Any, Optional


class SelfEvalTool(BaseTool):
    """自我评估工具"""

    @property
    def name(self) -> str:
        return "self_evaluate"

    @property
    def description(self) -> str:
        return "Evaluate the quality and correctness of your own response"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "response": {
                    "type": "string",
                    "description": "The response to evaluate"
                },
                "criteria": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Evaluation criteria: accuracy, completeness, clarity, safety"
                }
            },
            "required": ["response"]
        }

    def execute(self, **kwargs) -> ToolResult:
        response = kwargs.get("response", "")
        criteria = kwargs.get("criteria", ["accuracy", "completeness", "clarity"])

        # 评估维度
        evaluation = {}

        for criterion in criteria:
            if criterion == "accuracy":
                # 检查事实性陈述是否有依据
                evaluation["accuracy"] = {
                    "score": "N/A (requires LLM judgment)",
                    "note": "Check if factual claims are supported"
                }
            elif criterion == "completeness":
                # 检查是否回答了所有问题
                evaluation["completeness"] = {
                    "score": "N/A (requires LLM judgment)",
                    "note": "Check if all parts of the query are addressed"
                }
            elif criterion == "clarity":
                # 检查表达是否清晰
                evaluation["clarity"] = {
                    "score": "N/A (requires LLM judgment)",
                    "note": "Check if the response is well-organized"
                }
            elif criterion == "safety":
                # 检查是否有安全问题
                sensitive_topics = ["暴力", "违法", "危险", "伤害"]
                has_sensitive = any(topic in response for topic in sensitive_topics)
                evaluation["safety"] = {
                    "passed": not has_sensitive,
                    "note": "No sensitive content detected" if not has_sensitive else "Review needed"
                }

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "evaluation": evaluation,
                "criteria_used": criteria,
            }
        )


class ErrorDetectionTool(BaseTool):
    """错误检测工具"""

    @property
    def name(self) -> str:
        return "detect_errors"

    @property
    def description(self) -> str:
        return "Detect potential errors or issues in reasoning or output"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "content": {
                    "type": "string",
                    "description": "Content to check for errors"
                },
                "error_types": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Types of errors to check: logic, calculation, fact, consistency"
                }
            },
            "required": ["content"]
        }

    def execute(self, **kwargs) -> ToolResult:
        content = kwargs.get("content", "")
        error_types = kwargs.get("error_types", ["logic", "consistency"])

        errors = []

        # 检查逻辑一致性
        if "logic" in error_types:
            # 检测自相矛盾的陈述
            if ("总是" in content and "从不" in content) or \
               ("所有" in content and "有些" not in content and "不是所有" in content):
                errors.append({
                    "type": "logic",
                    "severity": "medium",
                    "description": "Potential logical inconsistency detected"
                })

        # 检查计算错误（简单的数字一致性）
        if "calculation" in error_types:
            import re
            numbers = re.findall(r'\d+(?:\.\d+)?', content)
            # 检测是否有明显计算错误
            if "1+1=3" in content or "2*2=5" in content:
                errors.append({
                    "type": "calculation",
                    "severity": "high",
                    "description": "Obvious calculation error detected"
                })

        # 检查一致性（相同概念是否使用相同术语）
        if "consistency" in error_types:
            # 简单检查：同一术语的不同表达
            if ("AI" in content and "人工智能" in content):
                pass  # 这是可以接受的

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "errors": errors,
                "error_count": len(errors),
            }
        )


class ImprovementSuggestTool(BaseTool):
    """改进建议工具"""

    @property
    def name(self) -> str:
        return "suggest_improvements"

    @property
    def description(self) -> str:
        return "Suggest ways to improve the quality of a response"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "response": {
                    "type": "string",
                    "description": "The response to improve"
                },
                "focus_areas": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Areas to focus on: structure, detail, examples, clarity"
                }
            },
            "required": ["response"]
        }

    def execute(self, **kwargs) -> ToolResult:
        response = kwargs.get("response", "")
        focus_areas = kwargs.get("focus_areas", ["structure", "clarity"])

        suggestions = []

        # 结构建议
        if "structure" in focus_areas:
            if len(response.split("\n")) < 3:
                suggestions.append("Consider adding paragraph breaks for better readability")
            if not any(marker in response for marker in ["1.", "2.", "-", "*"]):
                suggestions.append("Consider using bullet points or numbered lists for clarity")

        # 细节建议
        if "detail" in focus_areas:
            if len(response) < 100:
                suggestions.append("Response is brief; consider adding more details or examples")

        # 示例建议
        if "examples" in focus_areas:
            if "例如" not in response and "for example" not in response.lower():
                suggestions.append("Consider adding examples to illustrate key points")

        # 清晰度建议
        if "clarity" in focus_areas:
            complex_words = ["因此", "然而", "尽管", "综上所述", "furthermore", "however"]
            if not any(word in response for word in complex_words):
                suggestions.append("Consider using transition words for better flow")

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "suggestions": suggestions,
                "suggestion_count": len(suggestions),
            }
        )


class ReflectionSkill(Skill):
    """反思技能"""

    @property
    def name(self) -> str:
        return "reflection"

    @property
    def description(self) -> str:
        return "自我反思和改进能力，提升输出质量"

    @property
    def system_prompt(self) -> str:
        return """
你是一个具有自我反思能力的 AI 助手。在回答问题时，你应该：

【反思流程】
1. **初步回答**：首先给出你的初步答案
2. **自我评估**：使用 self_evaluate 工具评估回答质量
3. **错误检测**：使用 detect_errors 工具检查潜在错误
4. **改进建议**：使用 suggest_improvements 获取改进建议
5. **最终输出**：根据反思结果，输出改进后的答案

【评估标准】
- **准确性**：信息是否准确、有依据
- **完整性**：是否回答了所有问题
- **清晰度**：表达是否清晰、有条理
- **安全性**：内容是否安全、无害

【输出格式】
在需要高质量回答时，按以下格式输出：

---
**初步思考**：[简要说明你的初步想法]

**自我评估**：
- 准确性：[评估]
- 完整性：[评估]
- 清晰度：[评估]

**改进措施**：[根据评估结果提出的改进]

**最终答案**：[改进后的完整答案]
---

【适用场景】
- 复杂问题的回答
- 需要准确性的专业问题
- 用户明确要求高质量回答
"""

    @property
    def triggers(self) -> List[str]:
        return [
            "请仔细思考", "详细分析", "深入探讨",
            "请确保准确", "高质量回答", "认真回答",
            "think carefully", "analyze deeply", "careful answer",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            SelfEvalTool(),
            ErrorDetectionTool(),
            ImprovementSuggestTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[Reflection] 反思技能已激活")

    def on_deactivate(self) -> None:
        print(f"[Reflection] 反思技能已停用")
```

### 参考输出示例

```python
# 测试反思技能
from src.skills import ReflectionSkill

skill = ReflectionSkill()
print(f"技能名称：{skill.name}")
print(f"提供的工具：{[t.name for t in skill.get_tools()]}")
print(f"触发词示例：{skill.triggers[:5]}")
```

**参考输出**：
```
技能名称：reflection
提供的工具：['self_evaluate', 'detect_errors', 'suggest_improvements']
触发词示例：['请仔细思考', '详细分析', '深入探讨', '请确保准确', '高质量回答']
```

---

## 7.2 规划技能（Planning Skill）

### 设计目的

让 Agent 能够处理复杂任务，将大目标分解为可执行的小步骤，并有效调度和跟踪进度。

### 实现方法

使用任务树（Task Tree）+ 优先级队列（Priority Queue）：

```
复杂任务："分析公司年度销售数据并生成报告"
                    │
                    ▼
            ┌───────────────┐
            │   主任务       │
            │ (复杂度：高)   │
            └───────┬───────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │子任务 1  │ │子任务 2   │ │子任务 3  │
   │数据加载 │ │数据分析 │ │报告生成 │
   │优先级：高│ │优先级：中│ │优先级：低│
   └─────────┘ └─────────┘ └─────────┘
```

### 完整代码实现

```python
"""
规划技能（Planning Skill）完整实现
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Any, Optional
from dataclasses import dataclass, field
from enum import Enum
import json
from datetime import datetime


class TaskStatus(Enum):
    """任务状态"""
    PENDING = "pending"       # 待处理
    IN_PROGRESS = "in_progress"  # 进行中
    COMPLETED = "completed"   # 已完成
    FAILED = "failed"         # 失败
    BLOCKED = "blocked"       # 被阻塞


@dataclass
class Task:
    """任务定义"""
    id: str
    name: str
    description: str
    status: TaskStatus = TaskStatus.PENDING
    priority: int = 0  # 越高越优先
    parent_id: Optional[str] = None
    subtasks: List[str] = field(default_factory=list)
    result: Optional[str] = None
    error: Optional[str] = None
    created_at: datetime = field(default_factory=datetime.now)
    completed_at: Optional[datetime] = None


class TaskDecomposeTool(BaseTool):
    """任务分解工具"""

    @property
    def name(self) -> str:
        return "decompose_task"

    @property
    def description(self) -> str:
        return "Break down a complex task into smaller, manageable subtasks"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "task": {
                    "type": "string",
                    "description": "The complex task to decompose"
                },
                "max_depth": {
                    "type": "integer",
                    "description": "Maximum depth of decomposition (default: 3)"
                }
            },
            "required": ["task"]
        }

    def execute(self, **kwargs) -> ToolResult:
        task = kwargs.get("task", "")
        max_depth = kwargs.get("max_depth", 3)

        # 示例：分解任务
        # 实际应用应使用 LLM 进行智能分解

        decomposed = {
            "main_task": task,
            "subtasks": [],
            "depth": 0
        }

        # 简单启发式分解（实际应用应更智能）
        if "分析" in task and "报告" in task:
            decomposed["subtasks"] = [
                {"name": "数据收集", "priority": 1, "description": "收集相关数据"},
                {"name": "数据清洗", "priority": 2, "description": "清理和整理数据"},
                {"name": "数据分析", "priority": 3, "description": "执行分析"},
                {"name": "结果总结", "priority": 4, "description": "总结发现"},
                {"name": "报告撰写", "priority": 5, "description": "撰写最终报告"},
            ]
            decomposed["depth"] = 1
        elif "开发" in task or "创建" in task:
            decomposed["subtasks"] = [
                {"name": "需求分析", "priority": 1, "description": "理解需求"},
                {"name": "设计方案", "priority": 2, "description": "制定设计方案"},
                {"name": "实现编码", "priority": 3, "description": "编写代码"},
                {"name": "测试验证", "priority": 4, "description": "测试功能"},
                {"name": "部署上线", "priority": 5, "description": "部署应用"},
            ]
            decomposed["depth"] = 1
        else:
            decomposed["subtasks"] = [
                {"name": "准备阶段", "priority": 1, "description": "准备工作"},
                {"name": "执行阶段", "priority": 2, "description": "执行任务"},
                {"name": "收尾阶段", "priority": 3, "description": "完成收尾"},
            ]
            decomposed["depth"] = 1

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=decomposed
        )


class PriorityAssignTool(BaseTool):
    """优先级分配工具"""

    @property
    def name(self) -> str:
        return "assign_priorities"

    @property
    def description(self) -> str:
        return "Assign priorities to subtasks based on dependencies and importance"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "subtasks": {
                    "type": "array",
                    "items": {"type": "object"},
                    "description": "List of subtasks"
                },
                "criteria": {
                    "type": "string",
                    "description": "Priority criteria: urgency, importance, dependency"
                }
            },
            "required": ["subtasks"]
        }

    def execute(self, **kwargs) -> ToolResult:
        subtasks = kwargs.get("subtasks", [])
        criteria = kwargs.get("criteria", "importance")

        # 简单优先级分配（实际应用应更智能）
        prioritized = []
        for i, task in enumerate(subtasks):
            task_copy = dict(task)
            # 按顺序递增优先级（前面的任务优先级更高）
            task_copy["priority"] = len(subtasks) - i
            prioritized.append(task_copy)

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "prioritized_tasks": prioritized,
                "criteria": criteria,
            }
        )


class ProgressTrackerTool(BaseTool):
    """进度跟踪工具"""

    @property
    def name(self) -> str:
        return "track_progress"

    @property
    def description(self) -> str:
        return "Track the progress of task execution"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "task_id": {
                    "type": "string",
                    "description": "Task ID to track"
                },
                "status": {
                    "type": "string",
                    "description": "New status: pending, in_progress, completed, failed"
                },
                "result": {
                    "type": "string",
                    "description": "Task result (if completed)"
                },
                "error": {
                    "type": "string",
                    "description": "Error message (if failed)"
                }
            },
            "required": ["task_id"]
        }

    def execute(self, **kwargs) -> ToolResult:
        task_id = kwargs.get("task_id", "")
        new_status = kwargs.get("status")
        result = kwargs.get("result")
        error = kwargs.get("error")

        # 更新任务状态
        update_info = {
            "task_id": task_id,
            "previous_status": "unknown",
            "new_status": new_status or "pending",
            "updated_at": datetime.now().isoformat(),
        }

        if result:
            update_info["result"] = result
        if error:
            update_info["error"] = error

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=update_info
        )


class PlanningSkill(Skill):
    """规划技能"""

    @property
    def name(self) -> str:
        return "planning"

    @property
    def description(self) -> str:
        return "复杂任务规划与执行管理能力"

    @property
    def system_prompt(self) -> str:
        return """
你是一个专业的任务规划师，擅长将复杂任务分解为可执行的步骤并跟踪进度。

【核心能力】
1. **任务分解**：将大目标分解为小的、可管理的子任务
2. **优先级排序**：根据依赖关系和重要性排序
3. **进度跟踪**：实时监控任务执行状态
4. **灵活调整**：根据执行情况调整计划

【规划流程】
1. 理解任务目标和约束条件
2. 使用 decompose_task 进行任务分解
3. 使用 assign_priorities 分配优先级
4. 按优先级顺序执行子任务
5. 使用 track_progress 跟踪进度
6. 所有子任务完成后，汇总结果

【输出格式】
接到复杂任务时，按以下格式输出：

## 任务规划

**主任务**：[任务描述]

### 任务分解
| 序号 | 子任务 | 优先级 | 状态 |
|------|--------|--------|------|
| 1    | ...    | 高     | 待处理 |
| 2    | ...    | 中     | 待处理 |
| ...  | ...    | ...    | ...  |

### 执行进度
[当前完成的任务和结果]

### 下一步计划
[接下来要执行的任务]

【适用场景】
- 多步骤的复杂任务
- 需要长时间执行的任务
- 涉及多个子目标的复合任务
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 中文触发词
            "规划", "计划", "步骤", "分解",
            "如何开始", "怎么做", "流程",
            # 英文触发词
            "plan", "planning", "steps", "break down",
            "how to start", "process", "roadmap",
            # 复杂任务暗示
            "复杂的", "多步骤", "长期",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            TaskDecomposeTool(),
            PriorityAssignTool(),
            ProgressTrackerTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[Planning] 规划技能已激活")

    def on_deactivate(self) -> None:
        print(f"[Planning] 规划技能已停用")
```

---

## 7.3 记忆检索技能（Memory Retrieval Skill）

### 设计目的

让 Agent 能够智能检索相关历史信息，提高多轮对话的连贯性和知识复用能力。

### 实现方法

使用混合检索：向量相似度 + 关键词匹配

```python
"""
记忆检索技能（Memory Retrieval Skill）完整实现
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Any, Optional
import math


class SemanticSearchTool(BaseTool):
    """语义搜索工具（向量相似度）"""

    @property
    def name(self) -> str:
        return "semantic_search"

    @property
    def description(self) -> str:
        return "Search memories by semantic similarity using embeddings"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query"
                },
                "top_k": {
                    "type": "integer",
                    "description": "Number of results to return (default: 5)"
                }
            },
            "required": ["query"]
        }

    def execute(self, **kwargs) -> ToolResult:
        query = kwargs.get("query", "")
        top_k = kwargs.get("top_k", 5)

        # 模拟向量检索结果
        # 实际应用应使用真实的 embedding 模型和向量数据库

        mock_results = [
            {
                "content": f"与「{query}」相关的历史记忆 {i+1}",
                "similarity": round(0.9 - i * 0.1, 3),
                "timestamp": "2024-01-15 10:30:00",
                "context": "之前的对话",
            }
            for i in range(min(top_k, 5))
        ]

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "results": mock_results,
                "search_type": "semantic",
            }
        )


class KeywordSearchTool(BaseTool):
    """关键词搜索工具"""

    @property
    def name(self) -> str:
        return "keyword_search"

    @property
    def description(self) -> str:
        return "Search memories by keyword matching"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "keywords": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Keywords to search for"
                },
                "match_type": {
                    "type": "string",
                    "description": "Match type: exact, partial, regex",
                    "enum": ["exact", "partial", "regex"]
                }
            },
            "required": ["keywords"]
        }

    def execute(self, **kwargs) -> ToolResult:
        keywords = kwargs.get("keywords", [])
        match_type = kwargs.get("match_type", "partial")

        # 模拟关键词搜索结果
        mock_results = []
        for kw in keywords:
            mock_results.append({
                "content": f"包含关键词「{kw}」的记忆",
                "matched_keyword": kw,
                "match_count": 3,
                "timestamp": "2024-01-14 15:20:00",
            })

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "results": mock_results,
                "search_type": "keyword",
                "match_type": match_type,
            }
        )


class ReRankTool(BaseTool):
    """重排序工具"""

    @property
    def name(self) -> str:
        return "rerank_results"

    @property
    def description(self) -> str:
        return "Re-rank search results by relevance"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "results": {
                    "type": "array",
                    "items": {"type": "object"},
                    "description": "Search results to re-rank"
                },
                "strategy": {
                    "type": "string",
                    "description": "Re-ranking strategy: recency, relevance, combined"
                }
            },
            "required": ["results"]
        }

    def execute(self, **kwargs) -> ToolResult:
        results = kwargs.get("results", [])
        strategy = kwargs.get("strategy", "combined")

        # 简单重排序（实际应用应更智能）
        if strategy == "recency":
            # 按时间排序（最新的在前）
            sorted_results = sorted(results, key=lambda x: x.get("timestamp", ""), reverse=True)
        elif strategy == "relevance":
            # 按相关性排序
            sorted_results = sorted(results, key=lambda x: x.get("similarity", 0), reverse=True)
        else:
            # 综合排序
            sorted_results = results  # 保持原顺序

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "reranked_results": sorted_results,
                "strategy": strategy,
                "result_count": len(sorted_results),
            }
        )


class MemoryRetrievalSkill(Skill):
    """记忆检索技能"""

    @property
    def name(self) -> str:
        return "memory_retrieval"

    @property
    def description(self) -> str:
        return "智能检索相关历史信息，提升对话连贯性"

    @property
    def system_prompt(self) -> str:
        return """
你是一个擅长记忆检索的 AI 助手，能够快速找到相关的历史信息。

【核心能力】
1. **语义搜索**：理解查询的语义，找到相关记忆
2. **关键词匹配**：精确匹配关键词
3. **混合检索**：结合语义和关键词的优势
4. **智能排序**：按相关性和时效性排序

【检索流程】
1. 分析查询意图，提取关键词
2. 使用 semantic_search 进行语义搜索
3. 使用 keyword_search 进行关键词匹配
4. 使用 rerank_results 合并和排序结果
5. 返回最相关的记忆

【适用场景】
- 用户询问之前的对话内容
- 需要引用历史信息
- 多轮对话中的上下文关联

【注意事项】
- 优先返回高相关性的记忆
- 注意记忆的时效性
- 如未找到相关记忆，诚实告知用户
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 中文触发词
            "之前", "记得", "说过", "提到",
            "上次", "以前", "历史", "回忆",
            # 英文触发词
            "before", "remember", "mentioned", "earlier",
            "previously", "history",
            # 上下文相关
            "上下文", "前面", "刚才",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            SemanticSearchTool(),
            KeywordSearchTool(),
            ReRankTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[Memory] 记忆检索技能已激活")

    def on_deactivate(self) -> None:
        print(f"[Memory] 记忆检索技能已停用")
```

---

## 7.4 元认知技能（Meta-Cognition Skill）

### 设计目的

让 Agent 了解自己的知识边界和能力范围，能够准确表达不确定性，避免"胡编乱造"。

### 完整代码实现

```python
"""
元认知技能（Meta-Cognition Skill）完整实现
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Optional


class KnowledgeBoundaryTool(BaseTool):
    """知识边界检测工具"""

    @property
    def name(self) -> str:
        return "check_knowledge_boundary"

    @property
    def description(self) -> str:
        return "Check if a topic is within your knowledge domain"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "topic": {
                    "type": "string",
                    "description": "Topic to check"
                },
                "domain": {
                    "type": "string",
                    "description": "Knowledge domain to check against"
                }
            },
            "required": ["topic"]
        }

    def execute(self, **kwargs) -> ToolResult:
        topic = kwargs.get("topic", "")
        domain = kwargs.get("domain", "general")

        # 知识截止日期检查
        KNOWLEDGE_CUTOFF = "2024 年初"

        # 检测是否是近期事件
        recent_keywords = ["最新", "最近", "今天", "yesterday", "latest", "breaking"]
        is_recent = any(kw in topic.lower() for kw in recent_keywords)

        if is_recent:
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output={
                    "within_boundary": False,
                    "reason": "Query is about recent events; knowledge cutoff is " + KNOWLEDGE_CUTOFF,
                    "suggestion": "Search for latest information"
                }
            )

        # 检测是否是专业领域
        specialized_domains = ["医疗诊断", "法律咨询", "投资建议", "psychology", "law"]
        is_specialized = any(d in topic for d in specialized_domains)

        if is_specialized:
            return ToolResult(
                status=ToolResultStatus.SUCCESS,
                output={
                    "within_boundary": "partial",
                    "reason": "This is a specialized domain requiring expert knowledge",
                    "suggestion": "Consult a qualified professional"
                }
            )

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "within_boundary": True,
                "reason": "Topic is within general knowledge",
            }
        )


class UncertaintyEstimatorTool(BaseTool):
    """不确定性评估工具"""

    @property
    def name(self) -> str:
        return "estimate_uncertainty"

    @property
    def description(self) -> str:
        return "Estimate the uncertainty level of your knowledge on a topic"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "statement": {
                    "type": "string",
                    "description": "Statement or claim to evaluate"
                },
                "factors": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Uncertainty factors: source_quality, recency, consensus, expertise"
                }
            },
            "required": ["statement"]
        }

    def execute(self, **kwargs) -> ToolResult:
        statement = kwargs.get("statement", "")
        factors = kwargs.get("factors", ["source_quality", "consensus"])

        uncertainty_level = "low"  # 默认低不确定性
        reasons = []

        # 检查是否有不确定性标记词
        uncertainty_markers = ["可能", "也许", "大概", "approximately", "might", "could", "uncertain"]
        if any(marker in statement for marker in uncertainty_markers):
            uncertainty_level = "medium"
            reasons.append("Statement contains uncertainty markers")

        # 检查是否有绝对化表述
        absolute_markers = ["一定", "绝对", "肯定", "always", "never", "definitely"]
        if any(marker in statement for marker in absolute_markers):
            uncertainty_level = "high"
            reasons.append("Absolute claims have higher uncertainty")

        # 检查是否有具体数据支持
        import re
        if re.search(r'\d+%|\d+/\d+|\d+ percent', statement):
            reasons.append("Contains specific data (may increase confidence)")
        else:
            reasons.append("No specific data cited")

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "uncertainty_level": uncertainty_level,
                "factors": factors,
                "reasons": reasons,
            }
        )


class MetaCognitionSkill(Skill):
    """元认知技能"""

    @property
    def name(self) -> str:
        return "meta_cognition"

    @property
    def description(self) -> str:
        return "元认知能力：了解知识边界，准确表达不确定性"

    @property
    def system_prompt(self) -> str:
        return """
你是一个具有元认知能力的 AI 助手，了解自己的知识边界和能力范围。

【核心原则】
1. **诚实表达**：不知道就说不知道
2. **准确评估**：正确评估自己的不确定性
3. **知识边界**：清楚自己的知识截止日期和领域限制
4. **专业谦逊**：在专业领域建议咨询专家

【元认知流程】
1. 接到问题时，首先使用 check_knowledge_boundary 检查是否在知识范围内
2. 对于知识范围内的问题，使用 estimate_uncertainty 评估确定性
3. 根据评估结果，选择合适的回答方式：
   - 高确定性：直接回答
   - 中等确定性：回答但注明不确定性
   - 低确定性：建议用户咨询专家或查证

【回答模式】

**高确定性回答**：
"根据我的知识，[答案]。"

**中等确定性回答**：
"据我所知，[答案]。但请注意，这个信息可能不完全准确，建议进一步核实。"

**低确定性回答**：
"这个问题超出了我的可靠知识范围。我建议 [专业建议]。"

【知识限制】
- 知识截止日期：2024 年初
- 不提供：医疗诊断、法律建议、投资建议
- 对于实时信息（新闻、天气、股价等），建议用户查询最新来源
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 知识相关
            "你知道吗", "是否知道", "清楚吗",
            "确定吗", "肯定吗", "准确吗",
            # 建议请求
            "建议", "推荐", "应该",
            # 专业领域
            "医疗", "法律", "投资", "诊断",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            KnowledgeBoundaryTool(),
            UncertaintyEstimatorTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[MetaCognition] 元认知技能已激活")

    def on_deactivate(self) -> None:
        print(f"[MetaCognition] 元认知技能已停用")
```

---

## 7.5 工具学习技能（Tool Learning Skill）

### 设计目的

让 Agent 能够从文档或示例中自动学习如何使用新工具，无需硬编码工具调用逻辑。

### 完整代码实现

```python
"""
工具学习技能（Tool Learning Skill）完整实现
"""

from src.skills.base import Skill, SkillContext
from src.tools.base import BaseTool, ToolResult, ToolResultStatus
from typing import List, Dict, Optional


class DocParserTool(BaseTool):
    """文档解析工具"""

    @property
    def name(self) -> str:
        return "parse_documentation"

    @property
    def description(self) -> str:
        return "Parse tool documentation to extract usage information"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "documentation": {
                    "type": "string",
                    "description": "Tool documentation text"
                },
                "extract_fields": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "Fields to extract: name, description, parameters, examples, errors"
                }
            },
            "required": ["documentation"]
        }

    def execute(self, **kwargs) -> ToolResult:
        doc = kwargs.get("documentation", "")
        extract_fields = kwargs.get("extract_fields", ["name", "description", "parameters"])

        # 简单解析文档（实际应用应使用更智能的解析）
        parsed = {
            "name": "unknown",
            "description": "",
            "parameters": [],
            "examples": [],
            "errors": [],
        }

        # 提取名称
        import re
        name_match = re.search(r'(?:tool|function)\s+(?:named\s+)?["\']?(\w+)["\']?', doc, re.I)
        if name_match:
            parsed["name"] = name_match.group(1)

        # 提取描述（第一段）
        paragraphs = doc.split("\n\n")
        if paragraphs:
            parsed["description"] = paragraphs[0].strip()

        # 提取参数
        if "parameters:" in doc.lower():
            param_section = doc.lower().split("parameters:")[1]
            param_lines = param_section.split("\n")[:10]
            parsed["parameters"] = [line.strip() for line in param_lines if line.strip()]

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=parsed
        )


class ExampleExtractorTool(BaseTool):
    """示例提取工具"""

    @property
    def name(self) -> str:
        return "extract_examples"

    @property
    def description(self) -> str:
        return "Extract usage examples from documentation or code"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "content": {
                    "type": "string",
                    "description": "Content to extract examples from"
                },
                "example_type": {
                    "type": "string",
                    "description": "Type of examples: code, api_call, natural_language",
                    "enum": ["code", "api_call", "natural_language"]
                }
            },
            "required": ["content"]
        }

    def execute(self, **kwargs) -> ToolResult:
        content = kwargs.get("content", "")
        example_type = kwargs.get("example_type", "code")

        examples = []

        # 提取代码示例
        if example_type == "code":
            import re
            code_blocks = re.findall(r'```(?:\w+)?\n(.*?)```', content, re.DOTALL)
            examples = [{"type": "code", "content": block.strip()} for block in code_blocks]

        # 提取 API 调用示例
        elif example_type == "api_call":
            import re
            api_calls = re.findall(r'(?:api|call|request)\s*[:\s](.+)', content, re.I)
            examples = [{"type": "api_call", "content": call.strip()} for call in api_calls]

        # 提取自然语言示例
        elif example_type == "natural_language":
            # 提取引号中的内容作为示例
            import re
            quotes = re.findall(r'["\']([^"\']+)["\']', content)
            examples = [{"type": "natural_language", "content": quote} for quote in quotes[:10]]

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={
                "examples": examples,
                "count": len(examples),
            }
        )


class UsageGeneratorTool(BaseTool):
    """用法生成工具"""

    @property
    def name(self) -> str:
        return "generate_usage"

    @property
    def description(self) -> str:
        return "Generate usage instructions from parsed documentation"

    @property
    def parameters(self) -> dict:
        return {
            "type": "object",
            "properties": {
                "tool_info": {
                    "type": "object",
                    "description": "Parsed tool information"
                },
                "format": {
                    "type": "string",
                    "description": "Output format: markdown, json, natural_language",
                    "enum": ["markdown", "json", "natural_language"]
                }
            },
            "required": ["tool_info"]
        }

    def execute(self, **kwargs) -> ToolResult:
        tool_info = kwargs.get("tool_info", {})
        output_format = kwargs.get("format", "markdown")

        name = tool_info.get("name", "unknown")
        description = tool_info.get("description", "")
        parameters = tool_info.get("parameters", [])
        examples = tool_info.get("examples", [])

        if output_format == "markdown":
            usage = f"""## {name}

**描述**：{description}

**参数**：
"""
            for param in parameters:
                usage += f"- {param}\n"

            if examples:
                usage += "\n**示例**：\n"
                for ex in examples[:3]:
                    usage += f"```\n{ex.get('content', '')}\n```\n"

        elif output_format == "json":
            usage = {
                "name": name,
                "description": description,
                "parameters": parameters,
                "examples": examples,
            }
        else:
            usage = f"工具 {name} 用于{description}。参数包括：{', '.join(parameters)}。"

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output={"usage": usage}
        )


class ToolLearningSkill(Skill):
    """工具学习技能"""

    @property
    def name(self) -> str:
        return "tool_learning"

    @property
    def description(self) -> str:
        return "从文档和示例中自动学习工具用法"

    @property
    def system_prompt(self) -> str:
        return """
你是一个擅长学习新工具使用方法的 AI 助手。

【核心能力】
1. **文档解析**：从文档中提取工具信息
2. **示例学习**：从示例中学习用法
3. **用法生成**：生成清晰的使用说明

【学习流程】
1. 接收工具文档或使用示例
2. 使用 parse_documentation 解析文档结构
3. 使用 extract_examples 提取使用示例
4. 使用 generate_usage 生成使用说明
5. 根据学习内容，正确使用工具

【输出格式】
学习新工具后，按以下格式输出：

## 工具学习：[工具名称]

**功能描述**：[工具是做什么的]

**参数说明**：
- 参数 1：说明
- 参数 2：说明

**使用示例**：
```
[示例代码或调用方式]
```

**注意事项**：
- 注意点 1
- 注意点 2

【适用场景】
- 面对陌生的工具或 API
- 需要从文档中快速学习
- 需要生成使用说明
"""

    @property
    def triggers(self) -> List[str]:
        return [
            # 学习相关
            "学习", "如何使用", "怎么用", "用法",
            "这个工具", "这个函数", "这个 API",
            # 英文触发词
            "learn", "how to use", "usage", "this tool",
        ]

    def get_tools(self) -> List[BaseTool]:
        return [
            DocParserTool(),
            ExampleExtractorTool(),
            UsageGeneratorTool(),
        ]

    def on_activate(self, context: SkillContext) -> None:
        print(f"[ToolLearning] 工具学习技能已激活")

    def on_deactivate(self) -> None:
        print(f"[ToolLearning] 工具学习技能已停用")
```

---

# 8. 实践练习与挑战

## 8.1 基础练习

### 练习 1：修改 CalculatorSkill 支持更多运算

**任务**：扩展现有的 CalculatorSkill，支持以下运算：
- 幂运算（^或**）
- 平方根（sqrt）
- 百分比（%）
- 三角函数（sin, cos, tan）

**要求**：
1. 修改 CalculatorTool 的 execute 方法
2. 更新 parameters 定义
3. 添加适当的错误处理
4. 测试所有新功能

**提示代码**：
```python
import math

def execute(self, **kwargs):
    expression = kwargs.get("expression", "")
    # 处理幂运算
    expression = expression.replace("^", "**")
    # 处理平方根
    expression = expression.replace("sqrt", "math.sqrt")
    # ... 继续处理其他运算
    try:
        result = eval(expression)
        return ToolResult(status=ToolResultStatus.SUCCESS, output=result)
    except Exception as e:
        return ToolResult(status=ToolResultStatus.ERROR, error=str(e))
```

---

### 练习 2：为 SearchSkill 添加自定义搜索源

**任务**：扩展 SearchSkill，支持从指定网站搜索：
- 添加 `site` 参数限制搜索范围
- 支持多个网站的白名单
- 返回结果中包含来源网站信息

**要求**：
1. 修改 SearchEngineTool 的参数定义
2. 实现网站过滤逻辑
3. 更新 system_prompt 说明新功能

---

### 练习 3：创建简单的 WeatherSkill

**任务**：创建一个天气查询技能（使用模拟数据）

**要求**：
1. 定义 WeatherSkill 类，继承自 Skill
2. 实现 WeatherTool，包含以下功能：
   - 查询指定城市的天气
   - 返回温度、湿度、天气状况
3. 编写 system_prompt，说明如何使用
4. 设置合适的触发词

**示例代码框架**：
```python
class WeatherSkill(Skill):
    @property
    def name(self) -> str:
        return "weather"

    @property
    def description(self) -> str:
        return "查询天气信息"

    @property
    def system_prompt(self) -> str:
        return """你是天气查询助手..."""

    @property
    def triggers(self) -> List[str]:
        return ["天气", "weather", "气温", "下雨"]

    def get_tools(self) -> List[BaseTool]:
        return [WeatherTool()]


class WeatherTool(BaseTool):
    @property
    def name(self) -> str:
        return "get_weather"

    # ... 实现其他必需的方法和 execute
```

---

## 8.2 进阶练习

### 练习 4：创建带配置的 ConfigurableSkill

**任务**：实现一个支持动态配置的 Skill

**要求**：
1. 支持通过配置字典自定义行为
2. 配置可以在运行时更新
3. 配置可以持久化（保存到文件）

**示例接口**：
```python
class ConfigurableSkill(Skill):
    def __init__(self, config: Dict[str, Any] = None):
        self.config = config or {}

    def update_config(self, updates: Dict[str, Any]):
        self.config.update(updates)

    def save_config(self, file_path: str):
        import json
        with open(file_path, 'w') as f:
            json.dump(self.config, f)

    @classmethod
    def load_config(cls, file_path: str) -> 'ConfigurableSkill':
        import json
        with open(file_path, 'r') as f:
            config = json.load(f)
        return cls(config)
```

---

### 练习 5：实现技能组合（Skill Chain）

**任务**：创建一个可以将多个技能串联执行的系统

**要求**：
1. 定义 SkillChain 类
2. 支持按顺序执行多个技能
3. 前一个技能的输出可以作为后一个技能的输入
4. 支持条件分支（根据结果选择下一个技能）

**示例用法**：
```python
chain = SkillChain()
chain.add_step(TranslationSkill(), {"target_lang": "en"})
chain.add_step(CodeReviewSkill(), condition=lambda x: x["language"] == "en")
chain.add_step(TranslationSkill(), {"target_lang": "zh"})

result = chain.run("审查这段中文代码")
```

---

### 练习 6：为技能添加单元测试

**任务**：为 CalculatorSkill 编写完整的单元测试

**要求**：
1. 测试正常情况下的计算
2. 测试边界情况（除零、大数等）
3. 测试错误输入处理
4. 测试技能激活逻辑

**示例测试代码**：
```python
import unittest
from src.skills import CalculatorSkill

class TestCalculatorSkill(unittest.TestCase):
    def setUp(self):
        self.skill = CalculatorSkill()
        self.tools = {t.name: t for t in self.skill.get_tools()}

    def test_basic_addition(self):
        tool = self.tools["calculator"]
        result = tool.execute(expression="2 + 2")
        self.assertEqual(result.output, 4)

    def test_division_by_zero(self):
        tool = self.tools["calculator"]
        result = tool.execute(expression="1 / 0")
        self.assertEqual(result.status, ToolResultStatus.ERROR)

    def test_skill_triggers(self):
        self.assertTrue(self.skill.should_activate("请计算 1+1"))
        self.assertTrue(self.skill.should_activate("calculate 2*3"))
        self.assertFalse(self.skill.should_activate("今天天气不错"))

if __name__ == "__main__":
    unittest.main()
```

---

## 8.3 挑战练习

### 挑战 1：实现完整的 EmailAssistantSkill

**任务**：创建一个邮件写作助手技能

**功能要求**：
1. 支持多种邮件类型（正式、非正式、商务、感谢等）
2. 支持语气调整（友好、专业、紧急等）
3. 支持邮件润色和语法检查
4. 支持邮件模板

**工具设计**：
- EmailGeneratorTool：生成邮件草稿
- ToneAdjusterTool：调整语气
- GrammarCheckerTool：语法检查
- TemplateTool：模板管理

**完整代码量**：约 300-500 行

---

### 挑战 2：设计与实现 InterviewSkill（面试模拟）

**任务**：创建一个面试模拟技能，帮助用户准备面试

**功能要求**：
1. 支持多种职位类型（开发、产品、设计等）
2. 根据用户水平调整问题难度
3. 提供问题回答反馈
4. 记录面试历史和改进建议

**工具设计**：
- QuestionGeneratorTool：生成面试问题
- AnswerEvaluatorTool：评估回答质量
- FeedbackTool：提供改进反馈
- ProgressTrackerTool：跟踪进步

**完整代码量**：约 400-600 行

---

### 挑战 3：设计与实现通用的 PluginLoader 系统

**任务**：创建一个完整的插件加载系统，支持：
1. 从目录自动发现技能
2. 热加载/热卸载插件
3. 插件依赖管理
4. 插件沙箱隔离

**功能要求**：
1. 插件目录结构规范
2. 插件元数据（名称、版本、依赖）
3. 插件生命周期管理
4. 插件错误隔离

**完整代码量**：约 500-800 行

---

# 9. 常见问题与最佳实践

## 9.1 常见问题

### Q1: 技能不自动激活怎么办？

**可能原因**：
1. 触发词与查询不匹配
2. 大小写问题
3. 中文字符编码问题

**解决方法**：
```python
# 1. 检查触发词定义
print(skill.triggers)

# 2. 调试 should_activate 方法
query = "计算 1+1"
print(f"查询：'{query}'")
for trigger in skill.triggers:
    if trigger.lower() in query.lower():
        print(f"匹配触发词：'{trigger}'")

# 3. 重写 should_activate 增加灵活性
def should_activate(self, query: str) -> bool:
    query_lower = query.lower().strip()
    for trigger in self.triggers:
        trigger_clean = trigger.lower().strip()
        if trigger_clean in query_lower:
            return True
        # 尝试模糊匹配
        if any(word in query_lower for word in trigger_clean.split()):
            return True
    return False
```

---

### Q2: 多个技能提示词冲突如何处理？

**问题示例**：
- CalculatorSkill 说"始终展示计算步骤"
- 另一个技能说"直接给出答案"

**解决方法**：

**方法 1：优先级覆盖**
```python
class SkillManager:
    def get_combined_prompt(self, skills=None):
        if skills is None:
            skills = self.get_active_skills()

        # 按优先级排序
        sorted_skills = sorted(
            skills,
            key=lambda s: getattr(s, "_priority", 0),
            reverse=True
        )

        prompts = []
        for skill in sorted_skills:
            if skill.system_prompt:
                prompts.append(f"[{skill.name}]\n{skill.system_prompt}")

        # 添加冲突解决说明
        if len(prompts) > 1:
            prompts.insert(0, "注意：如果不同技能有冲突的指示，请优先遵循排在前面的技能。")

        return "\n\n".join(prompts)
```

**方法 2：提示词合并优化**
```python
def merge_prompts(prompts: List[str]) -> str:
    """合并提示词，去除重复和冲突"""
    merged = []
    seen_instructions = set()

    for prompt in prompts:
        lines = prompt.split("\n")
        for line in lines:
            if line.strip() and line.strip() not in seen_instructions:
                merged.append(line)
                seen_instructions.add(line.strip())

    return "\n".join(merged)
```

---

### Q3: 技能工具太多影响性能怎么办？

**问题**：注册了大量技能导致工具列表很长，影响 LLM 理解和响应速度

**解决方法**：

**方法 1：动态工具注册**
```python
class DynamicToolAgent(SkilledReActAgent):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._registered_tools = {}  # 缓存所有技能的工具
        self._active_tools = set()   # 当前激活的工具

    def activate_skill(self, skill_name: str):
        # 只在技能激活时注册对应工具
        skill = self.skill_manager.get_skill(skill_name)
        if skill:
            for tool in skill.get_tools():
                self.tools.register(tool)
                self._active_tools.add(tool.name)

    def deactivate_skill(self, skill_name: str):
        # 停用时移除工具
        skill = self.skill_manager.get_skill(skill_name)
        if skill:
            for tool in skill.get_tools():
                if tool.name not in self._active_tools:
                    continue
                self.tools.unregister(tool.name)
                self._active_tools.discard(tool.name)
```

**方法 2：工具分组**
```python
class GroupedToolRegistry:
    def __init__(self):
        self._groups: Dict[str, ToolRegistry] = {}
        self._active_group: Optional[str] = None

    def create_group(self, name: str):
        self._groups[name] = ToolRegistry()

    def register_to_group(self, group: str, tool: BaseTool):
        if group not in self._groups:
            self.create_group(group)
        self._groups[group].register(tool)

    def activate_group(self, group: str):
        self._active_group = group

    def get_active_tools(self) -> List[BaseTool]:
        if self._active_group:
            return list(self._groups[self._active_group]._tools.values())
        return []
```

---

### Q4: 如何实现技能的持久化配置？

**方法 1：JSON 配置文件**

```python
import json
from pathlib import Path

class PersistableSkillManager(SkillManager):
    """支持持久化的 SkillManager"""

    def save_config(self, file_path: str):
        """保存配置到 JSON 文件"""
        config = {
            "skills": {},
            "active_skills": list(self._active_skills),
            "priority_order": self._priority_order,
        }

        for name, skill in self._skills.items():
            config["skills"][name] = {
                "class": skill.__class__.__name__,
                "priority": getattr(skill, "_priority", 0),
            }

        with open(file_path, 'w', encoding='utf-8') as f:
            json.dump(config, f, ensure_ascii=False, indent=2)

        print(f"配置已保存到：{file_path}")

    def load_config(self, file_path: str, skill_factory: dict):
        """
        从 JSON 文件加载配置

        Args:
            file_path: 配置文件路径
            skill_factory: 技能工厂字典，如 {"calculator": CalculatorSkill}
        """
        with open(file_path, 'r', encoding='utf-8') as f:
            config = json.load(f)

        # 加载技能
        for name, skill_info in config.get("skills", {}).items():
            class_name = skill_info.get("class")
            if class_name in skill_factory:
                skill = skill_factory[class_name]()
                self.register_skill(skill, priority=skill_info.get("priority", 0))

        # 恢复激活状态
        for name in config.get("active_skills", []):
            if name in self._skills:
                self.activate_skill(name)

        print(f"配置已从 {file_path} 加载")


# 使用示例
if __name__ == "__main__":
    from src.skills import CalculatorSkill, SearchSkill

    manager = PersistableSkillManager()
    manager.register_skill(CalculatorSkill())
    manager.register_skill(SearchSkill())
    manager.activate_skill("calculator")

    # 保存配置
    manager.save_config("skill_config.json")

    # 加载配置
    new_manager = PersistableSkillManager()
    new_manager.load_config("skill_config.json", {
        "CalculatorSkill": CalculatorSkill,
        "SearchSkill": SearchSkill,
    })
```

**方法 2：YAML 配置（更友好）**

```python
import yaml

# skill_config.yaml
"""
skills:
  - name: calculator
    class: CalculatorSkill
    priority: 1
    enabled: true

  - name: search
    class: SearchSkill
    priority: 0
    enabled: true

  - name: code_review
    class: CodeReviewSkill
    priority: 2
    enabled: false

settings:
  max_active_skills: 3
  auto_activate: true
"""

# 加载 YAML 配置
def load_yaml_config(file_path: str, skill_factory: dict) -> SkillManager:
    with open(file_path, 'r', encoding='utf-8') as f:
        config = yaml.safe_load(f)

    manager = SkillManager()

    for skill_config in config.get("skills", []):
        if skill_config.get("enabled", True):
            class_name = skill_config.get("class")
            if class_name in skill_factory:
                skill = skill_factory[class_name]()
                manager.register_skill(
                    skill,
                    priority=skill_config.get("priority", 0)
                )

    return manager
```

---

## 9.2 最佳实践

### 技能命名规范

```python
# 推荐 ✓
name = "code_review"       # 小写 + 下划线
name = "data_analysis"
name = "translation"

# 不推荐 ✗
name = "CodeReview"        # 驼峰式
name = "code-review"       # 连字符
name = "skill_code_review" # 冗余前缀
```

### 提示词设计原则

**1. 角色定义清晰**
```python
# 好 ✓
"You are a professional code reviewer with 10 years of experience."

# 不好 ✗
"You are helpful."
```

**2. 步骤具体明确**
```python
# 好 ✓
"""
执行步骤：
1. 首先分析代码结构
2. 检查命名规范
3. 识别潜在 bug
4. 提供改进建议
"""

# 不好 ✗
"""
好好审查代码。
"""
```

**3. 输出格式规范**
```python
# 好 ✓
"""
输出格式：
## 审查结果

### 问题列表
- [严重] 问题 1
- [一般] 问题 2

### 建议
...
"""
```

### 工具粒度控制

**原则**：一个工具一个职责

```python
# 好 ✓
class CodeAnalyzerTool(BaseTool):
    """只负责代码分析"""

class StyleLinterTool(BaseTool):
    """只负责风格检查"""

# 不好 ✗
class EverythingTool(BaseTool):
    """什么都做，代码冗长难维护"""
```

### 错误处理模式

```python
def execute(self, **kwargs) -> ToolResult:
    try:
        # 1. 参数验证
        required_params = ["code"]
        for param in required_params:
            if param not in kwargs:
                return ToolResult(
                    status=ToolResultStatus.ERROR,
                    error=f"Missing required parameter: {param}"
                )

        # 2. 执行逻辑
        result = self._do_something(kwargs)

        # 3. 结果验证
        if result is None:
            return ToolResult(
                status=ToolResultStatus.ERROR,
                error="Execution returned None"
            )

        return ToolResult(
            status=ToolResultStatus.SUCCESS,
            output=result
        )

    except ValueError as e:
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error=f"Invalid input: {str(e)}"
        )
    except Exception as e:
        return ToolResult(
            status=ToolResultStatus.ERROR,
            error=f"Unexpected error: {str(e)}"
        )
```

### 测试策略

```python
import unittest

class TestSkillBase(unittest.TestCase):
    """Skill 测试基类"""

    def test_skill_has_required_properties(self):
        """测试技能有必需的属性"""
        skill = self.skill_class()

        self.assertIsInstance(skill.name, str)
        self.assertIsInstance(skill.description, str)
        self.assertIsInstance(skill.triggers, list)
        self.assertIsNotNone(skill.system_prompt)

    def test_tools_return_list(self):
        """测试 get_tools 返回列表"""
        skill = self.skill_class()
        tools = skill.get_tools()

        self.assertIsInstance(tools, list)
        for tool in tools:
            self.assertIsInstance(tool, BaseTool)

    def test_skill_activation(self):
        """测试技能激活逻辑"""
        skill = self.skill_class()

        # 测试应该激活的情况
        for trigger_query in ["查询 1", "查询 2"]:
            self.assertTrue(
                skill.should_activate(trigger_query),
                f"Skill should activate for: {trigger_query}"
            )

        # 测试不应该激活的情况
        for non_trigger_query in ["无关查询 1", "无关查询 2"]:
            self.assertFalse(
                skill.should_activate(non_trigger_query),
                f"Skill should not activate for: {non_trigger_query}"
            )


class TestCalculatorSkill(TestSkillBase):
    def setUp(self):
        from src.skills import CalculatorSkill
        self.skill_class = CalculatorSkill
        self.skill = CalculatorSkill()

    def test_calculation_accuracy(self):
        """测试计算准确性"""
        tools = {t.name: t for t in self.skill.get_tools()}
        result = tools["calculator"].execute(expression="2 + 2")
        self.assertEqual(result.output, 4)

    def test_division_by_zero(self):
        """测试除零错误处理"""
        tools = {t.name: t for t in self.skill.get_tools()}
        result = tools["calculator"].execute(expression="1 / 0")
        self.assertEqual(result.status, ToolResultStatus.ERROR)


if __name__ == "__main__":
    unittest.main()
```

---

# 总结

恭喜你完成了第六章 Skills 系统详解的学习！

## 本章回顾

### 核心知识点

1. **Skills 基础概念**
   - Skills vs Tools vs Capabilities 的区别
   - Skills 的四大核心特征：有状态、可配置、组合性、领域专精
   - 业界主流 Skills/Plugins 系统对比

2. **Skill 核心架构**
   - Skill 抽象类的六大核心组件
   - SkillContext 上下文类的五个字段
   - FunctionSkill 快速封装模式
   - 内置 Skills 源码解析

3. **SkillManager 技能管理器**
   - 注册机制与优先级系统
   - 技能检测算法
   - 组合提示词生成
   - 生命周期管理

4. **实战案例**
   - 代码审查 Skill
   - 翻译助手 Skill
   - 数据分析 Skill
   - Web 搜索 Skill

5. **Skill 与 Agent 集成**
   - 三种集成模式
   - 调试技巧
   - 常见问题排查

6. **高级话题**
   - 技能冲突与优先级
   - 技能组合与编排
   - 动态技能加载（插件系统）
   - 技能评估与优化
   - 安全考虑

7. **前沿技能设计**
   - 反思技能（Reflection）
   - 规划技能（Planning）
   - 记忆检索技能（Memory Retrieval）
   - 元认知技能（Meta-Cognition）
   - 工具学习技能（Tool Learning）

## 技能树

学完本章后，你应该能够：

```
┌─────────────────────────────────────────────────────────────┐
│                    Skills 技能树                             │
│                                                             │
│  理解层                                                     │
│  ├── 理解 Skills 与 Tools 的区别                             │
│  ├── 理解 Skills 在 Agent 架构中的位置                       │
│  └── 了解业界主流方案                                        │
│                                                             │
│  应用层                                                     │
│  ├── 能够创建自定义 Skills                                   │
│  ├── 能够使用 SkillManager 管理技能                          │
│  └── 能够将 Skills 集成到 Agent                              │
│                                                             │
│  进阶层                                                     │
│  ├── 能够设计技能组合和编排                                  │
│  ├── 能够实现动态技能加载                                    │
│  └── 能够进行技能评估和优化                                  │
│                                                             │
│  创新层                                                     │
│  ├── 能够设计前沿技能（反思、规划等）                        │
│  └── 能够根据需求创造新的技能类型                            │
└─────────────────────────────────────────────────────────────┘
```

## 下一步学习建议

1. **动手实践**：完成本章的练习和挑战，至少实现 2-3 个自定义 Skills
2. **阅读源码**：深入阅读 `src/skills/` 目录下的源代码
3. **扩展项目**：将学到的 Skills 应用到你的实际项目中
4. **学习前沿**：关注 Agent 领域的最新进展，了解新的技能设计模式

## 完整教程系列

本教程共六章，涵盖 AI Agent 的完整知识体系：

| 章节 | 主题 | 核心内容 |
|------|------|---------|
| 第一章 | Agent 基础 | Agent 组成、ReAct 模式、状态管理 |
| 第二章 | 工具系统 | BaseTool、ToolRegistry、工具创建 |
| 第三章 | 记忆系统 | 短期记忆、长期记忆、向量检索 |
| 第四章 | MCP 协议 | Model Context Protocol、服务集成 |
| 第五章 | 完整 Agent | 综合应用、项目实战 |
| **第六章** | **Skills 系统** | **技能架构、管理器、实战、前沿设计** |

---

**本章代码位置**：
- Skill 基类：`src/skills/base.py`
- SkillManager：`src/skills/manager.py`
- 内置 Skills：`src/skills/base.py` (CalculatorSkill, SearchSkill, FilesystemSkill)

**相关资源**：
- 项目 GitHub：[待补充]
- 讨论区：[待补充]

---

*至此，你已经完成了整个 AI Agent 教程的学习。希望这套教程能够帮助你从零构建强大的 AI Agent 系统！*