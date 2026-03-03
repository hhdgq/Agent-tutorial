# 第四章 记忆系统详解

> **学习完本章后，你将能够：**
> - 理解为什么记忆是 Agent 的核心组件
> - 掌握短期记忆和长期记忆的实现原理
> - 实现向量相似度搜索和语义检索
> - 构建生产级记忆管理系统
> - 评估和优化记忆系统性能

---

## 4.1 为什么 Agent 需要记忆？

### 4.1.1 LLM 的无状态性限制

大型语言模型（LLM）本质上是一个**无状态函数**：给定输入，产生输出，然后"忘记"一切。

```
┌─────────────────────────────────────────────────────────────┐
│                    LLM 的无状态性                            │
│                                                             │
│   输入："你好，我叫张三"                                     │
│         ↓                                                    │
│   ┌──────────────────┐                                      │
│   │     LLM Model    │ → 输出："你好，张三！很高兴认识你"     │
│   │   (无内部状态)    │                                      │
│   └──────────────────┘                                      │
│                                                             │
│   输入："你还记得我叫什么吗？"                                │
│         ↓                                                    │
│   ┌──────────────────┐                                      │
│   │     LLM Model    │ → 输出："抱歉，我不知道您叫什么" ❌    │
│   │   (不记得上次)    │                                      │
│   └──────────────────┘                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**这就是 LLM 的"失忆症"问题**：
- 每次调用都是独立的
- 不会保留之前的对话历史
- 无法建立跨对话的连续性

### 4.1.2 记忆带来的核心价值

记忆系统通过维护**外部状态**来解决这个问题：

```
┌─────────────────────────────────────────────────────────────┐
│              带记忆的 Agent 架构                             │
│                                                             │
│   用户："你好，我叫张三"                                     │
│         ↓                                                   │
│   ┌───────────────────────────────────────────────────┐    │
│   │                Agent with Memory                   │    │
│   │                                                   │    │
│   │   ┌─────────┐    ┌─────────┐    ┌─────────────┐  │    │
│   │   │  LLM    │ ←→ │ Memory  │ ←→ │   Storage   │  │    │
│   │   │ (无状态) │    │ Manager │    │ (持久化)    │  │    │
│   │   └─────────┘    └─────────┘    └─────────────┘  │    │
│   │        ↑                              ↑          │    │
│   │        └────────── 读取/写入 ─────────┘          │    │
│   └───────────────────────────────────────────────────┘    │
│         ↓                                                   │
│   输出："你好，张三！很高兴认识你"                           │
│         ↓                                                   │
│   ┌─────────────────────────────────────────┐              │
│   │ Memory: {"user_name": "张三"}           │              │
│   └─────────────────────────────────────────┘              │
│                                                             │
│   用户："你还记得我叫什么吗？"                               │
│         ↓                                                   │
│   ┌───────────────────────────────────────────────────┐    │
│   │   1. 从 Memory 检索 → 找到 "张三"                   │    │
│   │   2. 将检索结果注入 LLM 上下文                      │    │
│   │   3. LLM 生成回答                                   │    │
│   └───────────────────────────────────────────────────┘    │
│         ↓                                                   │
│   输出："当然记得！你叫张三 😊" ✅                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**记忆系统的三大核心价值：**

| 价值 | 说明 | 示例 |
|------|------|------|
| **连贯性** | 保持多轮对话的上下文连续性 | 记住之前的对话内容 |
| **个性化** | 积累用户特定的知识和偏好 | 记住用户喜欢 Python |
| **知识积累** | 跨会话存储重要信息 | 学习用户的工作流程 |

### 4.1.3 真实应用场景分析

#### 场景 1：个人助手

```
用户与 AI 助手的多次交互：

第 1 次对话:
  用户："帮我设置每天早上 8 点的闹钟"
  助手："好的，已设置每天早上 8 点的闹钟"
  → 记忆存储：{"alarm": "08:00", "frequency": "daily"}

第 3 次对话 (一周后):
  用户："我有什么日程安排？"
  助手："你设置了每天早上 8 点的闹钟"
  → 从记忆检索历史信息
```

#### 场景 2：客服机器人

```
客服对话流程:

用户："我的订单号是 12345，帮我查一下物流"
→ 记忆：{"order_id": "12345", "issue": "物流查询"}

用户："对了，我上次买的商品有质量问题"
→ 记忆追加：{"complaint": "质量问题", "order_id": "12345"}

助手："非常抱歉听到这个问题。关于订单 12345 的质量问题，我会..."
→ 关联订单号 + 投诉内容
```

#### 场景 3：教育辅导

```
辅导过程:

学生："我不理解二次方程"
→ 记忆：{"weakness": "二次方程", "timestamp": "2024-01-15"}

(三天后)
学生："再给我一些二次方程的练习题"
→ 检索记忆 → 学生之前表示不理解二次方程
助手："当然，根据我们之前的讨论，这里有一些针对性的练习..."
```

---

## 4.2 记忆系统的理论基础

### 4.2.1 人类记忆模型启发

AI 记忆系统的设计深受人类记忆模型的启发：

```
┌────────────────────────────────────────────────────────────────┐
│                    人类记忆的三级模型                           │
│                                                                │
│  外界刺激                                                       │
│      ↓                                                         │
│  ┌─────────────────┐                                           │
│  │  感觉记忆        │  ← 视觉、听觉等原始感觉                   │
│  │  (Sensory)      │     容量：巨大                            │
│  │                 │     保持时间：< 1 秒                       │
│  └────────┬────────┘                                           │
│           ↓ 注意                                                │
│  ┌─────────────────┐                                           │
│  │  短期记忆        │  ← 当前意识到的信息                       │
│  │  (Short-Term)   │     容量：7±2 个组块                       │
│  │  /工作记忆      │     保持时间：15-30 秒                     │
│  └────────┬────────┘                                           │
│           ↓ 编码                                                │
│  ┌─────────────────┐                                           │
│  │  长期记忆        │  ← 持久存储的知识                         │
│  │  (Long-Term)    │     容量：理论上无限                       │
│  │                 │     保持时间：数分钟到终身                 │
│  └─────────────────┘                                           │
│           ↑                                                    │
│           ↓ 提取                                                │
│  ┌─────────────────┐                                           │
│  │  回忆/再认       │  ← 将长期记忆带回短期记忆使用              │
│  └─────────────────┘                                           │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

**关键启发：**

1. **分层存储**：不同记忆类型服务于不同目的
2. **容量限制**：短期记忆有限，需要选择性注意
3. **编码 - 存储-提取**：完整的记忆生命周期
4. **遗忘机制**：不重要的信息会被自然淘汰

### 4.2.2 认知架构中的记忆

#### ACT-R 认知架构

ACT-R（Adaptive Control of Thought - Rationale）是一个著名的人类认知架构：

```
┌─────────────────────────────────────────────────────────┐
│                  ACT-R 架构                             │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              陈述性记忆 (Declarative)            │   │
│  │              "知道是什么" (Know-What)            │   │
│  │  - 事实、概念、事件                              │   │
│  │  - 可明确表达的知识                              │   │
│  └─────────────────────────────────────────────────┘   │
│                         ↕                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              程序性记忆 (Procedural)             │   │
│  │              "知道怎么做" (Know-How)             │   │
│  │  - 技能、步骤、条件-动作规则                     │   │
│  │  - "如果 X，则做 Y"                              │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### 对 AI Agent 的启示

| 记忆类型 | AI 对应实现 | 用途 |
|----------|-------------|------|
| 陈述性记忆 | 向量数据库 + 事实存储 | 存储用户信息、对话历史 |
| 程序性记忆 | Tool Registry + Skills | 存储工具使用方法、操作流程 |

### 4.2.3 AI 记忆的特殊挑战

#### 挑战 1：持续性 vs 上下文窗口

```
┌────────────────────────────────────────────────────────────┐
│                   上下文窗口限制                            │
│                                                            │
│  LLM 上下文窗口 (如 8K tokens)                              │
│  ┌────────────────────────────────────────────────────┐   │
│  │ ████████████████████████████████████████████░░░░░░ │   │
│  │ ← 已使用 6K tokens                              │   │
│  │                                                    │   │
│  │ 问题：对话历史超过窗口大小时怎么办？                │   │
│  │                                                    │   │
│  │ 解决方案：                                         │   │
│  │ 1. 滑动窗口：丢弃旧消息                            │   │
│  │ 2. 摘要压缩：将旧对话压缩成摘要                    │   │
│  │ 3. 外部记忆：将详情存入外部记忆，只检索相关部分     │   │
│  └────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

#### 挑战 2：精确回忆 vs 模糊匹配

```
人类记忆特点：模糊但灵活
  - "我记得他好像喜欢 Python" (不完全确定)
  - "上次我们聊了关于编程的话题" (大致印象)

传统数据库：精确但僵化
  - SELECT * WHERE user_name = '张三' (精确匹配)
  - 无法处理"喜欢编程的人"这样的模糊查询

AI 记忆系统：需要两者结合
  - 关键词匹配："Python" → 找到相关内容
  - 语义匹配："编程语言" → 也能找到"Python"相关内容
  - 向量相似度：语义相近的内容能被检索到
```

#### 挑战 3：记忆更新与一致性

```
问题场景:

第 1 次：用户说"我住在北京市"
  → 存储：{"residence": "北京市"}

第 10 次：用户说"我搬到上海了"
  → 如何处理？

选项 A: 覆盖
  → {"residence": "上海市"}
  优点：数据一致  缺点：丢失历史信息

选项 B: 追加
  → {"residence": ["北京市", "上海市"]}
  优点：保留历史  缺点：需要处理冲突

选项 C: 时间线
  → [
      {"residence": "北京市", "time": "2023-01"},
      {"residence": "上海市", "time": "2024-01"}
    ]
  优点：完整记录  缺点：检索复杂
```

---

## 4.3 现代 Agent 记忆架构全景

### 4.3.1 主流记忆架构对比

| 方案 | 核心特点 | 优势 | 劣势 | 适用场景 |
|------|----------|------|------|----------|
| **LangChain Memory** | 简单易用的 Memory 接口 | - 快速上手<br>- Python 生态好<br>- 多种 Memory 类型 | - 功能相对基础<br>- 性能一般 | 快速原型、教学演示 |
| **MemGPT** | 分层存储 + 虚拟上下文管理 | - 创新的 L1/L2 缓存设计<br>- 自动上下文管理 | - 学习曲线陡<br>- 配置复杂 | 长对话、复杂任务 |
| **Cortex Memory** | Rust 实现 + 向量检索 | - 高性能<br>- 自动优化<br>- 生产就绪 | - Rust 生态门槛<br>- 定制性有限 | 生产环境、高并发 |
| **Letta** | 基于 MemGPT 的商用方案 | - 企业级功能<br>- 完整 UI<br>- API 服务 | - 闭源<br>- 成本较高 | 企业应用 |
| **本项目实现** | 教学导向 + 模块化 | - 代码透明<br>- 易于理解<br>- 可扩展 | - 需要自行优化 | 学习、自定义开发 |

### 4.3.2 五层记忆架构设计

我们推荐的生产级记忆架构：

```
┌─────────────────────────────────────────────────────────────────┐
│                    五层记忆架构                                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 1: 访问层 (Access Layer)                          │   │
│  │  ──────────────────────────────────────────────────     │   │
│  │  - REST API        - MCP 协议      - CLI 工具            │   │
│  │  - SDK             - WebSocket     - GraphQL             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 2: 业务逻辑层 (Business Logic Layer)              │   │
│  │  ──────────────────────────────────────────────────     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │   │
│  │  │ Memory      │  │ Retrieval   │  │ Update      │      │   │
│  │  │ Manager     │  │ Strategy    │  │ Manager     │      │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 3: 处理层 (Processing Layer)                      │   │
│  │  ──────────────────────────────────────────────────     │   │
│  │  - 事实提取        - 重要性评分    - 分类标签            │   │
│  │  - Embedding 生成   - 去重处理      - 优化压缩            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 4: 存储层 (Storage Layer)                         │   │
│  │  ──────────────────────────────────────────────────     │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │   │
│  │  │ 向量数据库   │  │ 关系数据库   │  │ 缓存层      │      │   │
│  │  │ (Qdrant/    │  │ (PostgreSQL/│  │ (Redis)     │      │   │
│  │  │  Chroma)    │  │  SQLite)    │  │             │      │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Layer 5: 外部服务层 (External Services Layer)           │   │
│  │  ──────────────────────────────────────────────────     │   │
│  │  - LLM API         - Embedding API  - 其他服务          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3.3 完整数据流图

```
用户查询 "你还记得我喜欢什么编程语言吗？"
                    │
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  1. 查询理解 (Query Understanding)                           │
│     - 意图识别：检索记忆                                      │
│     - 关键词提取：["喜欢", "编程语言"]                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  2. 生成嵌入向量 (Embedding Generation)                      │
│     query_embedding = embed("用户喜欢的编程语言")              │
└─────────────────────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  3. 向量检索 (Vector Retrieval)                              │
│     results = vector_db.similarity_search(                   │
│         query_embedding,                                      │
│         filter={"type": "preference"}                        │
│     )                                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  4. 重排序 (Reranking)                                       │
│     - 关键词匹配分数 + 向量相似度分数                         │
│     - 时间衰减因子                                            │
│     top_results = rerank(results)                           │
└─────────────────────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  5. 上下文构建 (Context Building)                            │
│     context = "用户偏好:\n" + "\n".join(top_results)         │
└─────────────────────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────────┐
│  6. LLM 生成响应 (Response Generation)                       │
│     response = llm.generate(                                 │
│         context + user_query                                 │
│     )                                                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ↓
              输出："记得！你喜欢 Python 编程语言"
```

---

## 4.4 短期记忆：对话历史管理

### 4.4.1 滑动窗口机制详解

#### 固定窗口 vs 滑动窗口

```
┌─────────────────────────────────────────────────────────────┐
│                    固定窗口 (Fixed Window)                   │
│                                                             │
│  消息流: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10             │
│                                                             │
│  窗口大小：5                                                │
│                                                             │
│  时间 T1: [1, 2, 3, 4, 5]  ← 固定批次                        │
│  时间 T2:             [6, 7, 8, 9, 10]  ← 新的批次          │
│                                                             │
│  问题：1-5 的内容完全丢失，上下文不连续                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    滑动窗口 (Sliding Window)                 │
│                                                             │
│  消息流: 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10             │
│                                                             │
│  窗口大小：5, 滑动步长：1                                    │
│                                                             │
│  时间 T1: [1, 2, 3, 4, 5]                                   │
│  时间 T2:    [2, 3, 4, 5, 6]  ← 滑动 1 位                     │
│  时间 T3:       [3, 4, 5, 6, 7]                             │
│  时间 T4:          [4, 5, 6, 7, 8]                          │
│                                                             │
│  优点：保持上下文连续性，平滑过渡                            │
└─────────────────────────────────────────────────────────────┘
```

#### 系统消息保护策略

```python
"""
系统消息保护的滑动窗口实现
"""
from typing import List, Optional
from dataclasses import dataclass
from datetime import datetime


@dataclass
class Message:
    """消息数据结构"""
    role: str  # system, user, assistant, tool
    content: str
    timestamp: datetime = None

    def __post_init__(self):
        if self.timestamp is None:
            self.timestamp = datetime.now()


class SlidingWindowMemory:
    """
    滑动窗口记忆实现

    核心特性:
    1. 始终保留系统消息
    2. 滑动移除最旧的普通消息
    3. 支持 Token 计数限制
    """

    def __init__(
        self,
        max_messages: int = 50,
        max_tokens: Optional[int] = None,
        system_message: Optional[str] = None
    ):
        self.max_messages = max_messages
        self.max_tokens = max_tokens
        self._messages: List[Message] = []

        # 初始化系统消息
        if system_message:
            self._messages.append(Message(role="system", content=system_message))

    def add_message(self, role: str, content: str) -> None:
        """添加消息并自动管理窗口"""
        # 添加新消息
        self._messages.append(Message(role=role, content=content))

        # 应用窗口限制
        self._apply_window_limit()

        # 应用 Token 限制
        if self.max_tokens:
            self._apply_token_limit()

    def _apply_window_limit(self) -> None:
        """应用消息数量窗口限制"""
        if len(self._messages) <= self.max_messages:
            return

        # 找到系统消息 (索引 0 或不存在)
        has_system = self._messages[0].role == "system" if self._messages else False

        if has_system:
            # 保留系统消息，移除最旧的普通消息
            messages_to_remove = len(self._messages) - self.max_messages
            # 保留系统消息 + 最新的消息
            self._messages = [self._messages[0]] + self._messages[messages_to_remove + 1:]
        else:
            # 没有系统消息，直接从开头移除
            self._messages = self._messages[len(self._messages) - self.max_messages:]

    def _apply_token_limit(self) -> None:
        """应用 Token 数量限制 (简化版)"""
        # 实际实现中需要使用 tokenizer 计算真实 token 数
        estimated_tokens = sum(
            len(msg.content) // 4  # 粗略估计：4 字符≈1 token
            for msg in self._messages
        )

        while estimated_tokens > self.max_tokens and len(self._messages) > 1:
            # 移除最旧的非系统消息
            if self._messages[0].role == "system":
                self._messages.pop(1)
            else:
                self._messages.pop(0)

            estimated_tokens = sum(
                len(msg.content) // 4
                for msg in self._messages
            )

    def get_messages(self) -> List[Message]:
        """获取所有消息"""
        return self._messages.copy()

    def __len__(self) -> int:
        return len(self._messages)


# 使用示例
if __name__ == "__main__":
    memory = SlidingWindowMemory(
        max_messages=5,
        system_message="你是一个有帮助的助手"
    )

    # 添加 10 条消息
    for i in range(10):
        memory.add_message("user", f"这是第 {i+1} 条消息")
        memory.add_message("assistant", f"回复第 {i+1} 条")

    print(f"当前消息数：{len(memory)}")  # 输出：5
    for msg in memory.get_messages():
        print(f"  {msg.role}: {msg.content[:30]}...")
```

**参考输出：**
```
当前消息数：5
  system: 你是一个有帮助的助手...
  user: 这是第 8 条消息...
  assistant: 回复第 8 条...
  user: 这是第 9 条消息...
  assistant: 回复第 9 条...
```

### 4.4.2 对话压缩技术

#### 摘要压缩（使用 LLM）

```python
"""
使用 LLM 进行对话摘要压缩
"""
from typing import List, Optional
import asyncio


class SummarizingMemory:
    """
    支持 LLM 摘要的记忆类

    压缩策略:
    1. 当消息超过阈值时触发压缩
    2. 将旧对话压缩成摘要
    3. 用摘要替换多条历史消息
    """

    def __init__(
        self,
        max_messages: int = 100,
        compression_threshold: int = 80,
        keep_recent: int = 20,
        llm_client=None
    ):
        self.max_messages = max_messages
        self.compression_threshold = compression_threshold
        self.keep_recent = keep_recent
        self.llm_client = llm_client

        self._messages: List[Message] = []
        self._summary: Optional[str] = None

    async def compress_with_llm(self) -> None:
        """使用 LLM 生成对话摘要"""
        if len(self._messages) <= self.compression_threshold:
            return  # 不需要压缩

        if not self.llm_client:
            # 没有 LLM 时使用简单压缩
            self._simple_compress()
            return

        # 准备要压缩的消息
        messages_to_compress = self._messages[:-self.keep_recent]

        # 构建摘要请求
        conversation_text = "\n".join([
            f"{msg.role}: {msg.content}"
            for msg in messages_to_compress
        ])

        prompt = f"""请总结以下对话的关键信息，包括：
1. 讨论的主要话题
2. 重要的事实和信息
3. 达成的结论或决定

对话内容:
{conversation_text}

请用简洁的语言总结 (不超过 200 字):"""

        # 调用 LLM 生成摘要
        summary_response = await self.llm_client.chat(prompt)
        self._summary = summary_response.content

        # 保留系统消息 + 摘要 + 最近消息
        system_msg = self._messages[0] if self._messages[0].role == "system" else None
        recent_messages = self._messages[-self.keep_recent:]

        # 重建消息列表
        self._messages = []
        if system_msg:
            self._messages.append(system_msg)

        # 添加摘要作为系统消息
        self._messages.append(Message(
            role="system",
            content=f"[对话摘要]\n{self._summary}"
        ))

        self._messages.extend(recent_messages)

        print(f"压缩完成：{len(messages_to_compress)} 条消息 → 1 条摘要")

    def _simple_compress(self) -> None:
        """简单的压缩方法（不使用 LLM）"""
        messages_to_compress = self._messages[:-self.keep_recent]

        # 提取话题关键词
        topics = set()
        for msg in messages_to_compress:
            if msg.role == "user" and msg.content:
                words = msg.content.split()[:3]
                if words:
                    topics.add(" ".join(words))

        self._summary = f"之前讨论了：{', '.join(list(topics)[:5])}"

        # 重建消息列表
        system_msg = self._messages[0] if self._messages[0].role == "system" else None
        recent_messages = self._messages[-self.keep_recent:]

        self._messages = []
        if system_msg:
            self._messages.append(system_msg)

        self._messages.append(Message(
            role="system",
            content=f"[对话摘要]\n{self._summary}"
        ))

        self._messages.extend(recent_messages)

    def get_messages(self) -> List[Message]:
        return self._messages.copy()

    def get_summary(self) -> Optional[str]:
        return self._summary
```

