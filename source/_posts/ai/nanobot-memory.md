---
title: nanobot的记忆机制 
tag: agent
date: 2026-06-12 15:03:00
---
# nanobot 记忆系统学习笔记

  

本文整理前面对 nanobot 记忆系统的讨论，重点说明：哪些文件保存记忆、各自用途是什么、什么时候写入、如何恢复中断中的会话，以及 `runtime_checkpoint`、`last_consolidated`、`history.jsonl` 之间的关系。

  

## 一、总体设计

  

nanobot 的记忆系统不是单一文件，而是分成三层：

  

```text

sessions/*.jsonl

= 每个会话的完整上下文和运行状态

  

memory/history.jsonl

= 跨会话的全局历史归档队列

  

SOUL.md / USER.md / memory/MEMORY.md / skills/<name>/SKILL.md

= Dream 提炼后的长期记忆和可复用知识

```

  

可以理解为：

  

```text

会话消息先进入 sessions/*.jsonl

↓

旧消息太多时，归档/压缩进 memory/history.jsonl

↓

Dream 周期性读取 history.jsonl

↓

把有长期价值的信息整理进 SOUL.md / USER.md / MEMORY.md / SKILL.md

```

  

这个设计把“运行时上下文”和“长期记忆”分开：

  

- `sessions/*.jsonl` 负责让当前会话能继续。

- `memory/history.jsonl` 负责保存被压缩出去的历史线索。

- `MEMORY.md` 等长期文件负责保存提炼后的稳定知识。

  

## 二、关键文件作用

  

### 1. `sessions/<session-key>.jsonl`

  

作用：保存每个聊天会话的完整上下文。

  

位置由 `SessionManager` 管理，目录是：

  

```text

workspace/sessions/

```

  

每个 session 一个 JSONL 文件，正常格式如下：

  

```jsonl

{"_type":"metadata","metadata":{},"last_consolidated":0}

{"role":"user","content":"..."}

{"role":"assistant","content":"..."}

```

  

第一行是 metadata，后面是消息。

  

metadata 中可能保存：

  

- WebUI 标题

- sustained goal 状态

- `runtime_checkpoint`

- `_pending_user_turn`

- 其他会话级状态

  

`last_consolidated` 表示这个 session 前面多少条消息已经被归档/压缩过。

  

例如：

  

```text

messages[0:40] 已经归档

messages[40:100] 仍作为近期上下文使用

  

last_consolidated = 40

```

  

构建历史时会从这里开始：

  

```python

unconsolidated = self.messages[self.last_consolidated:]

```

  

所以它的作用是避免旧消息重复进入上下文，也避免重复归档。

  

### 2. `memory/history.jsonl`

  

作用：跨会话的全局历史归档队列。

  

它不是“每一行一个完整会话”，而是：

  

```text

一行 = 一次 append_history()

```

  

一条记录可能来自：

  

- 某个 session 中被压缩掉的一段旧消息摘要

- idle session 被 AutoCompact 归档的一段消息

- LLM 归档失败时的 `[RAW]` 原始片段

- 旧版 `HISTORY.md` 迁移过来的历史块

  

格式示例：

  

```json

{"cursor":1,"timestamp":"2026-06-11 10:30","content":"用户询问 nanobot 记忆机制..."}

```

  

字段含义：

  

- `cursor`：全局递增游标。

- `timestamp`：写入时间。

- `content`：归档摘要或原始片段。

  

它不是最终长期记忆，而是 Dream 的原料池。

  

### 3. `memory/.cursor`

  

作用：记录 `history.jsonl` 最新写入到哪个 cursor。

  

`append_history()` 写入一条历史时，会分配新的 cursor，并把最新 cursor 写到 `.cursor`。

  

这个文件用于保证历史条目的全局递增编号。

  

### 4. `memory/.dream_cursor`

  

作用：记录 Dream 已经处理到 `history.jsonl` 的哪一条。

  

Dream 运行时会读取：

  

```python

last_cursor = self.get_last_dream_cursor()

entries = self.read_unprocessed_history(since_cursor=last_cursor)

```

  

也就是说，Dream 只处理：

  

```text

cursor > .dream_cursor

```

  

Dream 成功完成后，才会更新 `.dream_cursor`。

  

如果 Dream 失败，cursor 不会前进，避免历史被误标记为已处理。

  

### 5. `memory/MEMORY.md`

  

作用：保存项目级长期记忆。

  

例如：

  

- 项目目标

- 架构理解

- 长期技术决策

- 基础设施概览

- 重要背景信息

  

它由 Dream 自动维护，不建议手动编辑。

  

### 6. `USER.md`

  

作用：保存用户画像和偏好。

  

例如：

  

- 用户身份

- 交流偏好

- 语言偏好

- 回复长度偏好

- 稳定的个人习惯

  

例如“用户偏好中文回答”应该进入 `USER.md`。

  

