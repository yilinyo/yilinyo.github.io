---
title: langchain models 实战 Agent 智能体
tag: langchain
date: 2026-03-30 22:12:00
---

# 🤖 Agent 智能体 —— 能思考、能行动的 AI

> **所在目录**：`agent/agent_hello.ipynb`、`agent_task/todolistagent.py`、`agent_task/test_todolistagent.py`  
> **核心依赖**：`langchain.agents`、`langgraph`

---

## 前言

如果说前面几个模块（模型、消息、工具）是 AI 应用的零件，那么 **Agent（智能体）** 就是把这些零件组装起来、让它们协同运转的引擎。

Agent 的核心能力：**自主决策**——给它一个目标，它会自己决定用什么工具、用几次、用什么顺序，直到完成任务。

---

## Agent 的工作原理：ReAct 循环

LangChain 中的 Agent 基于经典的 **ReAct（Reasoning + Acting）** 模式：

```
┌─────────────────────────────────────────────────────┐
│                    Agent Loop                       │
│                                                     │
│   用户输入                                           │
│       │                                             │
│       ▼                                             │
│   LLM 思考（需要调用哪个工具？参数是什么？）             │
│       │                                             │
│       ├─ 有工具调用 ──► 执行工具 ──► 结果返回给 LLM  ─┐│
│       │                                             ││
│       └─ 无工具调用 ──► 生成最终回复 ◄───────────────┘│
└─────────────────────────────────────────────────────┘
```

---

## 基础 Agent：快速上手

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """获取城市天气"""
    return f"{city}：晴天，28°C"

@tool
def search_web(query: str) -> str:
    """搜索网络信息"""
    return f"搜索结果：关于'{query}'的相关信息..."

# 创建 Agent
agent = create_agent(
    model="gpt-4o-mini",              # 使用的大模型
    tools=[get_weather, search_web],   # 可用工具列表
    system_prompt="你是一个全能助手"    # 系统提示词
)

# 运行 Agent
result = agent.invoke({
    "messages": [{"role": "user", "content": "北京今天天气怎样？"}]
})

print(result["messages"][-1].content)
```

---

## 自定义状态的 Agent

Agent 支持扩展默认的 `AgentState`，添加业务专属字段：

```python
from langchain.agents import AgentState, create_agent
from dataclasses import dataclass, field

@dataclass
class Task:
    priority: str = "中等"
    description: str = ""

@dataclass
class TaskManagerState(AgentState):
    current_tasks: list[Task] = field(default_factory=list)   # 待办任务
    completed_tasks: list[Task] = field(default_factory=list)  # 已完成任务
    user_name: str = ""

agent = create_agent(
    model="gpt-4o-mini",
    tools=[add_task, show_task, complete_task, delete_task],
    system_prompt="你是一个任务待办智能助手",
    state_schema=TaskManagerState  # 注入自定义状态
)
```

---

## 实战：任务管理 Agent

`agent_task/todolistagent.py` 实现了一个功能完整的**待办事项管理 Agent**，是本项目的核心实战案例。

### 架构设计

```
TaskManagerAgent
├── State: TaskManagerState
│   ├── current_tasks    ← 待办任务列表
│   ├── completed_tasks  ← 已完成任务列表
│   └── user_name        ← 用户名
│
└── Tools
    ├── add_task       ← 添加任务（支持优先级：高/中等/低）
    ├── show_task      ← 展示所有任务
    ├── complete_task  ← 将任务标记为完成
    └── delete_task    ← 彻底删除任务
```

### 任务优先级展示

```python
from collections import defaultdict

ORDER = ["高", "中等", "低"]  # 展示顺序

def format_tasks(tasks: list[Task]) -> str:
    """按优先级分组展示任务"""
    grouped = defaultdict(list)
    for t in tasks:
        grouped[t.priority].append(t.description)

    lines = []
    for p in ORDER:
        if p in grouped:
            names = grouped[p]
            lines.append(f"{p} ({len(names)}): {', '.join(names)}")
    return "\n".join(lines)