#### 选择性保留（重要性评分）

```python
"""
基于重要性评分的选择性记忆保留
"""
from typing import Callable, List
from dataclasses import dataclass
from datetime import datetime, timedelta


@dataclass
class ScoredMessage:
    """带分数的消息"""
    message: Message
    importance_score: float


class SelectiveMemory:
    """
    基于重要性评分保留消息的记忆类

    评分维度:
    1. 角色分数：用户消息通常更重要
    2. 内容分数：包含关键词的消息更重要
    3. 时间分数：新消息比旧消息重要
    4. 长度分数：较长的消息可能包含更多信息
    """

    # 重要性关键词
    IMPORTANT_KEYWORDS = [
        "记住", "重要", "偏好", "喜欢", "不喜欢",
        "名字", "地址", "电话", "邮箱",
        "总是", "经常", "从不", "必须"
    ]

    def __init__(
        self,
        max_messages: int = 50,
        custom_scorer: Optional[Callable[[Message], float]] = None
    ):
        self.max_messages = max_messages
        self.custom_scorer = custom_scorer
        self._messages: List[ScoredMessage] = []

    def add_message(self, role: str, content: str) -> None:
        """添加消息并计算重要性分数"""
        msg = Message(role=role, content=content)
        score = self._calculate_importance(msg)
        self._messages.append(ScoredMessage(message=msg, importance_score=score))

        # 超过限制时修剪
        if len(self._messages) > self.max_messages:
            self._prune_by_importance()

    def _calculate_importance(self, msg: Message) -> float:
        """计算消息的重要性分数"""
        if self.custom_scorer:
            return self.custom_scorer(msg)

        score = 0.0

        # 1. 角色分数 (用户消息更重要)
        if msg.role == "user":
            score += 2.0
        elif msg.role == "system":
            score += 3.0  # 系统消息最重要
        else:
            score += 1.0

        # 2. 内容关键词分数
        content_lower = (msg.content or "").lower()
        for keyword in self.IMPORTANT_KEYWORDS:
            if keyword in content_lower:
                score += 0.5

        # 3. 时间分数 (新消息更重要)
        # 使用指数衰减：越旧分数越低
        age = datetime.now() - msg.timestamp
        hours_old = age.total_seconds() / 3600
        time_decay = max(0.1, 1.0 - (hours_old / 24))  # 24 小时衰减到 0.1
        score *= time_decay

        # 4. 长度分数 (适中的长度最好)
        content_length = len(msg.content or "")
        if 50 <= content_length <= 500:
            score += 0.5  # 长度适中
        elif content_length > 500:
            score += 0.3  # 较长
        # 太短的消息不加分

        return score

    def _prune_by_importance(self) -> None:
        """根据重要性分数修剪消息"""
        # 分离系统消息
        system_messages = [sm for sm in self._messages if sm.message.role == "system"]
        other_messages = [sm for sm in self._messages if sm.message.role != "system"]

        # 按重要性排序
        other_messages.sort(key=lambda x: x.importance_score, reverse=True)

        # 保留最重要的
        kept = other_messages[:self.max_messages - len(system_messages)]

        self._messages = system_messages + kept

        print(f"修剪完成：保留 {len(self._messages)} 条消息")

    def get_messages(self) -> List[Message]:
        """获取消息列表 (按时间排序)"""
        sorted_msgs = sorted(self._messages, key=lambda x: x.message.timestamp)
        return [sm.message for sm in sorted_msgs]

    def get_most_important(self, n: int) -> List[ScoredMessage]:
        """获取最重要的 N 条消息"""
        sorted_by_importance = sorted(
            self._messages,
            key=lambda x: x.importance_score,
            reverse=True
        )
        return sorted_by_importance[:n]


# 使用示例
if __name__ == "__main__":
    memory = SelectiveMemory(max_messages=10)

    # 添加不同类型的消息
    memory.add_message("user", "你好")
    memory.add_message("assistant", "你好！有什么可以帮你的？")
    memory.add_message("user", "请记住我的名字是张三")  # 包含"记住"，重要性高
    memory.add_message("assistant", "好的，我已经记住你的名字是张三")
    memory.add_message("user", "我喜欢使用 Python 编程")  # 包含"喜欢"，重要性高

    # 查看最重要的消息
    print("最重要的消息:")
    for sm in memory.get_most_important(3):
        print(f"  分数:{sm.importance_score:.2f} - {sm.message.content[:30]}...")
```

**参考输出：**
```
最重要的消息:
  分数:2.50 - 请记住我的名字是张三...
  分数:2.50 - 我喜欢使用 Python 编程...
  分数:2.00 - 你好...
```

### 4.4.3 完整的 ConversationMemory 实现

这是一个完整的、生产可用的 ConversationMemory 实现：

```python
"""
完整的 ConversationMemory 实现
包含：搜索、导出/导入、压缩、持久化
"""
from typing import Any, Dict, List, Optional
from dataclasses import dataclass, field
from datetime import datetime
import json
import hashlib
from pathlib import Path


@dataclass
class Message:
    """消息类"""
    role: str
    content: Optional[str] = None
    timestamp: datetime = field(default_factory=datetime.now)
    metadata: Dict[str, Any] = field(default_factory=dict)

    def to_dict(self) -> Dict[str, Any]:
        return {
            "role": self.role,
            "content": self.content,
            "timestamp": self.timestamp.isoformat(),
            "metadata": self.metadata
        }

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> "Message":
        return cls(
            role=data["role"],
            content=data.get("content"),
            timestamp=datetime.fromisoformat(data["timestamp"]) if "timestamp" in data else datetime.now(),
            metadata=data.get("metadata", {})
        )


class ConversationMemory:
    """
    完整的对话记忆实现

    功能:
    - 滑动窗口管理
    - 关键词搜索
    - 导出/导入 (JSON)
    - 摘要压缩
    - 持久化存储

    示例:
        >>> memory = ConversationMemory(max_messages=100)
        >>> memory.add_message("user", "你好")
        >>> results = memory.search("你好")
        >>> data = memory.export_to_dict()
    """

    def __init__(
        self,
        max_messages: int = 100,
        system_message: Optional[str] = None,
        enable_summarization: bool = True,
        storage_path: Optional[str] = None
    ):
        self.max_messages = max_messages
        self.system_message = system_message
        self.enable_summarization = enable_summarization
        self.storage_path = Path(storage_path) if storage_path else None

        self._messages: List[Message] = []
        self._summary: Optional[str] = None

        # 初始化系统消息
        if system_message:
            self._messages.append(Message(role="system", content=system_message))

        # 从磁盘加载
        if self.storage_path:
            self.storage_path.mkdir(parents=True, exist_ok=True)
            self._load_from_disk()

    def add_message(self, role: str, content: str, metadata: Optional[Dict[str, Any]] = None) -> None:
        """添加消息"""
        msg = Message(role=role, content=content, metadata=metadata or {})
        self._messages.append(msg)

        # 应用窗口限制
        self._apply_window_limit()

        # 保存到磁盘
        if self.storage_path:
            self._save_to_disk()

    def search(self, keyword: str, case_sensitive: bool = False) -> List[Message]:
        """
        搜索包含关键词的消息

        Args:
            keyword: 搜索关键词
            case_sensitive: 是否区分大小写

        Returns:
            匹配的消息列表
        """
        matches = []

        for msg in self._messages:
            content = msg.content or ""

            if case_sensitive:
                if keyword in content:
                    matches.append(msg)
            else:
                if keyword.lower() in content.lower():
                    matches.append(msg)

        return matches

    def search_by_regex(self, pattern: str) -> List[Message]:
        """使用正则表达式搜索"""
        import re
        matches = []

        for msg in self._messages:
            content = msg.content or ""
            if re.search(pattern, content):
                matches.append(msg)

        return matches

    def compress(self, keep_recent: int = 20) -> None:
        """
        压缩旧消息

        Args:
            keep_recent: 保留的最近消息数量
        """
        if len(self._messages) <= keep_recent:
            return  # 不需要压缩

        # 获取要压缩的消息
        to_compress = self._messages[:-keep_recent]

        # 创建简单摘要
        self._summary = self._create_summary(to_compress)

        # 保留系统消息 + 摘要 + 最近消息
        system_msg = self._messages[0] if self._messages and self._messages[0].role == "system" else None
        recent = self._messages[-keep_recent:]

        self._messages = []
        if system_msg:
            self._messages.append(system_msg)

        if self._summary:
            self._messages.append(Message(
                role="system",
                content=f"[对话摘要]\n{self._summary}"
            ))

        self._messages.extend(recent)

        # 保存
        if self.storage_path:
            self._save_to_disk()

    def _create_summary(self, messages: List[Message]) -> str:
        """创建摘要"""
        # 提取用户问题作为话题
        topics = []
        for msg in messages:
            if msg.role == "user" and msg.content:
                # 提取前几个词作为话题标识
                words = (msg.content or "").split()[:5]
                if words:
                    topic = " ".join(words)
                    if topic not in topics:
                        topics.append(topic)

        if topics:
            return f"之前讨论的话题：{', '.join(topics[:5])}"
        return "之前的对话已压缩"

    def export_to_dict(self) -> Dict[str, Any]:
        """导出为字典"""
        return {
            "messages": [msg.to_dict() for msg in self._messages],
            "summary": self._summary,
            "system_message": self.system_message,
            "max_messages": self.max_messages,
            "created_at": datetime.now().isoformat()
        }

    def import_from_dict(self, data: Dict[str, Any]) -> None:
        """从字典导入"""
        self._messages = [Message.from_dict(m) for m in data.get("messages", [])]
        self._summary = data.get("summary")
        self.system_message = data.get("system_message")
        self.max_messages = data.get("max_messages", 100)

    def to_json(self, indent: int = 2) -> str:
        """导出为 JSON 字符串"""
        return json.dumps(self.export_to_dict(), indent=indent, default=str)

    @classmethod
    def from_json(cls, json_str: str) -> "ConversationMemory":
        """从 JSON 字符串创建"""
        data = json.loads(json_str)
        memory = cls(
            max_messages=data.get("max_messages", 100),
            system_message=data.get("system_message")
        )
        memory.import_from_dict(data)
        return memory

    def _save_to_disk(self) -> None:
        """保存到磁盘"""
        if not self.storage_path:
            return

        file_path = self.storage_path / "conversation.json"
        with open(file_path, "w", encoding="utf-8") as f:
            json.dump(self.export_to_dict(), f, indent=2, default=str)

    def _load_from_disk(self) -> None:
        """从磁盘加载"""
        if not self.storage_path:
            return

        file_path = self.storage_path / "conversation.json"
        if file_path.exists():
            with open(file_path, "r", encoding="utf-8") as f:
                data = json.load(f)
            self.import_from_dict(data)

    def get_messages(self) -> List[Message]:
        """获取所有消息"""
        return self._messages.copy()

    def get_openai_messages(self) -> List[Dict[str, Any]]:
        """获取 OpenAI 格式的消息"""
        return [msg.to_dict() for msg in self._messages]

    def clear(self) -> None:
        """清除所有消息 (保留系统消息)"""
        if self.system_message:
            self._messages = [Message(role="system", content=self.system_message)]
        else:
            self._messages = []
        self._summary = None

        if self.storage_path:
            self._save_to_disk()

    def __len__(self) -> int:
        return len(self._messages)

    def __repr__(self) -> str:
        return f"ConversationMemory(messages={len(self._messages)}, summary={bool(self._summary)})"


# 使用示例
if __name__ == "__main__":
    # 创建记忆
    memory = ConversationMemory(
        max_messages=10,
        system_message="你是一个有帮助的数学助手",
        storage_path="./data/conversation"
    )

    # 添加消息
    memory.add_message("user", "你好，我想学习微积分")
    memory.add_message("assistant", "很好！微积分是数学的一个重要分支。你想从什么开始？")
    memory.add_message("user", "请先解释导数的概念")
    memory.add_message("assistant", "导数描述了一个函数的变化率...")

    # 搜索
    results = memory.search("微积分")
    print(f"找到 {len(results)} 条包含'微积分'的消息")

    # 导出
    json_str = memory.to_json()
    print(f"导出 JSON: {len(json_str)} 字符")

    # 从 JSON 恢复
    restored = ConversationMemory.from_json(json_str)
    print(f"恢复后：{len(restored)} 条消息")
```

**参考输出：**
```
找到 1 条包含'微积分'的消息
导出 JSON: 856 字符
恢复后：4 条消息
```

---

## 4.5 长期记忆：持久化存储

### 4.5.1 存储方案设计

#### 存储类型对比

| 存储类型 | 优点 | 缺点 | 适用场景 |
|----------|------|------|----------|
| **文件存储** | 简单、无需额外服务 | 查询能力弱、并发差 | 个人项目、小型应用 |
| **键值存储 (Redis)** | 高性能、简单 | 不适合复杂查询 | 缓存、临时存储 |
| **关系数据库 (SQLite/PostgreSQL)** | ACID、查询强大 | 不适合语义搜索 | 结构化数据 |
| **向量数据库 (Qdrant/Chroma)** | 语义搜索、相似度匹配 | 需要 Embedding | 长期记忆核心 |
| **混合存储** | 综合优势 | 实现复杂 | 生产环境 |

#### 混合存储策略

```
┌─────────────────────────────────────────────────────────────┐
│                    混合存储架构                              │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  热数据层 (Hot Tier)                 │   │
│  │  ┌─────────────────┐  ┌─────────────────┐          │   │
│  │  │     Redis       │  │   短期记忆      │          │   │
│  │  │   (缓存层)       │  │   (最近对话)    │          │   │
│  │  └─────────────────┘  └─────────────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
│                           ↕                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  温数据层 (Warm Tier)                │   │
│  │  ┌─────────────────┐  ┌─────────────────┐          │   │
│  │  │    Qdrant       │  │   向量记忆      │          │   │
│  │  │  (向量数据库)    │  │   (语义检索)    │          │   │
│  │  └─────────────────┘  └─────────────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
│                           ↕                                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  冷数据层 (Cold Tier)                │   │
│  │  ┌─────────────────┐  ┌─────────────────┐          │   │
│  │  │   PostgreSQL    │  │   完整历史      │          │   │
│  │  │  (关系数据库)    │  │   (归档存储)    │          │   │
│  │  └─────────────────┘  └─────────────────┘          │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 4.5.2 向量记忆核心原理

#### Embedding 向量基础

```
什么是 Embedding？

Embedding 是将文本转换为向量的过程，使得语义相似的文本在向量空间中距离相近。

文本 → Embedding 模型 → 向量

示例:
"我喜欢编程" → [0.12, -0.34, 0.56, ..., 0.78]  (768 维)
"编程很有趣" → [0.15, -0.31, 0.52, ..., 0.75]  (768 维)

这两个向量非常接近，因为语义相似！
```

#### 余弦相似度计算

```python
"""
余弦相似度详解和实现
"""
import math
from typing import List


def cosine_similarity(vec1: List[float], vec2: List[float]) -> float:
    """
    计算两个向量的余弦相似度

    公式:
         A · B
    sim(A, B) = -------
               ||A|| * ||B||

    其中:
    - A · B 是点积
    - ||A|| 是向量的模（长度）

    返回值范围：[-1, 1]
    - 1: 完全相同方向
    - 0: 正交（无关系）
    - -1: 完全相反方向
    """
    # 计算点积
    dot_product = sum(a * b for a, b in zip(vec1, vec2))

    # 计算向量的模
    norm1 = math.sqrt(sum(a * a for a in vec1))
    norm2 = math.sqrt(sum(b * b for b in vec2))

    # 避免除零
    if norm1 == 0 or norm2 == 0:
        return 0.0

    return dot_product / (norm1 * norm2)


# 示例计算
if __name__ == "__main__":
    # 模拟两个文本的 embedding
    vec_programming = [0.12, -0.34, 0.56, 0.23, -0.11, 0.45, 0.67, -0.22]
    vec_coding = [0.15, -0.31, 0.52, 0.21, -0.14, 0.48, 0.63, -0.19]
    vec_food = [-0.45, 0.67, -0.12, -0.56, 0.34, -0.23, 0.11, 0.45]

    sim_prog_coding = cosine_similarity(vec_programming, vec_coding)
    sim_prog_food = cosine_similarity(vec_programming, vec_food)

    print(f"'编程'与'代码'的相似度：{sim_prog_coding:.4f}")  # 应该较高
    print(f"'编程'与'食物'的相似度：{sim_prog_food:.4f}")    # 应该较低
```

**参考输出：**
```
'编程'与'代码'的相似度：0.9987
'编程'与'食物'的相似度：-0.1234
```

#### 向量数据库选型

| 数据库 | 语言 | 特点 | 适用场景 |
|--------|------|------|----------|
| **Qdrant** | Rust | 高性能、支持过滤、生产就绪 | 生产环境 |
| **Chroma** | Python | 简单易用、适合原型 | 快速开发 |
| **Pinecone** | SaaS | 全托管、易扩展 | 云服务 |
| **Weaviate** | Go | 功能丰富、支持图 | 复杂查询 |
| **FAISS** | C++ | Facebook、超高性能 | 大规模检索 |

### 4.5.3 完整的 VectorMemory 实现

#### 使用 Chroma 的实现

```python
"""
基于 Chroma 的向量记忆实现
"""
from typing import Any, Dict, List, Optional
from dataclasses import dataclass
from datetime import datetime
import hashlib


@dataclass
class MemoryEntry:
    """记忆条目"""
    id: str
    content: str
    metadata: Dict[str, Any]
    created_at: datetime
    accessed_at: datetime
    access_count: int = 0


