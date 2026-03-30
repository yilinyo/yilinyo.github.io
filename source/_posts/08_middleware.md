---
title: langchain models 实战 Middleware 中间件
tag: langchain
date: 2026-03-30 22:12:00
---

# 🛡️ Middleware 中间件 —— Agent 的"护城河"

> **所在目录**：`middleware/bultin_middleware.ipynb`、`middleware/custom_middleware.ipynb`  
> **核心依赖**：`langchain.agents.middleware`

---

## 前言

中间件是软件工程中的经典模式：在核心逻辑执行前后插入横切关注点（Logging、权限校验、限流、重试……）。

LangChain 的 **Middleware** 系统将同样的思想引入 Agent：在工具调用和模型调用的前后插入拦截逻辑，无需修改 Agent 或工具的任何业务代码，就能实现：

- 👤 **人工审批**：敏感操作需要人类点"确认"才能执行
- 🔢 **调用限制**：防止 Agent 无限循环或超额消费
- 🔄 **自动重试**：网络抖动时自动重试，提升稳定性
- 🔏 **隐私脱敏**：自动屏蔽输入中的手机号、身份证等敏感信息
- 🧠 **智能工具筛选**：用模型决策哪些工具适合当前场景

LangChain 提供了两种使用方式：
1. **内置中间件** —— 开箱即用，覆盖常见场景
2. **自定义中间件** —— 通过 Hooks 机制按需定制逻辑

---

## 注册中间件

中间件通过 `create_agent()` 的 `middleware=` 参数注册，支持**链式叠加**：

```python
from langchain.agents import create_agent

agent = create_agent(
    model="gpt-4o-mini",
    tools=[...],
    middleware=[
        middleware_1,
        middleware_2,
        middleware_3,
    ]
)
```

执行顺序：`middleware_1 → middleware_2 → middleware_3 → 核心逻辑 → middleware_3 → middleware_2 → middleware_1`（类似洋葱模型）

---

# 📦 第一部分：内置中间件

---

## 1. HumanInTheLoopMiddleware —— 人工审批

对于发邮件、删数据等**高风险操作**，让人类在执行前进行审批：

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-4o-mini",
    tools=[send_email, read_emails],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
                "read_emails": False  # False = 无需审批
            }
        )
    ],
    checkpointer=InMemorySaver()  # 人机协作必须有 checkpointer
)
```

**审批流程：**

```python
import uuid
from langgraph.types import Command

config = {"configurable": {"thread_id": str(uuid.uuid4())}}

# 第一次调用：触发中断
result = agent.invoke(
    {"messages": [{"role": "user", "content": "发邮件给老板"}]},
    config=config
)

if result.get("__interrupt__"):
    print("⏸️ 等待人工审批...")
    # 人工决策：批准
    result = agent.invoke(
        Command(resume={"decisions": [{"type": "approve"}]}),
        config=config  # 必须用相同 config 恢复
    )
    print("✅ 已批准，邮件已发送")
```

---

## 2. ToolCallLimitMiddleware / ModelCallLimitMiddleware —— 调用限制

防止 Agent 进入无限循环或超额消费 API：

```python
from langchain.agents.middleware import ToolCallLimitMiddleware, ModelCallLimitMiddleware

