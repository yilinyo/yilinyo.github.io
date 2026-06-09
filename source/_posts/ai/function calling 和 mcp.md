---
title: function calling 和 mcp 的关系
tag: 大模型应用
date: 2026-04-01 20:12:00
---
## LLM 应用现状

大语言模型已经为我们提供了很好的自然语言对话功能，但使用deepseek或者chatgpt等等大模型我们或多或少都有这种感觉，好像他只能在言语上为我们解决一些问题，不能在实际生活帮我做一些事。比如今天我想要通过一句话让ai订一张上海到北京的机票（或者自动完成航班选择发送到手机上等我支付）。
然而单纯的大模型仅仅是一个空有一身知识的决策高手，在语言能力上很强，可是缺乏动手能力啊。
但是，既然他是一个决策高手，为何不发挥这一特点呢？我们是否可以给他配一些贴身随从，然后大模型将决策出来的任务吩咐给这些随从做不就行了吗。

## 什么是FunctionCall？
Function Calling应运而生。它就像为大语言模型配备了“手脚”，让它们能够调用外部工具 (Tools，早先叫Function Calling，后来改成了Tools，一个意思)，执行实际操作，与现实世界互动。
比如：我想查看 某个旅游城市的天气？以此作为参考决定去不去旅游。此时我们定义一个查询天气的function，定义好传入的参数 以及内部逻辑 和返回结果，以及对这个函数的功能的语义描述

```python
tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get current temperature for a given location.",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "City and country e.g. Bogotá, Colombia"
                }
            },
            "required": [
                "location"
            ],
            "additionalProperties": False
        },
        "strict": True
    }
}]
```

然后实现这个工具的实际逻辑：

```python
def get_weather(location: str) -> str:
    """
    根据城市名获取天气信息
    内部通过地理编码 API 将城市名转换为经纬度，再查询气象数据
    """
    try:
        # 1. 地理编码：城市名 → 经纬度
        geo = requests.get(
            f"https://geocoding-api.open-meteo.com/v1/search?name={location}&count=1"
        ).json()
        lat = geo['results'][0]['latitude']
        lon = geo['results'][0]['longitude']

        # 2. 查询天气
        response = requests.get(
            f"https://api.open-meteo.com/v1/forecast?"
            f"latitude={lat}&longitude={lon}&current=temperature_2m,wind_speed_10m"
        )
        data = response.json()
        temp = data['current']['temperature_2m']
        return f"{location} 当前温度: {temp}°C"
    except Exception as e:
        return f"获取天气信息时出错: {str(e)}"
```

### 调用流程

关键点：**模型本身不执行函数**。它只是"发号施令"，真正干活的是你的后端服务。

```
用户: "上海今天天气怎么样？"
    │
    ▼
┌─────────────────────────────────────────────────┐
│  LLM 接收 prompt + tools schema                  │
│  语义匹配 → 决定调用 get_weather                    │
│  提取参数 → {"location": "Shanghai, China"}       │
│  输出 tool_call JSON（不是最终回答）                 │
└────────────────────┬────────────────────────────┘
                     │ tool_call
                     ▼
┌─────────────────────────────────────────────────┐
│  你的后端服务执行 get_weather("Shanghai, China")   │
│  拿到结果: "Shanghai, China 当前温度: 24°C"        │
└────────────────────┬────────────────────────────┘
                     │ tool_result
                     ▼
┌─────────────────────────────────────────────────┐
│  LLM 拿到工具返回结果                              │
│  整合生成自然语言回答:                               │
│  "上海现在 24°C，天气不错，适合出门~"                  │
└─────────────────────────────────────────────────┘
```