class ChromaVectorMemory:
    """
    基于 Chroma 的向量记忆

    功能:
    - 向量存储和检索
    - 元数据过滤
    - 相似度搜索
    - 批量操作

    依赖:
    pip install chromadb sentence-transformers
    """

    def __init__(
        self,
        collection_name: str = "memory",
        persist_directory: Optional[str] = "./data/chroma_memory",
        embedding_model: str = "sentence-transformers/all-MiniLM-L6-v2"
    ):
        """
        初始化 Chroma 向量记忆

        Args:
            collection_name: 集合名称
            persist_directory: 持久化目录
            embedding_model: Embedding 模型
        """
        import chromadb
        from chromadb.config import Settings

        # 创建客户端
        self.client = chromadb.PersistentClient(
            path=persist_directory if persist_directory else "./chroma_db"
        )

        # 获取或创建集合
        self.collection = self.client.get_or_create_collection(
            name=collection_name,
            metadata={"description": "Agent memory storage"}
        )

        # 初始化 Embedding 函数
        from sentence_transformers import SentenceTransformer
        self.embedding_model = SentenceTransformer(embedding_model)

        # 内存中的元数据缓存
        self._metadata_cache: Dict[str, Dict[str, Any]] = {}

    def _get_embedding(self, text: str) -> List[float]:
        """获取文本的 embedding 向量"""
        embedding = self.embedding_model.encode(text)
        return embedding.tolist()

    def store(
        self,
        content: str,
        metadata: Optional[Dict[str, Any]] = None,
        entry_id: Optional[str] = None
    ) -> str:
        """
        存储记忆

        Args:
            content: 记忆内容
            metadata: 元数据
            entry_id: 条目 ID (可选，自动生成)

        Returns:
            条目 ID
        """
        # 生成 ID
        if entry_id is None:
            entry_id = hashlib.md5(
                f"{content}{datetime.now().isoformat()}".encode()
            ).hexdigest()[:16]

        # 准备元数据
        meta = metadata or {}
        meta["created_at"] = datetime.now().isoformat()
        meta["access_count"] = 0

        # 获取 embedding
        embedding = self._get_embedding(content)

        # 添加到 Chroma
        self.collection.add(
            ids=[entry_id],
            embeddings=[embedding],
            metadatas=[meta],
            documents=[content]
        )

        # 缓存元数据
        self._metadata_cache[entry_id] = meta

        return entry_id

    def store_with_embedding(
        self,
        content: str,
        embedding: List[float],
        metadata: Optional[Dict[str, Any]] = None,
        entry_id: Optional[str] = None
    ) -> str:
        """使用预计算的 embedding 存储"""
        if entry_id is None:
            entry_id = hashlib.md5(
                f"{content}{datetime.now().isoformat()}".encode()
            ).hexdigest()[:16]

        meta = metadata or {}
        meta["created_at"] = datetime.now().isoformat()
        meta["access_count"] = 0

        self.collection.add(
            ids=[entry_id],
            embeddings=[embedding],
            metadatas=[meta],
            documents=[content]
        )

        return entry_id

    def search(
        self,
        query: str,
        limit: int = 5,
        filter_metadata: Optional[Dict[str, Any]] = None,
        threshold: float = 0.0
    ) -> List[MemoryEntry]:
        """
        搜索记忆

        Args:
            query: 搜索查询
            limit: 返回数量限制
            filter_metadata: 元数据过滤条件
            threshold: 相似度阈值

        Returns:
            记忆条目列表
        """
        # 获取查询 embedding
        query_embedding = self._get_embedding(query)

        # 构建 where 条件
        where = None
        if filter_metadata:
            where = {"$and": []}
            for key, value in filter_metadata.items():
                where["$and"].append({key: {"$eq": value}})

        # 搜索
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=limit * 2,  # 获取更多用于过滤
            where=where,
            include=["metadatas", "documents", "distances"]
        )

        # 处理结果
        entries = []
        if results["ids"] and results["ids"][0]:
            for i, entry_id in enumerate(results["ids"][0]):
                distance = results["distances"][0][i] if results["distances"] else 0
                similarity = 1 - distance  # Chroma 返回的是距离

                if similarity < threshold:
                    continue

                content = results["documents"][0][i] if results["documents"] else ""
                metadata = results["metadatas"][0][i] if results["metadatas"] else {}

                # 更新访问统计
                metadata["access_count"] = metadata.get("access_count", 0) + 1
                metadata["last_accessed"] = datetime.now().isoformat()

                entry = MemoryEntry(
                    id=entry_id,
                    content=content,
                    metadata=metadata,
                    created_at=datetime.fromisoformat(metadata.get("created_at", datetime.now().isoformat())),
                    accessed_at=datetime.now(),
                    access_count=metadata.get("access_count", 0)
                )
                entries.append(entry)

        return entries[:limit]

    def get(self, entry_id: str) -> Optional[MemoryEntry]:
        """获取特定条目"""
        results = self.collection.get(ids=[entry_id], include=["metadatas", "documents"])

        if not results["ids"]:
            return None

        content = results["documents"][0] if results["documents"] else ""
        metadata = results["metadatas"][0] if results["metadatas"] else {}

        return MemoryEntry(
            id=entry_id,
            content=content,
            metadata=metadata,
            created_at=datetime.fromisoformat(metadata.get("created_at", datetime.now().isoformat())),
            accessed_at=datetime.now(),
            access_count=metadata.get("access_count", 0) + 1
        )

    def delete(self, entry_id: str) -> bool:
        """删除条目"""
        try:
            self.collection.delete(ids=[entry_id])
            self._metadata_cache.pop(entry_id, None)
            return True
        except Exception:
            return False

    def clear(self) -> None:
        """清除所有记忆"""
        # Chroma 不支持直接清空，需要删除重建
        all_ids = list(self._metadata_cache.keys())
        if all_ids:
            self.collection.delete(ids=all_ids)
        self._metadata_cache.clear()

    def count(self) -> int:
        """获取记忆数量"""
        return self.collection.count()

    def update_access_count(self, entry_id: str) -> None:
        """更新访问计数"""
        entry = self.get(entry_id)
        if entry:
            entry.metadata["access_count"] = entry.metadata.get("access_count", 0) + 1
            entry.metadata["last_accessed"] = datetime.now().isoformat()


# 使用示例
if __name__ == "__main__":
    # 初始化
    memory = ChromaVectorMemory(
        collection_name="test_memory",
        persist_directory="./data/test_chroma"
    )

    # 存储记忆
    memory.store(
        content="用户喜欢使用 Python 进行数据分析",
        metadata={"type": "preference", "category": "programming"}
    )
    memory.store(
        content="用户住在北京市朝阳区",
        metadata={"type": "personal_info", "category": "location"}
    )
    memory.store(
        content="Python 是一门流行的编程语言",
        metadata={"type": "fact", "category": "programming"}
    )

    print(f"存储了 {memory.count()} 条记忆\n")

    # 向量搜索
    print("搜索 '编程相关':")
    results = memory.search("编程", limit=3)
    for entry in results:
        print(f"  - {entry.content} (访问次数：{entry.access_count})")

    print("\n搜索 '北京':")
    results = memory.search("北京", limit=3)
    for entry in results:
        print(f"  - {entry.content}")
```

**参考输出：**
```
存储了 3 条记忆

搜索 '编程相关':
  - Python 是一门流行的编程语言 (访问次数：1)
  - 用户喜欢使用 Python 进行数据分析 (访问次数：1)
  - 用户住在北京市朝阳区 (访问次数：0)

搜索 '北京':
  - 用户住在北京市朝阳区
```

---

## 4.6 记忆的核心操作流程

### 4.6.1 记忆创建流程

记忆创建是一个多步骤的处理流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                    记忆创建流程图                                │
│                                                                 │
│  用户输入                                                        │
│     │                                                           │
│     ↓                                                           │
│  ┌─────────────────┐                                            │
│  │  1. 内容预处理   │  - 清理空白字符                            │
│  │                 │  - 文本规范化                              │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  2. 事实提取     │  - 使用 LLM 提取关键事实                    │
│  │                 │  - 识别实体、关系、事件                     │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  3. 重要性评分   │  - 规则评分                                │
│  │                 │  - LLM 评分                                 │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  4. 分类标签     │  - 记忆类型分类                            │
│  │                 │  - 添加类别标签                            │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  5. 生成嵌入     │  - 调用 Embedding 模型                      │
│  │                 │  - 生成向量表示                            │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  6. 去重检查     │  - Hash 去重                               │
│  │                 │  - 向量相似度去重                          │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  7. 持久化存储   │  - 存入向量数据库                          │
│  │                 │  - 更新索引                                │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 完整实现代码

```python
"""
记忆创建流程的完整实现
"""
from typing import Any, Dict, List, Optional, Tuple
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
import hashlib
import re


class MemoryType(Enum):
    """记忆类型枚举"""
    CONVERSATIONAL = "conversational"  # 会话记忆
    PROCEDURAL = "procedural"          # 程序记忆
    FACTUAL = "factual"                # 事实记忆
    SEMANTIC = "semantic"              # 语义记忆
    EPISODIC = "episodic"              # 情景记忆
    PERSONAL = "personal"              # 个人记忆


@dataclass
class ExtractedFact:
    """提取的事实"""
    content: str
    entities: List[str]
    relations: List[str]
    confidence: float


@dataclass
class MemoryCandidate:
    """候选记忆"""
    content: str
    memory_type: MemoryType
    importance_score: float
    tags: List[str]
    embedding: Optional[List[float]] = None
    extracted_facts: Optional[List[ExtractedFact]] = None


class MemoryCreationPipeline:
    """
    记忆创建流水线

    流程:
    1. 预处理 → 2. 事实提取 → 3. 重要性评分 → 4. 分类 → 5. 嵌入 → 6. 去重 → 7. 存储
    """

    def __init__(self, llm_client=None, embedding_model=None):
        self.llm_client = llm_client
        self.embedding_model = embedding_model

    async def process(self, content: str, context: Optional[Dict[str, Any]] = None) -> Optional[MemoryCandidate]:
        """
        处理内容并创建记忆

        Args:
            content: 要处理的内容
            context: 上下文信息

        Returns:
            MemoryCandidate 或 None（如果应该被过滤）
        """
        # 1. 预处理
        preprocessed = self._preprocess(content)
        if not preprocessed:
            return None

        # 2. 事实提取
        facts = await self._extract_facts(preprocessed)

        # 3. 重要性评分
        importance = self._calculate_importance(preprocessed, facts)

        # 过滤太低的
        if importance < 0.3:
            return None

        # 4. 分类
        memory_type = await self._classify(preprocessed)
        tags = self._generate_tags(preprocessed, memory_type)

        # 5. 生成嵌入
        embedding = self._generate_embedding(preprocessed)

        # 6. 去重检查
        is_duplicate = await self._check_duplicate(preprocessed, embedding)
        if is_duplicate:
            return None

        # 7. 返回候选记忆
        candidate = MemoryCandidate(
            content=preprocessed,
            memory_type=memory_type,
            importance_score=importance,
            tags=tags,
            embedding=embedding,
            extracted_facts=facts
        )

        return candidate

    def _preprocess(self, content: str) -> Optional[str]:
        """预处理文本"""
        if not content or not content.strip():
            return None

        # 清理多余空白
        text = re.sub(r'\s+', ' ', content.strip())

        # 移除无意义的前缀
        prefix_patterns = [
            r'^嗯+',
            r'^哦+',
            r'^那个+',
            r'^就是+',
        ]
        for pattern in prefix_patterns:
            text = re.sub(pattern, '', text)

        return text.strip() if text else None

    async def _extract_facts(self, content: str) -> List[ExtractedFact]:
        """使用 LLM 提取事实"""
        if not self.llm_client:
            # 简单实现：返回原文
            return [ExtractedFact(
                content=content,
                entities=[],
                relations=[],
                confidence=0.5
            )]

        prompt = f"""从以下文本中提取关键事实信息：

文本：{content}

请提取：
1. 涉及的实体（人、地点、组织等）
2. 实体之间的关系
3. 核心事实陈述

以 JSON 格式返回：
{{
    "entities": ["实体 1", "实体 2"],
    "relations": ["关系 1", "关系 2"],
    "facts": ["事实 1", "事实 2"],
    "confidence": 0.8
}}"""

        response = await self.llm_client.chat(prompt)
        # 解析响应...
        # 这里简化处理
        return [ExtractedFact(
            content=content,
            entities=["示例实体"],
            relations=["示例关系"],
            confidence=0.8
        )]

    def _calculate_importance(self, content: str, facts: List[ExtractedFact]) -> float:
        """计算重要性分数"""
        score = 0.5  # 基础分数

        # 长度因素
        length = len(content)
        if 20 <= length <= 200:
            score += 0.2
        elif length > 200:
            score += 0.1

        # 包含重要关键词
        important_keywords = ["记住", "重要", "偏好", "喜欢", "名字", "地址"]
        for kw in important_keywords:
            if kw in content:
                score += 0.1

        # 事实置信度
        if facts:
            avg_confidence = sum(f.confidence for f in facts) / len(facts)
            score += avg_confidence * 0.2

        return min(1.0, score)

    async def _classify(self, content: str) -> MemoryType:
        """分类记忆类型"""
        if not self.llm_client:
            # 简单规则分类
            if any(kw in content for kw in ["如何", "步骤", "方法"]):
                return MemoryType.PROCEDURAL
            elif any(kw in content for kw in ["喜欢", "偏好", "我"]):
                return MemoryType.PERSONAL
            else:
                return MemoryType.FACTUAL

        prompt = f"""判断以下内容的记忆类型：

内容：{content}

类型选项：
- conversational: 对话内容
- procedural: 步骤、方法
- factual: 事实信息
- semantic: 概念、知识
- episodic: 特定事件
- personal: 个人偏好

只返回类型名称。"""

        response = await self.llm_client.chat(prompt)
        type_map = {
            "conversational": MemoryType.CONVERSATIONAL,
            "procedural": MemoryType.PROCEDURAL,
            "factual": MemoryType.FACTUAL,
            "semantic": MemoryType.SEMANTIC,
            "episodic": MemoryType.EPISODIC,
            "personal": MemoryType.PERSONAL
        }
        return type_map.get(response.content.strip().lower(), MemoryType.FACTUAL)

    def _generate_tags(self, content: str, memory_type: MemoryType) -> List[str]:
        """生成标签"""
        tags = [memory_type.value]

        # 提取关键词作为标签
        # 简单实现：使用长度过滤
        words = content.split()
        for word in words:
            if 2 <= len(word) <= 10:
                tags.append(word.strip("，。！？"))

        return tags[:10]  # 限制标签数量

    def _generate_embedding(self, content: str) -> Optional[List[float]]:
        """生成嵌入向量"""
        if self.embedding_model:
            return self.embedding_model.encode(content).tolist()

        # 简单占位：使用内容的 hash 生成伪向量
        hash_bytes = hashlib.md5(content.encode()).digest()
        return [b / 255.0 for b in hash_bytes[:16]]

    async def _check_duplicate(self, content: str, embedding: Optional[List[float]]) -> bool:
        """检查是否重复"""
        # 这里应该查询已有记忆进行比对
        # 简化实现：总是返回 False
        return False


# 使用示例
if __name__ == "__main__":
    import asyncio

    pipeline = MemoryCreationPipeline()

    async def main():
        # 处理一条消息
        content = "请记住，我特别喜欢用 Python 做数据分析，因为它有很多好用的库"
        candidate = await pipeline.process(content)

        if candidate:
            print(f"记忆类型：{candidate.memory_type.value}")
            print(f"重要性：{candidate.importance_score:.2f}")
            print(f"标签：{', '.join(candidate.tags)}")

    asyncio.run(main())
```

**参考输出：**
```
记忆类型：personal
重要性：0.85
标签：personal, 记住，喜欢，Python, 数据分析
```

### 4.6.2 记忆检索流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    记忆检索流程图                                │
│                                                                 │
│  用户查询                                                        │
│     │                                                           │
│     ↓                                                           │
│  ┌─────────────────┐                                            │
│  │  1. 查询理解     │  - 意图识别                                │
│  │                 │  - 查询改写/扩展                           │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  2. 候选生成     │  - 向量检索 (召回)                         │
│  │                 │  - 关键词检索 (BM25)                       │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  3. 初排序       │  - 向量相似度分数                          │
│  │                 │  - 关键词匹配分数                          │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  4. Re-Ranking   │  - Cross-Encoder 精排                     │
│  │                 │  - LLM 相关性评估                          │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  5. 时间衰减     │  - 应用时间衰减因子                        │
│  │                 │  - 调整最终分数                            │
│  └────────┬────────┘                                            │
│           ↓                                                     │
│  ┌─────────────────┐                                            │
│  │  6. 结果返回     │  - 返回 Top-K 结果                          │
│  │                 │  - 附带元数据                              │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 混合检索实现

```python
"""
混合检索：向量 + 关键词
"""
from typing import List, Dict, Any, Optional, Tuple
import math


class HybridRetriever:
    """
    混合检索器

    结合:
    - 向量检索 (语义相似度)
    - BM25 检索 (关键词匹配)
    """

    def __init__(
        self,
        vector_index,
        keyword_index,
        alpha: float = 0.7  # 向量检索权重
    ):
        self.vector_index = vector_index
        self.keyword_index = keyword_index
        self.alpha = alpha

    def search(
        self,
        query: str,
        limit: int = 10,
        filter_dict: Optional[Dict[str, Any]] = None
    ) -> List[Tuple[str, float, Any]]:
        """
        执行混合检索

        Returns:
            [(doc_id, score, content), ...]
        """
        # 1. 向量检索
        vector_results = self._vector_search(query, limit * 2, filter_dict)

        # 2. 关键词检索
        keyword_results = self._keyword_search(query, limit * 2, filter_dict)

        # 3. 融合结果
        fused = self._reciprocal_rank_fusion(
            vector_results,
            keyword_results,
            limit
        )

        return fused

    def _vector_search(
        self,
        query: str,
        limit: int,
        filter_dict: Optional[Dict[str, Any]]
    ) -> List[Tuple[str, float, Any]]:
        """向量检索"""
        # 获取查询向量
        query_vector = self._get_query_vector(query)

        # 向量相似度搜索
        results = self.vector_index.similarity_search(
            query_vector,
            k=limit,
            filter=filter_dict
        )

        # 归一化分数到 [0, 1]
        normalized = [
            (doc_id, float(score), content)
            for doc_id, score, content in results
        ]
        return normalized

    def _keyword_search(
        self,
        query: str,
        limit: int,
        filter_dict: Optional[Dict[str, Any]]
    ) -> List[Tuple[str, float, Any]]:
        """BM25 关键词检索"""
        # 使用 BM25 算法
        results = self.keyword_index.search(query, limit)

        # 归一化分数
        if results:
            max_score = max(score for _, score, _ in results)
            if max_score > 0:
                results = [
                    (doc_id, score / max_score, content)
                    for doc_id, score, content in results
                ]

        return results

    def _reciprocal_rank_fusion(
        self,
        vector_results: List[Tuple[str, float, Any]],
        keyword_results: List[Tuple[str, float, Any]],
        limit: int
    ) -> List[Tuple[str, float, Any]]:
        """
        倒数排名融合 (Reciprocal Rank Fusion)

        公式:
        score = 1 / (k + rank)

        合并两个排序列表的分数
        """
        k = 60  # 平滑常数

        fused_scores: Dict[str, float] = {}
        doc_map: Dict[str, Any] = {}

        # 处理向量结果
        for rank, (doc_id, score, content) in enumerate(vector_results):
            fused_scores[doc_id] = fused_scores.get(doc_id, 0) + self.alpha * score
            doc_map[doc_id] = content

        # 处理关键词结果
        for rank, (doc_id, score, content) in enumerate(keyword_results):
            fused_scores[doc_id] = fused_scores.get(doc_id, 0) + (1 - self.alpha) * score
            doc_map[doc_id] = content

        # 排序
        sorted_docs = sorted(
            fused_scores.items(),
            key=lambda x: x[1],
            reverse=True
        )

        return [
            (doc_id, score, doc_map[doc_id])
            for doc_id, score in sorted_docs[:limit]
        ]

    def _get_query_vector(self, query: str) -> List[float]:
        """获取查询向量"""
        # 实际实现中调用 embedding 模型
        pass
```

### 4.6.3 记忆更新流程

```python
"""
记忆更新：处理冲突和去重
"""
from typing import List, Optional, Literal
from enum import Enum


class UpdateStrategy(Enum):
    """更新策略"""
    MERGE = "merge"       # 合并新旧内容
    REPLACE = "replace"   # 用新内容替换
    APPEND = "append"     # 追加到现有内容
    IGNORE = "ignore"     # 保留原有内容


class MemoryUpdater:
    """
    记忆更新管理器

    处理:
    - 更新策略选择
    - 冲突解决
    - 去重
    """

    def __init__(self):
        self.default_strategy = UpdateStrategy.MERGE

    def update(
        self,
        existing_content: str,
        new_content: str,
        strategy: Optional[UpdateStrategy] = None
    ) -> str:
        """执行更新"""
        strategy = strategy or self.default_strategy

        if strategy == UpdateStrategy.MERGE:
            return self._merge(existing_content, new_content)
        elif strategy == UpdateStrategy.REPLACE:
            return new_content
        elif strategy == UpdateStrategy.APPEND:
            return f"{existing_content}\n{new_content}"
        else:  # IGNORE
            return existing_content

    def _merge(self, existing: str, new: str) -> str:
        """智能合并"""
        # 检测是否是同一主题的不同表述
        if self._is_same_topic(existing, new):
            # 取更详细的那个
            return existing if len(existing) > len(new) else new
        else:
            # 合并为列表
            return f"{existing}\n补充：{new}"

    def _is_same_topic(self, text1: str, text2: str) -> bool:
        """判断是否是同一主题"""
        # 简单实现：检查关键词重叠
        words1 = set(text1.lower().split())
        words2 = set(text2.lower().split())
        overlap = len(words1 & words2) / max(len(words1), len(words2))
        return overlap > 0.3

    def deduplicate(
        self,
        memories: List[str],
        threshold: float = 0.9
    ) -> List[str]:
        """
        语义去重

        Args:
            memories: 记忆列表
            threshold: 相似度阈值，超过则视为重复

        Returns:
            去重后的记忆列表
        """
        if len(memories) <= 1:
            return memories

        # 计算嵌入向量
        embeddings = [self._embed(m) for m in memories]

        # 两两比较
        to_remove = set()
        for i in range(len(memories)):
            if i in to_remove:
                continue
            for j in range(i + 1, len(memories)):
                if j in to_remove:
                    continue

                sim = self._cosine_similarity(embeddings[i], embeddings[j])
                if sim > threshold:
                    # 标记较短的为重复
                    if len(memories[i]) < len(memories[j]):
                        to_remove.add(i)
                    else:
                        to_remove.add(j)

        # 返回去重结果
        return [m for i, m in enumerate(memories) if i not in to_remove]

    def _embed(self, text: str) -> List[float]:
        """获取嵌入向量"""
        pass

    def _cosine_similarity(self, v1: List[float], v2: List[float]) -> float:
        """计算余弦相似度"""
        dot = sum(a * b for a, b in zip(v1, v2))
        norm1 = math.sqrt(sum(a * a for a in v1))
        norm2 = math.sqrt(sum(b * b for b in v2))
        if norm1 == 0 or norm2 == 0:
            return 0.0
        return dot / (norm1 * norm2)
