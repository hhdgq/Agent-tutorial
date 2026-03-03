# 第二章 ReAct 模式详解

## 2.1 什么是 ReAct？

### 2.1.1 ReAct 的起源

ReAct（Reasoning + Acting，推理 + 行动）是一种让大型语言模型（LLM）交替进行推理和行动的模式。这个框架由普林斯顿大学的 Yao 等人在 2022 年发表的论文 ["ReAct: Synergizing Reasoning and Acting in Language Models"](https://arxiv.org/abs/2210.03629) 中首次提出。

**论文核心发现**：推理和行动结合比单独推理或单独行动效果更好。

ReAct 在多个基准测试上表现出色：

| 基准测试 | 任务类型 | ReAct 表现 |
|----------|----------|------------|
| **HotpotQA** | 多跳问答 | 减少了事实幻觉，能够动态检索信息 |
| **FEVER** | 事实验证 | 超越 CoT 和 Act-only 方法 |
| **ALFWorld** | 文本环境决策 | 能更好地分解复杂目标 |
| **WebShop** | 在线购物 | 表现出更好的导航和决策能力 |

论文的关键结论是：**ReAct + CoT + Self-Consistency** 组合通常超越所有其他方法。

### 2.1.2 ReAct 的核心思想

ReAct 的核心是 **Thought-Action-Observation（思考 - 行动 - 观察）** 循环：

```
Thought (思考) → Action (行动) → Observation (观察) → Thought → ...
```

这个循环持续进行，直到 Agent 得出最终答案。

#### 为什么"边想边做"比"只想不做"或"只做不想"更好？

| 组件 | 作用 | 优势 |
|------|------|------|
| **Thought（推理）** | 生成语言形式的推理轨迹，帮助模型规划、跟踪和更新行动计划 | 使决策过程可追溯、可理解 |
| **Action（行动）** | 执行具体操作，与外部环境交互（如搜索、计算、API 调用） | 获取外部信息，弥补 LLM 知识局限 |
| **Observation（观察）** | 获取行动结果，为下一步推理提供新信息 | 形成反馈闭环，支持迭代优化 |

**ReAct 的关键优势**：
1. 检索信息支持推理（应对知识截止和幻觉问题）
2. 推理帮助定位下一步检索目标（提高信息获取效率）

#### ReAct 循环流程图

```
┌─────────────────────────────────────────────────────────┐
│                     用户输入                             │
│                 "计算 15 × 7 + 20"                      │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Thought (思考)                                         │
│  "我需要分步计算：先算 15×7，再加 20"                   │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Action (行动)                                          │
│  选择工具：calculate                                    │
│  参数：expression = "15 * 7"                            │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  Observation (观察)                                     │
│  工具返回：105                                          │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │ 是否完成？            │
              └───────────┬───────────┘
                    │           │
                   否          是
                    │           │
                    ▼           ▼
            继续循环...    ┌─────────────────┐
                           │ Final Answer    │
                           │ "答案是 125"    │
                           └─────────────────┘
```

### 2.1.3 ReAct vs 其他推理方法

| 方法 | 特点 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| **Standard Prompting** | 直接回答 | 简单快速 | 容易出错，无法验证，事实幻觉 | 简单问题、常识问答 |
| **Chain of Thought (CoT)** | 逐步推理 | 提高推理准确性 | 无法获取外部信息，容易幻觉 | 数学题、逻辑推理 |
| **Act-only** | 只行动不思考 | 直接执行 | 无法分解复杂目标 | 简单工具调用 |
| **ReAct** | 推理 + 行动 | 结合内外信息，可追溯 | 实现复杂度较高，依赖检索质量 | 复杂问答、多步任务 |
| **Tree of Thoughts (ToT)** | 多路径探索 | 全局最优解 | 计算成本高 | 需要规划的任务 |
| **Reflexion** | 自我反思 | 从错误中学习 | 需要额外反馈 | 迭代优化任务 |

---

## 2.2 为什么需要 ReAct？

### 2.2.1 LLM 的局限性问题

大型语言模型虽然强大，但存在一些固有局限：

#### 1. 不擅长精确计算

```
用户：15 乘以 7 再加 20 等于多少？

LLM（直接回答）：135（错误！）
```

LLM 是基于概率预测的模型，在做数学计算时容易出错。

#### 2. 知识截止问题

```
用户：2024 年诺贝尔物理学奖得主是谁？

LLM：抱歉，我的知识截止于 2023 年，无法回答这个问题。
```

LLM 的训练数据有截止时间，无法获取最新信息。

#### 3. 幻觉（Hallucination）问题

```
用户：Python 语言的创始人是谁？

LLM（幻觉）：Python 是由 John Smith 于 1995 年创建的。（错误！）
```

LLM 可能生成看似合理但不准确的信息。

#### 4. 如何通过工具调用来弥补这些局限

ReAct 通过引入工具调用机制，让 LLM 能够：
- 使用计算器进行精确计算
- 使用搜索引擎获取最新信息
- 使用数据库验证事实

### 2.2.2 ReAct 解决的问题

| 问题 | ReAct 解决方案 |
|------|----------------|
| 分步推理减少错误 | Thought 步骤让模型显式规划，避免跳跃式错误 |
| 工具使用保证准确性 | Action 步骤调用可靠工具，如计算器、搜索引擎 |
| 过程可追踪、可调试 | 每个 Thought-Action-Observation 都可记录和分析 |
| 支持复杂任务分解 | 通过多轮循环逐步解决复杂问题 |

### 2.2.3 实际案例对比

#### 案例 1：数学计算

**直接回答（容易出错）**：
```
用户：15 乘以 7 再加 20 等于多少？
LLM: 135（错误）
```

**ReAct（准确可靠）**：
```
用户：15 乘以 7 再加 20 等于多少？

Thought: 我需要先计算 15×7，然后加 20
Action: calculate(expression="15 * 7")
Observation: 105
Thought: 现在我知道 15×7=105，需要加 20
Action: calculate(expression="105 + 20")
Observation: 125
Thought: 我得到了最终答案
Final Answer: 15 乘以 7 再加 20 等于 125
```

#### 案例 2：信息查询

**直接回答（可能幻觉）**：
```
用户：Python 是什么时候创建的？
LLM: Python 是由 John Smith 于 1995 年创建的。（幻觉）
```

**ReAct（事实验证）**：
```
用户：Python 是什么时候创建的？

Thought: 我需要查询 Python 创建的确切信息
Action: search(query="Python programming language created when")
Observation: Python was created by Guido van Rossum and first released in 1991.
Thought: 我现在有了准确信息
Final Answer: Python 由 Guido van Rossum 创建，于 1991 年首次发布。
```

#### 案例 3：多步骤任务

**没有 ReAct（混乱）**：
```
用户：Python 创始人今年的年龄是多少？
LLM: 大约 60 岁。（模糊且可能错误）
```

**ReAct（有序分解）**：
```
用户：Python 创始人今年的年龄是多少？

Thought: 我需要先找出 Python 创始人是谁，然后查询他的出生年份，最后计算年龄
Action: search(query="Python programming language creator")
Observation: Python was created by Guido van Rossum.
Thought: Python 创始人是 Guido van Rossum，现在需要他的出生年份
Action: search(query="Guido van Rossum birth year")
Observation: Guido van Rossum was born on January 31, 1956.
Thought: 他出生于 1956 年，现在是 2024 年，需要计算 2024-1956
Action: calculate(expression="2024 - 1956")
Observation: 68
Thought: 我得到了最终答案
Final Answer: Python 创始人 Guido van Rossum 今年 68 岁。
```

---

## 2.3 ReAct 核心组件详解

### 2.3.1 Thought（思考）

**Thought 的作用**：
- 分析当前状态和问题
- 评估已有信息是否足够
- 规划下一步行动（使用哪个工具或给出答案）

**如何生成好的 Thought**：
1. 明确当前目标
2. 分析已有信息
3. 识别信息缺口
4. 决定下一步行动

**Thought 的格式和风格**：
- 使用自然语言
- 简洁但完整
- 包含推理过程

**示例：不同场景下的 Thought 样例**

```
# 数学计算场景
Thought: 这个表达式需要先计算乘法再加法，我分两步计算。

# 信息查询场景
Thought: 我不知道这个事实，需要搜索可靠信息源。

# 多步骤任务场景
Thought: 要回答这个问题，我需要先找出 A，然后用 A 查询 B，最后计算结果。
```

### 2.3.2 Action（行动）

**Action 的定义**：
- 工具调用：执行具体操作（计算、搜索、文件读写等）
- 最终答案：当有足够信息时给出回答

**Action 的格式**（OpenAI Function Calling 格式）：
```json
{
  "id": "call_abc123",
  "type": "function",
  "function": {
    "name": "calculator",
    "arguments": {
      "expression": "15 * 7"
    }
  }
}
```

**如何设计 Action 空间**：
1. 根据任务需求选择工具
2. 工具描述要清晰具体
3. 参数定义要准确（使用 JSON Schema）

**示例：各种类型的 Action**

```python
# 计算动作
Action: calculate(expression="105 + 20")

# 搜索动作
Action: search(query="Python creator birth year")

# 文件读取动作
Action: read_file(path="data/config.json")

# 最终答案
Final Answer: 15 乘以 7 再加 20 等于 125。
```

### 2.3.3 Observation（观察）

**Observation 的来源**：
- 工具执行结果（成功或失败）
- 外部环境反馈

**如何格式化 Observation**：
- 成功时：返回工具输出
- 失败时：返回错误信息，指导 LLM 调整策略

**错误处理：当工具失败时**
```
Observation: 工具执行失败：文件不存在。请尝试其他路径或使用默认值。
```

**示例：成功和失败的 Observation**

```
# 成功的 Observation
Observation: 105
Observation: Python was created by Guido van Rossum in 1991.

# 失败的 Observation
Observation: Error: 除零错误。请检查表达式。
Observation: Error: 未找到相关信息。请尝试其他关键词。
```

### 2.3.4 状态管理

**Agent 状态机**：

```
IDLE → THINKING → EXECUTING → WAITING → COMPLETE/ERROR
  ↑       ↓           ↓           ↓
  └───────┴───────────┴───────────┘
          (循环直到完成)
```

**状态说明**：

| 状态 | 说明 |
|------|------|
| `IDLE` | 空闲，等待任务 |
| `THINKING` | 正在调用 LLM 进行思考 |
| `EXECUTING` | 正在执行工具调用 |
| `WAITING` | 等待工具结果 |
| `COMPLETE` | 任务完成 |
| `ERROR` | 发生错误 |

**如何追踪执行历史**：
```python
# 使用列表记录每个迭代的信息
trace = []
trace.append({
    "iteration": 1,
    "thought": "...",
    "action": "...",
    "observation": "..."
})
```

**如何判断任务完成**：
1. LLM 输出包含 "Final Answer" 标识
2. 没有更多工具调用
3. 达到最大迭代次数（强制结束）

---

## 2.4 从零实现 ReAct Agent

### 2.4.1 项目结构

```
react-agent/
├── agent/
│   ├── base.py          # 基础 Agent 类
│   ├── react.py         # ReAct Agent 实现
│   └── __init__.py
├── llm/
│   ├── client.py        # LLM 客户端
│   └── types.py         # 类型定义
├── tools/
│   ├── base.py          # 工具基类
│   ├── registry.py      # 工具注册表
│   └── builtins.py      # 内置工具
├── memory/
│   ├── short_term.py    # 短期记忆
│   └── __init__.py
├── config.py            # 配置
├── main.py              # 入口
└── requirements.txt
```

### 2.4.2 核心类型定义（完整代码 + 注释）

```python
# llm/types.py
from dataclasses import dataclass
from typing import List, Dict, Any, Optional

@dataclass
class Message:
    """消息类"""
    role: str  # "system", "user", "assistant"
    content: str

@dataclass
class ToolCall:
    """工具调用类"""
    id: str
    name: str
    arguments: Dict[str, Any]

@dataclass
class LLMResponse:
    """LLM 响应类"""
    content: str
    tool_calls: List[ToolCall]
    finish_reason: str
```

### 2.4.3 工具基类实现（完整代码 + 注释）

```python
# tools/base.py
from abc import ABC, abstractmethod
from typing import Any, Dict

class BaseTool(ABC):
    """工具基类"""

    @property
    @abstractmethod
    def name(self) -> str:
        """工具名称"""
        pass

    @property
    @abstractmethod
    def description(self) -> str:
        """工具描述"""
        pass

    @property
    @abstractmethod
    def parameters(self) -> Dict:
        """JSON Schema 参数定义"""
        pass

    @abstractmethod
    def execute(self, **kwargs) -> Any:
        """执行工具"""
        pass

    def to_openai_format(self) -> Dict:
        """转换为 OpenAI Function Calling 格式"""
        return {
            "type": "function",
            "function": {
                "name": self.name,
                "description": self.description,
                "parameters": self.parameters
            }
        }
```

### 2.4.4 工具注册表实现（完整代码 + 注释）

```python
# tools/registry.py
from typing import List, Optional
from .base import BaseTool

class ToolRegistry:
    """工具注册表"""

    def __init__(self):
        self._tools: Dict[str, BaseTool] = {}

    def register(self, tool: BaseTool):
        """注册工具"""
        self._tools[tool.name] = tool
        print(f"工具 '{tool.name}' 已注册")

    def get(self, name: str) -> Optional[BaseTool]:
        """获取工具"""
        return self._tools.get(name)

    def list_tools(self) -> List[Dict]:
        """列出所有工具（OpenAI 格式）"""
        return [tool.to_openai_format() for tool in self._tools.values()]

    def execute(self, name: str, **kwargs) -> Any:
        """执行工具"""
        tool = self.get(name)
        if not tool:
            raise ValueError(f"工具 '{name}' 不存在")
        return tool.execute(**kwargs)
```

### 2.4.5 记忆系统实现（完整代码 + 注释）

```python
# memory/short_term.py
from typing import List
from dataclasses import dataclass, field

@dataclass
class Message:
    role: str
    content: str

class ShortTermMemory:
    """短期记忆 - 滑动窗口实现"""

    def __init__(self, max_messages: int = 50):
        self.max_messages = max_messages
        self._messages: List[Message] = []

    def add_message(self, role: str, content: str):
        """添加消息"""
        self._messages.append(Message(role=role, content=content))
        # 滑动窗口：超过限制时移除最旧的消息
        while len(self._messages) > self.max_messages:
            self._messages.pop(0)

    def get_messages(self) -> List[Message]:
        """获取所有消息"""
        return self._messages

    def to_openai_messages(self) -> List[Dict]:
        """转换为 OpenAI 格式"""
        return [{"role": m.role, "content": m.content} for m in self._messages]

    def clear(self):
        """清空记忆"""
        self._messages.clear()
```

### 2.4.6 LLM 客户端实现（完整代码 + 注释）

```python
# llm/client.py
import os
import json
from openai import OpenAI
from typing import List, Dict
from .types import LLMResponse, ToolCall

class LLMClient:
    """LLM 客户端 - 支持 OpenAI 兼容 API"""

    def __init__(
        self,
        api_key: str = None,
        base_url: str = "https://dashscope.aliyuncs.com/compatible-mode/v1",
        model: str = "qwen-plus",
    ):
        self.api_key = api_key or os.getenv("DASHSCOPE_API_KEY")
        self.base_url = base_url
        self.model = model
        self._client = OpenAI(api_key=self.api_key, base_url=self.base_url)

    def chat(
        self,
        messages: List[Dict],
        tools: List[Dict] = None,
        temperature: float = 0.7,
    ) -> LLMResponse:
        """发送聊天请求"""
        kwargs = {
            "model": self.model,
            "messages": messages,
            "temperature": temperature,
        }

        if tools:
            kwargs["tools"] = tools
            kwargs["tool_choice"] = "auto"

        response = self._client.chat.completions.create(**kwargs)
        choice = response.choices[0]
        message = choice.message

        # 解析工具调用
        tool_calls = []
        if message.tool_calls:
            for tc in message.tool_calls:
                tool_calls.append(ToolCall(
                    id=tc.id,
                    name=tc.function.name,
                    arguments=json.loads(tc.function.arguments),
                ))

        return LLMResponse(
            content=message.content or "",
            tool_calls=tool_calls,
            finish_reason=choice.finish_reason,
        )
```

### 2.4.7 ReAct Agent 核心实现（完整代码 + 详细注释）

```python
# agent/react.py
import time
from typing import List, Optional
from .base import BaseAgent
from ..llm.client import LLMClient
from ..llm.types import LLMResponse
from ..tools.registry import ToolRegistry
from ..memory.short_term import ShortTermMemory

class ReActAgent(BaseAgent):
    """ReAct Agent 实现"""

    def __init__(
        self,
        llm_client: LLMClient,
        tools: ToolRegistry,
        memory: ShortTermMemory = None,
        system_prompt: str = None,
        max_iterations: int = 10,
        verbose: bool = False,
    ):
        self.llm_client = llm_client
        self.tools = tools
        self.memory = memory or ShortTermMemory()
        self.system_prompt = system_prompt or self._default_system_prompt()
        self.max_iterations = max_iterations
        self.verbose = verbose

        # 状态追踪
        self.iteration = 0
        self._trace: List[dict] = []

    def _default_system_prompt(self) -> str:
        """默认系统提示词"""
        return """你是一个使用 ReAct 框架的 AI 助手。

你遵循以下循环：
1. Thought（思考）：分析当前情况，决定下一步
2. Action（行动）：执行工具调用或给出最终答案
3. Observation（观察）：观察工具执行结果

当你需要获取信息或执行计算时，使用可用工具。
当你有足够信息时，给出最终答案。

可用工具：
{tools}

请用中文思考和回答。
"""

    def run(self, query: str, **kwargs) -> "AgentResult":
        """运行 ReAct 循环"""
        # 重置状态
        self.iteration = 0
        self._trace = []

        # 添加系统提示词
        tools_desc = "\n".join([
            f"- {tool.name}: {tool.description}"
            for tool in self.tools._tools.values()
        ])
        self.memory.add_message("system", self.system_prompt.format(tools=tools_desc))

        # 添加用户查询
        self.memory.add_message("user", query)

        if self.verbose:
            print(f"\n{'='*50}")
            print(f"开始执行 ReAct Agent")
            print(f"查询：{query}")
            print(f"{'='*50}\n")

        # ReAct 主循环
        while self.iteration < self.max_iterations:
            self.iteration += 1

            if self.verbose:
                print(f"\n--- Iteration {self.iteration} ---\n")

            # Step 1: Thought - 调用 LLM 获取响应
            if self.verbose:
                print("【Thought】调用 LLM 进行思考...")

            response = self._call_llm()

            # 记录追踪信息
            self._trace.append({
                "iteration": self.iteration,
                "thought": response.content,
                "tool_calls": response.tool_calls,
            })

            if self.verbose:
                print(f"思考内容：{response.content[:200]}...")

            # Step 2: 检查是否有最终答案
            if "Final Answer" in response.content or not response.tool_calls:
                if self.verbose:
                    print("\n【Final Answer】获得最终答案")

                return AgentResult(
                    success=True,
                    output=response.content,
                    iterations=self.iteration,
                    trace=self._trace,
                )

            # Step 3: Action - 执行工具调用
            tool_call = response.tool_calls[0]

            if self.verbose:
                print(f"\n【Action】调用工具：{tool_call.name}")
                print(f"参数：{tool_call.arguments}")

            try:
                result = self.tools.execute(tool_call.name, **tool_call.arguments)

                if self.verbose:
                    print(f"工具返回：{result}")

                # Step 4: Observation - 记录结果到记忆
                self.memory.add_message("assistant", response.content)
                self._add_tool_result_to_memory(tool_call, result)

                self._trace[-1]["observation"] = str(result)

            except Exception as e:
                if self.verbose:
                    print(f"工具执行错误：{e}")

                # 错误处理：将错误信息反馈给 LLM
                error_msg = f"工具执行失败：{str(e)}。请尝试其他方法。"
                self.memory.add_message("user", error_msg)
                self._trace[-1]["observation"] = error_msg

        # 超过最大迭代次数
        return AgentResult(
            success=False,
            error=f"超过最大迭代次数 ({self.max_iterations})",
            iterations=self.iteration,
            trace=self._trace,
        )

    def _call_llm(self) -> LLMResponse:
        """调用 LLM"""
        messages = self.memory.to_openai_messages()
        tools = self.tools.list_tools()

        return self.llm_client.chat(messages=messages, tools=tools)

    def _add_tool_result_to_memory(self, tool_call, result):
        """将工具结果添加到记忆"""
        # 有些实现会使用 tool role，这里简化处理
        self.memory.add_message("user", f"工具 {tool_call.name} 返回：{result}")

    def get_trace(self) -> List[dict]:
        """获取执行追踪"""
        return self._trace
```

### 2.4.8 Agent 结果类（完整代码）

```python
# agent/base.py
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class AgentResult:
    """Agent 执行结果"""
    success: bool
    output: str = None
    error: str = None
    iterations: int = 0
    trace: List[dict] = None

class BaseAgent:
    """Agent 基类"""
    def run(self, query: str, **kwargs) -> AgentResult:
        raise NotImplementedError
```

---

## 2.5 完整运行示例

### 2.5.1 环境准备

```bash
# 创建项目目录
mkdir react-agent-demo && cd react-agent-demo

# 创建虚拟环境
python -m venv venv
venv\Scripts\activate  # Windows
source venv/bin/activate  # Linux/Mac

# 安装依赖
pip install openai
```

### 2.5.2 配置文件

```python
# config.py
import os

# LLM 配置
LLM_CONFIG = {
    "api_key": os.getenv("DASHSCOPE_API_KEY", "your-api-key"),
    "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
    "model": "qwen-plus",
}

# Agent 配置
AGENT_CONFIG = {
    "max_iterations": 10,
    "verbose": True,
}
```

### 2.5.3 内置工具实现

```python
# tools/builtins.py
import json
import math
from .base import BaseTool

class CalculatorTool(BaseTool):
    """计算器工具"""

    @property
    def name(self):
        return "calculator"

    @property
    def description(self):
        return "计算数学表达式。支持加减乘除、幂运算、三角函数等。"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，如 '2 + 3 * 4' 或 'sin(pi/2)'"
                }
            },
            "required": ["expression"]
        }

    def execute(self, expression: str):
        # 安全的表达式求值
        allowed_names = {
            "abs": abs, "round": round,
            "sin": math.sin, "cos": math.cos, "tan": math.tan,
            "sqrt": math.sqrt, "log": math.log, "log10": math.log10,
            "pi": math.pi, "e": math.e,
        }
        try:
            result = eval(expression, {"__builtins__": {}}, allowed_names)
            return result
        except Exception as ex:
            return f"计算错误：{ex}"


class SearchTool(BaseTool):
    """搜索工具（模拟）"""

    @property
    def name(self):
        return "search"

    @property
    def description(self):
        return "搜索信息。用于查找事实性知识。"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索关键词"
                }
            },
            "required": ["query"]
        }

    def execute(self, query: str):
        # 模拟搜索结果
        mock_results = {
            "python": "Python 是一种高级编程语言，由 Guido van Rossum 于 1991 年创建。",
            "react": "ReAct 是一种 AI 框架，结合推理（Reasoning）和行动（Acting）。",
            "北京": "北京是中国的首都，人口约 2154 万（2020 年）。",
        }
        return mock_results.get(query.lower(), f"未找到关于 '{query}' 的信息")
```

### 2.5.4 完整运行脚本

```python
# main.py
from agent.react import ReActAgent
from llm.client import LLMClient
from tools.registry import ToolRegistry
from tools.builtins import CalculatorTool, SearchTool
from memory.short_term import ShortTermMemory

def main():
    # 初始化组件
    llm = LLMClient()
    tools = ToolRegistry()
    tools.register(CalculatorTool())
    tools.register(SearchTool())
    memory = ShortTermMemory(max_messages=50)

    # 创建 Agent
    agent = ReActAgent(
        llm_client=llm,
        tools=tools,
        memory=memory,
        max_iterations=10,
        verbose=True,
    )

    # 运行示例
    print("\n" + "="*60)
    print("示例 1：数学计算")
    print("="*60)
    result = agent.run("15 乘以 7 再加 23 等于多少？")
    print(f"\n最终答案：{result.output}")

    print("\n" + "="*60)
    print("示例 2：信息查询")
    print("="*60)
    result = agent.run("Python 是什么时候创建的？")
    print(f"\n最终答案：{result.output}")

    print("\n" + "="*60)
    print("示例 3：复杂问题")
    print("="*60)
    result = agent.run("Python 的创始人今年多少岁？")
    print(f"\n最终答案：{result.output}")

    # 显示执行追踪
    print("\n" + "="*60)
    print("执行追踪")
    print("="*60)
    for i, step in enumerate(result.trace, 1):
        print(f"\n--- Step {i} ---")
        print(f"Thought: {step['thought'][:100]}...")
        if step.get('tool_calls'):
            tc = step['tool_calls'][0]
            print(f"Action: {tc.name}({tc.arguments})")
        if step.get('observation'):
            print(f"Observation: {step['observation'][:100]}...")

if __name__ == "__main__":
    main()
```

### 2.5.5 参考输出样例

```
============================================================
示例 1：数学计算
============================================================

开始执行 ReAct Agent
查询：15 乘以 7 再加 23 等于多少？
============================================================

--- Iteration 1 ---

【Thought】调用 LLM 进行思考...
思考内容：我需要计算 15×7，然后加 23。让我先计算 15×7。

【Action】调用工具：calculator
参数：{'expression': '15 * 7'}
工具返回：105

--- Iteration 2 ---

【Thought】调用 LLM 进行思考...
思考内容：现在我知道 15×7=105，接下来需要加 23。

【Action】调用工具：calculator
参数：{'expression': '105 + 23'}
工具返回：128

--- Iteration 3 ---

【Thought】调用 LLM 进行思考...
思考内容：我现在知道了最终答案。

【Final Answer】获得最终答案

最终答案：15 乘以 7 再加 23 等于 128。

============================================================
执行追踪
============================================================

--- Step 1 ---
Thought: 我需要计算 15×7，然后加 23。让我先计算 15×7。
Action: calculator({'expression': '15 * 7'})
Observation: 105

--- Step 2 ---
Thought: 现在我知道 15×7=105，接下来需要加 23。
Action: calculator({'expression': '105 + 23'})
Observation: 128

--- Step 3 ---
Thought: 我现在知道了最终答案。
```

---

## 2.6 提示词工程详解

### 2.6.1 系统提示词的作用

系统提示词是 ReAct Agent 的"操作手册"，它定义了：
- **Agent 行为模式**：告诉 LLM 如何思考和行动
- **指定可用工具**：列出所有可调用的工具及其用途
- **设置输出格式**：规范 Thought、Action、Observation 的格式
- **约束行为边界**：设定最大迭代次数、错误处理规则等

### 2.6.2 提示词模板设计

```python
REACT_SYSTEM_PROMPT = """
你是一个使用 ReAct 框架的智能助手。

## 工作流程

你遵循以下循环进行推理和行动：

1. **Thought（思考）**：分析当前情况，推理下一步该做什么
   - 理解用户问题
   - 评估已有信息
   - 决定是否需要工具

2. **Action（行动）**：执行具体操作
   - 如果需要信息/计算：调用工具
   - 如果有足够信息：给出 Final Answer

3. **Observation（观察）**：查看行动结果
   - 工具返回了什么
   - 是否解决了问题
   - 是否需要进一步行动

## 可用工具

{tools_description}

## 输出格式

思考时使用：
Thought: [你的推理过程]

需要工具时使用：
[调用相应工具]

有最终答案时使用：
Thought: 我现在有足够信息回答用户问题了。
Final Answer: [完整、友好的回答]

## 指导原则

1. 逐步思考，不要跳跃
2. 不确定时使用工具
3. 工具失败时尝试其他方法
4. 最终答案要完整、准确、友好
5. 最多进行 {max_iterations} 次迭代

让我们开始！
"""
```

### 2.6.3 Few-Shot 示例（来自 ReAct 论文）

ReAct 论文中使用的 few-shot 示例格式：

```
Question: What is the elevation range for the area that the eastern sector of the
Colorado orogeny extends into?
Thought 1: I need to search Colorado orogeny, find the area that the eastern sector
of the Colorado orogeny extends into, then find the elevation range of the area.
Action 1: Search[Colorado orogeny]
Observation 1: The Colorado orogeny was an episode of mountain building (an orogeny) in
Colorado and surrounding areas.
Thought 2: It does not mention the eastern sector. So I need to look up eastern sector.
Action 2: Lookup[eastern sector]
Observation 2: (Result 1 / 1) The eastern sector extends into the High Plains and is called
the Central Plains orogeny.
Thought 3: The eastern sector of Colorado orogeny extends into the High Plains. So I
need to search High Plains and find its elevation range.
Action 3: Search[High Plains]
Observation 3: High Plains refers to one of two distinct land regions
Thought 4: I need to instead search High Plains (United States).
Action 4: Search[High Plains (United States)]
Observation 4: The High Plains are a subregion of the Great Plains. From east to west, the
High Plains rise in elevation from around 1,800 to 7,000 ft (550 to 2,130 m).[3]
Thought 5: High Plains rise in elevation from around 1,800 to 7,000 ft, so the answer
is 1,800 to 7,000 ft.
Action 5: Finish[1,800 to 7,000 ft]
```

### 2.6.4 LangChain ReAct 示例

LangChain 中的 ReAct Agent 实现示例：

```python
from langchain.llms import OpenAI
from langchain.agents import load_tools, initialize_agent

# 初始化 LLM
llm = OpenAI(model_name="text-davinci-003", temperature=0)

# 加载工具（搜索 + 计算器）
tools = load_tools(["google-serper", "llm-math"], llm=llm)

# 初始化 ReAct Agent
agent = initialize_agent(
    tools,
    llm,
    agent="zero-shot-react-description",
    verbose=True
)

# 运行查询
agent.run("Who is Olivia Wilde's boyfriend? What is his current age raised to the 0.23 power?")
```

执行追踪输出：
```
> Entering new AgentExecutor chain...
I need to find out who Olivia Wilde's boyfriend is and then calculate his age raised to the 0.23 power.
Action: Search
Action Input: "Olivia Wilde boyfriend"
Observation: Olivia Wilde started dating Harry Styles after ending her years-long engagement to Jason Sudeikis.
Thought: I need to find out Harry Styles' age.
Action: Search
Action Input: "Harry Styles age"
Observation: 29 years
Thought: I need to calculate 29 raised to the 0.23 power.
Action: Calculator
Action Input: 29^0.23
Observation: Answer: 2.169459462491557
Thought: I now know the final answer.
Final Answer: Harry Styles, Olivia Wilde's boyfriend, is 29 years old and his age raised to the 0.23 power is 2.169459462491557.
> Finished chain.
```

### 2.6.5 提示词优化技巧

| 技巧 | 说明 | 示例 |
|------|------|------|
| **具体化 vs 泛化** | 工具描述要具体，包含使用场景和示例 | "计算数学表达式，如 '2 + 3 * 4'" |
| **使用示例（few-shot prompting）** | 提供 1-3 个完整示例效果最佳 | 在提示词中包含完整的 Thought-Action-Observation 示例 |
| **添加约束条件** | 限制最大迭代次数，防止无限循环 | "最多进行 10 次迭代" |
| **设置明确的结束条件** | Final Answer 格式要清晰标识 | "Final Answer: [你的答案]" |
| **错误处理指导** | 告诉 LLM 工具失败时如何应对 | "如果工具失败，尝试其他方法或向用户求助" |

### 2.6.6 针对不同任务的提示词设计

| 任务类型 | 设计要点 | 示例 |
|----------|----------|------|
| **知识密集型任务**（如 HotpotQA） | 使用多个 thought-action-observation 步骤，鼓励信息检索 | "当不确定时，使用搜索工具验证" |
| **决策型任务**（如 ALFWorld） | 稀疏使用 thought，重点在行动执行 | "直接执行可行的行动，遇到障碍时再思考" |
| **复杂推理任务** | 结合 CoT，先内部推理再外部验证 | "先分析问题结构，再决定使用哪些工具" |

---

## 2.7 调试与问题排查

### 2.7.1 常见问题及解决方案

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| Agent 不调用工具 | 提示词不明确/工具描述模糊 | 优化工具描述，添加使用场景示例 |
| 无限循环 | 结束条件不清晰 | 设置 max_iterations，改进提示词 |
| 工具参数错误 | LLM 不理解参数格式 | 添加参数说明和示例 |
| 回答不完整 | 过早结束 | 改进 Final Answer 的提示 |
| 工具选择错误 | 工具描述重叠或不清晰 | 明确每个工具的专属使用场景 |
| Observation 被忽略 | 记忆管理问题 | 检查是否正确将 Observation 添加到对话历史 |

### 2.7.2 调试技巧

1. **启用 verbose 模式**
   ```python
   agent = ReActAgent(..., verbose=True)
   ```

2. **打印完整消息历史**
   ```python
   for msg in agent.memory.get_messages():
       print(f"{msg.role}: {msg.content}")
   ```

3. **记录 LLM 原始响应**
   ```python
   logging.basicConfig(level=logging.DEBUG)
   ```

4. **使用追踪可视化**
   ```python
   trace = agent.get_trace()
   for step in trace:
       print(f"Step {step['iteration']}: {step['thought'][:50]}...")
   ```

### 2.7.3 日志系统

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('agent.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger('ReActAgent')

# 在 Agent 中使用
logger.info(f"Starting iteration {iteration}")
logger.debug(f"Thought: {thought}")
logger.error(f"Tool execution failed: {error}")
```

---

## 2.8 进阶话题

### 2.8.1 多工具协调

当 Agent 有多个工具可用时，需要解决：

**工具选择策略**：
- 基于语义匹配：选择与当前目标最相关的工具
- 基于历史经验：优先使用过去成功的工具
- 基于成本考虑：优先使用低成本工具

**依赖关系处理**：
```python
# 某些工具需要其他工具的输出作为输入
# 例如：先搜索获取信息，再计算处理数据
```

**并行执行**：
```python
# 对于独立的工具调用，可以并行执行
# 例如：同时搜索多个关键词
```

### 2.8.2 错误恢复

**超时处理**：
```python
import signal

def timeout_handler(signum, frame):
    raise TimeoutError("Tool execution timed out")

signal.signal(signal.SIGALRM, timeout_handler)
signal.alarm(30)  # 30 秒超时
```

**重试机制**：
```python
def execute_with_retry(tool, args, max_retries=3):
    for i in range(max_retries):
        try:
            return tool.execute(**args)
        except Exception as e:
            if i == max_retries - 1:
                raise
            time.sleep(2 ** i)  # 指数退避
```

**降级策略**：
```python
# 当主要工具失败时，使用备用方案
try:
    result = search_tool.execute(query)
except:
    result = fallback_search_tool.execute(query)
```

### 2.8.3 性能优化

**缓存工具结果**：
```python
from functools import lru_cache

class CachedTool(BaseTool):
    def __init__(self, wrapped_tool):
        self.wrapped = wrapped_tool
        self._cache = {}

    def execute(self, **kwargs):
        key = str(sorted(kwargs.items()))
        if key in self._cache:
            return self._cache[key]
        result = self.wrapped.execute(**kwargs)
        self._cache[key] = result
        return result
```

**批量工具调用**：
```python
# 将多个相似的查询合并为一次调用
# 例如：批量搜索多个关键词
```

**流式响应**：
```python
# 对于长任务，使用流式输出逐步返回结果
for chunk in agent.stream_run(query):
    print(chunk, end='')
```

### 2.8.4 安全考虑

**输入验证**：
```python
def validate_tool_input(tool_name: str, input_data: dict) -> bool:
    # 检查输入是否符合预期格式
    # 防止注入攻击
    pass
```

**权限控制**：
```python
class PermissionedTool(BaseTool):
    def __init__(self, required_permission: str):
        self.required_permission = required_permission

    def execute(self, user_permissions: list, **kwargs):
        if self.required_permission not in user_permissions:
            return ToolResult(status=ERROR, error="Permission denied")
        # 继续执行
```

**敏感操作确认**：
```python
# 对于删除文件、发送消息等操作，要求用户确认
if tool.requires_confirmation:
    confirm = ask_user("确认执行此操作？(y/n)")
    if confirm != 'y':
        return "操作已取消"
```

---

## 2.9 实践练习

### 练习 1：实现天气查询 Agent

创建一个能够查询天气的 ReAct Agent：

1. 创建 `WeatherTool` 类（可以使用模拟数据或真实 API）
2. 将工具注册到 Agent
3. 测试各种查询，如"北京今天天气怎么样？"

```python
class WeatherTool(BaseTool):
    @property
    def name(self):
        return "weather"

    @property
    def description(self):
        return "查询指定城市的天气情况"

    @property
    def parameters(self):
        return {
            "type": "object",
            "properties": {
                "city": {"type": "string", "description": "城市名称"},
                "date": {"type": "string", "description": "日期（可选）"}
            },
            "required": ["city"]
        }

    def execute(self, city: str, date: str = None):
        # 实现天气查询逻辑
        pass
```

### 练习 2：实现多轮对话

让 Agent 能够保持上下文，处理连续问题：

1. 使用 `ConversationMemory` 保持对话历史
2. 测试指代消解（如"它"、"那个"）
3. 处理连续问题

```python
memory = ConversationMemory(max_messages=100)
agent = ReActAgent(..., memory=memory)

# 第一轮
result1 = agent.run("Python 是什么时候创建的？")
print(result1.output)

# 第二轮 - Agent 应该知道"它"指的是 Python
result2 = agent.run("它的创始人现在还活跃吗？")
print(result2.output)
```

### 练习 3：自定义提示词

设计一个专业领域的 ReAct Agent：

1. 选择一个专业领域（如医疗、法律、金融）
2. 设计领域专用的系统提示词
3. 调整语气和风格
4. 对比效果差异

```python
MEDICAL_AGENT_PROMPT = """
你是一个医疗咨询助手，使用 ReAct 框架提供准确的医疗信息。

指导原则：
1. 对于医疗问题，始终建议咨询专业医生
2. 使用可靠的医疗信息源进行查询
3. 避免给出诊断建议，仅提供信息参考

可用工具：
{tools_description}

...
"""
```

---

## 参考资料

### 核心论文

1. **ReAct 原论文**: Yao et al., 2022. "ReAct: Synergizing Reasoning and Acting in Language Models"
   - https://arxiv.org/abs/2210.03629
   - 核心发现：ReAct 在 HotpotQA、FEVER、ALFWorld、WebShop 等基准测试上表现出色
   - ReAct 结合 CoT+Self-Consistency 通常超越所有其他方法

### 技术文章

2. **Prompt Engineering Guide - ReAct**
   - https://www.promptingguide.ai/techniques/react
   - 详细的 ReAct 工作原理和 LangChain 使用示例

3. **ReAct Pattern: Interleaving Reasoning and Action**
   - https://mbrenndoerfer.com/writing/react-pattern-llm-reasoning-action-agents
   - 深入讲解 ReAct 模式的实现细节

### 其他推理方法对比

4. **Chain of Thought (CoT)**: Wei et al., 2022 - 逐步推理但无法获取外部信息
5. **Tree of Thoughts (ToT)**: 多路径探索，计算成本高
6. **Program-Aided Language Models (PAL)**: 使用代码执行进行推理

### 实验结果摘要

| 任务类型 | 方法对比 | 结果 |
|----------|----------|------|
| HotpotQA (问答) | ReAct > Act, CoT > ReAct (部分) | ReAct 减少事实幻觉 |
| FEVER (事试验证) | ReAct > CoT, ReAct > Act | ReAct 表现最佳 |
| ALFWorld (决策) | ReAct > Act | ReAct 能更好分解目标 |
| WebShop (决策) | ReAct > Act | 但仍不如专家人类 |

### 关键洞察

- **CoT 的问题**: 容易事实幻觉，无法获取外部信息
- **Act-only 的问题**: 无法正确分解子目标
- **ReAct 的优势**: 结合内部知识和外部信息检索
- **ReAct 的局限**: 结构约束降低灵活性，依赖检索质量

---

## 小测验

1. ReAct 模式的三个核心步骤是什么？
2. 为什么 ReAct 比直接让 LLM 回答更准确？
3. 如何防止 Agent 陷入无限循环？
4. 在设计工具描述时，应该注意什么？
5. Few-shot prompting 对 ReAct 有什么帮助？

---

## 下一步

掌握 ReAct 模式后，继续学习 [工具系统](/docs/03_tools.md)，深入了解如何创建和管理工具。