### 7. `SOUL.md`

  

作用：保存 Agent 自身行为规则和交互策略。

  

例如：

  

- 工具使用策略

- 与用户协作方式

- 行为护栏

- agent 的长期沟通风格

  

例如“回答前优先查源码”更适合进入 `SOUL.md`。

  

### 8. `skills/<name>/SKILL.md`

  

作用：保存可复用工作流、命令、步骤和具体操作模板。

  

Dream 发现某些信息不是普通记忆，而是可复用流程时，可能迁移到 skill。

  

例如：

  

- 固定排查步骤

- 重复出现的命令

- 某服务的操作流程

- 某类任务的模板

  

## 三、保存机制

  

### 1. Session 如何保存

  

`SessionManager.save()` 会重写整个 session 文件：

  

```text

1. 写入临时文件 .jsonl.tmp

2. 第一行写 metadata

3. 后续行写 session.messages

4. os.replace(tmp, 正式文件)

```

  

所以 session 文件的 metadata 正常只在第一行。

  

加载时实现更宽松：它会逐行扫描整个 JSONL，只要遇到 `_type == "metadata"` 就当作 metadata。如果异常出现多条 metadata，后面的会覆盖前面的。

  

### 2. `history.jsonl` 如何追加

  

`append_history()` 会：

  

```text

1. 清理 entry 中的 think/template 泄漏

2. 对超长内容做截断

3. 分配新的 cursor

4. 追加一行 JSON 到 memory/history.jsonl

5. 更新 memory/.cursor

```

  

每一行是一条独立归档记录，不按 session 单独分文件。

  

### 3. 什么时候写入 `history.jsonl`

  

主要情况：

  

```text

session 太长，需要 token 级压缩

idle session 被 AutoCompact 压缩

session 文件超过上限，需要硬裁剪

归档 LLM 失败，需要 raw_archive 兜底

旧版 HISTORY.md 迁移

```

  

正常情况下，归档会先让 LLM 总结旧消息，再把摘要写入 `history.jsonl`。

  

如果 LLM 总结失败，则写入 `[RAW]` 截断片段，避免信息完全丢失。

  

## 四、Dream 的作用

  

Dream 是长期记忆提炼器。

  

它不会直接读取所有 session，而是读取：

  

```text

memory/history.jsonl 中 .dream_cursor 之后的新记录

```

  

然后根据 `templates/agent/dream.md` 的规则，把信息路由到：

  

```text

SOUL.md

USER.md

memory/MEMORY.md

skills/<name>/SKILL.md

```

  

Dream 的核心原则是 MECE：

  

- 用户偏好进入 `USER.md`

- agent 行为规则进入 `SOUL.md`

- 项目长期背景进入 `MEMORY.md`

- 可复用流程进入 `SKILL.md`

- 不重复保存同一个事实

- 过期或短暂信息应删除或忽略

  

Dream 成功后：

  

```text

1. 更新 memory/.dream_cursor

2. compact_history() 压缩 history.jsonl

3. 如果 GitStore 已初始化，可能自动 commit 记忆变更

```

  

## 五、`runtime_checkpoint` 是什么

  

`runtime_checkpoint` 是保存在 `session.metadata` 里的临时恢复点。

  

它不是长期记忆，而是用于防止一轮对话跑到一半中断后丢状态。

  

它会在这些阶段生成：

  

```text

awaiting_tools

模型已经返回 tool_calls，但工具还没执行

  

tools_completed

工具已经执行完，但还没进入下一轮 LLM

  

final_response

最终回复已经生成，但还没完全保存到 session

```

  

checkpoint 一生成就会马上写 session 文件：

  

```python

session.metadata["runtime_checkpoint"] = payload

self.sessions.save(session)

```

  

所以它不是只存在内存里。

  

正常 turn 结束后会清掉：

  

```python

self._clear_runtime_checkpoint(session)

self.sessions.save(session)

```

  

因此正常情况下它只短暂存在。

  

## 六、checkpoint 如何恢复

  

恢复的判断很简单：

  

```text

进入 session 时检查 session.metadata["runtime_checkpoint"]

```

  

如果它是 dict，就说明上次 turn 中断后留下了 checkpoint。

  

恢复时不是恢复 Python 协程，而是把 checkpoint 中的半截 turn 转成正式消息，追加进 `session.messages`。

  

也就是：

  

```text

metadata["runtime_checkpoint"]

↓

session.messages

```

  

恢复逻辑会取出：

  

```text

assistant_message

completed_tool_results

pending_tool_calls

```

  

然后：

  

- `assistant_message` 追加为 assistant 消息。

- `completed_tool_results` 追加为 tool 消息。

- `pending_tool_calls` 补成失败的 tool 消息。

  

未完成工具会补成：

  

```json

{

"role": "tool",

"tool_call_id": "...",

"name": "tool_name",

"content": "Error: Task interrupted before this tool finished."

}

```

  