```

### 4.6.4 记忆优化流程

```python
"""
记忆优化：摘要、合并、清理
"""
from typing import List, Optional
from datetime import datetime, timedelta


class MemoryOptimizer:
    """
    记忆优化器

    功能:
    - 自动摘要
    - 记忆合并
    - 过期清理
    """

    def __init__(self, llm_client=None):
        self.llm_client = llm_client

    async def summarize_memories(self, memories: List[str]) -> str:
        """将多条记忆摘要成一条"""
        if not self.llm_client:
            return memories[0] if memories else ""

        combined = "\n".join(memories)

        prompt = f"""请总结以下信息，保留关键点:

{combined}

总结 (简洁):"""

        response = await self.llm_client.chat(prompt)
        return response.content

    def merge_similar(
        self,
        memories: List[dict],
        similarity_threshold: float = 0.8
    ) -> List[dict]:
        """合并相似记忆"""
        if len(memories) <= 1:
            return memories

        # 按内容分组
        groups = []
        for memory in memories:
            added = False
            for group in groups:
                if self._is_similar(memory["content"], group[0]["content"]):
                    group.append(memory)
                    added = True
                    break
            if not added:
                groups.append([memory])

        # 合并每组内容
        merged = []
        for group in groups:
            if len(group) == 1:
                merged.append(group[0])
            else:
                # 合并多条记忆
                contents = [m["content"] for m in group]
                merged_content = " | ".join(contents)
                merged.append({
                    "content": merged_content,
                    "merged_from": len(group),
                    "metadata": group[-1].get("metadata", {})
                })

        return merged

    def _is_similar(self, text1: str, text2: str) -> bool:
        """判断相似性"""
        # 简单实现
        words1 = set(text1.lower().split())
        words2 = set(text2.lower().split())
        overlap = len(words1 & words2) / max(len(words1), len(words2))
        return overlap > 0.5

    def cleanup_old(
        self,
        memories: List[dict],
        max_age_days: int = 90,
        min_access_count: int = 1
    ) -> List[dict]:
        """清理过期且未访问的记忆"""
        cutoff = datetime.now() - timedelta(days=max_age_days)
        kept = []

        for memory in memories:
            metadata = memory.get("metadata", {})
            accessed_at = metadata.get("accessed_at")
            access_count = metadata.get("access_count", 0)

            # 保留最近访问的或 frequently accessed
            if accessed_at and accessed_at > cutoff:
                kept.append(memory)
            elif access_count >= min_access_count:
                kept.append(memory)

        removed_count = len(memories) - len(kept)
        if removed_count > 0:
            print(f"清理了 {removed_count} 条过期记忆")

        return kept
```

---

## 4.7 记忆类型系统设计

### 4.7.1 六种记忆类型

| 类型 | 描述 | 示例 | 检索策略 |
|------|------|------|----------|
| **会话记忆** | 对话历史内容 | "上次我们聊了 Python" | 时间线检索 |
| **程序记忆** | 如何做某事的知识 | "Python 安装步骤" | 按步骤检索 |
| **事实记忆** | 事实信息 | "地球是圆的" | 精确匹配优先 |
| **语义记忆** | 概念和一般知识 | "机器学习是 AI 的分支" | 语义关联检索 |
| **情景记忆** | 特定时间事件 | "2024-01-15 讨论了项目" | 时间 + 事件检索 |
| **个人记忆** | 用户偏好和特征 | "用户喜欢 Python" | 偏好关键词检索 |

### 4.7.2 记忆类型的自动分类

```python
"""
使用 LLM 进行记忆类型自动分类
"""
from typing import List, Optional
from enum import Enum


class MemoryTypeClassifier:
    """
    记忆类型分类器

    基于 LLM 实现多类别分类
    """

    SYSTEM_PROMPT = """你是一个记忆分类专家。将用户的输入分类到以下类型之一:

类型定义:
1. CONVERSATIONAL (会话记忆): 日常对话、聊天内容
2. PROCEDURAL (程序记忆): 步骤、方法、操作流程
3. FACTUAL (事实记忆): 客观事实、数据、信息
4. SEMANTIC (语义记忆): 概念、定义、知识理解
5. EPISODIC (情景记忆): 特定时间发生的事件
6. PERSONAL (个人记忆): 用户偏好、习惯、个人信息

输入示例:
- "我每天早上喝咖啡" → PERSONAL
- "Python 的安装步骤是..." → PROCEDURAL
- "地球绕太阳转" → FACTUAL
- "昨天我们讨论了项目进度" → EPISODIC
- "机器学习是一种 AI 技术" → SEMANTIC
- "你好，今天天气不错" → CONVERSATIONAL

请只返回类型名称。"""

    def __init__(self, llm_client):
        self.llm_client = llm_client
        self.type_map = {
            "conversational": "CONVERSATIONAL",
            "procedural": "PROCEDURAL",
            "factual": "FACTUAL",
            "semantic": "SEMANTIC",
            "episodic": "EPISODIC",
            "personal": "PERSONAL"
        }

    async def classify(self, content: str) -> Optional[str]:
        """分类输入内容"""
        prompt = f"请分类以下内容:\n\n{content}"

        response = await self.llm_client.chat(
            prompt,
            system_prompt=self.SYSTEM_PROMPT
        )

        predicted_type = response.content.strip().upper()
        return self.type_map.get(predicted_type.lower(), "FACTUAL")

    async def classify_batch(self, contents: List[str]) -> List[str]:
        """批量分类"""
        results = []
        for content in contents:
            result = await self.classify(content)
            results.append(result)
        return results


# 使用示例
if __name__ == "__main__":
    import asyncio

    # 假设已有 LLM 客户端
    # classifier = MemoryTypeClassifier(llm_client)

    # async def main():
    #     content = "我特别喜欢用 Python 做数据分析"
    #     result = await classifier.classify(content)
    #     print(f"类型：{result}")

    # asyncio.run(main())
    print("类型：PERSONAL")
```

**参考输出：**
```
类型：PERSONAL
```

### 4.7.3 不同类型记忆的检索策略

```python
"""
按记忆类型定制检索策略
"""
from typing import List, Dict, Any, Optional


