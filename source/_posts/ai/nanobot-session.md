---
title: nanobot的session和workspace机制 
tag: agent
date: 2026-06-22 15:03:00
---
# nanobot Session、SessionManager 与 Workspace 机制

  

本文结合 `nanobot.md` 的执行流程梳理，说明 nanobot 中 Session、SessionManager 与 Workspace 的设计关系，以及一条会话从创建、使用、持久化、压缩到删除的生命周期。

  

## 1. 总体关系

  

nanobot 的会话系统可以按下面的层次理解：

  

```text

Workspace

-> SessionManager

-> Session

```

  

- `Workspace` 是 nanobot 的状态与上下文根目录。

- `SessionManager` 绑定到某一个 workspace，管理该 workspace 下的所有 session 文件。

- `Session` 表示一条具体聊天线程，通常对应一个 `channel:chat_id`。

  

普通 CLI/gateway 模式中，workspace 同时承担这些职责：

  

```text

<workspace>/

sessions/ # 会话文件

memory/ # 长期记忆与历史归档

skills/ # workspace 级技能

AGENTS.md # 工作区指令

USER.md

SOUL.md

HEARTBEAT.md

```

  

默认 workspace 是：

  

```text

~/.nanobot/workspace

```

  

启动时可以通过 `--workspace` 或 `-w` 覆盖：

  

```bash

nanobot agent -w /path/to/project

nanobot gateway -w /path/to/project

```

  

## 2. Workspace 与 Session 的绑定关系

  

Session 文件存放在 workspace 下：

  

```text

<workspace>/sessions/<safe_session_key>.jsonl

```

  

因此，同一个 session key 在不同 workspace 下是不同会话：

  

```text

~/.nanobot/workspace/sessions/cli_direct.jsonl

/path/to/project/sessions/cli_direct.jsonl

```

  

它们都可以对应 `cli:direct`，但内容彼此独立。

  

所以 workspace 与 session 的关系不是 session 自己携带一个独立 workspace 字段，而是：

  

```text

workspace -> SessionManager -> sessions/*.jsonl

```

  

## 3. Session 是什么

  

`Session` 定义在 `session/manager.py`，核心结构如下：

  

```python

@dataclass

class Session:

key: str

messages: list[dict[str, Any]]

created_at: datetime

updated_at: datetime

metadata: dict[str, Any]

last_consolidated: int = 0

```

  

字段含义：

  

- `key`：会话唯一标识，通常是 `channel:chat_id`。

- `messages`：会话中持久化保存的消息。

- `created_at`：创建时间。

- `updated_at`：最近一次更新或维护时间。

- `metadata`：扩展状态，例如 WebUI 标题、workspace scope、goal state、摘要、checkpoint 等。

- `last_consolidated`：已经被归档/摘要过的消息前缀长度。

  

未归档、仍可能进入近期上下文的消息是：

  

```python

session.messages[session.last_consolidated:]

```

  

常见 session key：

  

```text

cli:direct

websocket:<chat_id>

telegram:<chat_id>

discord:<chat_id>

heartbeat

```

  

`nanobot agent` 默认使用：

  

```text

cli:direct

```

  

也可以手动指定：

  

```bash

nanobot agent -s cli:project-a

```

  

## 4. SessionManager 是什么

  

`SessionManager` 是当前 workspace 下所有 session 的仓库和持久化层。

  

初始化时：

  

```python

SessionManager(workspace)

```

  

内部会设置：

  

```python

self.workspace = workspace

self.sessions_dir = workspace / "sessions"

self._cache: dict[str, Session]

```

  

它管理的是：

  

```text

<workspace>/sessions/*.jsonl

```

  

注意：它不是全局管理所有 workspace。换一个 workspace，就会有另一个 `SessionManager`。

  

核心职责：

  

```text

get_or_create(key) 获取或创建 Session

_load(key) 从 JSONL 加载 Session

_repair(key) 尝试修复损坏 JSONL

save(session) 原子保存 Session

flush_all() 退出时 fsync 保存缓存中的 Session

invalidate(key) 清除内存缓存

delete_session(key) 删除 Session 文件

read_session_file(key) 只读加载，供 WebUI/API 使用

list_sessions() 扫描 sessions 目录，生成会话列表

```

  

典型使用方式：

  

