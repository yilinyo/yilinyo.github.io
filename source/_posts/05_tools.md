---
title: langchain models 实战 Tools 工具系统
tag: langchain
date: 2026-03-30 22:12:00
---

# 🔧 Tools 工具系统 —— 给 Agent 装上"手"

> **所在目录**：`tools/tool_demo.ipynb`、`tools/tool_runtime.ipynb`  
> **核心依赖**：`langchain.tools`、`langchain.agents`

---

## 前言

大模型天生只能"说话"，它拥有海量知识，却无法执行代码、查询数据库、调用外部 API。**Tools（工具）** 解决了这个问题——它让 Agent 拥有了"手"，可以真正地和外部世界交互。

本模块涵盖两个层次：
1. **基础工具定义**（`tool_demo`）：如何把一个普通 Python 函数变成 Agent 可调用的工具
2. **Runtime 进阶**（`tool_runtime`）：工具如何读取和修改 Agent 的运行时状态

---

## Part 1：定义工具

### 最简单的方式：`@tool` 装饰器

```python
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气信息。

    Args:
        city: 城市名称，如：北京、上海
    """
    # 实际使用中这里会调用天气 API
    return f"{city}：晴天，温度 28°C，湿度 65%"

# 工具的元数据
print(get_weather.name)         # get_weather
print(get_weather.description)  # 获取指定城市的天气信息...
print(get_weather.args)         # {"city": {"type": "string"}}

# 直接调用（测试用）
result = get_weather.invoke("北京")
```

> ⚠️ **docstring 至关重要**！模型根据 docstring 来决定何时调用这个工具、如何填写参数。写得越清楚，模型的决策越准确。

---

### 带类型标注的工具

```python
from langchain.tools import tool
from typing import Optional

@tool
def search_database(
    query: str,
    table: str,
    limit: Optional[int] = 10
) -> str:
    """在数据库中搜索记录。

    Args:
        query: 搜索关键词
        table: 要查询的表名（users/orders/products）
        limit: 返回结果数量上限，默认 10
    """
    return f"在 {table} 表中搜索 '{query}'，返回 {limit} 条结果"
```

LangChain 会自动从**类型标注**和 **docstring 的 Args 段落**生成 JSON Schema，传给模型。

---

## Part 2：注册到 Agent

工具列表通过 `create_agent()` 的 `tools=` 参数注册：

```python
from langchain.agents import create_agent

agent = create_agent(
    model="gpt-4o-mini",
    tools=[get_weather, search_database],
    system_prompt="你是一个智能助手，可以查询天气和数据库信息。"
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "北京今天天气怎么样？"}]
})
print(result["messages"][-1].content)
```

工具调用的完整流程：

```
用户输入 → LLM 决定调用哪个工具 → 执行工具函数 → 结果返回给 LLM → LLM 生成最终回复
```

---

## Part 3：ToolRuntime —— 工具访问 Agent 状态

普通工具只能通过函数参数接收数据，无法知道 Agent 的内部状态。**`ToolRuntime`** 打破了这个限制，让工具可以访问 Agent 的：

- **`runtime.state`**：当前 Agent 的可变状态
- **`runtime.context`**：不可变的请求上下文（如当前用户 ID）
- **`runtime.store`**：跨会话的持久化存储
- **`runtime.tool_call_id`**：当前工具调用的 ID

### 读取 State

```python
from langchain.tools import tool, ToolRuntime
from langchain.agents import AgentState

@tool
def summarize_conversation(runtime: ToolRuntime) -> str:
    """统计当前对话的消息数量。"""
    messages = runtime.state["messages"]
    human_count = sum(1 for m in messages if m.__class__.__name__ == "HumanMessage")
    ai_count = sum(1 for m in messages if m.__class__.__name__ == "AIMessage")
    return f"对话共 {human_count} 条用户消息，{ai_count} 条 AI 回复"
```

---

### 使用自定义 State（泛型注解）

`ToolRuntime[ContextType, StateType]` 提供类型安全的访问：

```python
from dataclasses import dataclass
from langchain.agents import AgentState

@dataclass
class UserContext:      # 不可变：每次请求固定传入
    user_id: str

class AppState(AgentState):  # 可变：工具可以修改
    user_name: str = ""
    user_level: str = "basic"

@tool
def check_user_level(runtime: ToolRuntime[UserContext, AppState]) -> str:
    """查询当前用户等级。"""
    user_id = runtime.context.user_id   # 读不可变上下文
    level = runtime.state["user_level"] # 读可变状态
    return f"用户 {user_id} 的等级是：{level}"
```

---

### 通过 Command 修改 State

工具不仅可以读取状态，还可以通过返回 **`Command`** 对象来更新状态：

```python
from langgraph.types import Command
from langchain_core.messages import ToolMessage

@tool
def upgrade_user(runtime: ToolRuntime[UserContext, AppState]) -> Command:
    """将用户升级到更高等级。"""
    current_level = runtime.state["user_level"]
    new_level = "premium" if current_level == "basic" else "vip"

    return Command(
        update={
            "user_level": new_level,       # 更新状态字段
            "messages": [
                ToolMessage(
                    content=f"✅ 已升级到 {new_level}",
                    tool_call_id=runtime.tool_call_id  # 必填！
                )
            ]
        }
    )
```

> 💡 返回 `Command` 时，**必须**在 `messages` 中包含 `ToolMessage`，且 `tool_call_id` 要与 `runtime.tool_call_id` 一致。

---

### 访问不可变 Context

```python
USER_DB = {"user123": {"name": "Alice", "balance": 5000}}

@tool
def get_account_info(runtime: ToolRuntime[UserContext, AppState]) -> dict:
    """查询当前登录用户的账户信息。"""
    uid = runtime.context.user_id  # 从不可变 context 获取用户 ID
    return USER_DB.get(uid, "用户不存在")

# 调用时传入 context
result = agent.invoke(
    {"messages": [{"role": "user", "content": "查看我的账户"}]},
    context=UserContext(user_id="user123")
)
```

---

## 工具设计最佳实践

| 原则 | 说明 |
|------|------|
| 单一职责 | 每个工具只做一件事，职责清晰 |
| 清晰的 docstring | 是模型理解工具的唯一依据，务必详细 |
| 类型标注 | 帮助 LangChain 生成准确的 JSON Schema |
| 边界情况处理 | 工具应该优雅处理找不到数据等边界情况 |
| 避免副作用歧义 | 读操作和写操作分开定义，非必要不混用 |

---

## 小结

| 特性 | 基础工具 | ToolRuntime 工具 |
|------|----------|-----------------|
| 参数来源 | LLM 生成的参数 | LLM 参数 + Agent 状态/上下文 |
| 读取 State | ❌ | ✅ `runtime.state` |
| 修改 State | ❌ | ✅ 返回 `Command` |
| 访问 Context | ❌ | ✅ `runtime.context` |
| 跨会话存储 | ❌ | ✅ `runtime.store` |

> 🚀 **下一步**：学习 `agent` 和 `agent_task` 模块，了解如何将多个工具组合成一个能完成复杂任务的智能体。