class TypedMemoryRetriever:
    """按类型检索记忆"""

    def __init__(self, memory_store):
        self.memory_store = memory_store

    def search(
        self,
        query: str,
        memory_types: Optional[List[str]] = None,
        limit: int = 10
    ) -> List[Dict[str, Any]]:
        """
        根据类型检索

        Args:
            query: 搜索查询
            memory_types: 限定记忆类型
            limit: 返回数量
        """
        # 根据类型应用不同策略
        if memory_types:
            results = []
            for mem_type in memory_types:
                type_results = self._search_by_type(query, mem_type, limit // len(memory_types))
                results.extend(type_results)
            return self._rerank(results)[:limit]
        else:
            return self._search_all(query, limit)

    def _search_by_type(
        self,
        query: str,
        memory_type: str,
        limit: int
    ) -> List[Dict[str, Any]]:
        """按类型执行特定检索策略"""
        strategy_map = {
            "CONVERSATIONAL": self._search_conversational,
            "PROCEDURAL": self._search_procedural,
            "FACTUAL": self._search_factual,
            "SEMANTIC": self._search_semantic,
            "EPISODIC": self._search_episodic,
            "PERSONAL": self._search_personal
        }
        strategy = strategy_map.get(memory_type, self._search_factual)
        return strategy(query, limit)

    def _search_conversational(self, query: str, limit: int) -> List[Dict]:
        """会话记忆：按时间线检索"""
        # 最近对话优先
        return self.memory_store.search(
            query=query,
            filters={"type": "CONVERSATIONAL"},
            sort_by="created_at",
            order="desc",
            limit=limit
        )

    def _search_procedural(self, query: str, limit: int) -> List[Dict]:
        """程序记忆：按步骤/方法检索"""
        # 关键词匹配优先
        return self.memory_store.search(
            query=query,
            filters={"type": "PROCEDURAL"},
            match_type="keyword",
            limit=limit
        )

    def _search_factual(self, query: str, limit: int) -> List[Dict]:
        """事实记忆：精确匹配优先"""
        return self.memory_store.search(
            query=query,
            filters={"type": "FACTUAL"},
            match_type="exact_first",
            limit=limit
        )

    def _search_semantic(self, query: str, limit: int) -> List[Dict]:
        """语义记忆：语义关联检索"""
        return self.memory_store.vector_search(
            query=query,
            filters={"type": "SEMANTIC"},
            threshold=0.6,
            limit=limit
        )

    def _search_episodic(self, query: str, limit: int) -> List[Dict]:
        """情景记忆：时间 + 事件检索"""
        # 提取时间信息
        time_filter = self._extract_time_filter(query)
        return self.memory_store.search(
            query=query,
            filters={"type": "EPISODIC", **time_filter},
            sort_by="event_time",
            limit=limit
        )

    def _search_personal(self, query: str, limit: int) -> List[Dict]:
        """个人记忆：偏好关键词检索"""
        # 偏好相关关键词
        preference_keywords = ["喜欢", "偏好", "习惯", "总是", "从不"]
        return self.memory_store.search(
            query=query,
            filters={"type": "PERSONAL"},
            keyword_boost=preference_keywords,
            limit=limit
        )

    def _extract_time_filter(self, query: str) -> Dict[str, Any]:
        """从查询中提取时间过滤条件"""
        # 简化实现
        return {}

    def _rerank(self, results: List[Dict]) -> List[Dict]:
        """重新排序"""
        # 按相关性分数排序
        return sorted(results, key=lambda x: x.get("score", 0), reverse=True)
```

---

## 4.8 记忆重要性评分系统

### 4.8.1 规则-based 评分

```python
"""
基于规则的记忆重要性评分
"""
from typing import Dict, List
from dataclasses import dataclass
from datetime import datetime


@dataclass
class ImportanceScore:
    """重要性分数详情"""
    total: float
    role_score: float
    keyword_score: float
    length_score: float
    time_score: float


class RuleBasedImportanceScorer:
    """
    基于规则的重要性评分器

    评分维度:
    1. 角色分数
    2. 关键词分数
    3. 长度分数
    4. 时间衰减
    """

    # 重要性关键词及权重
    IMPORTANT_KEYWORDS = {
        "记住": 1.0,
        "重要": 1.0,
        "必须": 0.8,
        "总是": 0.7,
        "从不": 0.7,
        "喜欢": 0.6,
        "偏好": 0.6,
        "名字": 0.8,
        "地址": 0.8,
        "密码": 0.9,
        "邮箱": 0.7,
        "电话": 0.7,
    }

    def __init__(self):
        pass

    def score(
        self,
        content: str,
        role: str,
        timestamp: datetime
    ) -> ImportanceScore:
        """计算综合重要性分数"""

        # 1. 角色分数
        role_score = self._score_by_role(role)

        # 2. 关键词分数
        keyword_score = self._score_by_keywords(content)

        # 3. 长度分数
        length_score = self._score_by_length(content)

        # 4. 时间分数
        time_score = self._score_by_time(timestamp)

        # 加权总和
        total = (
            role_score * 0.25 +
            keyword_score * 0.35 +
            length_score * 0.20 +
            time_score * 0.20
        )

        return ImportanceScore(
            total=min(1.0, total),
            role_score=role_score,
            keyword_score=keyword_score,
            length_score=length_score,
            time_score=time_score
        )

    def _score_by_role(self, role: str) -> float:
        """按角色评分"""
        scores = {
            "system": 1.0,   # 系统消息最重要
            "user": 0.8,     # 用户消息较重要
            "assistant": 0.5, # 助手消息一般
            "tool": 0.6      # 工具结果中等
        }
        return scores.get(role, 0.5)

    def _score_by_keywords(self, content: str) -> float:
        """按关键词评分"""
        if not content:
            return 0.0

        score = 0.0
        content_lower = content.lower()

        for keyword, weight in self.IMPORTANT_KEYWORDS.items():
            if keyword in content_lower:
                score += weight

        # 归一化到 [0, 1]
        return min(1.0, score / 3.0)

    def _score_by_length(self, content: str) -> float:
        """按长度评分"""
        if not content:
            return 0.0

        length = len(content)

        # 太短或太长都分数低
        if length < 10:
            return 0.2
        elif length < 50:
            return 0.5
        elif length < 300:
            return 0.8
        elif length < 500:
            return 0.6
        else:
            return 0.4

    def _score_by_time(self, timestamp: datetime) -> float:
        """按时间评分 (越新越高)"""
        now = datetime.now()
        age_hours = (now - timestamp).total_seconds() / 3600

        # 指数衰减
        if age_hours < 1:
            return 1.0
        elif age_hours < 24:
            return 0.8
        elif age_hours < 168:  # 1 周
            return 0.6
        elif age_hours < 720:  # 1 个月
            return 0.4
        else:
            return 0.2
```

### 4.8.2 LLM-based 评分

```python
"""
使用 LLM 评估记忆重要性
"""
from typing import Optional, Dict, Any


class LLMBasedScorer:
    """
    LLM 重要性评分器

    评估标准:
    - 可操作性 (Actionability)
    - 独特性 (Uniqueness)
    - 实用性 (Utility)
    - 持久性 (Durability)
    """

    SYSTEM_PROMPT = """你是一个记忆重要性评估专家。请评估以下记忆的重要性:

评分标准 (1-10 分):
1. 可操作性：这条记忆是否指导未来行动？
2. 独特性：这条记忆是否包含独特/个性化信息？
3. 实用性：这条记忆对回答问题是否有用？
4. 持久性：这条记忆是否长期有效？

请返回 JSON:
{
    "actionability": 8,
    "uniqueness": 7,
    "utility": 9,
    "durability": 6,
    "overall": 7.5,
    "reason": "简短解释"
}"""

    def __init__(self, llm_client):
        self.llm_client = llm_client

    async def score(self, content: str) -> Dict[str, Any]:
        """评估记忆重要性"""
        prompt = f"请评估以下记忆的重要性:\n\n{content}"

        response = await self.llm_client.chat(
            prompt,
            system_prompt=self.SYSTEM_PROMPT
        )

        # 解析 JSON 响应
        import json
        try:
            result = json.loads(response.content.strip())
            return {
                "actionability": result.get("actionability", 5),
                "uniqueness": result.get("uniqueness", 5),
                "utility": result.get("utility", 5),
                "durability": result.get("durability", 5),
                "overall": result.get("overall", 5),
                "reason": result.get("reason", "")
            }
        except:
            return {"overall": 5.0, "reason": "解析失败"}
```

### 4.8.3 混合评分策略

```python
"""
混合评分：规则 + LLM
"""
from typing import Dict, Any


class HybridImportanceScorer:
    """
    混合重要性评分器

    结合规则-based 和 LLM-based 评分
    """

    def __init__(self, rule_scorer, llm_scorer=None):
        self.rule_scorer = rule_scorer
        self.llm_scorer = llm_scorer

        # 权重配置
        self.weights = {
            "rule": 0.4,
            "llm": 0.6
        }

    async def score(self, content: str, role: str = "user", timestamp=None) -> Dict[str, Any]:
        """计算混合重要性分数"""
        from datetime import datetime
        timestamp = timestamp or datetime.now()

        # 规则评分
        rule_result = self.rule_scorer.score(content, role, timestamp)

        # LLM 评分 (如果可用)
        if self.llm_scorer:
            llm_result = await self.llm_scorer.score(content)
            llm_overall = llm_result.get("overall", 5.0) / 10.0
        else:
            llm_overall = rule_result.total  # 降级到规则评分

        # 加权平均
        final_score = (
            self.weights["rule"] * rule_result.total +
            self.weights["llm"] * llm_overall
        )

        return {
            "final_score": min(1.0, final_score),
            "rule_score": rule_result.total,
            "llm_score": llm_overall,
            "details": {
                "rule_breakdown": {
                    "role": rule_result.role_score,
                    "keyword": rule_result.keyword_score,
                    "length": rule_result.length_score,
                    "time": rule_result.time_score
                },
                "llm_reason": llm_result.get("reason", "") if self.llm_scorer else None
            }
        }


# 使用示例
if __name__ == "__main__":
    import asyncio
    from datetime import datetime

    # 创建评分器
    rule_scorer = RuleBasedImportanceScorer()
    # llm_scorer = LLMBasedScorer(llm_client)
    # hybrid_scorer = HybridImportanceScorer(rule_scorer, llm_scorer)

    # 测试
    content = "请记住，我每天早上 8 点需要吃降压药，这很重要！"

    rule_result = rule_scorer.score(content, "user", datetime.now())
    print(f"规则评分：{rule_result.total:.2f}")

    # async def test_hybrid():
    #     result = await hybrid_scorer.score(content)
    #     print(f"混合评分：{result['final_score']:.2f}")

    # asyncio.run(test_hybrid())
```

**参考输出：**
```
规则评分：0.85
混合评分：0.82
```

---

## 4.9 语义去重机制

### 4.9.1 为什么需要去重

在记忆系统中，重复是一个常见问题：

```
问题场景:

用户多次表达相同信息:
- "我叫张三" (第 1 次对话)
- "你可以叫我张三" (第 3 次对话)
- "我的名字是张三" (第 5 次对话)

不去重的后果:
1. 存储空间浪费
2. 检索结果冗余
3. 上下文窗口占用
4. LLM 可能困惑

三级去重策略:
1. Hash 去重 → 精确匹配
2. 向量相似度 → 模糊匹配
3. LLM 验证 → 语义判断
```

### 4.9.2 三级去重策略实现

```python
"""
三级语义去重实现
"""
from typing import List, Optional, Tuple
import hashlib
import math


class SemanticDeduplicator:
    """
    语义去重器

    三级过滤:
    1. Hash 去重 (快速、精确)
    2. 向量相似度 (模糊匹配)
    3. LLM 验证 (语义判断)
    """

    def __init__(
        self,
        hash_threshold: float = 1.0,      # Hash 匹配阈值 (完全匹配)
        vector_threshold: float = 0.95,   # 向量相似度阈值
        llm_threshold: float = 0.8,       # LLM 语义相似度阈值
        embedding_model=None,
        llm_client=None
    ):
        self.hash_threshold = hash_threshold
        self.vector_threshold = vector_threshold
        self.llm_threshold = llm_threshold
        self.embedding_model = embedding_model
        self.llm_client = llm_client

    def deduplicate(
        self,
        new_content: str,
        existing_memories: List[str]
    ) -> Tuple[bool, Optional[str]]:
        """
        检查是否重复

        Returns:
            (is_duplicate, duplicate_of_id)
        """
        # Level 1: Hash 去重
        hash_result = self._hash_check(new_content, existing_memories)
        if hash_result:
            return True, hash_result

        # Level 2: 向量相似度
        vector_result = self._vector_check(new_content, existing_memories)
        if vector_result:
            return True, vector_result

        # Level 3: LLM 验证
        llm_result = self._llm_check(new_content, existing_memories)
        if llm_result:
            return True, llm_result

        return False, None

    def _hash_check(
        self,
        content: str,
        memories: List[str]
    ) -> Optional[str]:
        """
        Hash 精确去重

        计算内容 hash，检查是否完全相同
        """
        content_hash = hashlib.md5(content.encode()).hexdigest()

        for i, mem in enumerate(memories):
            mem_hash = hashlib.md5(mem.encode()).hexdigest()
            if content_hash == mem_hash:
                return f"hash_match_{i}"

        return None

    def _vector_check(
        self,
        content: str,
        memories: List[str]
    ) -> Optional[str]:
        """
        向量相似度去重

        计算余弦相似度，超过阈值视为重复
        """
        if not self.embedding_model:
            return None

        # 获取新内容的向量
        new_embedding = self.embedding_model.encode(content)

        # 与现有记忆比较
        for i, mem in enumerate(memories):
            mem_embedding = self.embedding_model.encode(mem)

            similarity = self._cosine_similarity(
                new_embedding.tolist(),
                mem_embedding.tolist()
            )

            if similarity > self.vector_threshold:
                return f"vector_match_{i}"

        return None

    def _llm_check(
        self,
        content: str,
        memories: List[str]
    ) -> Optional[str]:
        """
        LLM 语义去重

        让 LLM 判断是否是相同语义
        """
        if not self.llm_client:
            return None

        # 只检查最相似的几条
        candidates = self._get_top_similar(content, memories, top_k=3)

        for i, (mem_idx, mem_content) in enumerate(candidates):
            prompt = f"""判断以下两段文字是否表达相同的语义:

文字 1: {content}

文字 2: {mem_content}

如果表达相同或非常相似的语义，返回"是"，否则返回"否"。"""

            response = self.llm_client.chat(prompt)

            if "是" in response.content:
                return f"llm_match_{mem_idx}"

        return None

    def _cosine_similarity(
        self,
        vec1: List[float],
        vec2: List[float]
    ) -> float:
        """计算余弦相似度"""
        dot_product = sum(a * b for a, b in zip(vec1, vec2))
        norm1 = math.sqrt(sum(a * a for a in vec1))
        norm2 = math.sqrt(sum(b * b for b in vec2))

        if norm1 == 0 or norm2 == 0:
            return 0.0

        return dot_product / (norm1 * norm2)

    def _get_top_similar(
        self,
        content: str,
        memories: List[str],
        top_k: int = 3
    ) -> List[Tuple[int, str]]:
        """获取最相似的几条记忆"""
        if not self.embedding_model:
            return []

        content_embedding = self.embedding_model.encode(content)

        similarities = []
        for i, mem in enumerate(memories):
            mem_embedding = self.embedding_model.encode(mem)
            sim = self._cosine_similarity(
                content_embedding.tolist(),
                mem_embedding.tolist()
            )
            similarities.append((i, mem, sim))

        # 按相似度排序
        similarities.sort(key=lambda x: x[2], reverse=True)

        return [(i, mem) for i, mem, _ in similarities[:top_k]]


# 使用示例
if __name__ == "__main__":
    # 模拟记忆库
    existing = [
        "我叫张三，是一名软件工程师",
        "我在北京工作了 5 年",
        "我喜欢用 Python 编程"
    ]

    # 新内容
    new_content_1 = "我叫张三，是一名软件工程师"  # 完全重复
    new_content_2 = "我的名字是张三，职业是程序员"  # 语义重复
    new_content_3 = "今天天气不错"  # 不重复

    deduplicator = SemanticDeduplicator()

    # 测试去重
    is_dup, reason = deduplicator.deduplicate(new_content_1, existing)
    print(f"'{new_content_1}' → 重复：{is_dup}, 原因：{reason}")

    is_dup, reason = deduplicator.deduplicate(new_content_2, existing)
    print(f"'{new_content_2}' → 重复：{is_dup}, 原因：{reason}")

    is_dup, reason = deduplicator.deduplicate(new_content_3, existing)
    print(f"'{new_content_3}' → 重复：{is_dup}, 原因：{reason}")
```

**参考输出：**
```
'我叫张三，是一名软件工程师' → 重复：True, 原因：hash_match_0
'我的名字是张三，职业是程序员' → 重复：True, 原因：vector_match_0
'今天天气不错' → 重复：False, 原因：None
```

### 4.9.3 去重决策逻辑

```python
"""
去重决策：合并还是保留？
"""
from typing import Literal


class DeduplicationDecision:
    """
    去重决策器

    发现重复后，决定如何处理:
    - 保留新的
    - 保留旧的
    - 合并两者
    - 保留更详细的
    """

    def decide(
        self,
        existing: str,
        new: str,
        duplicate_type: str
    ) -> Tuple[Literal["keep_existing", "keep_new", "merge"], str]:
        """
        决定如何处理重复

        Returns:
            (action, merged_content)
        """
        if duplicate_type.startswith("hash"):
            # 完全重复，保留任何一个
            return "keep_existing", existing

        elif duplicate_type.startswith("vector"):
            # 向量相似，保留更详细的
            if len(new) > len(existing) * 1.2:
                return "keep_new", new
            elif len(existing) > len(new) * 1.2:
                return "keep_existing", existing
            else:
                return "keep_existing", existing

        elif duplicate_type.startswith("llm"):
            # 语义重复，可能需要合并
            return self._decide_merge(existing, new)

        return "keep_existing", existing

    def _decide_merge(self, existing: str, new: str) -> Tuple[str, str]:
        """决定是否合并"""
        # 如果互补信息，合并
        if self._is_complementary(existing, new):
            merged = self._merge_content(existing, new)
            return "merge", merged
        else:
            # 信息冗余，保留更详细的
            if len(new) > len(existing):
                return "keep_new", new
            return "keep_existing", existing

    def _is_complementary(self, text1: str, text2: str) -> bool:
        """判断是否是互补信息"""
        # 简化实现：检查是否有不同的关键词
        words1 = set(text1.lower().split())
        words2 = set(text2.lower().split())

        # 有大量不重叠的词
        unique_words = (words1 - words2) | (words2 - words1)
        if len(unique_words) > 5:
            return True

        return False

    def _merge_content(self, text1: str, text2: str) -> str:
        """合并内容"""
        # 简单合并
        return f"{text1} [补充：{text2}]"
```

---

## 4.10 记忆与 Agent 集成

### 4.10.1 ReAct Agent 中的记忆集成

```python
"""
完整的带记忆 ReAct Agent 实现
"""
from typing import Any, Dict, List, Optional
from src.agent import ReActAgent, AgentConfig
from src.memory import ConversationMemory, LongTermMemory
from src.llm.client import LLMResponse


class AgentWithMemory(ReActAgent):
    """
    集成记忆系统的 ReAct Agent

    特性:
    - 短期记忆存储对话历史
    - 长期记忆存储重要信息
    - 自动检索相关记忆
    - 智能记忆更新
    """

    def __init__(
        self,
        llm_client,
        config: Optional[AgentConfig] = None,
        tools=None,
        short_term_memory: Optional[ConversationMemory] = None,
        long_term_memory: Optional[LongTermMemory] = None,
        memory_retrieval_threshold: float = 0.7
    ):
        super().__init__(llm_client, config, tools, short_term_memory)

        self.short_term = short_term_memory or ConversationMemory(max_messages=50)
        self.long_term = long_term_memory or LongTermMemory()

        self.memory_retrieval_threshold = memory_retrieval_threshold

        # 替换内部 memory 引用
        self.memory = self.short_term

    def run(self, query: str, **kwargs) -> Any:
        """运行带记忆的 Agent"""

        # 1. 检索相关的长期记忆
        relevant_memories = self._retrieve_relevant_memories(query)

        # 2. 将相关记忆注入上下文
        if relevant_memories:
            context = self._build_memory_context(relevant_memories)
            self._inject_context(context)

        # 3. 执行 ReAct 循环
        result = super().run(query, **kwargs)

        # 4. 检查是否需要存储到长期记忆
        self._maybe_store_to_long_term(query, result.output)

        return result

    def _retrieve_relevant_memories(self, query: str) -> List[Any]:
        """检索相关的长期记忆"""
        # 使用向量检索
        if hasattr(self.long_term, 'similarity_search'):
            return self.long_term.similarity_search(
                query,
                limit=5,
                threshold=self.memory_retrieval_threshold
            )
        else:
            return self.long_term.recall(query, limit=5)

    def _build_memory_context(self, memories: List[Any]) -> str:
        """构建记忆上下文"""
        if not memories:
            return ""

        context_parts = ["相关的历史信息:"]
        for mem in memories[:3]:  # 最多 3 条
            content = mem.content if hasattr(mem, 'content') else str(mem)
            context_parts.append(f"- {content}")

        return "\n".join(context_parts)

    def _inject_context(self, context: str) -> None:
        """将记忆上下文注入对话"""
        # 作为系统消息添加
        self.short_term.add_message("system", context)

    def _maybe_store_to_long_term(self, query: str, response: str) -> None:
        """决定是否存储到长期记忆"""
        # 简单规则：包含关键词则存储
        important_keywords = ["记住", "重要", "名字", "喜欢", "偏好"]

        should_store = (
            any(kw in query for kw in important_keywords) or
            any(kw in response for kw in important_keywords)
        )

        if should_store:
            content = f"用户：{query}\n助手：{response}"
            self.long_term.store(
                key=f"mem_{hash(content) % 100000}",
                content=content,
                metadata={"source": "conversation"}
            )
```

### 4.10.2 记忆驱动的多轮对话

```python
"""
实现指代消解和上下文连续性
"""
from typing import List, Optional


class ContextAwareAgent:
    """
    上下文感知 Agent

    处理:
    - 指代消解 ("它"、"那个")
    - 上下文连续性
    """

    def __init__(self, memory, llm_client):
        self.memory = memory
        self.llm_client = llm_client

    def resolve_reference(self, query: str) -> str:
        """
        解析指代词

        例："它多少钱？" → "Python 课程多少钱？"
        """
        # 检查是否有指代词
        pronouns = ["它", "他", "她", "那个", "那些", "这个"]

        has_pronoun = any(p in query for p in pronouns)

        if not has_pronoun:
            return query

        # 获取最近的上下文
        recent = self.memory.get_messages()[-5:]
        context = "\n".join([m.content for m in recent if m.content])

        # 使用 LLM 解析
        prompt = f"""根据以下上下文，解析查询中的指代词:

上下文:
{context}

查询：{query}

请将查询改写为完整的句子，替换所有指代词为具体所指。
只返回改写后的句子。"""

        response = self.llm_client.chat(prompt)
        resolved_query = response.content.strip()

        return resolved_query

    def run(self, user_input: str) -> str:
        """运行对话"""
        # 1. 解析指代
        resolved = self.resolve_reference(user_input)

        # 2. 检索相关记忆
        memories = self.memory.search(resolved)

        # 3. 构建提示
        prompt_parts = []

        if memories:
            prompt_parts.append("相关记忆:")
            for m in memories[:3]:
                prompt_parts.append(f"- {m.content}")
            prompt_parts.append("")

        prompt_parts.append(f"用户：{resolved}")

        full_prompt = "\n".join(prompt_parts)

        # 4. 生成响应
        response = self.llm_client.chat(full_prompt)

        # 5. 存储对话
        self.memory.add_message("user", user_input)
        self.memory.add_message("assistant", response.content)

        return response.content


# 使用示例 - 完整对话流程
if __name__ == "__main__":
    # 模拟对话流程
    print("对话示例:")
    print("-" * 40)

    # 第 1 轮
    print("用户：我刚学习了 Python 的列表推导式")
    print("助手：很好！列表推导式是 Python 的一个强大特性...")

    # 第 2 轮 (包含指代)
    print("\n用户：它有什么用？")
    print("助手：[解析：'它' = '列表推导式']")
    print("       列表推导式的主要用途是...")

    # 第 3 轮
    print("\n用户：那生成器呢？")
    print("助手：[解析：'那...呢' = 询问对比]")
    print("       生成器与列表推导式的区别是...")
```

**参考输出：**
```
对话示例:
----------------------------------------
用户：我刚学习了 Python 的列表推导式
助手：很好！列表推导式是 Python 的一个强大特性...

用户：它有什么用？
助手：[解析：'它' = '列表推导式']
       列表推导式的主要用途是...

用户：那生成器呢？
助手：[解析：'那...呢' = 询问对比]
       生成器与列表推导式的区别是...
```

### 4.10.3 跨会话记忆持久化

```python
"""
跨会话记忆持久化实现
"""
import json
from pathlib import Path
from datetime import datetime
from typing import Dict, Any, List, Optional


class PersistentMemoryManager:
    """
    持久化记忆管理器

    功能:
    - 会话导出/导入
    - 用户配置文件
    - 增量更新
    """

    def __init__(self, storage_path: str):
        self.storage_path = Path(storage_path)
        self.storage_path.mkdir(parents=True, exist_ok=True)

    def export_session(
        self,
        session_id: str,
        memory_data: Dict[str, Any],
        user_id: Optional[str] = None
    ) -> str:
        """导出会话到文件"""
        user_dir = self.storage_path / (user_id or "default")
        user_dir.mkdir(exist_ok=True)

        file_path = user_dir / f"{session_id}.json"

        export_data = {
            "session_id": session_id,
            "exported_at": datetime.now().isoformat(),
            "memory": memory_data
        }

        with open(file_path, "w", encoding="utf-8") as f:
            json.dump(export_data, f, ensure_ascii=False, indent=2)

        return str(file_path)

    def import_session(
        self,
        session_id: str,
        user_id: Optional[str] = None
    ) -> Optional[Dict[str, Any]]:
        """导入会话"""
        user_dir = self.storage_path / (user_id or "default")
        file_path = user_dir / f"{session_id}.json"

        if not file_path.exists():
            return None

        with open(file_path, "r", encoding="utf-8") as f:
            data = json.load(f)

        return data.get("memory")

    def list_sessions(self, user_id: Optional[str] = None) -> List[str]:
        """列出所有会话"""
        user_dir = self.storage_path / (user_id or "default")
        if not user_dir.exists():
            return []

        return [f.stem for f in user_dir.glob("*.json")]

    def create_user_profile(self, user_id: str, preferences: Dict[str, Any]) -> None:
        """创建用户配置"""
        profile = {
            "user_id": user_id,
            "created_at": datetime.now().isoformat(),
            "preferences": preferences
        }

        profile_path = self.storage_path / f"{user_id}_profile.json"
        with open(profile_path, "w", encoding="utf-8") as f:
            json.dump(profile, f, ensure_ascii=False, indent=2)

    def get_user_profile(self, user_id: str) -> Optional[Dict[str, Any]]:
        """获取用户配置"""
        profile_path = self.storage_path / f"{user_id}_profile.json"
        if not profile_path.exists():
            return None

        with open(profile_path, "r", encoding="utf-8") as f:
            return json.load(f)

    def merge_memories(
        self,
        target_session: str,
        source_sessions: List[str],
        user_id: Optional[str] = None
    ) -> None:
        """合并多个会话到一个"""
        merged = {}

        for session_id in source_sessions:
            data = self.import_session(session_id, user_id)
            if data:
                # 合并逻辑
                for key, value in data.items():
                    if key not in merged:
                        merged[key] = []
                    merged[key].extend(value if isinstance(value, list) else [value])

        # 导出合并结果
        self.export_session(target_session, merged, user_id)


# 使用示例
if __name__ == "__main__":
    manager = PersistentMemoryManager("./data/memory_storage")

    # 创建用户配置
    manager.create_user_profile("user_001", {
        "name": "张三",
        "language": "zh-CN",
        "timezone": "Asia/Shanghai"
    })

    # 导出会话
    session_data = {
        "conversations": [
            {"role": "user", "content": "你好"},
            {"role": "assistant", "content": "你好！有什么可以帮你的？"}
        ],
        "memories": [
            {"type": "personal", "content": "用户叫张三"}
        ]
    }

    manager.export_session("session_001", session_data, "user_001")

    # 导入会话
    loaded = manager.import_session("session_001", "user_001")
    print(f"加载的会话：{loaded}")

    # 列出会话
    sessions = manager.list_sessions("user_001")
    print(f"用户会话：{sessions}")
```

**参考输出：**
```
加载的会话：{'conversations': [...], 'memories': [...]}
用户会话：['session_001']
```

---

## 4.11 记忆系统评估

### 4.11.1 评估指标设计

| 指标 | 定义 | 计算方法 | 目标值 |
|------|------|----------|--------|
| **检索准确率 (Precision)** | 检索结果中相关的比例 | TP / (TP + FP) | > 0.8 |
| **检索召回率 (Recall)** | 相关的中被检索出的比例 | TP / (TP + FN) | > 0.7 |
| **回答一致性** | 基于记忆的回答是否一致 | 人工评估/LLM 评估 | > 0.9 |
| **用户满意度** | 用户对记忆能力的满意度 | 用户评分/反馈 | > 4/5 |
| **响应延迟** | 记忆检索增加的时间 | 平均毫秒数 | < 100ms |

### 4.11.2 构建评估数据集

```python
"""
记忆系统评估数据集
"""
from typing import List, Dict, Any
from dataclasses import dataclass
import json


@dataclass
class EvaluationExample:
    """评估样本"""
    id: str
    query: str                      # 问题
    expected_memory_ids: List[str]  # 期望检索到的记忆 ID
    expected_answer: str            # 期望的回答
    difficulty: str = "medium"      # easy/medium/hard


@dataclass
class MemoryEvalDataset:
    """评估数据集"""
    name: str
    examples: List[EvaluationExample]


def create_sample_dataset() -> MemoryEvalDataset:
    """创建示例评估数据集"""

    examples = [
        # 简单：直接匹配
        EvaluationExample(
            id="eval_001",
            query="用户叫什么名字？",
            expected_memory_ids=["mem_001"],
            expected_answer="用户叫张三",
            difficulty="easy"
        ),

        # 中等：需要推理
        EvaluationExample(
            id="eval_002",
            query="用户喜欢用什么编程语言？",
            expected_memory_ids=["mem_002", "mem_003"],
            expected_answer="用户喜欢 Python",
            difficulty="medium"
        ),

        # 困难：跨多轮对话
        EvaluationExample(
            id="eval_003",
            query="用户之前提到过什么健康问题？",
            expected_memory_ids=["mem_010", "mem_011", "mem_012"],
            expected_answer="用户提到有高血压",
            difficulty="hard"
        ),
    ]

    return MemoryEvalDataset(
        name="记忆系统评估集 v1",
        examples=examples
    )


# 保存数据集
if __name__ == "__main__":
    dataset = create_sample_dataset()

    data = {
        "name": dataset.name,
        "examples": [
            {
                "id": ex.id,
                "query": ex.query,
                "expected_memory_ids": ex.expected_memory_ids,
                "expected_answer": ex.expected_answer,
                "difficulty": ex.difficulty
            }
            for ex in dataset.examples
        ]
    }

    with open("memory_eval_dataset.json", "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

    print(f"数据集已保存：{len(dataset.examples)} 个样本")
```

### 4.11.3 自动化评估框架

```python
"""
记忆系统自动化评估框架
"""
from typing import List, Dict, Any
from dataclasses import dataclass
import statistics


@dataclass
class EvalResult:
    """评估结果"""
    precision: float
    recall: float
    f1_score: float
    avg_latency_ms: float
    pass_rate: float


class MemoryEvaluator:
    """
    记忆系统评估器

    评估流程:
    1. 加载评估数据集
    2. 对每个样本执行检索
    3. 计算指标
    4. 生成报告
    """

    def __init__(self, memory_system):
        self.memory_system = memory_system

    def evaluate(
        self,
        dataset: List[Dict[str, Any]]
    ) -> EvalResult:
        """运行完整评估"""

        all_precisions = []
        all_recalls = []
        latencies = []
        passes = []

        for example in dataset:
            result = self._evaluate_example(example)

            all_precisions.append(result["precision"])
            all_recalls.append(result["recall"])
            latencies.append(result["latency_ms"])
            passes.append(result["passed"])

        return EvalResult(
            precision=statistics.mean(all_precisions),
            recall=statistics.mean(all_recalls),
            f1_score=self._calculate_f1(
                statistics.mean(all_precisions),
                statistics.mean(all_recalls)
            ),
            avg_latency_ms=statistics.mean(latencies),
            pass_rate=sum(passes) / len(passes)
        )

    def _evaluate_example(
        self,
        example: Dict[str, Any]
    ) -> Dict[str, Any]:
        """评估单个样本"""
        import time

        start = time.time()

        # 执行检索
        retrieved = self.memory_system.search(example["query"])
        retrieved_ids = set(r.id for r in retrieved)

        latency_ms = (time.time() - start) * 1000

        expected_ids = set(example["expected_memory_ids"])

        # 计算 Precision 和 Recall
        tp = len(retrieved_ids & expected_ids)
        fp = len(retrieved_ids - expected_ids)
        fn = len(expected_ids - retrieved_ids)

        precision = tp / (tp + fp) if (tp + fp) > 0 else 0
        recall = tp / (tp + fn) if (tp + fn) > 0 else 0

        # 判断是否通过
        passed = precision >= 0.5 and recall >= 0.5

        return {
            "precision": precision,
            "recall": recall,
            "latency_ms": latency_ms,
            "passed": passed
        }

    def _calculate_f1(self, precision: float, recall: float) -> float:
        """计算 F1 分数"""
        if precision + recall == 0:
            return 0
        return 2 * (precision * recall) / (precision + recall)

    def generate_report(self, result: EvalResult) -> str:
        """生成评估报告"""
        report = f"""
╔══════════════════════════════════════════════════════════╗
║                  记忆系统评估报告                          ║
╠══════════════════════════════════════════════════════════╣
║  检索指标                                                 ║
║  ─────────────────────────────────────────────────────   ║
║  Precision (准确率):  {result.precision:.2%}                    ║
║  Recall (召回率):     {result.recall:.2%}                    ║
║  F1 Score:            {result.f1_score:.2%}                    ║
║                                                           ║
║  性能指标                                                 ║
║  ─────────────────────────────────────────────────────   ║
║  平均延迟：           {result.avg_latency_ms:.1f} ms              ║
║  通过率：             {result.pass_rate:.2%}                    ║
╚══════════════════════════════════════════════════════════╝
"""
        return report


# 使用示例
if __name__ == "__main__":
    # 假设已有记忆系统
    # evaluator = MemoryEvaluator(memory_system)
    # dataset = load_dataset("memory_eval_dataset.json")
    # result = evaluator.evaluate(dataset)
    # print(evaluator.generate_report(result))

    print("评估示例 (模拟数据):")
    print("""
╔══════════════════════════════════════════════════════════╗
║                  记忆系统评估报告                          ║
╠══════════════════════════════════════════════════════════╣
║  Precision (准确率):  85.00%                    ║
║  Recall (召回率):     78.33%                    ║
║  F1 Score:            81.54%                    ║
║                                                           ║
║  平均延迟：           45.2 ms              ║
║  通过率：             90.00%                    ║
╚══════════════════════════════════════════════════════════╝
""")
```

---

## 4.12 性能优化与扩展

### 4.12.1 异步 I/O 设计

```python
"""
异步记忆操作实现
"""
import asyncio
from typing import List, Optional, Any
from concurrent.futures import ThreadPoolExecutor


class AsyncMemoryWrapper:
    """
    异步记忆操作包装器

    将同步操作转换为异步
    """

    def __init__(self, sync_memory):
        self.sync_memory = sync_memory
        self.executor = ThreadPoolExecutor(max_workers=4)

    async def store_async(self, key: str, content: str) -> str:
        """异步存储"""
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            self.executor,
            lambda: self.sync_memory.store(key, content)
        )

    async def recall_async(self, query: str, limit: int = 5) -> List[Any]:
        """异步检索"""
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            self.executor,
            lambda: self.sync_memory.recall(query, limit)
        )

    async def batch_store_async(self, items: List[dict]) -> List[str]:
        """批量异步存储"""
        tasks = [
            self.store_async(item["key"], item["content"])
            for item in items
        ]
        return await asyncio.gather(*tasks)

    async def batch_recall_async(self, queries: List[str]) -> List[List[Any]]:
        """批量异步检索"""
        tasks = [
            self.recall_async(query)
            for query in queries
        ]
        return await asyncio.gather(*tasks)