这样做是为了保证 OpenAI-style tool call 历史结构完整：

  

```text

assistant 有 tool_calls

后面必须有对应 tool 消息

```

  

恢复成功后，会立即保存 session 文件：

  

```python

if self._restore_runtime_checkpoint(ctx.session):

self.sessions.save(ctx.session)

```

  

保存后文件会变成：

  

```text

metadata 里没有 runtime_checkpoint

messages 里多了补进去的 assistant/tool 消息

```

  

## 七、几个关键问题总结

  

### 1. `memory/history.jsonl` 是按会话追加的吗？

  

不是严格按会话。

  

它是跨会话的全局归档日志。不同 session 的归档内容都会追加到同一个 `history.jsonl`。

  

单条记录不是一个完整会话，而是一次归档产生的一段内容。

  

### 2. 一条 `history.jsonl` 记录是不是一个会话？

  

不是。

  

更准确是：

  

```text

一条 history.jsonl 记录 = 一次 append_history()

```

  

它通常对应某个会话的一部分历史，而不是完整会话。每条对应的是一个会话的部分。

  

一个长会话可能产生多条 `history.jsonl` 记录。

  

### 3. `session.metadata` 是内存里的还是文件里的？

  

两者都是。

  

运行时它是内存中的 Python dict：

  

```python

session.metadata

```

  

保存时会写入 session JSONL 第一行：

  

```json

{"_type":"metadata","metadata":{...}}

```

  

如果只改内存但不 save，重启后会丢。

但 `runtime_checkpoint` 写入时会马上 `save()`。

  

### 4. session 初始化 metadata 只取 JSONL 第一行吗？

  

正常文件格式是第一行 metadata。

  

但加载实现不是只读第一行，而是逐行扫描：

  

- 遇到 `_type == "metadata"` 就更新 metadata。

- 其他行当作 message。

  

所以如果异常出现多条 metadata，后面的会覆盖前面的。

  

### 5. checkpoint 每次都会生成吗？

  

不是。

  

只有 runner 到达可恢复阶段时才生成：

  

```text

awaiting_tools

tools_completed

final_response

```

  

普通无工具调用的 turn 通常只会在 `final_response` 生成一次。

  

如果很早就报错或取消，可能没有 checkpoint。

  

### 6. 只有 `/stop` 才会恢复 checkpoint 吗？

  

不是。

  

`/stop` 是典型场景，但只要 session 文件里残留 `runtime_checkpoint`，下次进入同一个 session 时都会尝试恢复。

  

可能来源包括：

  

```text

/stop

进程崩溃

服务重启

异常中断

任务取消

```

  

### 7. 恢复后会马上写文件吗？

  

会。

  

恢复成功后外层会调用：

  

```python

self.sessions.save(session)

```

  

所以补进 `messages` 的 assistant/tool 消息会立即写回 `sessions/<session-key>.jsonl`。

  

## 八、核心关系图

  

```text

用户消息

↓

SessionManager 加载 sessions/<session-key>.jsonl

↓

检查 metadata.runtime_checkpoint

├─ 有：恢复半截 turn 到 session.messages，然后保存

└─ 无：正常继续

↓

构建上下文：system prompt + memory + recent session history

↓

AgentRunner 执行 LLM / tools 循环

↓

运行中写 runtime_checkpoint

↓

正常完成：保存 messages，清掉 runtime_checkpoint

↓

如果 session 太长：旧消息归档到 memory/history.jsonl

↓

Dream 读取未处理 history

↓

更新 SOUL.md / USER.md / MEMORY.md / skills

```

  

## 九、源码阅读入口

  

建议按这个顺序读：

  

1. `session/manager.py`

- `Session`

- `SessionManager._load()`

- `SessionManager.save()`

  

2. `agent/loop.py`

- `_set_runtime_checkpoint()`

- `_restore_runtime_checkpoint()`

- `_clear_runtime_checkpoint()`

- `_state_restore()`

- `_state_save()`

  

3. `agent/runner.py`

- `_emit_checkpoint()`

- `awaiting_tools`

- `tools_completed`

- `final_response`

  

4. `agent/memory.py`

- `MemoryStore.append_history()`

- `MemoryStore.raw_archive()`

- `MemoryConsolidator.archive()`

- `MemoryStore.build_dream_prompt()`

- `MemoryStore.set_last_dream_cursor()`

  

5. `templates/agent/dream.md`

- Dream 如何把历史路由到 `SOUL.md`、`USER.md`、`MEMORY.md`、`SKILL.md`

  

## 十、一句话总结

  

nanobot 的记忆系统本质上是：

  

```text

session 文件保存当前对话状态；

history.jsonl 保存被压缩出去的历史原料；

Dream 把历史原料提炼成长期记忆；

runtime_checkpoint 保障半轮对话中断后可以恢复。

```