agent = create_agent(
    model="gpt-4o-mini",
    tools=[search_web, expensive_api],
    middleware=[
        # 模型调用限制
        ModelCallLimitMiddleware(
            thread_limit=10,   # 整个会话最多调用模型 10 次
            run_limit=3,       # 单次运行最多 3 次
            exit_behavior="end"
        ),
        # 工具调用限制
        ToolCallLimitMiddleware(
            run_limit=5,       # 单次运行最多调用工具 5 次
            exit_behavior="end"
        ),
        # 特定工具的精细限制
        ToolCallLimitMiddleware(
            tool_name="expensive_api",
            run_limit=1,       # 高成本 API 每次运行只能调用 1 次
            exit_behavior="continue"  # 超出后返回错误消息，不中断
        ),
    ],
    checkpointer=InMemorySaver()
)
```

| 参数 | 说明 |
|------|------|
| `thread_limit` | 整个会话（同一 thread_id）的累计上限 |
| `run_limit` | 单次 `invoke` 调用的上限 |
| `exit_behavior` | 达到上限时的行为：`"end"` 优雅退出，`"continue"` 继续但报错 |

---

## 3. ToolRetryMiddleware —— 自动重试

为不稳定的外部 API 调用添加指数退避重试：

```python
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="gpt-4o-mini",
    tools=[unreliable_api],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,           # 最多重试 3 次（共尝试 4 次）
            backoff_factor=2.0,      # 指数退避（1s → 2s → 4s）
            initial_delay=1.0,       # 初始等待 1 秒
            retry_on=(               # 触发重试的异常类型
                ConnectionError,
                TimeoutError,
            ),
            on_failure="error"       # 重试耗尽后：返回错误给 LLM
        )
    ]
)
```

**退避时间计算**：`delay = initial_delay × backoff_factor^(retry_count)`
- 第 1 次重试：等待 1s
- 第 2 次重试：等待 2s
- 第 3 次重试：等待 4s

---

## 4. PIIMiddleware —— 隐私信息脱敏

自动检测并屏蔽用户输入中的隐私数据，防止其进入 LLM：

```python
from langchain.agents.middleware import PIIMiddleware

pii_phone = PIIMiddleware(
    "phone_masker",                        # 中间件名称（任意）
    detector=r"\b1[3-9]\d{9}\b",          # 正则：中国手机号
    strategy="mask",                       # 策略：mask/remove/replace
    apply_to_input=True                    # 处理用户输入
)

agent = create_agent(
    model="gpt-4o-mini",
    tools=[],
    middleware=[pii_phone],
    checkpointer=InMemorySaver()
)

# 用户发送含手机号的消息
result = agent.invoke({
    "messages": {"role": "user", "content": "我的手机号是 18274519812，帮我记录一下"}
})
# 发送给 LLM 的实际内容：我的手机号是 ***********，帮我记录一下
```

---

## 5. LLMToolSelectorMiddleware —— 智能工具筛选

当工具列表很长时，让模型先"想一想"哪些工具与当前问题相关，只暴露相关工具，提升推理效率：

```python
from langchain.agents.middleware import LLMToolSelectorMiddleware

agent = create_agent(
    model="gpt-4o-mini",
    tools=[tool_a, tool_b, tool_c, tool_d, ...],  # 大量工具
    middleware=[
        LLMToolSelectorMiddleware(
            model="gpt-4o-mini",
            max_tools=3  # 每次最多选 3 个工具
        )
    ]
)
```

---

# 🔧 第二部分：自定义中间件

内置中间件覆盖了常见场景，但实际项目中你往往需要**定制自己的逻辑**：记录业务日志、做性能分析、实现积分扣费……

LangChain 提供了一套 **Hooks（钩子）** 机制，让你可以把自定义代码"钩"在 Agent 执行流程的各个节点上。

---

## Hooks 一览

自定义中间件的核心就是 6 个 Hook，分为两类：

### 节点式钩子 —— "在旁边观察"

顺序执行，适用于**日志、验证、状态更新**等场景。它们不控制核心逻辑是否执行，只是在执行前后做一些额外的事情。

| Hook | 触发时机 | 执行次数 |
|------|---------|---------|
| `before_agent` | 智能体启动前 | 整个调用期间 **1 次** |
| `before_model` | 每次模型调用前 | 每轮循环 **1 次** |
| `after_model` | 每次模型响应后 | 每轮循环 **1 次** |
| `after_agent` | 智能体完成后 | 整个调用期间 **1 次** |

### 包装式钩子 —— "接管执行"

可以**控制核心逻辑是否执行、执行几次、如何处理结果**，适用于重试、缓存、转换、降级等场景。

| Hook | 包装对象 |
|------|---------|
| `wrap_model_call` | 包装模型调用 |
| `wrap_tool_call` | 包装工具调用 |

### 执行顺序全景图

```
智能体调用开始
    ↓
before_agent (middleware 1, 2, 3)
    ↓