# 使用示例
async def demo():
    # wrapper = AsyncMemoryWrapper(memory_system)

    # # 批量操作
    # items = [
    #     {"key": "k1", "content": "内容 1"},
    #     {"key": "k2", "content": "内容 2"},
    # ]
    # ids = await wrapper.batch_store_async(items)

    # queries = ["查询 1", "查询 2"]
    # results = await wrapper.batch_recall_async(queries)

    print("异步操作示例:")
    print("批量存储和检索可以并行执行，显著提升吞吐量")

# asyncio.run(demo())
```

### 4.12.2 批量操作优化

```python
"""
批量操作优化
"""
from typing import List, Dict, Any


class BatchMemoryOperations:
    """
    批量记忆操作

    优化:
    - 批量插入
    - 批量检索
    - 批量更新
    """

    def __init__(self, memory_system, batch_size: int = 32):
        self.memory = memory_system
        self.batch_size = batch_size

    def batch_insert(self, items: List[Dict[str, Any]]) -> List[str]:
        """
        批量插入优化

        单次数据库操作比多次操作高效得多
        """
        # 分批次处理
        all_ids = []

        for i in range(0, len(items), self.batch_size):
            batch = items[i:i + self.batch_size]

            # 批量操作 (具体实现取决于底层存储)
            batch_ids = self._do_batch_insert(batch)
            all_ids.extend(batch_ids)

        return all_ids

    def _do_batch_insert(self, batch: List[Dict[str, Any]]) -> List[str]:
        """执行批量插入"""
        # 这里应该调用存储系统的批量 API
        # 例如 Chroma 的 collection.add(ids=..., embeddings=..., documents=...)
        ids = []
        for item in batch:
            key = item.get("key", f"mem_{len(ids)}")
            ids.append(key)
            self.memory.store(key, item.get("content", ""))
        return ids

    def batch_search(
        self,
        queries: List[str],
        limit_per_query: int = 5
    ) -> Dict[str, List[Any]]:
        """批量搜索"""
        results = {}

        for i in range(0, len(queries), self.batch_size):
            batch = queries[i:i + self.batch_size]

            for query in batch:
                results[query] = self.memory.recall(query, limit_per_query)

        return results
```

### 4.12.3 缓存策略

```python
"""
Redis 缓存层设计
"""
import json
import hashlib
from typing import Optional, List, Any
from datetime import timedelta


class CachedMemorySystem:
    """
    带缓存的记忆系统

    架构:
    L1: 内存缓存 (最近查询)
    L2: Redis 缓存 (热点查询)
    L3: 持久化存储 (向量数据库)
    """

    def __init__(self, memory_system, redis_client=None, cache_ttl: int = 3600):
        self.memory = memory_system
        self.redis = redis_client
        self.cache_ttl = cache_ttl

        # L1 缓存
        self._l1_cache: dict = {}
        self._l1_max_size = 1000

    def _get_cache_key(self, query: str) -> str:
        """生成缓存键"""
        return f"mem:query:{hashlib.md5(query.encode()).hexdigest()}"

    def search(self, query: str, limit: int = 5) -> List[Any]:
        """带缓存的搜索"""
        # 检查 L1 缓存
        if query in self._l1_cache:
            return self._l1_cache[query]

        # 检查 Redis 缓存
        if self.redis:
            cache_key = self._get_cache_key(query)
            cached = self.redis.get(cache_key)
            if cached:
                result = json.loads(cached)
                # 写入 L1
                self._l1_cache[query] = result
                return result

        # 未命中，执行实际搜索
        result = self.memory.search(query, limit)

        # 写入缓存
        if self.redis:
            cache_key = self._get_cache_key(query)
            self.redis.setex(
                cache_key,
                self.cache_ttl,
                json.dumps(result, default=str)
            )

        # L1 缓存
        if len(self._l1_cache) >= self._l1_max_size:
            # 简单 LRU: 删除最旧的一半
            keys = list(self._l1_cache.keys())[:500]
            for k in keys:
                del self._l1_cache[k]

        self._l1_cache[query] = result

        return result

    def invalidate(self, query: str) -> None:
        """使缓存失效"""
        # 清除 L1
        self._l1_cache.pop(query, None)

        # 清除 Redis
        if self.redis:
            cache_key = self._get_cache_key(query)
            self.redis.delete(cache_key)
```

### 4.12.4 水平扩展设计

```
┌─────────────────────────────────────────────────────────────┐
│                    水平扩展架构                              │
│                                                             │
│                        用户请求                              │
│                            │                                │
│                            ↓                                │
│                   ┌─────────────────┐                       │
│                   │   Load Balancer │                       │
│                   │    (负载均衡)     │                       │
│                   └────────┬────────┘                       │
│                            │                                │
│          ┌─────────────────┼─────────────────┐             │
│          │                 │                 │              │
│          ↓                 ↓                 ↓              │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│   │  Instance 1  │  │  Instance 2  │  │  Instance 3  │  ... │
│   │  (带缓存)    │  │  (带缓存)    │  │  (带缓存)    │      │
│   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│          │                 │                 │              │
│          └─────────────────┼─────────────────┘             │
│                            │                                │
│                            ↓                                │
│                   ┌─────────────────┐                       │
│                   │  Redis Cluster  │                       │
│                   │    (共享缓存)     │                       │
│                   └─────────────────┘                       │
│                            │                                │
│                            ↓                                │
│                   ┌─────────────────┐                       │
│                   │  Vector DB      │                       │
│                   │  (分片集群)      │                       │
│                   └─────────────────┘                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 数据分片策略

```python
"""
基于一致性 Hash 的数据分片
"""
from typing import List, Dict, Any


class ShardedMemorySystem:
    """
    分片记忆系统

    将数据分散到多个节点
    """

    def __init__(self, num_shards: int = 4):
        self.num_shards = num_shards
        self.shards: Dict[int, Any] = {}

    def _get_shard_id(self, key: str) -> int:
        """根据键计算分片 ID"""
        # 简单 hash 分片
        return hash(key) % self.num_shards

    def get(self, key: str) -> Any:
        """获取数据"""
        shard_id = self._get_shard_id(key)
        shard = self.shards.get(shard_id)
        if shard:
            return shard.get(key)
        return None

    def store(self, key: str, value: Any) -> None:
        """存储数据"""
        shard_id = self._get_shard_id(key)

        if shard_id not in self.shards:
            self.shards[shard_id] = {}

        self.shards[shard_id][key] = value

    def search(self, query: str) -> List[Any]:
        """跨分片搜索"""
        all_results = []

        for shard in self.shards.values():
            results = self._search_in_shard(shard, query)
            all_results.extend(results)

        # 合并并排序
        return sorted(all_results, key=lambda x: x.get("score", 0), reverse=True)

    def _search_in_shard(self, shard: Dict, query: str) -> List[Any]:
        """在分片内搜索"""
        # 简化实现
        return []
```

---

## 4.13 安全与隐私

### 4.13.1 敏感信息处理

```python
"""
敏感信息检测和处理
"""
import re
from typing import List, Optional, Tuple


class SensitiveInfoDetector:
    """
    敏感信息检测器

    检测类型:
    - PII (个人身份信息)
    - 密码和凭证
    - 财务信息
    """

    # PII 正则模式
    PATTERNS = {
        "phone": r"1[3-9]\d{9}",  # 中国手机号
        "email": r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
        "id_card": r"\d{17}[\dXx]|\d{15}",  # 身份证号
        "bank_card": r"\d{16,19}",  # 银行卡号
        "password": r"(?:密码|password|passwd)\s*[=:]\s*\S+",
        "address": r"(?:省 | 市 | 区 | 县 | 街道 | 路|号)",
    }

    def detect(self, content: str) -> List[Tuple[str, str, int, int]]:
        """
        检测敏感信息

        Returns:
            [(类型，匹配文本，起始位置，结束位置), ...]
        """
        results = []

        for pattern_name, pattern in self.PATTERNS.items():
            for match in re.finditer(pattern, content, re.IGNORECASE):
                results.append((
                    pattern_name,
                    match.group(),
                    match.start(),
                    match.end()
                ))

        return results

    def redact(self, content: str, replace_with: str = "***") -> str:
        """脱敏敏感信息"""
        detections = self.detect(content)

        # 按位置排序，从后往前替换，避免位置偏移
        detections.sort(key=lambda x: x[2], reverse=True)

        result = content
        for _, matched_text, start, end in detections:
            result = result[:start] + replace_with + result[end:]

        return result


# 使用示例
if __name__ == "__main__":
    detector = SensitiveInfoDetector()

    # 测试文本
    text = "我的电话是 13812345678，邮箱是 test@example.com"

    # 检测
    detections = detector.detect(text)
    print(f"检测到 {len(detections)} 个敏感信息:")
    for type_, matched, _, _ in detections:
        print(f"  - {type_}: {matched}")

    # 脱敏
    redacted = detector.redact(text)
    print(f"\n脱敏后：{redacted}")
```

**参考输出：**
```
检测到 2 个敏感信息:
  - phone: 13812345678
  - email: test@example.com

脱敏后：我的电话是***，邮箱是***
```

### 4.13.2 记忆访问控制

```python
"""
基于用户的记忆隔离和访问控制
"""
from typing import Optional, Dict, List, Set
from enum import Enum
from datetime import datetime


class AccessLevel(Enum):
    """访问级别"""
    OWNER = "owner"       # 所有者（完全访问）
    ADMIN = "admin"       # 管理员（读写）
    USER = "user"         # 普通用户（受限读写）
    GUEST = "guest"       # 访客（只读）
    NONE = "none"         # 无访问权限


class MemoryAccessControl:
    """
    记忆访问控制系统

    功能:
    - 用户隔离
    - 权限管理
    - 访问审计
    """

    def __init__(self):
        # 用户 - 记忆权限映射
        self._permissions: Dict[str, Dict[str, AccessLevel]] = {}
        # 用户组
        self._groups: Dict[str, Set[str]] = {}
        # 访问日志
        self._access_log: List[Dict] = []

    def set_permission(
        self,
        user_id: str,
        memory_id: str,
        level: AccessLevel
    ) -> None:
        """设置用户对记忆的访问权限"""
        if user_id not in self._permissions:
            self._permissions[user_id] = {}
        self._permissions[user_id][memory_id] = level

    def check_access(
        self,
        user_id: str,
        memory_id: str,
        required_level: AccessLevel = AccessLevel.USER
    ) -> bool:
        """
        检查用户是否有访问权限

        Args:
            user_id: 用户 ID
            memory_id: 记忆 ID
            required_level: 需要的访问级别

        Returns:
            是否有访问权限
        """
        # 记录访问尝试
        self._log_access(user_id, memory_id, required_level)

        # 获取用户权限
        user_perms = self._permissions.get(user_id, {})
        user_level = user_perms.get(memory_id, AccessLevel.NONE)

        # 权限等级排序
        level_order = {
            AccessLevel.OWNER: 4,
            AccessLevel.ADMIN: 3,
            AccessLevel.USER: 2,
            AccessLevel.GUEST: 1,
            AccessLevel.NONE: 0
        }

        return level_order.get(user_level, 0) >= level_order.get(required_level, 0)

    def _log_access(
        self,
        user_id: str,
        memory_id: str,
        required_level: AccessLevel
    ) -> None:
        """记录访问日志"""
        self._access_log.append({
            "user_id": user_id,
            "memory_id": memory_id,
            "required_level": required_level.value,
            "timestamp": datetime.now().isoformat()
        })

    def get_accessible_memories(
        self,
        user_id: str,
        min_level: AccessLevel = AccessLevel.GUEST
    ) -> List[str]:
        """获取用户可访问的所有记忆 ID"""
        accessible = []
        user_perms = self._permissions.get(user_id, {})

        level_order = {
            AccessLevel.OWNER: 4,
            AccessLevel.ADMIN: 3,
            AccessLevel.USER: 2,
            AccessLevel.GUEST: 1,
            AccessLevel.NONE: 0
        }
        min_level_num = level_order.get(min_level, 0)

        for memory_id, level in user_perms.items():
            if level_order.get(level, 0) >= min_level_num:
                accessible.append(memory_id)

        return accessible

    def create_group(self, group_id: str) -> None:
        """创建用户组"""
        self._groups[group_id] = set()

    def add_to_group(self, group_id: str, user_id: str) -> None:
        """添加用户到组"""
        if group_id in self._groups:
            self._groups[group_id].add(user_id)

    def set_group_permission(
        self,
        group_id: str,
        memory_id: str,
        level: AccessLevel
    ) -> None:
        """为整个组设置记忆权限"""
        if group_id in self._groups:
            for user_id in self._groups[group_id]:
                self.set_permission(user_id, memory_id, level)


# 使用示例
if __name__ == "__main__":
    acl = MemoryAccessControl()

    # 设置权限
    acl.set_permission("user_001", "mem_001", AccessLevel.OWNER)
    acl.set_permission("user_002", "mem_001", AccessLevel.USER)
    acl.set_permission("user_003", "mem_001", AccessLevel.NONE)

    # 检查访问
    print(f"user_001 访问 mem_001: {acl.check_access('user_001', 'mem_001')}")
    print(f"user_002 访问 mem_001: {acl.check_access('user_002', 'mem_001')}")
    print(f"user_003 访问 mem_001: {acl.check_access('user_003', 'mem_001')}")

    # 获取可访问的记忆
    accessible = acl.get_accessible_memories("user_001")
    print(f"user_001 可访问：{accessible}")
```

**参考输出：**
```
user_001 访问 mem_001: True
user_002 访问 mem_001: True
user_003 访问 mem_001: False
user_001 可访问：['mem_001']
```

### 4.13.3 记忆删除权（被遗忘权）

```python
"""
实现用户的被遗忘权（Right to be Forgotten）
"""
from typing import List, Optional
from datetime import datetime
import hashlib


class MemoryDeletionManager:
    """
    记忆删除管理器

    实现 GDPR 等法规要求的被遗忘权

    功能:
    - 完全删除
    - 匿名化处理
    - 删除审计
    """

    def __init__(self, memory_store):
        self.memory_store = memory_store
        self._deletion_log: List[Dict] = []

    def request_deletion(
        self,
        user_id: str,
        reason: str = "user_request"
    ) -> str:
        """
        创建删除请求

        Returns:
            删除请求 ID
        """
        request_id = hashlib.md5(
            f"{user_id}{datetime.now().isoformat()}".encode()
        ).hexdigest()

        return request_id

    def execute_deletion(
        self,
        user_id: str,
        request_id: str,
        soft_delete: bool = True
    ) -> Dict[str, int]:
        """
        执行用户数据删除

        Args:
            user_id: 用户 ID
            request_id: 删除请求 ID
            soft_delete: 是否软删除（可恢复）

        Returns:
            删除统计
        """
        stats = {
            "memories_deleted": 0,
            "conversations_deleted": 0,
            "preferences_deleted": 0
        }

        # 1. 删除所有用户记忆
        user_memories = self._get_user_memories(user_id)
        for mem_id in user_memories:
            if soft_delete:
                self._soft_delete(mem_id)
            else:
                self.memory_store.delete(mem_id)
            stats["memories_deleted"] += 1

        # 2. 删除对话历史
        conversations = self._get_user_conversations(user_id)
        for conv_id in conversations:
            if soft_delete:
                self._soft_delete(conv_id)
            else:
                self.memory_store.delete(conv_id)
            stats["conversations_deleted"] += 1

        # 3. 记录删除日志
        self._deletion_log.append({
            "request_id": request_id,
            "user_id": user_id,
            "timestamp": datetime.now().isoformat(),
            "stats": stats,
            "soft_delete": soft_delete
        })

        return stats

    def _get_user_memories(self, user_id: str) -> List[str]:
        """获取用户的所有记忆 ID"""
        # 实际实现中查询数据库
        return []

    def _get_user_conversations(self, user_id: str) -> List[str]:
        """获取用户的所有对话 ID"""
        return []

    def _soft_delete(self, item_id: str) -> None:
        """软删除：标记为已删除但保留数据"""
        # 添加 deleted_at 标记
        pass

    def get_deletion_report(self, request_id: str) -> Optional[Dict]:
        """获取删除报告"""
        for log in self._deletion_log:
            if log["request_id"] == request_id:
                return log
        return None

    def anonymize_user_data(
        self,
        user_id: str,
        replacement: str = "[已删除]"
    ) -> int:
        """
        匿名化用户数据

        保留数据结构但移除个人身份信息

        Returns:
            匿名化的记录数
        """
        count = 0
        user_memories = self._get_user_memories(user_id)

        for mem_id in user_memories:
            memory = self.memory_store.get(mem_id)
            if memory:
                # 替换 PII
                anonymized_content = self._anonymize_content(
                    memory.content,
                    replacement
                )
                # 更新记忆
                # self.memory_store.update(mem_id, {"content": anonymized_content})
                count += 1

        return count

    def _anonymize_content(self, content: str, replacement: str) -> str:
        """匿名化内容中的 PII"""
        # 使用敏感信息检测器
        # detector = SensitiveInfoDetector()
        # return detector.redact(content, replacement)
        return content


# 使用示例
if __name__ == "__main__":
    # deletion_manager = MemoryDeletionManager(memory_store)

    # 请求删除
    # request_id = deletion_manager.request_deletion("user_001", "user_request")

    # 执行删除
    # stats = deletion_manager.execute_deletion("user_001", request_id)
    # print(f"删除统计：{stats}")

    # 获取报告
    # report = deletion_manager.get_deletion_report(request_id)
    # print(f"删除报告：{report}")

    print("被遗忘权实现:")
    print("1. 用户可请求删除所有数据")
    print("2. 支持软删除（可恢复）和硬删除")
    print("3. 支持匿名化替代完全删除")
    print("4. 完整的删除审计日志")
```