```python

manager = SessionManager(workspace)

  

session = manager.get_or_create("cli:direct")

session.add_message("user", "hello")

manager.save(session)

```

  

`Session` 负责一条会话自身的数据和局部逻辑，`SessionManager` 负责多条会话的生命周期和磁盘持久化。

  

## 5. Session 文件格式

  

Session 文件是 JSONL：

  

```text

第一行：metadata

后续每行：一条消息

```

  

第一行大致如下：

  

```json

{

"_type": "metadata",

"key": "cli:direct",

"created_at": "...",

"updated_at": "...",

"metadata": {},

"last_consolidated": 0

}

```

  

后续消息行大致如下：

  

```json

{"role": "user", "content": "hello", "timestamp": "..."}

{"role": "assistant", "content": "hi", "timestamp": "..."}

```

  

保存时采用临时文件加原子替换：

  

```text

写入 .jsonl.tmp

-> os.replace(tmp, target)

```

  

进程退出时还会通过 `flush_all(fsync=True)` 尽量保证缓存中的 session 稳定落盘。

  

## 6. Session 生命周期

  

一个 session 的生命周期可以概括为：

  

```text

不存在

-> get_or_create(key)

内存 Session

-> save(session)

磁盘 JSONL Session

-> 进程重启后再次 get_or_create(key)

从磁盘恢复到内存

-> 每个 turn 更新 messages / metadata

持续保存

-> compact / archive

旧消息被压缩或归档，但 session 继续存在

-> delete_session(key)

从内存和磁盘删除

```

  

### 6.1 创建

  

当某个 `channel:chat_id` 第一次被处理时，AgentLoop 会调用：

  

```python

session = self.sessions.get_or_create(key)

```

  

如果内存缓存中已有，则直接返回。
如果磁盘存在对应 JSONL，则加载。
如果都不存在，则创建新的：

  

```python

Session(key=key)

```

  

### 6.2 使用

  

每次处理用户消息时，AgentLoop 会：

  

1. 根据 `InboundMessage` 得到 session key。

2. 使用 `SessionManager.get_or_create()` 获取 session。

3. 通过 `session.get_history()` 取近期历史。

4. 用 `ContextBuilder` 构建模型上下文。

5. 调用 `AgentRunner` 执行 LLM/tool 循环。

6. 将用户消息、assistant 回复、tool call/result 等写回 session。

7. 调用 `SessionManager.save()` 保存。

  

### 6.3 恢复

  

进程重启后，内存 cache 清空。

  

下一次访问同一个 key 时：

  

```python

get_or_create(key)

```

  

会重新从 JSONL 文件恢复 session。

  

如果 JSONL 中有损坏行，`SessionManager._repair()` 会尝试跳过坏行，尽量恢复可用消息。

  

### 6.4 删除

  

普通运行不会自动删除整个 session。

  

显式删除主要来自 WebUI 删除聊天，对应：

  

```python

delete_session(key)

```

  

删除时会：

  

1. 从内存 cache 移除。

2. 删除 `<workspace>/sessions/<safe_key>.jsonl`。

3. WebUI 还会清理自己的 thread/transcript 文件。

  

需要区分：

  

- 删除 session：整个会话文件消失。

- 压缩/归档 session：旧消息被摘要或迁移，但会话仍然存在。

  

## 7. AgentLoop 如何使用 Session

  

`nanobot.md` 中的一条请求主线是：

  

```text

Channel 收到用户消息

-> 转成 InboundMessage

-> MessageBus

-> AgentLoop

-> 恢复 Session

-> 构建上下文

-> AgentRunner 调 LLM / 执行工具

-> SAVE 写回 Session

-> OutboundMessage 返回 Channel

```

  

`AgentLoop` 是产品层编排器，负责：

  

- 从 `MessageBus` 接收消息。

- 按 session 串行、跨 session 并发地调度任务。

- 维护 turn 状态机。

- 构建历史、记忆、技能和运行时上下文。

- 调用 `AgentRunner`。

- 保存 session 并发布响应。

  

同一个 session 会用锁串行执行：

  

```text

同一个 Session：串行

不同 Session：可以并发

```

  

这样可以避免同一条聊天同时跑多个互相覆盖的 turn。

  

## 8. Session 不等于最终 Prompt

  

Session 是上下文来源之一，但最终发给模型的 prompt 不是简单地把 `session.messages` 全量塞进去。

  