┌─────── ReAct 循环 ───────┐
│                          │
│  before_model (1, 2, 3)  │
│         ↓                │
│  wrap_model_call (3→2→1) │
│         ↓                │
│     模型推理             │
│         ↓                │
│  wrap_model_call (1→2→3) │
│         ↓                │
│  after_model (3, 2, 1)   │
│         ↓                │
│  如果需要工具：          │
│    wrap_tool_call        │
│         ↓                │
│  重复直到完成            │
│                          │
└──────────────────────────┘
    ↓
after_agent (3, 2, 1)
```

> 💡 **注意**：节点式钩子按**注册顺序**执行 before，按**逆序**执行 after；包装式钩子则遵循洋葱模型——外层先进、内层先出。

---

## 方式一：函数式钩子（装饰器写法）

最简单的自定义方式——用装饰器把普通函数变成中间件。

### 节点式钩子的函数签名

```python
def hook_function(state, runtime) -> dict:
    # state: 当前智能体状态（包含 messages 等）
    # runtime: 运行时上下文（包含 config、store 等）
    return None  # 或返回状态更新 dict
```

### 示例：四个节点式钩子实战

```python
from langchain.agents import create_agent
from langchain.agents.middleware import before_model, after_model, before_agent, after_agent
from langchain.tools import tool
from langchain_core.messages import HumanMessage

@tool
def calculate(expression: str) -> str:
    """执行数学计算"""
    try:
        result = eval(expression)
        return f"计算结果: {result}"
    except Exception as e:
        return f"计算错误: {e}"

# ① before_model - 在模型调用前记录日志
@before_model
def log_before_model(state, runtime):
    """记录模型调用前的状态"""
    msg_count = len(state.get("messages", []))
    print(f"📝 准备调用模型，当前有 {msg_count} 条消息")
    return None  # 不修改状态

# ② after_model - 在模型响应后记录日志
@after_model
def log_after_model(state, runtime):
    """记录模型响应"""
    last_message = state.get("messages", [])[-1]
    print(f"✅ 模型已响应: {last_message.content[:50]}...")
    return None

# ③ before_agent - 智能体启动时执行一次
@before_agent
def log_agent_start(state, runtime):
    """记录智能体启动"""
    print("🚀 智能体开始执行")
    return None

# ④ after_agent - 智能体完成时执行一次
@after_agent
def log_agent_end(state, runtime):
    """记录智能体完成"""
    print("🏁 智能体执行完成")
    return None

# 注册中间件
agent_with_logs = create_agent(
    model="gpt-4o-mini",
    tools=[calculate],
    system_prompt="你是一个计算助手。",
    middleware=[
        log_agent_start,
        log_before_model,
        log_after_model,
        log_agent_end
    ]
)
```

**运行效果：**

```
🚀 智能体开始执行
📝 准备调用模型，当前有 1 条消息
✅ 模型已响应: ...
📝 准备调用模型，当前有 3 条消息
✅ 模型已响应: 25 * 4 的计算结果是 100。...
🏁 智能体执行完成
```

可以清晰地看到：`before_agent` 和 `after_agent` 各执行了一次，而 `before_model` 和 `after_model` 在 ReAct 循环中执行了两次（第一次调用工具，第二次生成回复）。

---

### 包装式钩子的函数签名

与节点式钩子不同，包装式钩子接管了整个执行过程，你可以决定**是否调用、何时调用、调用几次**：

```python
@wrap_model_call
def wrapper(request, handler) -> ModelResponse:
    # request: 模型调用请求（包含 messages 等）
    # handler: 实际执行模型调用的函数

    # 在调用前做某事
    response = handler(request)  # 调用模型
    # 在调用后做某事

    return response
```

### 示例：三个包装式钩子实战

```python
from langchain.agents.middleware import wrap_model_call, wrap_tool_call
import time

# ① 性能监控 - 测量模型调用耗时
@wrap_model_call
def measure_model_time(request, handler):
    """测量模型调用性能"""
    start_time = time.time()

    response = handler(request)  # 执行模型调用

    duration = time.time() - start_time
    print(f"⏱️  模型调用耗时: {duration:.3f}秒")

    return response

