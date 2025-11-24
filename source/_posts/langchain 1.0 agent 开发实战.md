---
title: langchain 1.0 agent 开发实战
tags: [LangChain,Agent]
date: 2025-11-24 11:48:00
---
# langchain 1.0 agent 开发实战

## 调取大模型



```python
import os

import dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI

dotenv.load_dotenv() #加载当前目录下的 .env 文件
os.environ['OPENAI_API_KEY'] = os.getenv("OPENAI_API_KEY2")

os.environ['OPENAI_BASE_URL'] = os.getenv("OPENAI_BASE_URL2")
# 创建大模型实例
llm = ChatOpenAI(model="gpt-4o-mini") # 默认使用 gpt-3.5-turbo
# 使用提示词调用

prompt = ChatPromptTemplate.from_messages([
("system", "你是世界级的技术文档编写者"),
("user", "{input}") # {input}为变量
])
# 我们可以把prompt和具体llm的调用和在一起。
chain = prompt | llm
message = chain.invoke({"input": "大模型中的LangChain是什么?"})
print(message)

```

    content='LangChain 是一个开源框架，旨在帮助开发者构建与大语言模型（LLMs）相关的应用程序。它提供了一系列工具和组件，使得用户能够更容易地集成和使用语言模型，比如 OpenAI 的 GPT 系列、Anthropic 的 Claude 等，从而简化自然语言处理（NLP）应用的开发过程。\n\nLangChain 的核心理念是将语言模型与其他数据源、工具和上下文结合起来，使其能够执行更复杂的任务。其主要组件包括：\n\n1. **链（Chains）**：将多个操作串联在一起，以实现复杂的逻辑。例如，可以将文本生成与数据检索结合起来，先从数据库中检索信息，然后再使用语言模型生成总结。\n\n2. **代理（Agents）**：通过实时决策，使得应用程序能够根据用户的输入动态调整其行为。它可以在多个操作（如API调用、数据库查询等）之间进行选择，以提供最佳响应。\n\n3. **内存（Memory）**：允许应用程序记住用户的上下文和会话历史，以便在后续的交互中提供更个性化和相关的回答。 \n\n4. **数据连接（Data Connectors）**：方便与外部数据源（如文档、数据库、API等）进行交互，使得大型语言模型能够基于最新和相关的数据提供信息。\n\nLangChain 的设计使得开发者能够快速构建从聊天机器人到智能助手等多种基于语言模型的应用，同时也提供了灵活性以满足不同场景的需求。通过组合这些组件，开发者可以创建出具有更高实用性和智能的应用。' additional_kwargs={'refusal': None} response_metadata={'token_usage': {'completion_tokens': 363, 'prompt_tokens': 29, 'total_tokens': 392, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_efad92c60b', 'id': 'chatcmpl-CfSzQigYp8TeDSNrUVdfEVFOyj30H', 'finish_reason': 'stop', 'logprobs': None} id='lc_run--bb7594a8-1658-4da8-b14c-6a131d2c1d8a-0' usage_metadata={'input_tokens': 29, 'output_tokens': 363, 'total_tokens': 392, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}


### 静态模型使用
Static models are configured once when creating the agent and remain unchanged throughout execution. This is the most common and straightforward approach.




```python
from langchain.agents import create_agent

agent = create_agent(
    "gpt-5",
    tools=None
)

```

直接使用模型提供商提供的包
provider:model


```python
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI


model = ChatOpenAI(
    model="gpt-4o-mini",
)
agent = create_agent(model)
result = agent.invoke({
    "messages": [
        {"role":"user",
         "content": "what is the weather in Beijing?",
         }
    ],
})
print(result)

```

    {'messages': [HumanMessage(content='what is the weather in Beijing?', additional_kwargs={}, response_metadata={}, id='3d1cb7b0-74d6-403d-b262-c1440896984b'), AIMessage(content="I don't have real-time data access to provide current weather conditions. To find the latest weather information for Beijing, I recommend checking a reliable weather website or using a weather app.", additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 36, 'prompt_tokens': 14, 'total_tokens': 50, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_efad92c60b', 'id': 'chatcmpl-CfSziNl4bwdbUE2fWR3PPxJ0igsdJ', 'finish_reason': 'stop', 'logprobs': None}, id='lc_run--d1053709-33ce-46da-872a-ccc6e8f09ab4-0', usage_metadata={'input_tokens': 14, 'output_tokens': 36, 'total_tokens': 50, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]}


一种统一的方式


```python
from langchain.chat_models import init_chat_model
gpt_4o = init_chat_model("gpt-4o",temperature=0.5)
res = gpt_4o.invoke("hello")
print(res)
```

    content='Hello! How can I assist you today?' additional_kwargs={'refusal': None} response_metadata={'token_usage': {'completion_tokens': 10, 'prompt_tokens': 8, 'total_tokens': 18, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-2024-11-20', 'system_fingerprint': 'fp_b54fe76834', 'id': 'chatcmpl-CfSXlu8P0yIQ4rLb5b27DsXmt3PyO', 'finish_reason': 'stop', 'logprobs': None} id='lc_run--6121df78-28d2-497d-879f-0c7dfd5bc131-0' usage_metadata={'input_tokens': 8, 'output_tokens': 10, 'total_tokens': 18, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}


### 动态模型使用
Dynamic models are selected at runtime based on the current state and context. This enables sophisticated routing logic and cost optimization.


```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse

basic_model = ChatOpenAI(model="gpt-3.5-turbo")
advanced_model = ChatOpenAI(model="gpt-4o")

@wrap_model_call
def dynamic_model_selection(request: ModelRequest, handler) -> ModelResponse:
    """Choose model based on conversation complexity."""
    message_count = len(request.state["messages"])

    if message_count > 0:
        # Use an advanced model for longer conversations
        model = advanced_model
    else:
        model = basic_model

    return handler(request.override(model=model))

agent = create_agent(
    model=basic_model,  # Default model
    tools=None,
    middleware=[dynamic_model_selection]
)


result = agent.invoke( {"messages": [
        {"role":"user",
         "content": "你具体是openai 那个模型 ",
         },

    ],
})

print(result["messages"][-1])

```

    content='我是 OpenAI 的 GPT-4 模型。具体版本是基于 GPT-4 架构的一个实例，我的知识截截止到 2023 年 10 月，能帮助你解决问题、提供建议或回答你的问题！如果你有任何需要，欢迎随时问我！' additional_kwargs={'refusal': None} response_metadata={'token_usage': {'completion_tokens': 64, 'prompt_tokens': 16, 'total_tokens': 80, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-2024-11-20', 'system_fingerprint': 'fp_b54fe76834', 'id': 'chatcmpl-CfT0CmGTUkaVvQgAdN5LmXPHk3pwr', 'finish_reason': 'stop', 'logprobs': None} id='lc_run--bddfac84-3715-44c3-b7bd-b96987355c1e-0' usage_metadata={'input_tokens': 16, 'output_tokens': 64, 'total_tokens': 80, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}


## 创建一个可以调用函数的agent智能体


```python
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """return the weather for a city"""
    print(f"getting weather for {city}")
    return f"Sunny ,30 degree"

agent = create_agent(
    model="gpt-4o-mini",
    tools=[get_weather],
    system_prompt="You are a helpful research assistant"
)

result = agent.invoke({
    "messages": [
        {"role":"user",
         "content": "what is the weather in Beijing?",
         }
    ],
})
print(result["messages"][-1])
```

    getting weather for Beijing
    content='The weather in Beijing is sunny with a temperature of 30 degrees Celsius.' additional_kwargs={'refusal': None} response_metadata={'token_usage': {'completion_tokens': 17, 'prompt_tokens': 85, 'total_tokens': 102, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_efad92c60b', 'id': 'chatcmpl-CfT0MQq6Qdmqk1lXU12to4jmyyFQB', 'finish_reason': 'stop', 'logprobs': None} id='lc_run--4299b8d5-423f-4da0-b29e-11eb8eeef565-0' usage_metadata={'input_tokens': 85, 'output_tokens': 17, 'total_tokens': 102, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}


## middleware 中间件的使用
用于智能体的高度个性化

 HumanInTheLoopMiddleware 函数调用询问
```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents.middleware import HumanInTheLoopMiddleware


def get_weather(city: str) -> str:
    """return the weather for a city"""
    print(f"getting weather for {city}")
    return f"Sunny ,30 degree"

agent = create_agent(
    model="gpt-4o-mini",
    tools=[get_weather],
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "get_weather": {
                    "allowed_decisions":["approve","edit","reject"]
                }
            }
        )
    ],
    checkpointer=InMemorySaver(),
    system_prompt="You are a helpful research assistant"
)

config = {"configurable" :{"thread_id":"some_other_id"}}
result = agent.invoke({
    "messages": [
        {"role":"user",
         "content": "what is the weather in Beijing?",
         }
    ],
},config=config)
#print(result["__interrupt__"])

from langgraph.types import Command
result = agent.invoke(
    Command(resume={"decisions":[{"type":"approve"}]}),
    config=config
)

print(result['messages'][-1])
```

    getting weather for Beijing
    content='The weather in Beijing is sunny with a temperature of 30 degrees Celsius.' additional_kwargs={'refusal': None} response_metadata={'token_usage': {'completion_tokens': 17, 'prompt_tokens': 85, 'total_tokens': 102, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-mini-2024-07-18', 'system_fingerprint': 'fp_efad92c60b', 'id': 'chatcmpl-CfSIkuYVg7TxtNZJia90lfZUJVk6p', 'finish_reason': 'stop', 'logprobs': None} id='lc_run--5d32a14d-2be2-418b-827e-2b27e0406a8a-0' usage_metadata={'input_tokens': 85, 'output_tokens': 17, 'total_tokens': 102, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}



函数调用 异常处理中间件
```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_tool_call
from langchain.messages import ToolMessage


def search(query: str) -> str:
    """Search for information."""
    print(f"searching for {query}")
    a = 10
    b = 0
    print(a / b)
    return f"Results for: {query}"

@wrap_tool_call
def handle_tool_errors(request, handler):
    """Handle tool execution errors with custom messages."""
    try:
        return handler(request)
    except Exception as e:
        # Return a custom error message to the model
        return ToolMessage(
            content=f"Tool error: Please check your input and try again. ({str(e)})",
            tool_call_id=request.tool_call["id"]
        )

agent = create_agent(
    model="gpt-4o",
    tools=[search, get_weather],
    middleware=[handle_tool_errors]
)

result = agent.invoke({
    "messages": [
        {"role":"user",
         "content": "调用search 函数搜索langchain",
         }
    ],
})
print(result["messages"])
```

    searching for langchain
    searching for langchain
    [HumanMessage(content='调用search 函数搜索langchain', additional_kwargs={}, response_metadata={}, id='322da991-4e81-4528-8271-cd80820e5efa'), AIMessage(content='', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 15, 'prompt_tokens': 69, 'total_tokens': 84, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-2024-11-20', 'system_fingerprint': 'fp_b54fe76834', 'id': 'chatcmpl-CfSt6ihYG2dQiIeR506Pe7t7WHQaM', 'finish_reason': 'tool_calls', 'logprobs': None}, id='lc_run--08d81b55-c001-46a3-b0f9-225963f15d6a-0', tool_calls=[{'name': 'search', 'args': {'query': 'langchain'}, 'id': 'call_VLZKXRNtupHGy98ekl9hFeeF', 'type': 'tool_call'}], usage_metadata={'input_tokens': 69, 'output_tokens': 15, 'total_tokens': 84, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}), ToolMessage(content='Tool error: Please check your input and try again. (division by zero)', id='d8c52ac4-0a4e-4951-bab1-d8c043704eb4', tool_call_id='call_VLZKXRNtupHGy98ekl9hFeeF'), AIMessage(content='It seems there was an issue when trying to search for "langchain." Let me try again.', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 37, 'prompt_tokens': 106, 'total_tokens': 143, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-2024-11-20', 'system_fingerprint': 'fp_b54fe76834', 'id': 'chatcmpl-CfSt7mSmbAH41fjknxZhOMfDnCKRR', 'finish_reason': 'tool_calls', 'logprobs': None}, id='lc_run--cc9586f4-dd26-41b7-ae90-327cbe206409-0', tool_calls=[{'name': 'search', 'args': {'query': 'langchain'}, 'id': 'call_aoYrhP3MVrt0AHRUVtYr6bG7', 'type': 'tool_call'}], usage_metadata={'input_tokens': 106, 'output_tokens': 37, 'total_tokens': 143, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}}), ToolMessage(content='Tool error: Please check your input and try again. (division by zero)', id='0797ce72-23f6-4825-9bbc-3efa4cbf4f41', tool_call_id='call_aoYrhP3MVrt0AHRUVtYr6bG7'), AIMessage(content='There seems to be a persistent issue with the search function when trying to look up "langchain". Would you like me to help with anything else or try an alternative query?', additional_kwargs={'refusal': None}, response_metadata={'token_usage': {'completion_tokens': 37, 'prompt_tokens': 167, 'total_tokens': 204, 'completion_tokens_details': {'accepted_prediction_tokens': 0, 'audio_tokens': 0, 'reasoning_tokens': 0, 'rejected_prediction_tokens': 0}, 'prompt_tokens_details': {'audio_tokens': 0, 'cached_tokens': 0}}, 'model_provider': 'openai', 'model_name': 'gpt-4o-2024-11-20', 'system_fingerprint': 'fp_b54fe76834', 'id': 'chatcmpl-CfSt96oY70Bv6BLKjvQ97MeqTnKIm', 'finish_reason': 'stop', 'logprobs': None}, id='lc_run--c6a8fcf9-b8a3-4f40-b6cd-8575f08dfdd0-0', usage_metadata={'input_tokens': 167, 'output_tokens': 37, 'total_tokens': 204, 'input_token_details': {'audio': 0, 'cache_read': 0}, 'output_token_details': {'audio': 0, 'reasoning': 0}})]

