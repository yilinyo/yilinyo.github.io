---
title: langchain models 实战 消息系统详解
tag: langchain
date: 2026-03-30 22:12:00
---

# 💬 消息系统详解 —— LangChain 的对话语言

> **所在目录**：`message/message_use.ipynb`  
> **核心依赖**：`langchain_core.messages`

---

## 前言

如果说大模型是引擎，那么**消息（Message）就是燃料**。LangChain 用一套精心设计的消息类型系统来描述"对话"这件事——谁说了什么、什么角色说的、工具返回了什么……每一条信息都有明确的类型。

理解消息系统，是构建多轮对话、Agent、记忆系统的基础。

---

## 消息类型全景图

```
对话历史（messages 列表）
├── SystemMessage      ← 系统指令，设定 AI 的角色和行为规则
├── HumanMessage       ← 用户说的话
├── AIMessage          ← AI 的回复
│   └── tool_calls     ← AI 决定调用某个工具时附带的调用信息
└── ToolMessage        ← 工具执行后返回的结果
```

---

## 核心消息类型详解

### SystemMessage —— 系统提示词

系统消息是"幕后指令"，用来告诉模型它是谁、应该怎么做。它通常放在消息列表的**最开头**。

```python
from langchain_core.messages import SystemMessage

system = SystemMessage(content="你是一个专业的 Python 编程助手，请用简洁的代码示例回答问题。")
```

> 💡 `SystemMessage` 不会显示给用户，但会深刻影响模型的输出风格和行为。

---

### HumanMessage —— 用户消息

用户发送的每一条输入，都封装为 `HumanMessage`：

```python
from langchain_core.messages import HumanMessage

human = HumanMessage(content="如何在 Python 中实现单例模式？")
```

也可以直接传字典，LangChain 会自动转换：
```python
{"role": "user", "content": "如何实现单例模式？"}
```

---

### AIMessage —— 模型回复

模型的每次回复都是一个 `AIMessage`。当模型决定调用工具时，`AIMessage` 会包含 `tool_calls` 字段：

```python
from langchain_core.messages import AIMessage

# 普通回复
ai_msg = AIMessage(content="单例模式可以通过类变量实现...")

# 带工具调用的回复（Agent 场景）
ai_msg_with_tool = AIMessage(
    content="",
    tool_calls=[{
        "id": "call_abc123",
        "name": "search_web",
        "args": {"query": "Python 单例模式"}
    }]
)
```

---

### ToolMessage —— 工具返回结果

当工具执行完毕，其结果需要用 `ToolMessage` 包装后放回消息列表，让模型知道"工具告诉了我什么"：

```python
from langchain_core.messages import ToolMessage

tool_result = ToolMessage(
    content="搜索结果：单例模式是一种创建型设计模式...",
    tool_call_id="call_abc123"  # 必须与 AIMessage.tool_calls 中的 id 对应
)
```

> ⚠️ `tool_call_id` 必须与触发调用的 `AIMessage` 中的 `tool_calls[i].id` 完全匹配，否则模型会报错。

---

## 构建多轮对话

消息系统的真正威力在于**有状态的多轮对话**。将消息列表作为历史记录不断追加，模型就拥有了"记忆"：

```python
from langchain_core.messages import HumanMessage, AIMessage
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# 构建对话历史
messages = []

# 第一轮
messages.append(HumanMessage("我叫小明"))
response = llm.invoke(messages)
messages.append(response)  # AIMessage

# 第二轮
messages.append(HumanMessage("我叫什么名字？"))
response = llm.invoke(messages)
print(response.content)  # 输出：你叫小明
```

这正是 `multi_round_chat.py` 的实现原理：

```python
msgs = []
while True:
    input_str = input("我: ")
    msgs.append(HumanMessage(input_str))
    resp = llm.invoke(msgs)
    print(f"AI: {resp.content}")
    msgs.append(AIMessage(resp.content))
```

---

## Prompt Template —— 消息模板化

实际应用中，消息内容往往需要动态填充：

```python
from langchain_core.prompts import ChatPromptTemplate

template = ChatPromptTemplate.from_messages([
    ("system", "你是一个{language}语言专家"),
    ("human", "请解释：{concept}"),
])

# 填充变量，生成消息列表
messages = template.format_messages(
    language="Python",
    concept="装饰器"
)
```

---

## 消息类型速查表

| 消息类型 | 角色标识 | 典型使用场景 |
|----------|----------|-------------|
| `SystemMessage` | `system` | 设置 AI 角色、行为规则 |
| `HumanMessage` | `human` / `user` | 用户输入 |
| `AIMessage` | `ai` / `assistant` | 模型回复、工具调用意图 |
| `ToolMessage` | `tool` | 工具执行结果 |

---

## 小结

LangChain 的消息系统提供了一套**类型安全的对话表示层**，核心价值在于：

1. **语义清晰**：每种角色的发言有专属类型，而不是混乱的字典
2. **工具链路追踪**：`AIMessage.tool_calls` ↔ `ToolMessage.tool_call_id` 形成闭环
3. **无缝序列化**：消息对象可以直接传入任何支持对话的 LangChain 组件

> 🚀 **下一步**：学习如何用 Memory 模块自动管理对话历史，而不需要手动维护 `messages` 列表。