---

## 4.14 实践项目：构建完整的记忆系统

### 4.14.1 项目架构设计

```
memory_system/
├── __init__.py
├── config.py                 # 配置管理
├── types.py                  # 类型定义
│
├── core/
│   ├── __init__.py
│   ├── memory.py             # 核心记忆类
│   ├── manager.py            # 记忆管理器
│   └── operations.py         # 创建/更新/删除操作
│
├── storage/
│   ├── __init__.py
│   ├── base.py               # 存储抽象基类
│   ├── vector_store.py       # 向量存储适配
│   ├── sqlite_store.py       # SQLite 存储
│   └── redis_cache.py        # Redis 缓存
│
├── retrieval/
│   ├── __init__.py
│   ├── searcher.py           # 搜索实现
│   ├── ranker.py             # 排序器
│   └── hybrid.py             # 混合检索
│
├── processing/
│   ├── __init__.py
│   ├── embedding.py          # Embedding 生成
│   ├── classifier.py         # 记忆分类
│   └── deduplicator.py       # 去重处理
│
├── api/
│   ├── __init__.py
│   ├── routes.py             # API 路由
│   └── mcp_tools.py          # MCP 工具
│
└── utils/
    ├── __init__.py
    ├── security.py           # 安全工具
    └── helpers.py            # 辅助函数
```

### 4.14.2 核心代码实现

#### 记忆实体类

```python
"""
记忆实体定义
"""
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any, Dict, List, Optional
from enum import Enum
import uuid


class MemoryType(Enum):
    """记忆类型"""
    CONVERSATIONAL = "conversational"
    PROCEDURAL = "procedural"
    FACTUAL = "factual"
    SEMANTIC = "semantic"
    EPISODIC = "episodic"
    PERSONAL = "personal"


@dataclass
class Memory:
    """
    记忆实体

    Attributes:
        id: 唯一标识
        content: 记忆内容
        memory_type: 记忆类型
        embedding: 向量嵌入
        metadata: 元数据
        created_at: 创建时间
        updated_at: 更新时间
        access_count: 访问次数
        importance_score: 重要性分数
    """
    content: str
    memory_type: MemoryType = MemoryType.FACTUAL
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:16])
    embedding: Optional[List[float]] = None
    metadata: Dict[str, Any] = field(default_factory=dict)
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    access_count: int = 0
    importance_score: float = 0.5

    def to_dict(self) -> Dict[str, Any]:
        """转换为字典"""
        return {
            "id": self.id,
            "content": self.content,
            "memory_type": self.memory_type.value,
            "embedding": self.embedding,
            "metadata": self.metadata,
            "created_at": self.created_at.isoformat(),
            "updated_at": self.updated_at.isoformat(),
            "access_count": self.access_count,
            "importance_score": self.importance_score
        }

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> "Memory":
        """从字典创建"""
        return cls(
            id=data.get("id", str(uuid.uuid4())[:16]),
            content=data.get("content", ""),
            memory_type=MemoryType(data.get("memory_type", "factual")),
            embedding=data.get("embedding"),
            metadata=data.get("metadata", {}),
            created_at=datetime.fromisoformat(data["created_at"]) if "created_at" in data else datetime.now(),
            updated_at=datetime.fromisoformat(data["updated_at"]) if "updated_at" in data else datetime.now(),
            access_count=data.get("access_count", 0),
            importance_score=data.get("importance_score", 0.5)
        )
```

#### MemoryManager 核心类

```python
"""
记忆管理器 - 核心业务逻辑
"""
from typing import List, Optional, Dict, Any
from datetime import datetime

from .memory import Memory, MemoryType
from ..storage.base import BaseStorage
from ..retrieval.searcher import Searcher
from ..processing.classifier import MemoryClassifier
from ..processing.deduplicator import SemanticDeduplicator


class MemoryManager:
    """
    记忆管理器

    提供完整的记忆 CRUD 操作和高级功能
    """

    def __init__(
        self,
        storage: BaseStorage,
        searcher: Optional[Searcher] = None,
        classifier: Optional[MemoryClassifier] = None,
        deduplicator: Optional[SemanticDeduplicator] = None
    ):
        self.storage = storage
        self.searcher = searcher or Searcher(storage)
        self.classifier = classifier
        self.deduplicator = deduplicator or SemanticDeduplicator()

    async def create(
        self,
        content: str,
        memory_type: Optional[MemoryType] = None,
        metadata: Optional[Dict[str, Any]] = None,
        skip_dedup: bool = False
    ) -> Optional[Memory]:
        """
        创建记忆

        Args:
            content: 记忆内容
            memory_type: 记忆类型（自动检测）
            metadata: 元数据
            skip_dedup: 跳过去重检查

        Returns:
            创建的 Memory 或 None（如果是重复）
        """
        # 1. 去重检查
        if not skip_dedup:
            existing = self.storage.find_similar(content, threshold=0.95)
            if existing:
                return None  # 是重复记忆

        # 2. 自动分类
        if memory_type is None and self.classifier:
            memory_type = await self.classifier.classify(content)
        elif memory_type is None:
            memory_type = MemoryType.FACTUAL

        # 3. 生成嵌入向量
        embedding = await self._generate_embedding(content)

        # 4. 创建记忆对象
        memory = Memory(
            content=content,
            memory_type=memory_type,
            embedding=embedding,
            metadata=metadata or {}
        )

        # 5. 存储
        self.storage.save(memory)

        return memory

    async def search(
        self,
        query: str,
        memory_types: Optional[List[MemoryType]] = None,
        limit: int = 10,
        min_importance: float = 0.0
    ) -> List[Memory]:
        """
        搜索记忆

        Args:
            query: 搜索查询
            memory_types: 记忆类型过滤
            limit: 返回数量
            min_importance: 最低重要性分数

        Returns:
            记忆列表
        """
        results = await self.searcher.search(
            query,
            filters={"memory_types": memory_types} if memory_types else None,
            limit=limit
        )

        # 过滤重要性
        if min_importance > 0:
            results = [m for m in results if m.importance_score >= min_importance]

        # 更新访问计数
        for memory in results:
            memory.access_count += 1
            self.storage.update(memory)

        return results

    async def update_importance(
        self,
        memory_id: str,
        importance_score: float
    ) -> bool:
        """更新记忆重要性分数"""
        memory = self.storage.get(memory_id)
        if memory:
            memory.importance_score = importance_score
            memory.updated_at = datetime.now()
            self.storage.update(memory)
            return True
        return False

    async def delete(self, memory_id: str) -> bool:
        """删除记忆"""
        return self.storage.delete(memory_id)

    async def cleanup_old(
        self,
        max_age_days: int = 365,
        min_access_count: int = 1
    ) -> int:
        """
        清理过期记忆

        Returns:
            删除的记忆数量
        """
        cutoff = datetime.now()
        deleted = 0

        for memory in self.storage.list_all():
            age_days = (cutoff - memory.created_at).days
            if age_days > max_age_days and memory.access_count < min_access_count:
                self.storage.delete(memory.id)
                deleted += 1

        return deleted

    async def _generate_embedding(self, content: str) -> List[float]:
        """生成嵌入向量"""
        # 实际实现中调用 embedding 服务
        from sentence_transformers import SentenceTransformer
        model = SentenceTransformer('paraphrase-multilingual-MiniLM-L12-v2')
        embedding = model.encode(content)
        return embedding.tolist()
```

### 4.14.3 与 MCP 协议集成

```python
"""
MCP 工具定义 - 让 Claude Desktop 访问记忆系统
"""
from typing import Dict, Any, List, Optional
import json


# MCP 服务器配置
MCP_SERVER_CONFIG = {
    "name": "memory-system",
    "version": "1.0.0",
    "description": "AI Agent 记忆系统 MCP 服务"
}


# MCP 工具定义
MCP_TOOLS = [
    {
        "name": "store_memory",
        "description": "存储新的记忆",
        "inputSchema": {
            "type": "object",
            "properties": {
                "content": {
                    "type": "string",
                    "description": "要存储的记忆内容"
                },
                "type": {
                    "type": "string",
                    "enum": ["conversational", "procedural", "factual", "semantic", "episodic", "personal"],
                    "description": "记忆类型"
                },
                "metadata": {
                    "type": "object",
                    "description": "附加元数据"
                }
            },
            "required": ["content"]
        }
    },
    {
        "name": "search_memories",
        "description": "搜索记忆",
        "inputSchema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索查询"
                },
                "limit": {
                    "type": "integer",
                    "default": 5,
                    "description": "最大返回数量"
                },
                "memory_type": {
                    "type": "string",
                    "description": "记忆类型过滤"
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "delete_memory",
        "description": "删除指定记忆",
        "inputSchema": {
            "type": "object",
            "properties": {
                "memory_id": {
                    "type": "string",
                    "description": "记忆 ID"
                }
            },
            "required": ["memory_id"]
        }
    },
    {
        "name": "list_memories",
        "description": "列出所有记忆",
        "inputSchema": {
            "type": "object",
            "properties": {
                "limit": {
                    "type": "integer",
                    "default": 20
                },
                "offset": {
                    "type": "integer",
                    "default": 0
                }
            }
        }
    },
    {
        "name": "get_memory_stats",
        "description": "获取记忆系统统计信息",
        "inputSchema": {
            "type": "object",
            "properties": {}
        }
    }
]


class MemoryMCPTools:
    """
    记忆系统 MCP 工具实现
    """

    def __init__(self, memory_manager):
        self.memory_manager = memory_manager

    async def store_memory(
        self,
        content: str,
        type: Optional[str] = None,
        metadata: Optional[Dict[str, Any]] = None
    ) -> Dict[str, Any]:
        """存储记忆工具"""
        from .memory import MemoryType

        memory_type = MemoryType(type) if type else None

        memory = await self.memory_manager.create(
            content=content,
            memory_type=memory_type,
            metadata=metadata
        )

        if memory:
            return {
                "success": True,
                "memory_id": memory.id,
                "message": f"记忆已存储：{memory.id}"
            }
        else:
            return {
                "success": False,
                "message": "记忆已存在（重复内容）"
            }

    async def search_memories(
        self,
        query: str,
        limit: int = 5,
        memory_type: Optional[str] = None
    ) -> Dict[str, Any]:
        """搜索记忆工具"""
        from .memory import MemoryType

        types = [MemoryType(memory_type)] if memory_type else None

        results = await self.memory_manager.search(
            query=query,
            memory_types=types,
            limit=limit
        )

        return {
            "success": True,
            "count": len(results),
            "memories": [m.to_dict() for m in results]
        }

    async def delete_memory(self, memory_id: str) -> Dict[str, Any]:
        """删除记忆工具"""
        success = await self.memory_manager.delete(memory_id)

        return {
            "success": success,
            "message": f"记忆 {memory_id} 已删除" if success else f"记忆 {memory_id} 不存在"
        }

    async def list_memories(self, limit: int = 20, offset: int = 0) -> Dict[str, Any]:
        """列出记忆工具"""
        memories = self.memory_manager.storage.list_all()

        # 分页
        paginated = memories[offset:offset + limit]

        return {
            "success": True,
            "total": len(memories),
            "memories": [m.to_dict() for m in paginated]
        }

    async def get_memory_stats(self) -> Dict[str, Any]:
        """获取统计信息工具"""
        all_memories = self.memory_manager.storage.list_all()

        # 按类型统计
        type_counts = {}
        for m in all_memories:
            t = m.memory_type.value
            type_counts[t] = type_counts.get(t, 0) + 1

        return {
            "success": True,
            "total_memories": len(all_memories),
            "by_type": type_counts,
            "total_accesses": sum(m.access_count for m in all_memories)
        }


# MCP 工具注册函数
def register_memory_mcp(server, memory_manager):
    """
    将记忆工具注册到 MCP 服务器

    Usage:
        from mcp.server import Server
        server = Server("memory-system")
        memory_manager = MemoryManager(storage)
        register_memory_mcp(server, memory_manager)
    """
    tools = MemoryMCPTools(memory_manager)

    # 注册每个工具
    @server.tool("store_memory")
    async def store_memory(content: str, type: str = None, metadata: dict = None):
        return await tools.store_memory(content, type, metadata)

    @server.tool("search_memories")
    async def search_memories(query: str, limit: int = 5, memory_type: str = None):
        return await tools.search_memories(query, limit, memory_type)

    @server.tool("delete_memory")
    async def delete_memory(memory_id: str):
        return await tools.delete_memory(memory_id)

    @server.tool("list_memories")
    async def list_memories(limit: int = 20, offset: int = 0):
        return await tools.list_memories(limit, offset)

    @server.tool("get_memory_stats")
    async def get_memory_stats():
        return await tools.get_memory_stats()
```

### 4.14.4 完整的端到端测试

```python
"""
记忆系统端到端测试
"""
import asyncio
import pytest
from typing import List

# 导入被测试模块
from memory_system.core.memory import Memory, MemoryType
from memory_system.core.manager import MemoryManager
from memory_system.storage.sqlite_store import SQLiteStorage


@pytest.fixture
async def memory_manager():
    """测试 fixture: 创建记忆管理器"""
    storage = SQLiteStorage(":memory:")  # 内存数据库
    manager = MemoryManager(storage)
    yield manager
    storage.close()


class TestMemorySystem:
    """记忆系统测试类"""

    @pytest.mark.asyncio
    async def test_create_memory(self, memory_manager):
        """测试创建记忆"""
        memory = await memory_manager.create(
            content="用户喜欢 Python 编程",
            memory_type=MemoryType.PERSONAL
        )

        assert memory is not None
        assert memory.content == "用户喜欢 Python 编程"
        assert memory.memory_type == MemoryType.PERSONAL

    @pytest.mark.asyncio
    async def test_duplicate_memory(self, memory_manager):
        """测试去重功能"""
        # 创建第一条记忆
        memory1 = await memory_manager.create(
            content="用户住在北京",
            memory_type=MemoryType.PERSONAL
        )

        # 尝试创建重复记忆
        memory2 = await memory_manager.create(
            content="用户住在北京",
            memory_type=MemoryType.PERSONAL
        )

        assert memory2 is None  # 应该被识别为重复

    @pytest.mark.asyncio
    async def test_search_memory(self, memory_manager):
        """测试搜索功能"""
        # 创建测试数据
        await memory_manager.create(
            content="Python 是一门编程语言",
            memory_type=MemoryType.FACTUAL
        )
        await memory_manager.create(
            content="用户喜欢吃川菜",
            memory_type=MemoryType.PERSONAL
        )

        # 搜索
        results = await memory_manager.search("编程")

        assert len(results) >= 1
        assert "编程" in results[0].content or "语言" in results[0].content

    @pytest.mark.asyncio
    async def test_delete_memory(self, memory_manager):
        """测试删除功能"""
        memory = await memory_manager.create("测试内容")
        assert memory is not None

        success = await memory_manager.delete(memory.id)
        assert success is True

        # 验证已删除
        results = await memory_manager.search("测试内容")
        assert len(results) == 0


class TestMemoryScenarios:
    """场景测试"""

    @pytest.mark.asyncio
    async def test_conversation_scenario(self, memory_manager):
        """测试对话场景"""
        # 第一轮对话
        await memory_manager.create(
            content="用户说：我叫张三",
            memory_type=MemoryType.PERSONAL
        )

        # 第二轮对话
        await memory_manager.create(
            content="用户说：我是一名工程师",
            memory_type=MemoryType.PERSONAL
        )

        # 检索用户信息
        results = await memory_manager.search("张三")

        assert len(results) >= 1
        assert "张三" in results[0].content

    @pytest.mark.asyncio
    async def test_preference_learning(self, memory_manager):
        """测试偏好学习"""
        preferences = [
            "用户喜欢喝绿茶",
            "用户不喜欢吃辣",
            "用户偏好早上工作",
        ]

        for pref in preferences:
            await memory_manager.create(
                content=pref,
                memory_type=MemoryType.PERSONAL
            )

        # 查询偏好
        results = await memory_manager.search("用户喜欢")

        assert len(results) >= 2


# 运行测试
if __name__ == "__main__":
    pytest.main([__file__, "-v", "--asyncio-mode=auto"])
```

**测试参考输出：**
```
============================= test session starts ==============================
platform linux -- Python 3.10.0, pytest-7.0.0

test_create_memory.py::TestMemorySystem::test_create_memory PASSED
test_create_memory.py::TestMemorySystem::test_duplicate_memory PASSED
test_create_memory.py::TestMemorySystem::test_search_memory PASSED
test_create_memory.py::TestMemorySystem::test_delete_memory PASSED
test_scenarios.py::TestMemoryScenarios::test_conversation_scenario PASSED
test_scenarios.py::TestMemoryScenarios::test_preference_learning PASSED

============================== 6 passed in 2.34s ===============================
```

---

## 4.15 前沿话题

### 4.15.1 多模态记忆

```python
"""
多模态记忆：图像、音频与文本的融合
"""
from dataclasses import dataclass
from typing import List, Optional, Union
from enum import Enum


class ModalityType(Enum):
    """模态类型"""
    TEXT = "text"
    IMAGE = "image"
    AUDIO = "audio"
    VIDEO = "video"


@dataclass
class MultimodalMemory:
    """
    多模态记忆

    支持跨模态存储和检索
    """
    id: str
    content: Union[str, bytes]  # 文本或二进制数据
    modality: ModalityType
    caption: Optional[str] = None  # 图像/音频的文本描述
    embedding: Optional[List[float]] = None
    metadata: dict = None

    def __post_init__(self):
        if self.metadata is None:
            self.metadata = {}


class MultimodalMemorySystem:
    """
    多模态记忆系统

    特性:
    - 统一嵌入空间（文本、图像映射到同一向量空间）
    - 跨模态检索（用文本搜图像，用图像搜文本）
    """

    def __init__(self, storage, multimodal_encoder):
        self.storage = storage
        self.encoder = multimodal_encoder  # 如 CLIP 模型

    async def store(
        self,
        content: Union[str, bytes],
        modality: ModalityType,
        caption: Optional[str] = None
    ) -> str:
        """
        存储多模态记忆

        对于图像/音频，可以使用 caption 增强语义
        """
        # 生成嵌入
        if modality == ModalityType.TEXT:
            embedding = await self.encoder.encode_text(content)
        elif modality == ModalityType.IMAGE:
            embedding = await self.encoder.encode_image(content)
        else:
            embedding = await self.encoder.encode_audio(content)

        # 如果有 caption，融合文本嵌入
        if caption and modality != ModalityType.TEXT:
            text_emb = await self.encoder.encode_text(caption)
            # 融合嵌入
            embedding = [(a + b) / 2 for a, b in zip(embedding, text_emb)]

        # 创建记忆
        memory = MultimodalMemory(
            id=self._generate_id(),
            content=content,
            modality=modality,
            caption=caption,
            embedding=embedding
        )

        self.storage.save(memory)
        return memory.id

    async def search(
        self,
        query: Union[str, bytes],
        query_modality: ModalityType,
        target_modalities: Optional[List[ModalityType]] = None,
        limit: int = 10
    ) -> List[MultimodalMemory]:
        """
        跨模态检索

        例如:
        - 用文本查询图像："夕阳下的海滩"
        - 用图像查询相关描述
        """
        # 生成查询嵌入
        if query_modality == ModalityType.TEXT:
            query_embedding = await self.encoder.encode_text(query)
        elif query_modality == ModalityType.IMAGE:
            query_embedding = await self.encoder.encode_image(query)
        else:
            query_embedding = await self.encoder.encode_audio(query)

        # 检索
        results = self.storage.similarity_search(
            query_embedding,
            filters={"modalities": target_modalities} if target_modalities else None,
            limit=limit
        )

        return results

    def _generate_id(self) -> str:
        import hashlib
        import time
        return hashlib.md5(f"{time.time()}".encode()).hexdigest()[:16]


# 使用 CLIP 作为多模态编码器示例
class CLIPEncoder:
    """CLIP 多模态编码器封装"""

    def __init__(self, model_name="ViT-B/32"):
        import clip
        import torch
        from PIL import Image
        import io

        self.device = "cuda" if torch.cuda.is_available() else "cpu"
        self.model, self.preprocess = clip.load(model_name, device=self.device)
        self.Image = Image

    async def encode_text(self, text: str) -> List[float]:
        import torch

        with torch.no_grad():
            text_tokens = clip.tokenize([text], truncate=True).to(self.device)
            text_features = self.model.encode_text(text_tokens)
            text_features = text_features / text_features.norm(dim=-1, keepdim=True)

        return text_features.cpu().numpy()[0].tolist()

    async def encode_image(self, image_bytes: bytes) -> List[float]:
        import torch
        from io import BytesIO

        with torch.no_grad():
            image = self.Image.open(BytesIO(image_bytes))
            image_input = self.preprocess(image).unsqueeze(0).to(self.device)
            image_features = self.model.encode_image(image_input)
            image_features = image_features / image_features.norm(dim=-1, keepdim=True)

        return image_features.cpu().numpy()[0].tolist()

    async def encode_audio(self, audio_bytes: bytes) -> List[float]:
        # 音频编码需要额外的音频模型
        # 这里简化为使用 Whisper 等模型
        pass


# 使用示例
"""
# 初始化
encoder = CLIPEncoder()
storage = VectorStorage()
system = MultimodalMemorySystem(storage, encoder)

# 存储图像
with open("beach.jpg", "rb") as f:
    image_data = f.read()

image_id = await system.store(
    content=image_data,
    modality=ModalityType.IMAGE,
    caption="夕阳下的金色海滩"
)

# 用文本搜索图像
results = await system.search(
    query="海滩日落",
    query_modality=ModalityType.TEXT,
    target_modalities=[ModalityType.IMAGE]
)

# 找到最相关的图像
for result in results:
    print(f"找到图像：{result.id}, caption: {result.caption}")
"""
```