![image.png](https://s2.loli.net/2025/12/02/uUMLShz7yopGiF3.png)

核心原理拆解：

> **函数描述**：向模型提供可用函数的详细描述 — 函数名、参数类型、功能说明。模型靠这些描述来"理解"工具的用途。
> 
> **意图识别**：模型解析用户输入，通过与工具描述的**语义相似度匹配**判断是否需要调用工具、调用哪个。
> 
> **参数生成**：模型从用户输入中提取信息，生成结构化的 JSON 参数。
> 
> **函数执行**：你的后端服务接收 tool_call 请求，执行实际函数，拿到结果。
> 
> **结果整合**：结果返回给模型，模型将其整合为自然语言呈现给用户。

所以描述写得好不好，直接决定了模型能不能正确匹配工具。`"description"` 字段就是工具的"简历" — 写得越清晰，模型越容易选对。

### 两种实现方式

**方式一：原生 Tool Calling**

现在主流大模型（GPT-4o、Claude、Gemini、DeepSeek-V3 等）都原生支持 Tool Calling 机制。通过 API 传入 tools schema，模型会在 response 中返回结构化的 `tool_calls` 列表。也可以通过 LangChain 等框架进行开发：
[可以参考此篇博客调用工具方法](https://yilinyo.github.io/2025/11/24/langchain%201.0%20agent%20开发实战/#创建一个可以调用函数的agent智能体)

**方式二：Prompt 黑魔法**

有些模型不支持原生 Tool Calling（如 GPT-3.5 早期版本、Embedding 模型、LLaMA 3 8B/70B base 版本等）。这种情况下可以用 **prompt engineering** 硬搞 — 把函数定义、参数格式、返回规范全塞进 system prompt，让模型"假装"自己会调用工具，输出结构化 JSON，然后你的后端去解析执行。

> 💡 其实方式一的本质也是方式二 — 只不过模型经过了专门的微调训练,使其在遵循工具调用格式上更加稳定可靠。记住：**大模型与外部系统的交互本质都是结构化文本。**

---

## MCP：工具调用的"USB-C 接口"

**Model Context Protocol（模型上下文协议）** — 由 Anthropic 提出，定义了 AI 应用如何与外部工具服务进行标准化交互的协议。

类比一下：Function Calling 相当于"模型知道自己可以使用工具"，而 MCP 则定义了"工具插头长什么样、怎么插、数据怎么流"。它是工具调用世界的 **USB-C** — 一个统一的接口标准。

### 架构三角：Host / Client / Server

MCP 官方定义了三个核心角色：

| 角色 | 职责 | 类比 |
|------|------|------|
| **MCP Host** | AI 应用本体，协调管理一个或多个 Client | 操作系统 |
| **MCP Client** | 与 MCP Server 保持连接，获取上下文 | USB 控制器 |
| **MCP Server** | 对外暴露工具/资源/提示词的服务程序 | USB 外设 |

我们日常使用的 AI 智能体（如 Cursor、Claude Desktop）就是 MCP Host。当 Host 需要调用 MCP 工具时，内部会创建一个 Client 与对应的 MCP Server 建立连接。

### MCP 的三大原语

MCP Server 不只是暴露工具，它实际上提供三种原语（Primitives）：

| 原语 | 说明 | 控制方 |
|------|------|--------|
| **Tools** | 可被模型调用的函数（如查天气、查数据库） | 模型决定何时调用 |
| **Resources** | 可被应用读取的数据源（如文件内容、数据库记录），类似 GET 接口 | 应用侧控制 |
| **Prompts** | 预定义的提示词模板，用户可以选择使用 | 用户侧控制 |

很多人只关注 Tools，但 Resources 和 Prompts 同样重要 — 它们让 MCP 不仅仅是"工具调用协议"，而是一个完整的**上下文供给协议**。

### 传输层：两种通道

**1. stdio（标准输入/输出）**

适用于 MCP Server 与 Host 在**同一台机器**上的场景。通过 `npx`、`docker` 等方式启动 Server 进程，Host 通过 stdin/stdout 直接通信，低延迟、零网络开销：

```json
{
  "leetcode": {
    "timeout": 60,
    "type": "stdio",
    "command": "npx",
    "args": ["-y", "@jinzcdev/leetcode-mcp-server"]
  }
}
```

**2. Streamable HTTP**

适用于 MCP Server 部署在**远程**的场景。这是 MCP `2025-03-26` 规范引入的新传输方式，取代了早期的 HTTP+SSE 双通道方案，改为**单一 HTTP 端点**，对负载均衡、代理、防火墙等现代基础设施更友好：

```json
{
  "remote_tools": {
    "timeout": 60,
    "type": "streamableHttp",
    "url": "http://127.0.0.1:8000/mcp"
  }
}
```

> ⚡ Streamable HTTP 还引入了 **会话管理**（`Mcp-Session-Id`）和 **OAuth 2.1 授权框架**，使远程 MCP 连接更安全、更健壮。

### 数据层：JSON-RPC 2.0

MCP 的数据层基于 [JSON-RPC 2.0](https://www.jsonrpc.org/) 协议，好处一目了然：

- `method` 可以动态变化，不需要提前定义 REST schema
- 结构统一（`jsonrpc` / `method` / `params` / `result`），LLM 容易理解
- 轻量级，跨语言解析无压力

---

##  Client ↔ Server 交互流程

一次完整的 MCP 会话，走的是这样一条链路：

```
Client                              Server
  │                                    │
  │──── initialize (握手) ────────▶   │
  │◀─── capabilities (能力声明) ────   │
  │                                    │
  │──── notifications/initialized ──▶  │
  │                                    │
  │──── tools/list (工具发现) ────▶    │
  │◀─── tool schemas ────────────────  │
  │                                    │
  │──── tools/call (工具调用) ────▶    │
  │◀─── tool result ─────────────────  │
  │                                    │
  │◀─── notifications/tools/          │
  │     list_changed (工具变更通知) ──  │
  │                                    │
  │──── tools/list (重新发现) ────▶    │
  │     ...                            │
```

### Step 1: 初始化握手

```json
// → Client Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "elicitation": {}
    },
    "clientInfo": {
      "name": "example-client",
      "version": "1.0.0"
    }
  }
}

// ← Server Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": {}
    },
    "serverInfo": {
      "name": "example-server",
      "version": "1.0.0"
    }
  }
}
```

双方在握手中交换 `capabilities` — 声明各自支持的能力（工具、资源等），以及协议版本号。

### Step 2: 连接确认

```json
// → Client Notification（无 id，不需要 response）
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

### Step 3: 工具发现

客户端通过 `tools/list` 原语发现服务端提供了哪些工具：

```json
// → Client Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list"
}

// ← Server Response
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "tools": [
      {
        "name": "calculator_arithmetic",
        "title": "Calculator",
        "description": "Perform mathematical calculations...",
        "inputSchema": {
          "type": "object",
          "properties": {
            "expression": {
              "type": "string",
              "description": "Mathematical expression to evaluate"
            }
          },
          "required": ["expression"]
        }
      },
      {
        "name": "weather_current",
        "title": "Weather Information",
        "description": "Get current weather information for any location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name, address, or coordinates"
            },
            "units": {
              "type": "string",
              "enum": ["metric", "imperial", "kelvin"],
              "default": "metric"
            }
          },
          "required": ["location"]
        }
      }
    ]
  }
}
```

> 🔑 这一步是 MCP 的杀手特性 — **自我发现（Self-Discovery）**。Client 不需要提前知道 Server 有什么工具，连上就能自动获取完整的工具清单和参数 schema。

### Step 4: 工具调用

```json
// → Client Request
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "weather_current",
    "arguments": {
      "location": "San Francisco",
      "units": "imperial"
    }
  }
}

