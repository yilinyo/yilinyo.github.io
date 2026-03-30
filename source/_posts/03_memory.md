---
title: langchain models 实战 Memory 记忆模块
tag: langchain
date: 2026-03-30 22:12:00
---

# 🧠 Memory 记忆模块 —— 让 AI 记住过去的对话

> **所在目录**：`memory/short_memory.ipynb`  
> **核心依赖**：`langgraph.checkpoint`、`langgraph.store`、`langchain.agents`

---

## 前言

一个没有记忆的 AI 助手，每次对话都像是第一次见面。它不知道你叫什么名字，不知道你上次讲过什么，更别说积累对你的了解。

LangChain/LangGraph 的 **Memory（记忆）系统** 解决了这个问题，并且区分了两个维度：

- **短期记忆（Short-term Memory）**：单次对话或跨对话的消息历史，通过 **Checkpointer（检查点）** 实现
- **长期记忆（Long-term Memory）**：跨会话持久化的用户偏好、知识库，通过 **Store（存储）** 实现

---

## 短期记忆：Checkpointer

### 原理

Checkpointer 的本质是**状态快照**。每次 Agent 运行时，LangGraph 都会把当前的完整状态（包括所有消息）保存到 Checkpointer 中。下次使用相同的 `thread_id` 调用时，自动从上次的状态恢复。

```
第一次调用 ──► [保存状态到 Checkpointer]
                      │
                      ▼
第二次调用 ──► [从 Checkpointer 恢复状态] ──► 继续对话
```

### InMemorySaver —— 内存存储（开发调试用）

```python
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent

# 创建内存检查点
checkpointer = InMemorySaver()

agent = create_agent(
    model="gpt-4o-mini",
    checkpointer=checkpointer  # 注入记忆
)

# 第一轮对话：告诉 AI 我的名字
config = {"configurable": {"thread_id": "session_001"}}
agent.invoke(
    {"messages": [{"role": "user", "content": "我叫小明"}]},
    config=config
)

# 第二轮对话：AI 还记得
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我叫什么名字？"}]},
    config=config  # 相同的 thread_id！
)
print(result["messages"][-1].content)  # → "你叫小明"
```

> 💡 **关键**：相同的 `thread_id` = 同一个会话。不同的 `thread_id` = 全新对话，互不干扰。

---

### PostgresSaver —— 数据库持久化（生产环境用）

`InMemorySaver` 在进程重启后数据丢失，生产环境需要用数据库持久化：

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:password@localhost:5432/mydb?sslmode=disable"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # 自动创建所需数据表

    agent = create_agent(
        model="gpt-4o-mini",
        checkpointer=checkpointer
    )
    # ... 正常使用
```

**PostgresSaver vs InMemorySaver 对比：**

| 特性 | InMemorySaver | PostgresSaver |
|------|---------------|---------------|
| 持久化 | ❌ 进程重启后丢失 | ✅ 永久保存 |
| 性能 | 🚀 极快 | ⚡ 较快（有 I/O） |
| 适用场景 | 开发、测试 | 生产环境 |
| 配置复杂度 | 零配置 | 需要数据库 |

---

## 长期记忆：Store

Checkpointer 保存的是"消息历史"，而 **Store** 是一个**跨会话的键值存储**，更适合保存用户偏好、个性化设置等长期知识。

### InMemoryStore —— 基础使用

```python
from langgraph.store.memory import InMemoryStore
from langchain.tools import tool, ToolRuntime

store = InMemoryStore()

@tool
def save_preference(key: str, value: str, runtime: ToolRuntime) -> str:
    """保存用户偏好设置"""
    user_id = runtime.context.user_id
    namespace = ("preferences", user_id)  # 按用户 ID 隔离数据
    runtime.store.put(namespace, key, {"value": value})
    return f"✅ 已保存：{key} = {value}"

@tool
def get_preference(key: str, runtime: ToolRuntime) -> str:
    """获取用户偏好设置"""
    user_id = runtime.context.user_id
    namespace = ("preferences", user_id)
    item = runtime.store.get(namespace, key)
    return f"您的偏好：{key} = {item.value['value']}" if item else "未找到"
```

### 跨会话读取演示

```python
agent = create_agent(model="gpt-4o-mini", tools=[save_preference, get_preference], store=store)

# 会话 1：保存偏好
agent.invoke(
    {"messages": [{"role": "user", "content": "帮我保存偏好：language=Chinese"}]},
    context=CustomContext(user_id="user_001")
)

# 会话 2（全新 thread）：读取上次保存的偏好
result = agent.invoke(
    {"messages": [{"role": "user", "content": "我的 language 偏好是什么？"}]},
    context=CustomContext(user_id="user_001")  # 相同用户 ID
)
print(result["messages"][-1].content)  # → "您的偏好：language = Chinese"
```

---

## 两种记忆的定位

```
Checkpointer（短期/对话级记忆）
└── 存储：完整的消息历史
└── 范围：同一个 thread_id 内
└── 用途：让 AI 记住"这次对话说了什么"

Store（长期/用户级记忆）
└── 存储：任意键值数据（偏好、笔记、知识点）
└── 范围：跨 thread，按用户 ID 隔离
└── 用途：让 AI 记住"这个用户的个性化信息"
```

---

## 小结

| 组件 | 类型 | 生命周期 | 典型数据 |
|------|------|---------|---------|
| `InMemorySaver` | 短期记忆 | 进程级 | 对话消息历史 |
| `PostgresSaver` | 短期记忆 | 永久 | 对话消息历史 |
| `InMemoryStore` | 长期记忆 | 进程级 | 用户偏好、设置 |

> 🚀 **下一步**：学习 `struct_output` 模块，了解如何让模型输出结构化数据（JSON / Pydantic 对象），而不是随意的文本。