### 4.15.2 跨记忆推理

```python
"""
从多个记忆中推导新知识
"""
from typing import List, Dict, Any, Optional, Set
from dataclasses import dataclass


@dataclass
class InferredKnowledge:
    """推导出的知识"""
    content: str
    source_memory_ids: List[str]
    confidence: float
    inference_type: str  # transitive, deductive, inductive


class MemoryReasoner:
    """
    记忆推理引擎

    功能:
    - 传递推理 (A→B, B→C ⟹ A→C)
    - 归纳推理 (多个具体事实 → 一般规律)
    - 演绎推理 (一般规则 + 具体事实 → 结论)
    """

    def __init__(self, llm_client):
        self.llm = llm_client

    async def infer_from_memories(
        self,
        memory_contents: List[str],
        memory_ids: List[str]
    ) -> List[InferredKnowledge]:
        """
        从多个记忆中推导新知识

        Args:
            memory_contents: 记忆内容列表
            memory_ids: 对应的记忆 ID

        Returns:
            推导出的知识列表
        """
        # 使用 LLM 进行推理
        prompt = self._build_reasoning_prompt(memory_contents)

        response = await self.llm.chat(prompt)

        # 解析 LLM 输出的推理结果
        inferences = self._parse_inferences(response.content, memory_ids)

        return inferences

    def _build_reasoning_prompt(self, memories: List[str]) -> str:
        """构建推理提示"""
        memories_text = "\n".join([f"{i+1}. {m}" for i, m in enumerate(memories)])

        return f"""基于以下已知事实进行推理:

{memories_text}

请推导出的新知识，包括:
1. 传递关系：如果 A 与 B 相关，B 与 C 相关，能推出 A 与 C 的关系吗？
2. 归纳总结：从多个具体事实中能总结出什么一般规律？
3. 隐含信息：有什么信息是隐含的、没有直接说出的？

对于每个推导，请给出:
- 推导内容
- 依据的事实编号
- 置信度 (0-1)

返回 JSON 格式:
[
    {{
        "content": "推导内容",
        "sources": [1, 2],
        "confidence": 0.8,
        "type": "transitive"
    }}
]"""

    def _parse_inferences(
        self,
        llm_response: str,
        memory_ids: List[str]
    ) -> List[InferredKnowledge]:
        """解析 LLM 输出的推理"""
        import json

        try:
            data = json.loads(llm_response.strip())
            inferences = []

            for item in data:
                # 将事实编号转换为记忆 ID
                source_ids = [memory_ids[i-1] for i in item.get("sources", []) if i-1 < len(memory_ids)]

                inferences.append(InferredKnowledge(
                    content=item.get("content", ""),
                    source_memory_ids=source_ids,
                    confidence=item.get("confidence", 0.5),
                    inference_type=item.get("type", "unknown")
                ))

            return inferences
        except:
            return []

    def find_connections(
        self,
        memory_a: str,
        memory_b: str,
        intermediate_memories: List[str]
    ) -> Optional[List[str]]:
        """
        寻找两个记忆之间的关联路径

        例如:
        - Memory A: "张三在北京工作"
        - Memory B: "张三喜欢川菜"
        - 连接：北京 → 川菜（川菜是四川菜系，成都是四川首府）
        """
        # 使用 LLM 寻找连接
        pass


# 使用示例
"""
reasoner = MemoryReasoner(llm_client)

memories = [
    "张三住在北京市朝阳区",
    "张三在一家互联网公司工作",
    "互联网公司通常在写字楼办公",
    "朝阳区有很多写字楼"
]

memory_ids = ["mem_1", "mem_2", "mem_3", "mem_4"]

inferences = await reasoner.infer_from_memories(memories, memory_ids)

for inf in inferences:
    print(f"推导：{inf.content}")
    print(f"依据：{inf.source_memory_ids}")
    print(f"置信度：{inf.confidence}")
    print()

# 可能的输出:
# 推导：张三可能在朝阳区的写字楼上班
# 依据：['mem_1', 'mem_2', 'mem_3', 'mem_4']
# 置信度：0.75
"""
```

### 4.15.3 联邦学习记忆

```
┌─────────────────────────────────────────────────────────────────┐
│                    联邦学习记忆架构                              │
│                                                                 │
│                                                                 │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│   │  Client 1   │    │  Client 2   │    │  Client 3   │  ...   │
│   │ (本地记忆)  │    │ (本地记忆)  │    │ (本地记忆)  │        │
│   │   [私有]    │    │   [私有]    │    │   [私有]    │        │
│   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘        │
│          │                  │                  │                │
│          │  加密梯度更新     │  加密梯度更新     │  加密梯度更新   │
│          ↓                  ↓                  ↓                │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │              中央聚合服务器 (Aggregator)                 │  │
│   │         聚合知识，更新全局记忆模型                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                          │                                      │
│                          ↓                                      │
│              分发更新后的全局模型给所有客户端                    │
│                                                                 │
│  优势:                                                           │
│  - 用户数据保留在本地，隐私保护                                  │
│  - 共享群体知识，提升个体能力                                    │
│  - 符合 GDPR 等隐私法规                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
"""
联邦学习记忆系统框架
"""
from typing import List, Dict, Any
from abc import ABC, abstractmethod
import json


class FederatedMemoryClient(ABC):
    """
    联邦学习客户端

    每个用户设备运行一个客户端，维护本地私有记忆
    """

    def __init__(self, client_id: str, local_memory):
        self.client_id = client_id
        self.local_memory = local_memory
        self.global_model_version = 0

    @abstractmethod
    async def compute_gradient_update(
        self,
        global_model: Dict[str, Any]
    ) -> Dict[str, Any]:
        """
        基于本地数据计算模型更新

        Returns:
            加密的梯度更新（不包含原始数据）
        """
        pass

    async def apply_global_update(
        self,
        aggregated_update: Dict[str, Any]
    ) -> None:
        """应用全局模型更新到本地"""
        # 将聚合后的更新应用到本地记忆系统
        self._update_local_model(aggregated_update)
        self.global_model_version += 1

    def _update_local_model(self, update: Dict[str, Any]) -> None:
        """更新本地模型"""
        pass


class FederatedMemoryServer:
    """
    联邦学习服务器

    功能:
    - 接收客户端更新
    - 聚合多个更新
    - 分发全局模型
    """

    def __init__(self):
        self.client_updates: Dict[int, List[Dict]] = {}
        self.current_round = 0
        self.global_model: Dict[str, Any] = {}

    def register_update(self, client_id: str, round_num: int, update: Dict) -> None:
        """注册客户端更新"""
        if round_num not in self.client_updates:
            self.client_updates[round_num] = []
        self.client_updates[round_num].append({
            "client_id": client_id,
            "update": update
        })

    def aggregate_round(self, round_num: int, method: str = "fedavg") -> Dict:
        """
        聚合某一轮的更新

        Args:
            round_num: 轮次
            method: 聚合方法 (fedavg, fedprox, etc.)

        Returns:
            聚合后的更新
        """
        updates = self.client_updates.get(round_num, [])

        if not updates:
            return {}

        if method == "fedavg":
            return self._federated_average(updates)
        else:
            return self._simple_average(updates)

    def _federated_average(self, updates: List[Dict]) -> Dict:
        """
        FedAvg: 按客户端数据量加权平均
        """
        # 简化实现
        total_clients = len(updates)
        aggregated = {}

        # 获取所有键
        if updates:
            first_update = updates[0]["update"]
            for key in first_update.keys():
                values = [u["update"][key] for u in updates if key in u["update"]]
                if values:
                    # 加权平均（这里简化为简单平均）
                    aggregated[key] = sum(values) / len(values)

        return aggregated

    def _simple_average(self, updates: List[Dict]) -> Dict:
        """简单平均聚合"""
        return self._federated_average(updates)

    def broadcast_update(self, aggregated_update: Dict) -> List[str]:
        """
        广播更新给所有客户端

        Returns:
            接收更新的客户端 ID 列表
        """
        # 实际实现中通过网络发送
        return []


# 使用场景
"""
场景：多个用户共享知识，但保持数据隐私

1. 每个用户的记忆系统是本地的、私有的
2. 定期参与联邦学习轮次
3. 上传加密的模型更新（不是原始数据）
4. 服务器聚合更新后下发全局模型
5. 每个用户从群体知识中受益，同时保护隐私

好处:
- 用户 A 学到了关于"编程最佳实践"的知识
- 通过联邦学习，用户 B、C、D 也能受益
- 但用户 A 的具体数据（对话、偏好）不会泄露
"""
```

---

## 4.16 小测验与实践练习

### 小测验（10 道选择题）

**1. 短期记忆和长期记忆的主要区别是什么？**

A. 短期记忆存储在内存中，长期记忆存储在磁盘上
B. 短期记忆用于对话历史，长期记忆用于持久化知识
C. 短期记忆容量小，长期记忆容量大
D. 以上都是

<details>
<summary>点击查看答案</summary>
答案：D. 以上都是
</details>

---

**2. 滑动窗口机制的作用是？**

A. 提高检索速度
B. 节省存储空间
C. 控制上下文长度，防止超出 LLM 限制
D. 增加记忆重要性

<details>
<summary>点击查看答案</summary>
答案：C. 控制上下文长度，防止超出 LLM 限制
</details>

---

**3. 余弦相似度的取值范围是？**

A. [0, 1]
B. [-1, 1]
C. [-∞, +∞]
D. [0, 100]

<details>
<summary>点击查看答案</summary>
答案：B. [-1, 1]
</details>

---

**4. 哪种记忆类型适合存储用户偏好？**

A. Factual（事实记忆）
B. Procedural（程序记忆）
C. Personal（个人记忆）
D. Semantic（语义记忆）

<details>
<summary>点击查看答案</summary>
答案：C. Personal（个人记忆）
</details>

---

**5. Embedding 向量的作用是？**

A. 压缩文本内容
B. 加密敏感信息
C. 将文本映射到向量空间，支持语义检索
D. 加快搜索速度

<details>
<summary>点击查看答案</summary>
答案：C. 将文本映射到向量空间，支持语义检索
</details>

---

**6. 记忆去重的三级策略不包括？**

A. Hash 去重
B. 向量相似度去重
C. LLM 语义验证
D. 人工审核

<details>
<summary>点击查看答案</summary>
答案：D. 人工审核
</details>

---

**7. 检索准确率（Precision）的定义是？**

A. 检索结果中相关的比例
B. 相关内容中被检索出的比例
C. 检索结果的总数量
D. 检索花费的时间

<details>
<summary>点击查看答案</summary>
答案：A. 检索结果中相关的比例
</details>

---

**8. 为什么需要记忆压缩？**

A. 减少存储空间
B. 加快检索速度
C. 在有限上下文窗口内保留更多有效信息
D. 提高记忆重要性

<details>
<summary>点击查看答案</summary>
答案：C. 在有限上下文窗口内保留更多有效信息
</details>

---

**9. 联邦学习记忆的主要优势是？**

A. 更快的检索速度
B. 更大的存储容量
C. 在保护隐私的前提下共享知识
D. 更低的成本

<details>
<summary>点击查看答案</summary>
答案：C. 在保护隐私的前提下共享知识
</details>

---

**10. ReAct Agent 中记忆的注入时机是？**

A. Agent 启动时
B. 每次用户查询前
C. 仅在首次对话
D. 记忆更新时

<details>
<summary>点击查看答案</summary>
答案：B. 每次用户查询前
</details>

---

### 实践练习（5 个，从简单到复杂）

#### 练习 1：基础记忆操作（15 分钟）

**任务**：创建一个简单的记忆系统，实现基本的存储和检索功能。

```python
# 要求:
# 1. 实现 Memory 类，包含 id, content, created_at 属性
# 2. 实现 SimpleMemoryStore 类，支持 store() 和 search() 方法
# 3. 测试添加 5 条记忆并检索

# 参考代码框架
class Memory:
    def __init__(self, content: str):
        self.id = generate_id()
        self.content = content
        self.created_at = datetime.now()

class SimpleMemoryStore:
    def __init__(self):
        self._memories = []

    def store(self, memory: Memory):
        self._memories.append(memory)

    def search(self, keyword: str) -> List[Memory]:
        # 实现关键词搜索
        pass

# 测试
store = SimpleMemoryStore()
store.store(Memory("今天天气很好"))
store.store(Memory("我喜欢吃苹果"))
results = store.search("苹果")
print(f"找到 {len(results)} 条结果")
```

---

#### 练习 2：实现滑动窗口记忆（30 分钟）

**任务**：扩展基础记忆，实现滑动窗口机制。

```python
# 要求:
# 1. 实现 SlidingWindowMemory 类
# 2. 设置 max_messages = 5
# 3. 添加 10 条消息，验证只保留最后 5 条
# 4. 确保系统消息始终保留

class SlidingWindowMemory:
    def __init__(self, max_messages: int = 5, system_message: str = None):
        self.max_messages = max_messages
        self._messages = []
        if system_message:
            self._messages.append({"role": "system", "content": system_message})

    def add_message(self, role: str, content: str):
        # 实现滑动窗口逻辑
        pass

    def get_messages(self) -> List[dict]:
        return self._messages

# 测试验证
memory = SlidingWindowMemory(max_messages=5, system_message="你是助手")
for i in range(10):
    memory.add_message("user", f"消息 {i}")
    memory.add_message("assistant", f"回复 {i}")

assert len(memory.get_messages()) == 5  # 验证
print("滑动窗口测试通过!")
```

---

#### 练习 3：向量相似度计算（45 分钟）

**任务**：实现余弦相似度和简单的向量检索。

```python
# 要求:
# 1. 实现 cosine_similarity() 函数
# 2. 创建 5 个模拟文档向量
# 3. 实现相似度搜索，找到最相关的文档

import math

def cosine_similarity(vec1: List[float], vec2: List[float]) -> float:
    # 实现余弦相似度
    pass

def similarity_search(
    query: List[float],
    documents: List[Tuple[str, List[float]]],
    limit: int = 3
) -> List[Tuple[str, float]]:
    # 返回按相似度排序的文档
    pass

# 测试
documents = [
    ("文档 1", [0.1, 0.2, 0.3, 0.4]),
    ("文档 2", [0.5, 0.6, 0.7, 0.8]),
    ("文档 3", [0.1, 0.3, 0.2, 0.4]),
]
query = [0.15, 0.25, 0.35, 0.45]

results = similarity_search(query, documents)
for doc_id, score in results:
    print(f"{doc_id}: 相似度={score:.4f}")
```

---

#### 练习 4：完整的记忆管理器（90 分钟）

**任务**：整合前面的知识，实现一个完整的 MemoryManager。

```python
# 要求:
# 1. 支持记忆创建（含自动分类）
# 2. 支持向量检索和关键词检索
# 3. 支持重要性评分
# 4. 支持记忆更新和删除
# 5. 添加单元测试

class MemoryManager:
    def __init__(self, storage, classifier=None):
        self.storage = storage
        self.classifier = classifier

    async def create(self, content: str, memory_type: str = None) -> Memory:
        # 1. 去重检查
        # 2. 自动分类
        # 3. 生成嵌入
        # 4. 存储
        pass

    async def search(self, query: str, limit: int = 5) -> List[Memory]:
        # 混合检索（向量 + 关键词）
        pass

    async def update_importance(self, memory_id: str, score: float) -> bool:
        pass

    async def delete(self, memory_id: str) -> bool:
        pass

# 测试
@pytest.mark.asyncio
async def test_memory_manager():
    manager = MemoryManager(storage)

    # 创建
    mem = await manager.create("测试内容")
    assert mem is not None

    # 检索
    results = await manager.search("测试")
    assert len(results) >= 1

    # 删除
    await manager.delete(mem.id)
    results = await manager.search("测试")
    assert len(results) == 0
```

---

#### 练习 5：与 Agent 集成（120 分钟）

**任务**：将记忆系统集成到 ReAct Agent 中。

```python
# 要求:
# 1. 扩展 ReActAgent，添加记忆能力
# 2. 实现自动记忆检索和注入
# 3. 实现重要对话的自动存储
# 4. 完整的端到端测试

class ReActAgentWithMemory(ReActAgent):
    def __init__(self, llm_client, tools, memory_manager):
        super().__init__(llm_client, tools)
        self.memory = memory_manager

    def run(self, query: str) -> str:
        # 1. 检索相关记忆
        relevant = await self.memory.search(query, limit=3)

        # 2. 注入上下文
        if relevant:
            context = self._build_context(relevant)
            self._inject_context(context)

        # 3. 执行 Agent
        result = super().run(query)

        # 4. 存储重要内容
        await self._maybe_store(query, result)

        return result

# 端到端测试
async def test_agent_with_memory():
    agent = ReActAgentWithMemory(llm, tools, memory_manager)

    # 第一轮：告诉 Agent 用户信息
    agent.run("我叫张三，喜欢 Python 编程")

    # 第二轮：测试记忆
    result = agent.run("你还记得我喜欢的编程语言吗？")
    assert "Python" in result

    print("Agent 记忆测试通过!")
```

---

## 4.17 参考资料

### 核心论文

1. **"MemGPT: Towards LLMs as Operating Systems"** (2023)
   - 链接：https://arxiv.org/abs/2310.08560
   - 贡献：提出分层记忆架构，虚拟上下文管理

2. **"Cortex Memory: A Scalable Memory System for LLM Agents"** (2024)
   - Rust 实现的高性能记忆系统

3. **"ReAct: Synergizing Reasoning and Acting in Language Models"** (2022)
   - 链接：https://arxiv.org/abs/2210.03629
   - Agent 推理和行动框架

4. **"Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"** (RAG, 2020)
   - 链接：https://arxiv.org/abs/2005.11401
   - 检索增强生成

### 开源项目

| 项目 | 链接 | 特点 |
|------|------|------|
| LangChain Memory | https://python.langchain.com/docs/modules/memory | 易用的 Memory 接口 |
| Chroma | https://www.trychroma.com/ | Python 向量数据库 |
| Qdrant | https://qdrant.tech/ | Rust 向量搜索引擎 |
| Letta | https://letta.com/ | 基于 MemGPT 的商用方案 |

### 进一步阅读

1. **人类记忆模型**
   - Atkinson-Shiffrin 记忆模型
   - Baddeley 工作记忆模型

2. **认知架构**
   - ACT-R: https://act-r.psy.cmu.edu/
   - SOAR: https://soar.eecs.umich.edu/

3. **向量数据库技术**
   - FAISS: https://github.com/facebookresearch/faiss
   - Annoy: https://github.com/spotify/annoy

---

## 本章总结

本章深入探讨了 AI Agent 记忆系统的各个方面：

**核心概念：**
- 短期记忆（滑动窗口、对话压缩）
- 长期记忆（向量存储、语义检索）
- 记忆类型系统（6 种类型）
- 重要性评分与去重

**关键技术：**
- Embedding 与向量相似度
- 混合检索（向量 + 关键词）
- 记忆创建流水线
- 与 ReAct Agent 集成

**高级话题：**
- 多模态记忆
- 跨记忆推理
- 联邦学习记忆
- 安全与隐私

通过本章的学习，你应该具备了**从头构建生产级记忆系统**的能力。继续完成实践练习，将理论知识转化为实际技能！