# ② 自动重试 - 模型调用失败时重试
@wrap_model_call
def retry_on_error(request, handler):
    """失败时自动重试（最多3次）"""
    max_retries = 3

    for attempt in range(max_retries):
        try:
            response = handler(request)
            if attempt > 0:
                print(f"✅ 第 {attempt + 1} 次尝试成功")
            return response
        except Exception as e:
            if attempt == max_retries - 1:
                print(f"❌ 重试 {max_retries} 次后仍然失败")
                raise
            print(f"⚠️  第 {attempt + 1} 次尝试失败，重试中...")
            time.sleep(0.5)

# ③ 工具调用包装 - 记录工具执行详情
@wrap_tool_call
def log_tool_call(request, handler):
    """记录工具调用详情"""
    tool_name = request.tool_call["name"]
    tool_args = request.tool_call["args"]

    print(f"🔧 调用工具: {tool_name}")
    print(f"   参数: {tool_args}")

    result = handler(request)

    print(f"   结果: {result}")

    return result

# 组合注册
agent_with_wraps = create_agent(
    model="gpt-4o-mini",
    tools=[calculate],
    system_prompt="你是一个计算助手。",
    middleware=[
        measure_model_time,  # 第 3 顺序进入，第 1 顺序退出
        retry_on_error,
        log_tool_call
    ]
)
```

**运行效果：**

```
⏱️  模型调用耗时: 1.228秒
🔧 调用工具: calculate
   参数: {'expression': '100 / 5 + 10'}
   结果: content='计算结果: 30.0' name='calculate' ...
⏱️  模型调用耗时: 1.155秒
```

---

## 方式二：类式中间件（面向对象写法）

当你的中间件需要**维护内部状态**（如计数器、日志列表）或**定义多个相关联的钩子**时，类式写法更优雅。

只需继承 `AgentMiddleware`，实现对应的钩子方法即可：

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    def __init__(self, config_param):
        self.config = config_param

    def before_model(self, state, runtime):
        return None

    def after_model(self, state, runtime):
        return None

    def wrap_model_call(self, request, handler):
        return handler(request)
```

### 示例 1：详细日志中间件

一个类中聚合了 `before_agent`、`before_model`、`after_model`、`after_agent` 四个钩子，共享内部的 `logs` 列表：

```python
from langchain.agents.middleware import AgentMiddleware
from datetime import datetime

class DetailedLoggingMiddleware(AgentMiddleware):
    """详细的日志记录中间件"""

    def __init__(self, log_level="INFO"):
        self.log_level = log_level
        self.logs = []  # 内部状态：日志列表

    def _log(self, level, message, data=None):
        """内部日志方法"""
        if level == "DEBUG" and self.log_level != "DEBUG":
            return

        timestamp = datetime.now().strftime("%H:%M:%S")
        log_entry = {"timestamp": timestamp, "level": level, "message": message, "data": data}
        self.logs.append(log_entry)

        emoji = {"INFO": "ℹ️ ", "DEBUG": "🔍", "ERROR": "❌"}
        print(f"{emoji.get(level, '')} [{timestamp}] {message}")
        if data and self.log_level == "DEBUG":
            print(f"         数据: {data}")

    def before_agent(self, state, runtime):
        self._log("INFO", "智能体开始执行")
        return None

    def before_model(self, state, runtime):
        msg_count = len(state.get("messages", []))
        self._log("DEBUG", f"准备调用模型", {"message_count": msg_count})
        return None

    def after_model(self, state, runtime):
        last_msg = state.get("messages", [])[-1]
        content = last_msg.content[:30] + "..." if len(last_msg.content) > 30 else last_msg.content
        self._log("DEBUG", f"模型已响应", {"content": content})
        return None

    def after_agent(self, state, runtime):
        self._log("INFO", "智能体执行完成")
        print(f"\n📊 共记录 {len(self.logs)} 条日志")
        return None
```

### 示例 2：性能监控中间件

同时使用 `wrap_model_call`（包装式）和 `after_agent`（节点式），在同一个类中混合两种钩子：