// ← Server Response
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Current weather in San Francisco: 68°F, partly cloudy..."
      }
    ]
  }
}
```

### Step 5: 工具变更通知

Server 可以主动推送通知，告知工具列表发生了变化，Client 随即重新执行 `tools/list`：

```json
// ← Server Notification
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

整个交互流程：
![pasted-image-1775051639336.png|690](https://files.seeusercontent.com/2026/04/01/hGt4/pasted-image-1775051639336.png)

---

## 4 Function Calling vs MCP：不是替代，是分层


这两个概念经常被拿来比较，但它们根本不在同一个层面上：

| 维度 | Function Calling | MCP |
|------|-----------------|-----|
| **解决什么问题** | 让模型知道"我可以调用工具" | 定义"工具怎么发现、怎么连接、怎么调用" |
| **作用层级** | 模型能力层 — 输出结构化 tool_call | 通信协议层 — 规范 Client↔Server 交互 |
| **谁在用** | LLM 本身 | Agent 框架 / AI 应用 |
| **类比** | CPU 的指令集 | USB 协议标准 |

更直觉的理解：

- **没有 MCP 的时代**：你得手动编写 tools 列表硬编码给模型，模型返回 tool_call 后，你的框架（如 LangChain）通过**反射**找到对应的本地函数执行。工具和应用**强耦合**。

- **有了 MCP 之后**：工具列表通过 `tools/list` **动态发现**，工具调用通过 `tools/call` 的 JSON-RPC 消息**远程执行**。工具和应用**彻底解耦** — 你可以即插即用互联网上任何 MCP Server 提供的工具。

```
Before MCP:
  Agent ──硬编码──▶ [tools列表] ──反射调用──▶ 本地函数

After MCP:
  Agent ──tools/list──▶ MCP Server ──tools/call──▶ 任意远程服务
```


functioncall 强调的是 大语言模型 提供 工具调用的能力，能够在输出结构中显示告知agent要进行工具调用
而mcp 是定义 agent 代理如何 和 工具提供服务商 进行交互以及传输协议，如何进行工具调用。也就是说我们通过mcp 的的发现功能去获得以前我们编码写tools 列表 以及 讲通过function calling 返回的tool_call 具体如何去调用的方式。
试想一下，如果没有mcp,诸如langchain等智能体框架拿到函数名后就是会通过反射进行实际函数的执行，而有了mcp,我们就只要通过mcp client 发送一条 tools/call 的jsonrpc 2.0消息即可。

**所以这两个概念不存在替代关系 — 它们共同协作，让 LLM 从"只会说"进化到"能做事"。** Function Calling 是模型的"意愿表达"，MCP 是这个意愿的"执行通道"。