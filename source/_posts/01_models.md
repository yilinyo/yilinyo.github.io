---
title: langchain models 实战 模型初始化与调用
tag: langchain
date: 2026-03-30 22:12:00
---

# 🤖 模型初始化与调用 —— LangChain 的第一步

> **所在目录**：`models/initmodel_demo.ipynb`  
> **核心依赖**：`langchain_openai`、`langchain_anthropic`、`langchain_core`

---

## 前言

在使用 LangChain 构建 AI 应用之前，第一步永远是"选一个大模型，把它跑起来"。`models` 模块就是干这件事的——它演示了如何初始化不同厂商的大语言模型（LLM），以及如何通过统一的接口发起调用。

LangChain 最大的价值之一就是**屏蔽了底层模型差异**，无论你用的是 OpenAI、Anthropic 还是其他厂商，对上层代码来说调用姿势几乎一模一样。

---

## 核心概念

### 1. ChatModel 与 LLM 的区别

LangChain 将模型分为两类：

| 类型 | 说明 | 典型代表 |
|------|------|----------|
| `BaseChatModel` | 以"多轮对话消息"为输入/输出单元，适合对话场景 | `ChatOpenAI`、`ChatAnthropic` |
| `BaseLLM` | 以纯文本字符串为输入/输出，偏底层 | 已逐渐被 ChatModel 取代 |

现代大模型应用几乎都使用 **ChatModel**。

---

## 关键组件详解

### ChatOpenAI —— 接入 OpenAI 系列模型

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-4o-mini",      # 模型名称
    temperature=0.7,           # 创造性：0=精确，1=自由发挥
    max_tokens=1024,           # 最大输出 token 数
)

response = llm.invoke("什么是大模型？")
print(response.content)
```

**参数说明：**

| 参数 | 作用 | 默认值 |
|------|------|--------|
| `model` | 选择使用哪个模型版本 | `gpt-3.5-turbo` |
| `temperature` | 控制输出随机性 | `0.7` |
| `max_tokens` | 限制输出长度（节省费用） | None |
| `base_url` | 自定义 API 地址（代理/中转） | OpenAI 官方地址 |
| `api_key` | 认证密钥 | 读取环境变量 |

---

### ChatAnthropic —— 接入 Claude 系列模型

```python
from langchain_anthropic import ChatAnthropic

claude = ChatAnthropic(
    model="claude-3-5-sonnet-20241022",
    max_tokens=1024,
)

response = claude.invoke("用一句话解释量子纠缠")
print(response.content)
```

> 💡 **统一接口的威力**：从 `ChatOpenAI` 切换到 `ChatAnthropic`，业务代码零改动，只需换一行模型初始化。

---

### 环境变量管理 —— dotenv 最佳实践

敏感的 API Key 不应硬编码在代码里，推荐使用 `.env` 文件 + `python-dotenv` 管理：

```python
import dotenv
import os

dotenv.load_dotenv()  # 自动读取当前目录下的 .env 文件

os.environ['OPENAI_API_KEY'] = os.getenv("OPENAI_API_KEY")
os.environ['OPENAI_BASE_URL'] = os.getenv("OPENAI_BASE_URL")  # 若使用代理
```

`.env` 文件示例：
```
OPENAI_API_KEY=sk-xxxxxxxxxx
OPENAI_BASE_URL=https://your-proxy.com/v1
```

---

### 调用方式：invoke vs stream

```python
# 同步调用（等待完整响应）
response = llm.invoke("你好，介绍一下自己")
print(response.content)

# 流式调用（逐 token 输出，体验更流畅）
for chunk in llm.stream("讲一个故事"):
    print(chunk.content, end="", flush=True)
```

---

## 返回值结构

`llm.invoke()` 返回的是一个 `AIMessage` 对象，而不是纯字符串：

```python
response = llm.invoke("你好")

response.content        # 模型回复的文本内容 ✅ 最常用
response.response_metadata  # token 使用量、停止原因等元数据
response.id             # 本次请求的唯一 ID
```

---

## 小结

| 知识点 | 核心要点 |
|--------|----------|
| 模型初始化 | 通过 `ChatOpenAI(model=...)` 等方式创建模型实例 |
| 统一接口 | 不同厂商模型的 `.invoke()` 调用方式完全一致 |
| 环境变量 | 用 `dotenv` 管理 API Key，避免硬编码 |
| 返回值 | `AIMessage` 对象，通过 `.content` 获取文本 |

> 🚀 **下一步**：了解如何构造不同类型的消息（HumanMessage、SystemMessage 等），进入 `message` 模块。
