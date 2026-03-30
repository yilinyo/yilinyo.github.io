---
title: langchain models 实战 结构化输出
tag: langchain
date: 2026-03-30 22:12:00
---

# 📦 结构化输出 —— 让 AI 说"机器能懂的语言"

> **所在目录**：`struct_output/struct_op_test.ipynb`  
> **核心依赖**：`langchain_core`、`pydantic`

---

## 前言

大模型的默认输出是自由格式的文本，但在实际开发中，我们经常需要模型返回**固定格式的数据**：

- 从简历中提取姓名、邮箱、工作经历
- 分析情感，返回 `{"sentiment": "positive", "score": 0.95}`
- 生成结构化的商品信息，直接存入数据库

这就是**结构化输出（Structured Output）**的价值：告诉模型"请用这个格式回答我"，然后直接拿到 Python 对象。

---

## 核心方法：`with_structured_output()`

LangChain 提供了 `with_structured_output()` 方法，将大模型"包装"成一个结构化输出器：

```python
llm_structured = llm.with_structured_output(YourSchema)
result = llm_structured.invoke("...")
# result 是 YourSchema 的实例，不是字符串！
```

---

## 三种定义结构的方式

### 方式一：Pydantic 模型（推荐）✅

Pydantic 是 Python 生态中最流行的数据验证库，用它定义结构清晰、类型安全、自带校验：

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

class PersonInfo(BaseModel):
    """从文本中提取人物信息"""
    name: str = Field(description="人物姓名")
    age: int = Field(description="年龄")
    occupation: str = Field(description="职业")
    skills: list[str] = Field(description="技能列表", default_factory=list)

llm = ChatOpenAI(model="gpt-4o-mini")
extractor = llm.with_structured_output(PersonInfo)

text = "张伟，28岁，是一名后端工程师，熟悉 Python 和 Go 语言。"
result = extractor.invoke(text)

print(type(result))       # <class 'PersonInfo'>
print(result.name)        # 张伟
print(result.age)         # 28
print(result.skills)      # ['Python', 'Go']
```

> 💡 **Field(description=...)** 非常重要——这个描述会被发送给模型，帮助它理解每个字段的含义。

---

### 方式二：TypedDict

如果不想引入 Pydantic，可以用 `TypedDict`：

```python
from typing import TypedDict

class ProductInfo(TypedDict):
    product_name: str
    price: float
    category: str

extractor = llm.with_structured_output(ProductInfo)
result = extractor.invoke("iPhone 15 Pro，价格 7999 元，属于智能手机类别")
# result 是普通字典
print(result["product_name"])  # iPhone 15 Pro
```

---

### 方式三：JSON Schema

对于更动态的场景，可以直接传入 JSON Schema 字典：

```python
schema = {
    "name": "sentiment_analysis",
    "description": "分析文本情感",
    "properties": {
        "sentiment": {
            "type": "string",
            "enum": ["positive", "negative", "neutral"]
        },
        "confidence": {
            "type": "number",
            "description": "置信度 0-1"
        }
    },
    "required": ["sentiment", "confidence"]
}

analyzer = llm.with_structured_output(schema)
result = analyzer.invoke("这个产品真的太棒了！")
# {'sentiment': 'positive', 'confidence': 0.98}
```

---

## 进阶：嵌套结构

Pydantic 支持嵌套模型，可以表达复杂的层级结构：

```python
from pydantic import BaseModel

class Address(BaseModel):
    city: str
    province: str

class Company(BaseModel):
    name: str
    headquarters: Address
    employee_count: int
    founded_year: int

extractor = llm.with_structured_output(Company)
result = extractor.invoke(
    "字节跳动成立于2012年，总部位于北京市海淀区，员工约11万人。"
)

print(result.name)                     # 字节跳动
print(result.headquarters.city)        # 北京
print(result.founded_year)             # 2012
```

---

## 底层原理

`with_structured_output()` 在底层使用了两种技术之一（取决于模型支持情况）：

1. **Function Calling / Tool Calling**：让模型"调用"一个虚拟函数，参数就是结构化数据（OpenAI、Anthropic 等主流模型支持）
2. **JSON Mode**：强制模型只输出合法 JSON，再用 Schema 校验解析

```
用户输入
  │
  ▼
LLM（启用 Function Calling）
  │
  ▼
JSON 字符串（模型输出）
  │
  ▼
Pydantic/TypedDict 校验与解析
  │
  ▼
Python 对象 ✅
```

---

## 使用场景对比

| 场景 | 推荐方案 |
|------|----------|
| 信息提取（简历、合同、新闻） | Pydantic 模型 |
| 情感分析、分类任务 | Pydantic 或 TypedDict |
| 动态 Schema（运行时决定结构） | JSON Schema 字典 |
| 快速原型 | TypedDict |

---

## 小结

结构化输出是 LangChain 最实用的功能之一，它实现了从**"AI 生成文本"**到**"AI 生成数据"**的跨越：

1. 定义好 Pydantic 模型，每个字段写清楚 `description`
2. 用 `llm.with_structured_output(Model)` 包装模型
3. 直接 `.invoke()` 得到干净的 Python 对象

> 🚀 **下一步**：学习 `tools` 模块，了解如何为 Agent 配备各种工具，让它不仅能"说话"，还能"做事"。
