---
title: langchain models 实战 流式输出（Streaming）
tag: langchain
date: 2026-03-30 22:12:00
---

# ⚡ 流式输出（Streaming）—— 让回复"实时涌现"

> **所在目录**：`chunk/stream_mode.ipynb`  
> **核心依赖**：`langchain_openai`、`langchain.agents`

---

## 前言

ChatGPT 的回复是一个字一个字涌现出来的，而不是等全部生成完再展示——这就是**流式输出（Streaming）**。它让用户感受到"AI 正在实时思考"，而不是对着空屏幕等待。

| 对比维度 | 普通 `invoke` | 流式 `stream` |
|----------|--------------|--------------|
| 用户体验 | 长时等待 → 突然全文出现 | 内容实时涌现 |
| 首字节时间 | 等待全部生成 | 几乎即时 |
| 适用场景 | 批处理、后台任务 | 对话界面、实时展示 |

---

## 基础流式：`llm.stream()`

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

for chunk in llm.stream("给我讲一个关于程序员的笑话"):
    print(chunk.content, end="", flush=True)
```

返回的每个 `chunk` 是 `AIMessageChunk`，`.content` 是这次输出的文本片段。

---

## Agent 流式：`astream_events()`

Agent 执行涉及多个阶段（思考 → 调工具 → 再思考），用 `astream_events()` 可以感知每个阶段：

```python
async for event in agent.astream_events(input_data, version="v2"):
    event_type = event["event"]

    if event_type == "on_chat_model_stream":
        print(event["data"]["chunk"].content, end="", flush=True)

    elif event_type == "on_tool_start":
        print(f"\n🔧 调用工具：{event['name']}，参数：{event['data'].get('input')}")

    elif event_type == "on_tool_end":
        print(f"\n✅ 工具返回：{event['data'].get('output')}")
```

### 常见事件类型

| 事件名 | 触发时机 |
|--------|---------|
| `on_chat_model_stream` | 模型输出每个 chunk |
| `on_tool_start` | 工具开始执行 |
| `on_tool_end` | 工具执行完毕 |
| `on_chain_start/end` | 链的开始/结束 |

---

## 在 FastAPI 中使用流式

```python
from fastapi.responses import StreamingResponse

@app.get("/chat")
async def chat(question: str):
    async def generate():
        async for chunk in llm.astream(question):
            yield chunk.content
    return StreamingResponse(generate(), media_type="text/plain")
```

---

## 小结

1. **简单场景**：`llm.stream()` 即可，逐 chunk 打印
2. **Agent 场景**：`agent.astream_events()` 获取完整执行事件流
3. **Web 服务**：结合 `StreamingResponse` 实现真正的实时流

> 🚀 **下一步**：学习 `middleware` 模块，了解如何在 Agent 执行链路上添加限流、重试、审批、隐私保护等横切能力。
