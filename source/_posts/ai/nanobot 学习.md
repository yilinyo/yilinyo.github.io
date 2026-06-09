---
title: "nanobot 学习"
date: "2026-06-09 07:05:33"
---
---
title: nanobot的整体运行机制 
tag: agent
date: 2026-06-09 15:03:00
---  
本文以nanobot项目代码为准，重点梳理一条用户消息从 Channel 进入系统，到 Agent 调用 LLM、执行工具、保存 Session，再返回响应的完整流程。  

![展示用户消息进入 nanobot、经过 Agent 执行与工具调用后保存会话并返回响应的封面图](https://files.seeusercontent.com/2026/06/09/9iXc/01-cover.webp)
  
## 一、核心概念  
  
### 1. Turn  
  
一个 Turn 表示系统处理一条入站消息的完整过程：  
  
```text  
接收 InboundMessage  
→ 恢复 Session  
→ 构建上下文  
→ 运行 Agent  
→ 保存结果  
→ 生成 OutboundMessage  
```  
  
需要区分三个容易混淆的概念：  
  
- **AgentLoop**：长期存在的编排对象，不是一次 request。  
- **Turn**：处理一条入站消息的一轮业务流程。  
- **Iteration**：一个 Turn 内部，`AgentRunner` 主循环的一次迭代。  
  
一个 Turn 可能包含多次 Iteration，例如：  
  
```text  
Iteration 0：LLM 请求 read_file  
Iteration 1：LLM 根据文件内容请求 edit_file  
Iteration 2：LLM 返回最终回答  
```  
  
通常一次 Iteration 会发起一次主要 LLM 请求，但空响应恢复等逻辑可能在同一个 Iteration 内额外请求 LLM，因此 Iteration 数不一定严格等于 Provider API 请求总数。  
  
### 2. 消息与总线  
  
Channel 将外部平台消息转换为 `InboundMessage`，再发布到 `MessageBus`：  
  
```text  
Telegram / Slack / WebSocket / CLI  
→ InboundMessage  
→ MessageBus.inbound  
→ AgentLoop  
```  
  
Agent 处理完成后生成 `OutboundMessage`：  
  
```text  
AgentLoop  
→ OutboundMessage  
→ MessageBus.outbound  
→ 对应 Channel  
```  
  
`MessageBus` 的作用是解耦 Channel 与 Agent 核心。Channel 不需要直接调用 LLM，AgentLoop 也不需要了解各平台的收发实现。  
  
### 3. Session  
  
`Session` 表示一段对话的持久化状态，定义在 `session/manager.py`：  
  
```python  
@dataclass  
class Session:  
    key: str  
    messages: list[dict[str, Any]]  
    created_at: datetime    updated_at: datetime    metadata: dict[str, Any]  
    last_consolidated: int  
```  
  
各字段含义：  
  
- `key`：会话唯一标识，通常是 `channel:chat_id`。  
- `messages`：该会话保存的消息。  
- `created_at`：Session 创建时间。  
- `updated_at`：最近一次消息或维护操作的时间。  
- `metadata`：标题、摘要、checkpoint、持续目标等扩展状态。  
- `last_consolidated`：已经归档的消息前缀长度。  
  
未归档、仍可进入近期上下文的消息是：  
  
```python  
session.messages[session.last_consolidated:]  
```  
  
## 二、AgentLoop 的职责  

![展示 nanobot 外部接入、消息总线、产品层编排、执行循环与会话持久化边界的架构图](https://files.seeusercontent.com/2026/06/09/xxH4/02-architecture.webp)
  
`AgentLoop` 定义在 `agent/loop.py`，是产品层的核心编排器，主要负责：  
  
1. 从 `MessageBus` 接收消息。  
2. 按 Session 串行、跨 Session 并发地调度任务。  
3. 维护 Turn 状态机。  
4. 构建历史、记忆、技能和运行时上下文。  
5. 调用 `AgentRunner`。  
6. 保存 Session 并发布响应。  
  
`AgentLoop` 不直接实现通用的 ReAct 工具循环；真正的 LLM/工具迭代位于 `AgentRunner`。  
  
## 三、AgentLoop 的主要入口  
  
### 1. `run()`：持续消费 MessageBus  
  
```python  
async def run(self) -> None:  
    while self._running:        msg = await asyncio.wait_for(            self.bus.consume_inbound(),            timeout=1.0,  
        )```  
  
它适用于 Gateway 和各种持续运行的 Channel。  
  
等待入站消息超时后，系统会检查空闲 Session：  
  
```python  
self.auto_compact.check_expired(...)  
```  
  
收到普通消息后，`run()` 创建后台任务：  
  
```python  
task = asyncio.create_task(self._dispatch(msg))  
```  
  
这样 AgentLoop 不会因为某个 Session 正在等待 LLM，而停止接收其他 Session 的消息。  
  
### 2. `_dispatch()`：并发调度与消息发布  
  
`_dispatch()` 的并发原则是：  
  
```text  
同一个 Session：通过 asyncio.Lock 串行执行  
不同 Session：可以并发执行  
全局并发量：可通过 Semaphore 限制  
```  
  
核心结构：  
  
```python  
lock = self._session_locks.setdefault(session_key, asyncio.Lock())  
gate = self._concurrency_gate or nullcontext()  
  
async with lock, gate:  
    response = await self._process_message(...)  
```  
  
如果同一个 Session 已经有任务运行，后续普通消息会进入 `pending_queue`，供当前 Runner 在执行过程中注入，而不是再启动一个相互竞争的 Turn。  
  
`_dispatch()` 还负责：  
  
- 建立流式输出回调。  
- 调用 `_process_message()`。  
- 将最终 `OutboundMessage` 发布到总线。  
- 在取消时恢复 runtime checkpoint。  
- 将未消费的 pending 消息重新发布，避免丢失。  
  
### 3. `process_direct()`：直接处理一条消息  
  
```python  
async def process_direct(  
    self,  
    content: str,  
    session_key: str = "cli:direct",  
    ...) -> OutboundMessage | None:  
```  
  
它自行构造 `InboundMessage`，取得 Session 锁，然后调用 `_process_message()`。  
  
当前项目中它被 Python SDK、OpenAI 兼容 API、CLI 的部分直接调用路径以及 Dream 内部任务使用。它不经过 MessageBus 的持续消费循环，但仍复用同一套 Turn 状态机。  
  
### 4. `_process_message()`：Turn 状态机驱动器  
  
`_process_message()` 创建 `TurnContext`，然后循环调用当前状态对应的 handler：  
  
```python  
while ctx.state is not TurnState.DONE:  
    handler_name = f"_state_{ctx.state.name.lower()}"    handler = getattr(self, handler_name)    event = await handler(ctx)    ctx.state = self._TRANSITIONS[(ctx.state, event)]  
```  
  
每个状态的耗时、事件和异常会写入 `ctx.trace`。  
  
## 四、TurnContext 与状态机  

![展示 AgentLoop 外层 Turn 状态机与 RUN 阶段内部 AgentRunner 多次 Iteration 的流程图](https://files.seeusercontent.com/2026/06/09/w3kT/03-turn-flow.webp)
  
### 1. `TurnState`  
  
完整状态流转：  
  
```text  
RESTORE  
→ COMPACT  
→ COMMAND  
→ BUILD  
→ RUN  
→ SAVE  
→ RESPOND  
→ DONE  
```  
  
`COMMAND` 还可以通过 `"shortcut"` 直接进入 `DONE`。  
  
### 2. `StateTraceEntry`  
  
记录单个状态的执行情况：  
  
```python  
@dataclass  
class StateTraceEntry:  
    state: TurnState    started_at: float  
    duration_ms: float  
    event: str  
    error: str | None = None```  
  
例如可以记录：  
  
```text  
RESTORE：3 ms  
BUILD：20 ms  
RUN：5 s  
```  
  
### 3. `TurnContext`  
  
`TurnContext` 不是 AgentLoop 本身，而是某一个 Turn 的临时工作上下文。  
  
它保存：  
  
- 当前 `InboundMessage` 和 `Session`。  
- 当前状态及 trace。  
- 历史消息和待发送给模型的 `initial_messages`。  
- 最终回答、工具使用情况和停止原因。  
- pending queue、压缩摘要和流式回调。  
- 本轮是否为 `ephemeral` 内部运行。  
  
## 五、各状态详解  
  
### 1. RESTORE  
  
对应 `_state_restore()`。  
  
主要工作：  
  
1. 处理消息附件，必要时提取文档文本。  
2. 获取或创建 Session。  
3. 发布 Session Turn 开始事件。  
4. 保存消息对应的工作区范围。  
5. 恢复中断前的 runtime checkpoint。  
6. 恢复提前持久化但尚未完成的用户消息。  
  
checkpoint 可以保留中断前的 Assistant tool call 和已经完成的工具结果，使 `/stop` 或任务取消后不会完全丢失执行现场。  
  
### 2. COMPACT  
  
对应 `_state_compact()`：  
  
```python  
ctx.session, pending = self.auto_compact.prepare_session(  
    ctx.session,    ctx.session_key,)  
ctx.pending_summary = pending  
```  
  
这里通常不执行实际压缩，而是把后台空闲压缩的结果接回当前 Turn：  
  
- 重新取得可能已经被替换的 Session。  
- 读取 `_last_summary`。  
- 将格式化摘要保存到 `ctx.pending_summary`，供 BUILD 阶段注入 prompt。  
  
真正的空闲压缩由：  
  
```text  
AutoCompact.check_expired()  
→ AutoCompact._archive()  
→ Consolidator.compact_idle_session()  
```  
  
在后台执行。  
  
如果 `idleCompactAfterMinutes=0`，空闲压缩关闭，此状态通常只返回原 Session；但如果 Session 已经有 `_last_summary`，仍可能读取并注入摘要。  
  
### 3. COMMAND  
  
对应 `_state_command()`。  
  
它调用：  
  
```python  
result = await self.commands.dispatch(cmd_ctx)  
```  
  
如果是普通消息，返回 `"dispatch"`，进入 BUILD。  
  
如果命令已直接处理，返回 `"shortcut"`，跳过 BUILD、RUN、SAVE 和 RESPOND，直接进入 DONE。为了让 WebUI 能恢复命令历史，除 `/new` 外，命令和结果会在这里直接保存，并标记 `_command=True`，使其不进入普通 LLM 历史。  
  
需要注意：`/stop` 等优先级控制命令会在 `AgentLoop.run()` 中提前处理，不一定进入这个状态。  
  
### 4. BUILD  
  
对应 `_state_build()`，作用是构造本轮 LLM 输入。  
  
#### 4.1 Token consolidation  
  
非 `ephemeral` Turn 先调用：  
  
```python  
await self.consolidator.maybe_consolidate_by_tokens(...)```  
  
安全输入预算：  
  
```text  
context_window_tokens  
- max_completion_tokens  
- 1024 safety buffer  
```  
  
估算对象不仅是 Session 文本，还包括：  
  
- System prompt。  
- 未归档 Session 历史。  
- `_last_summary`。  
- Memory、skills 和运行时上下文。  
- Session metadata。  
- 工具 definitions。  
- 探测消息 `[token-probe]`。  
  
当：  
  
```text  
estimated >= input budget  
```  
  
系统会选择旧的完整用户轮次，通过 LLM 生成摘要，写入 `memory/history.jsonl`，推进 `last_consolidated`，并把最近摘要写入：  
  
```python  
session.metadata["_last_summary"]  
```  
  
默认目标是压到：  
  
```text  
input budget × consolidation_ratio  
```  
  
`consolidation_ratio` 默认是 `0.5`。  
  
#### 4.2 构建近期历史  
  
```python  
ctx.history = ctx.session.get_history(  
    max_messages=self._max_messages,    max_tokens=self._replay_token_budget(),    include_timestamps=True,)  
```  
  
`get_history()` 会：  
  
- 只读取 `last_consolidated` 后的消息。  
- 按消息数和 token 预算截取。  
- 尽量从 user 消息开始。  
- 删除开头孤立的 tool result。  
- 过滤 `_command` 消息。  
- 为附件补充文本 breadcrumb。  
- 给 user 消息添加时间戳。  
  
#### 4.3 构造完整 prompt  
  
```python  
ctx.initial_messages = self._build_initial_messages(  
    ctx.msg,    ctx.session,    ctx.history,    ctx.pending_summary,    include_memory_recent_history=not ctx.ephemeral,)  
```  
  
最终上下文大致为：  
  
```text  
System prompt  
+ 工作区与运行时说明  
+ Skills  
+ MEMORY.md 等长期记忆  
+ 压缩摘要  
+ 最近完整历史  
+ 当前用户消息  
```  
  
BUILD 还会：  
  
- 设置工具上下文。  
- 重置 `MessageTool` 的本轮状态。  
- 提前持久化当前用户消息，防止执行中断后丢失。  
- 创建进度和重试等待回调。  
  
### 5. RUN  
  
对应 `_state_run()`。  
  
它先发布 `"running"` 状态，然后调用：  
  
```python  
result = await self._run_agent_loop(...)  
```  
  
`_run_agent_loop()` 负责把产品层上下文适配成 `AgentRunSpec`：  
  
- 进度、流式和额外 Hook。  
- checkpoint callback。  
- pending message injection callback。  
- RequestContext、workspace scope 和 file state。  
- 模型、工具、上下文窗口与超时。  
- sustained goal 的自动继续条件。  
  
随后调用：  
  
```python  
result = await self.runner.run(AgentRunSpec(...))  
```  
  
`_state_run()` 将返回结果写回 `TurnContext`：  
  
```python  
ctx.final_content  
ctx.tools_used  
ctx.all_messages  
ctx.stop_reason  
ctx.had_injections  
```  
  
最后检查是否需要创建内部 continuation。  
  
### 6. SAVE  
  
对应 `_state_save()`。  
  
主要工作：  
  
1. 处理 continuation 的保存边界。  
2. 为意外空回答补充统一占位文本。  
3. 计算本轮用户可感知延迟。  
4. 将 Runner 产生的新消息追加到 Session。  
5. 记录运行时间。  
6. 执行 Session 文件消息数量上限保护。  
7. 后台再次安排 token consolidation，为下一轮提前整理。  
8. 清除 pending user turn 和 runtime checkpoint。  
9. 原子保存 Session。  
  
这里的后台 token consolidation 与 BUILD 前的同步检查互补：  
  
```text  
BUILD 前：确保当前请求发送前不会超预算  
SAVE 后：利用后台时间为下一轮提前整理  
```  
  
### 7. RESPOND  
  
对应 `_state_respond()`。  
  
如果 `suppress_response=True`，则不生成普通出站消息。  
  
否则调用 `_assemble_outbound()`，将最终文本包装为：  
  
```python  
OutboundMessage(  
    channel=ctx.msg.channel,    chat_id=ctx.msg.chat_id,    content=ctx.final_content,    metadata=...,)  
```  
  
如果本轮 `MessageTool` 已经主动发送消息，普通最终回复可能被抑制，避免重复发送。  
  
流式响应会在 metadata 中标记 `_streamed`，延迟会记录为 `latency_ms`。`ephemeral` 内部运行还会附带 `_stop_reason`。  
  
## 六、AgentRunner  
  
`AgentRunner` 定义在 `agent/runner.py`，目标是提供不依赖 Channel、SessionManager 等产品层细节的通用工具型 Agent 执行循环。  
  
### 1. `AgentRunSpec`  
  
描述一次 Runner 执行所需的配置：  
  
- `initial_messages`  
- `tools`  
- `model`  
- `max_iterations`  
- `max_tool_result_chars`  
- Hook 和回调  
- workspace、session key  
- context window  
- timeout  
- injection callback  
- sustained goal 条件  
  
### 2. `AgentRunResult`  
  
Runner 的统一返回结构：  
  
```python  
@dataclass  
class AgentRunResult:  
    final_content: str | None  
    messages: list[dict[str, Any]]  
    tools_used: list[str]  
    usage: dict[str, int]  
    stop_reason: str  
    error: str | None  
    tool_events: list[dict[str, str]]  
    had_injections: bool  
```  
  
### 3. `run()`  
  
`AgentRunner.run()` 管理 Hook 生命周期和异常边界：  
  
```text  
hook.before_run()  
→ _run_core()  
→ hook.after_run() / hook.on_error()  
→ hook.on_finally()  
```  
  
真正的 ReAct 循环位于 `_run_core()`。  
  
## 七、`_run_core()` 主循环  
  
核心结构：  
  
```python  
for iteration in range(spec.max_iterations):  
    messages_for_model = 治理后的上下文    response = await self._request_model(...)  
    if response.should_execute_tools:        执行工具        continue  
    处理恢复、错误和注入    保存最终回答    breakelse:  
    stop_reason = "max_iterations"  
```  
  
这里的 `else` 属于 `for...else`：只有循环自然耗尽且没有执行 `break` 时才进入。  
  
### 1. 每轮上下文治理  
  
每次请求模型前依次执行：  
  
```python  
_drop_orphan_tool_results()  
_backfill_missing_tool_results()  
_microcompact()  
_apply_tool_result_budget()  
_snip_history()  
```  
  
作用分别是：  
  
- 删除没有对应 assistant tool call 的孤立 tool result。  
- 为缺失结果的 tool call 插入合成错误结果。  
- 将较早的大型工具结果替换为一行提示。  
- 限制单条工具结果大小，必要时落盘。  
- 当整个 prompt 超预算时，只保留较新的合法历史。  
  
这些处理只作用于 `messages_for_model`，不直接改变用于持久化的原始 `messages`。  
  
`_snip_history()` 后可能再次产生孤立工具消息，因此还会重新执行结构修复。  
  
### 2. 请求模型  
  
```python  
response = await self._request_model(  
    spec,    messages_for_model,    hook,    context,)  
```  
  
响应包含：  
  
- `content`  
- `tool_calls`  
- `finish_reason`  
- reasoning/thinking 信息  
- token usage  
  
`finish_reason` 表示单次 LLM 生成停止的原因，例如：  
  
- `stop`：正常结束。  
- `tool_calls` / `function_call`：请求执行工具。  
- `length`：达到输出 token 上限。  
- `error`：Provider 调用失败。  
  
它不同于 Runner 的 `stop_reason`；后者描述整个 Agent 执行最终为何停止。  
  
### 3. 工具调用分支  
  
当：  
  
```python  
response.should_execute_tools  
```  
  
为真时，Runner 会：  
  
1. 将 Assistant tool call 追加到 `messages`。  
2. 保存 `awaiting_tools` checkpoint。  
3. 调用 `_execute_tools()`。  
4. 将每个结果包装为 `role="tool"` 消息。  
5. 保存 `tools_completed` checkpoint。  
6. 检查是否有 pending 消息需要注入。  
7. `continue` 进入下一次 Iteration。  
  
### 4. 工具执行链  
  
完整调用路径：  
  
```text  
_run_core()  
→ _execute_tools()  
→ _partition_tool_batches()  
→ _run_tool()  
→ ToolRegistry.prepare_call()  
→ 具体 Tool.execute()  
```  
  
`prepare_call()` 负责：  
  
- 按名称找到工具。  
- 转换参数类型。  
- 根据 JSON Schema 校验参数。  
  
随后正常路径直接调用：  
  
```python  
result = await tool.execute(**params)  
```  
  
如果传入的是不支持 `prepare_call()` 的兼容工具容器，则退化为：  
  
```python  
result = await spec.tools.execute(tool_call.name, params)  
```  
  
只读且 `concurrency_safe=True` 的工具可以通过 `asyncio.gather()` 并发执行；写文件、执行命令等有副作用工具通常单独执行。  
  
工具结果会被标准化为：  
  
```python  
{  
    "role": "tool",    "tool_call_id": tool_call.id,    "name": tool_call.name,    "content": normalized_result,}  
```  
  
下一轮 LLM 请求会看到该结果，并决定继续调用工具还是返回最终答案。  
  
### 5. 没有工具调用时  
  
没有工具调用不代表立即在分支处返回。Runner 还需要检查：  
  
- 空回答是否需要重试。  
- 输出是否被 `max_tokens` 截断并需要续写。  
- 是否有用户追加消息或子 Agent 结果。  
- sustained goal 是否仍需继续。  
- Provider 是否返回错误。  
  
如果这些情况都不存在，则：  
  
```python  
messages.append(assistant_message)  
final_content = clean  
break  
```  
  
循环退出后统一构造 `AgentRunResult`。  
  
### 6. 恢复与特殊分支  
  
#### 空回答  
  
空回答会先重试；多次为空时，再发起 finalization retry。仍为空则：  
  
```python  
stop_reason = "empty_final_response"  
```  
  
#### 输出截断  
  
当：  
  
```python  
response.finish_reason == "length"  
```  
  
Runner 会把当前部分回答加入消息，再加入“继续输出”提示，然后进入下一次 Iteration。恢复次数有上限。  
  
#### 中途消息注入  
  
用户在 Agent 执行过程中发送的新消息，或子 Agent 返回的结果，可以通过 `injection_callback` 注入当前消息链。  
  
有注入时：  
  
```python  
had_injections = True  
continue  
```  
  
#### LLM 或工具错误  
  
错误会被转换为统一文本和 `stop_reason`。如果此时还有可注入消息，Runner 仍可能继续；否则退出。  
  
#### 最大迭代次数  
  
如果一直调用工具、续写或处理注入，直到 `for` 循环自然耗尽：  
  
```python  
stop_reason = "max_iterations"  
```  
  
系统生成统一的迭代上限提示并返回。  
  
## 八、Session 压缩的三个层次  

![对比 nanobot Session 的空闲压缩、Token consolidation 与 Runner snip 三层机制](https://files.seeusercontent.com/2026/06/09/qr3E/05-compaction-comparison.webp)
  
### 1. 空闲压缩  
  
配置：  
  
```json  
{  
  "agents": {    "defaults": {      "idleCompactAfterMinutes": 30    }  }}  
```  
  
`0` 表示关闭，默认值也是 `0`。  
  
Session 空闲达到 TTL 后，后台调用 `compact_idle_session()`：  
  
- 将旧消息通过 LLM 归档。  
- 最多保留最近 8 条合法消息。  
- 真正替换 `session.messages`。  
- 将 `last_consolidated` 重置为 `0`。  
- 保存 `_last_summary`。  
  
空闲压缩不先检查 Session 是否 token 超限；触发条件是空闲时间。  
  
### 2. Token consolidation  
  
在 BUILD 前同步检查，并在 SAVE 后后台检查。  
  
触发条件是完整 prompt 的估算 token 达到安全输入预算。它通常不立即删除旧消息，而是：  
  
- 对旧消息生成摘要。  
- 写入 `history.jsonl`。  
- 推进 `last_consolidated`。  
- 让 `get_history()` 忽略已经归档的前缀。  
  
### 3. Runner 即时裁剪  
  
`AgentRunner._snip_history()` 是最后一道防线：  
  
- 不调用 LLM 摘要。  
- 不修改 Session。  
- 不更新 `last_consolidated`。  
- 只裁剪当前这一请求实际发送给模型的 `messages_for_model`。  
  
三者的区别：  
  
```text  
空闲压缩：按空闲时间触发，硬删除旧 Session 消息  
Token consolidation：按 prompt token 触发，持久化摘要并推进游标  
Runner snip：每次请求前的临时防线，只影响当前模型输入  
```  
  
## 九、一次完整请求示例  

![展示 Channel、MessageBus、AgentLoop、AgentRunner、LLM 与 ToolRegistry 完成一次请求的时序图](https://files.seeusercontent.com/2026/06/09/6Wwn/04-sequence.webp)
  
```text  
1. Channel 收到用户消息  
2. 发布 InboundMessage 到 MessageBus  
3. AgentLoop.run() 消费消息  
4. _dispatch() 获取 Session 锁  
5. _process_message() 创建 TurnContext  
6. RESTORE 恢复 Session 和 checkpoint  
7. COMPACT 接入后台空闲压缩结果  
8. COMMAND 判断是否为快捷命令  
9. BUILD 检查 token、读取历史、构建 prompt  
10. RUN 调用 AgentRunner  
11. LLM 请求 read_file  
12. Runner 执行 ReadFileTool.execute()  
13. 工具结果加入 messages  
14. Runner 再次请求 LLM  
15. LLM 返回最终回答  
16. SAVE 将新消息写入 Session  
17. RESPOND 构造 OutboundMessage  
18. _dispatch() 发布消息到 MessageBus  
19. Channel 将响应发送给用户  
```  
  
## 十、代码的边界  
  
```text  
Channel / MessageBus：消息传输  
AgentLoop：产品层 Turn 编排  
TurnContext：单个 Turn 的临时状态  
SessionManager：会话持久化  
ContextBuilder：Prompt 组装  
Consolidator / AutoCompact：历史压缩与摘要  
AgentRunner：通用 LLM + Tool 迭代  
ToolRegistry：工具注册、参数校验与执行分发  
Provider：具体模型 API 调用  
```  
  
理解这些边界后，阅读 nanobot 的主线可以简化为：  
  
```text  
消息进入  
→ AgentLoop 组织一轮 Turn  
→ ContextBuilder 准备模型输入  
→ AgentRunner 完成 ReAct 循环  
→ ToolRegistry 执行能力  
→ SessionManager 保存现场  
→ 消息返回  
```