# 输出示例：
# 高 (1): 完成 leetcode 刷题
# 低 (1): 完成 30min 运动
```

### 工具实现示例：`add_task`

```python
@tool
def add_task(
    priority: str,
    task_description: str,
    runtime: ToolRuntime[CustomContext, TaskManagerState]
) -> Command:
    """添加待办任务

    Args:
        priority: 优先级（低/中等/高），默认中等
        task_description: 任务描述，如：完成有氧运动 30min
    """
    current_tasks = runtime.state["current_tasks"]

    # 参数校验：优先级非法时降级为"中等"
    if priority not in ["低", "中等", "高"]:
        priority = "中等"

    task = Task(priority=priority, description=task_description)
    current_tasks.append(task)

    return Command(
        update={
            "current_tasks": current_tasks,
            "messages": [ToolMessage(
                content="Successfully set new task",
                tool_call_id=runtime.tool_call_id
            )]
        }
    )
```

### 完整使用流程

```python
# 初始化空状态
tasks, finished = [], []

# 添加任务
result = agent.invoke({
    "messages": [{"role": "user", "content": "帮我添加低优先级任务：完成30min运动"}],
    "current_tasks": tasks,
    "completed_tasks": finished,
})

# 继承状态，继续操作
tasks = result.get("current_tasks")
finished = result.get("completed_tasks")

# 查看任务
result = agent.invoke({
    "messages": [{"role": "user", "content": "查看我的待办任务"}],
    "current_tasks": tasks,
    "completed_tasks": finished,
})
print(result["messages"][-1].content)
```

---

## 单元测试：如何测试 Agent 工具

直接测试 Agent 需要调用 LLM，成本高且结果不确定。更好的方案是**Mock ToolRuntime**，直接测工具逻辑：

```python
from unittest.mock import MagicMock

def make_runtime(current_tasks=None, completed_tasks=None, tool_call_id="test-id"):
    """构造 Mock 的 ToolRuntime"""
    runtime = MagicMock()
    runtime.state = {
        "current_tasks": current_tasks or [],
        "completed_tasks": completed_tasks or [],
    }
    runtime.tool_call_id = tool_call_id
    return runtime

def test_add_task():
    runtime = make_runtime()
    result = _add_task_logic("高", "写技术文档", runtime)

    assert len(result.update["current_tasks"]) == 1
    assert result.update["current_tasks"][0].priority == "高"
    assert "Successfully" in result.update["messages"][0].content
```

这种测试策略的优势：
- 🚀 **无 LLM 调用**：测试速度快，零成本
- 🎯 **精准覆盖**：直接测工具逻辑，与模型行为无关
- 🔁 **结果确定**：不受随机性影响，CI/CD 友好

---

## `create_agent` 参数速查

```python
agent = create_agent(
    model="gpt-4o-mini",        # 模型名称或实例
    tools=[...],                 # 工具列表
    system_prompt="...",         # 系统提示词
    state_schema=MyState,        # 自定义状态 Schema
    context_schema=MyContext,    # 不可变上下文 Schema
    checkpointer=saver,          # 记忆/检查点（短期记忆）
    store=store,                 # 持久化存储（长期记忆）
    middleware=[...],            # 中间件列表
)
```

---

## 小结

| 概念 | 作用 |
|------|------|
| `create_agent()` | 创建具备推理和工具调用能力的 Agent |
| `AgentState` | Agent 的内部状态，工具可读写 |
| `CustomContext` | 不可变的请求上下文（如用户ID） |
| `Command` | 工具通过此对象修改 Agent 状态 |
| Mock ToolRuntime | 单元测试工具逻辑的最佳实践 |

> 🚀 **下一步**：学习 `chunk` 模块，了解流式输出（Streaming）的使用方式。