```python
class PerformanceMiddleware(AgentMiddleware):
    """性能监控中间件"""

    def __init__(self):
        self.model_call_count = 0
        self.total_duration = 0

    def wrap_model_call(self, request, handler):
        """测量每次模型调用的性能"""
        self.model_call_count += 1
        start_time = time.time()

        response = handler(request)

        duration = time.time() - start_time
        self.total_duration += duration

        print(f"⏱️  模型调用 #{self.model_call_count}: {duration:.3f}秒")

        return response

    def after_agent(self, state, runtime):
        """输出性能统计"""
        avg = self.total_duration / self.model_call_count if self.model_call_count > 0 else 0
        print(f"\n📈 性能统计:")
        print(f"   总调用次数: {self.model_call_count}")
        print(f"   总耗时: {self.total_duration:.3f}秒")
        print(f"   平均耗时: {avg:.3f}秒")
        return None
```

### 组合使用

```python
agent = create_agent(
    model="gpt-4o-mini",
    tools=[calculate],
    system_prompt="你是一个计算助手。",
    middleware=[
        DetailedLoggingMiddleware(log_level="DEBUG"),
        PerformanceMiddleware()
    ]
)
```

**运行效果：**

```
ℹ️  [20:38:14] 智能体开始执行
🔍 [20:38:14] 准备调用模型
         数据: {'message_count': 1}
⏱️  模型调用 #1: 1.326秒
🔍 [20:38:15] 模型已响应
         数据: {'content': ''}
🔍 [20:38:15] 准备调用模型
         数据: {'message_count': 3}
⏱️  模型调用 #2: 0.758秒
🔍 [20:38:16] 模型已响应
         数据: {'content': '计算结果是 90。'}

📈 性能统计:
   总调用次数: 2
   总耗时: 2.084秒
   平均耗时: 1.042秒
ℹ️  [20:38:16] 智能体执行完成

📊 共记录 6 条日志
```

---

## 方式三：扩展 Agent 状态 + 跳转控制

这是自定义中间件最强大的进阶用法——在钩子中**读写自定义状态字段**，甚至**控制 Agent 的执行流程**（提前结束、跳转到指定节点）。

### 核心概念

1. **`AgentState` 扩展**：定义额外的状态字段（如调用次数、积分余额），中间件可读写这些字段
2. **`state_schema` 参数**：告诉装饰器使用哪个状态 Schema
3. **`hook_config` + `can_jump_to`**：声明钩子有权跳转到哪些节点（`end`、`tools`、`model`）
4. **`jump_to`**：在返回的状态更新中指示 Agent 跳转到哪个节点

### 示例：调用次数限制 + 积分扣费

```python
from langchain.agents.middleware import AgentState, before_model, hook_config
from typing_extensions import NotRequired
from langchain_core.messages import AIMessage

# ① 定义扩展状态
class ExtendedState(AgentState):
    """扩展的智能体状态"""
    model_call_count: NotRequired[int]  # 模型调用次数
    user_credits: NotRequired[int]      # 用户剩余积分

# ② 调用次数限制（超过 5 次强制结束）
@before_model(state_schema=ExtendedState)
@hook_config(can_jump_to=["end"])
def limit_model_calls(state, runtime):
    """限制模型调用次数"""
    count = state.get("model_call_count", 0)

    if count >= 5:
        print(f"⛔ 达到调用限制（{count}/5），提前结束")
        return {
            "messages": [AIMessage(content="抱歉，已达到最大调用次数限制。")],
            "jump_to": "end"  # 👈 提前退出 Agent
        }

    print(f"📊 模型调用次数: {count + 1}/5")
    return {"model_call_count": count + 1}  # 更新计数

# ③ 积分扣费（每次调用扣 10 积分）
@before_model(state_schema=ExtendedState)
@hook_config(can_jump_to=["end"])
def deduct_credits(state, runtime):
    """每次模型调用扣除积分"""
    credits = state.get("user_credits", 100)

    if credits < 10:
        print(f"💳 积分不足（剩余 {credits}），停止执行")
        return {
            "messages": [AIMessage(content="您的积分不足，无法继续。")],
            "jump_to": "end"
        }

    new_credits = credits - 10
    print(f"💳 扣除 10 积分，剩余: {new_credits}")
    return {"user_credits": new_credits}

# ④ 创建 Agent
agent_with_state = create_agent(
    model="gpt-4o-mini",
    tools=[calculate],
    system_prompt="你是一个计算助手。",
    middleware=[limit_model_calls, deduct_credits]
)
```