`ContextBuilder` 会组合：

  

```text

System prompt

+ workspace 说明

+ AGENTS.md / USER.md / SOUL.md

+ tools contract

+ memory/MEMORY.md

+ skills

+ archived summary

+ 最近 session history

+ 当前用户消息

+ runtime context

```

  

`session.get_history()` 也会对历史做处理：

  

- 只取 `last_consolidated` 之后的未归档消息。

- 按 `max_messages` 截取尾部。

- 尽量从 user turn 开始。

- 避免孤立 tool result。

- 可按 token 预算裁剪。

- 图片消息回放时加占位 breadcrumb。

  

因此可以理解为：

  

```text

Session 保存现场

ContextBuilder 负责把现场和 workspace/memory/skills 组装成模型输入

```

  

## 9. 压缩、归档与 last_consolidated

  

nanobot 有多层上下文控制机制：

  

```text

AutoCompact

Token consolidation

Runner snip

Session 文件硬上限

```

  

### 9.1 AutoCompact

  

空闲时间达到配置阈值后，对 session 做后台压缩。

  

它可能会：

  

- 将旧消息通过 LLM 归档。

- 保留最近合法消息尾巴。

- 替换 `session.messages`。

- 保存 `_last_summary`。

  

### 9.2 Token consolidation

  

在 BUILD 前同步检查，并在 SAVE 后后台检查。

  

当历史重放窗口或 prompt token 接近预算时，它会：

  

- 对旧消息生成摘要。

- 写入 `memory/history.jsonl`。

- 推进 `last_consolidated`。

- 让 `get_history()` 忽略已归档前缀。

  

### 9.3 Runner snip

  

这是 AgentRunner 请求模型前的临时防线。

  

它只影响当前发送给模型的 `messages_for_model`：

  

- 不修改 Session。

- 不更新 `last_consolidated`。

- 不写入摘要。

  

### 9.4 Session 文件硬上限

  

SAVE 阶段还会执行：

  

```python

ctx.session.enforce_file_cap(

on_archive=self.context.memory.raw_archive,

)

```

  

它限制 session 文件中实际保存的消息数量。如果超过上限，会保留最近合法后缀，并把尚未归档的被删除消息 raw archive 到历史中。

  

## 10. WebUI 的 workspace_scope

  

普通 CLI 模式中，workspace 同时是：

  

```text

session 存储目录

memory/skills 目录

工具相对路径根

```

  

WebUI 多了一层 `workspace_scope`。

  

这表示：

  

```text

session 文件仍然存放在默认 workspace/sessions 下

但某个 WebUI 聊天可以指定自己的 project path

```

  

可以区分为：

  

```text

storage workspace:

session、memory、skills 存放在哪里

  

effective project workspace scope:

当前 WebUI 聊天这一轮工具在哪个项目目录工作

```

  

因此 WebUI 可以做到：

  

```text

默认 workspace 存 nanobot 状态

聊天 A 操作 project-a

聊天 B 操作 project-b

```

  

而 CLI 模式目前没有完整的 data-dir/project-dir 分离。

  

## 11. 总结

  

核心关系如下：

  

```text

Workspace

是 nanobot 的状态与上下文根目录

包含 sessions、memory、skills、bootstrap files

  

SessionManager(workspace)

管理 workspace/sessions

负责加载、缓存、保存、删除 Session

  

Session(key)

表示一条 channel/chat 维度的聊天线程

key 通常是 channel:chat_id

保存 messages、metadata、last_consolidated

  

AgentLoop

每次 turn 根据 InboundMessage 找 Session

读取历史并构造上下文

调用 AgentRunner

将结果写回 Session

  

ContextBuilder

不只读取 Session

还合并 workspace bootstrap、memory、skills、runtime context

  

WebUI workspace_scope

让某个 WebUI session 的有效项目目录可以不同于存储 workspace

```

  

一句话概括：

  

```text

workspace 是 nanobot 的状态与上下文根；

SessionManager 是绑定到某个 workspace 的会话仓库；

Session 是某个 channel/chat 的持久化线程；

AgentLoop 在每个 turn 中把 session 历史、workspace 信息、memory、skills 和当前消息组装起来交给 AgentRunner，

并在结束后通过 SessionManager 保存现场。

```