### 测试场景

```python
# 正常调用（积分充足）
result = agent_with_state.invoke({
    "messages": [HumanMessage(content="计算 5 + 5")],
    "model_call_count": 0,
    "user_credits": 100
})
# 📊 模型调用次数: 1/5
# 💳 扣除 10 积分，剩余: 90
# 📊 模型调用次数: 2/5
# 💳 扣除 10 积分，剩余: 80
# → 正常返回结果

# 积分不足（触发提前退出）
result = agent_with_state.invoke({
    "messages": [HumanMessage(content="计算 10 + 10")],
    "model_call_count": 0,
    "user_credits": 5  # 不足 10 积分
})
# 📊 模型调用次数: 1/5
# 💳 积分不足（剩余 5），停止执行
# → 返回 "您的积分不足，无法继续。"
```

### 关键要点

> 💡 钩子函数的返回值是一个**状态更新 dict**，LangChain 会将它 **merge** 到当前 Agent State 中，供后续节点继续使用。这是钩子与 Agent 通信的唯一方式。

---

## 函数式 vs 类式：如何选择？

| 场景 | 推荐方式 | 理由 |
|------|---------|------|
| 简单日志/打印 | 函数式 `@before_model` | 一个装饰器搞定，最轻量 |
| 需要维护内部状态 | 类式 `AgentMiddleware` | 类天然支持 `self.xxx` 状态管理 |
| 多个相关联的钩子 | 类式 `AgentMiddleware` | 一个类中聚合多个方法，逻辑更内聚 |
| 需要控制执行流程 | 函数式 `@wrap_model_call` | 包装式钩子更直观 |
| 需要扩展状态 + 跳转 | 函数式 + `state_schema` + `hook_config` | 目前仅函数式支持此高级特性 |

---

## 中间件组合策略

实际生产中，多个中间件往往组合使用：

```python
middleware=[
    PIIMiddleware(...),                  # 第一道：隐私脱敏
    HumanInTheLoopMiddleware(...),       # 第二道：人工审批
    ToolRetryMiddleware(...),            # 第三道：失败重试
    ToolCallLimitMiddleware(...),        # 第四道：防止超额
    DetailedLoggingMiddleware(...),      # 第五道：日志记录
    PerformanceMiddleware(),             # 第六道：性能监控
]
```

> 💡 **排列建议**：安全类（脱敏、审批）放前面，稳定性类（重试、限流）放中间，观测类（日志、性能）放后面。

---

## 小结

### 内置中间件

| 中间件 | 解决问题 | 关键参数 |
|--------|---------|---------| 
| `HumanInTheLoopMiddleware` | 敏感操作需人工审批 | `interrupt_on` |
| `ModelCallLimitMiddleware` | 防止模型调用超额 | `thread_limit`、`run_limit` |
| `ToolCallLimitMiddleware` | 防止工具调用超额 | `tool_name`、`run_limit` |
| `ToolRetryMiddleware` | 外部 API 不稳定时自动重试 | `max_retries`、`retry_on` |
| `PIIMiddleware` | 屏蔽输入中的隐私信息 | `detector`、`strategy` |
| `LLMToolSelectorMiddleware` | 工具太多时智能筛选 | `max_tools` |

### 自定义中间件

| 方式 | 适用场景 | 核心 API |
|------|---------|---------|
| 函数式节点钩子 | 简单的前/后处理 | `@before_model`、`@after_model`、`@before_agent`、`@after_agent` |
| 函数式包装钩子 | 控制执行流程（重试/缓存/降级） | `@wrap_model_call`、`@wrap_tool_call` |
| 类式中间件 | 有状态的复杂中间件 | 继承 `AgentMiddleware`，实现对应方法 |
| 状态扩展 + 跳转 | 需要自定义字段和流程控制 | `state_schema`、`hook_config`、`jump_to` |
