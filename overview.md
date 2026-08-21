# LangChain 概览

> 原文:[LangChain Overview](https://docs.langchain.com/oss/python/langchain/overview)

> LangChain 提供了 `create_agent`:一个极简、高度可配置的 Agent 运行框架(harness)。你可以从模型、工具、提示词和中间件中,精确组合出你的使用场景所需的 Agent。

**Agent = 模型 + 框架(Harness)。** LangChain 提供了 `create_agent`:一个极简、高度可配置的 harness。Harness 是围绕模型循环(model loop)的一切:提示词、工具,以及任何塑造行为的中间件(middleware)。从基础原语(primitives)出发,精确组合出你的使用场景所需的内容。支持 [OpenAI、Anthropic、Google 等](https://docs.langchain.com/oss/python/integrations/providers/overview)。

> **提示:LangChain vs. LangGraph vs. Deep Agents**
>
> - 如果你想要一个"开箱即用"(batteries-included)的 Agent,具备自动上下文压缩、虚拟文件系统、子 Agent 派生等特性,请从 [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview/) 开始。Deep Agents 正是构建在 LangChain [agents](https://docs.langchain.com/oss/python/langchain/agents/) 之上,你也可以直接使用后者。
> - 如果你需要一个高度可定制的运行框架,并能轻松适配你的使用场景和数据,请使用 [LangChain](https://docs.langchain.com/oss/python/langchain/agents)(`create_agent`)。
> - 如果你有将确定性工作流与 Agent 式(agentic)工作流相结合的高级需求,请使用我们的底层编排框架 [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview)。
> - 使用 [LangSmith](https://docs.langchain.com/langsmith/observability) 对以上任一框架构建的 Agent 进行追踪(trace)、调试和评估。可参考[追踪快速入门](https://docs.langchain.com/langsmith/trace-with-langchain)完成配置。我们还建议同时配置 [LangSmith Engine](https://docs.langchain.com/langsmith/engine),它可以监控你的追踪数据、检测问题并提出修复建议。

## 创建 Agent

以下示例演示如何创建一个带自定义工具的简单 LangChain Agent:

### OpenAI

```python
# pip install -qU langchain "langchain[openai]"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### Google Gemini

```python
# pip install -qU langchain "langchain[google-genai]"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="google_genai:gemini-2.5-flash-lite",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### Claude (Anthropic)

```python
# pip install -qU langchain "langchain[anthropic]"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### OpenRouter

```python
# pip install -qU langchain langchain-openrouter
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="openrouter:anthropic/claude-sonnet-4-6",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### Fireworks

```python
# pip install -qU langchain langchain-fireworks
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="fireworks:accounts/fireworks/models/qwen3p5-397b-a17b",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### Baseten

```python
# pip install -qU langchain langchain-baseten
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="baseten:zai-org/GLM-5.2",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### Ollama

```python
# pip install -qU langchain langchain-ollama
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="ollama:devstral-2",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### Azure

```python
# pip install -qU langchain "langchain[openai]"
import os
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

model = init_chat_model(
    "azure_openai:gpt-5.5",
    azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT_NAME"],
)
agent = create_agent(
    model=model,
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### AWS Bedrock

```python
# pip install -qU langchain langchain-aws
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

# 美国跨区域推理配置文件;如需全球路由,请使用 global.anthropic.claude-sonnet-4-6。
agent = create_agent(
    model="bedrock_converse:us.anthropic.claude-sonnet-4-6",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

### HuggingFace

```python
# pip install -qU langchain "langchain[huggingface]"
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="huggingface:microsoft/Phi-3-mini-4k-instruct",
    tools=[get_weather],
    system_prompt="You are a helpful assistant",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]}
)
print(result["messages"][-1].content_blocks)
```

请参考[安装说明](https://docs.langchain.com/oss/python/langchain/install)和[快速入门指南](https://docs.langchain.com/oss/python/langchain/quickstart),开始使用 LangChain 构建你自己的 Agent 和应用。

> **提示**
>
> 使用 [LangSmith](https://docs.langchain.com/langsmith/observability) 追踪请求、调试 Agent 行为并评估输出。只需设置 `LANGSMITH_TRACING=true` 和你的 API key 即可开始。

## 核心优势

| 特性 | 说明 |
|------|------|
| **统一的模型接口** | 跨提供商使用同一个接口调用对话模型(chat model)、嵌入模型(embeddings)等。以最少的代码改动切换模型,随着需求演进保持应用的可移植性。 |
| **高度可配置的 harness** | 以 `create_agent` 作为极简 harness 起步,通过中间件(middleware)渐进式添加能力。只组合你的使用场景所需的部分——从护栏(guardrails)和重试,到路由和自定义工具策略。 |
| **构建于 LangGraph 之上** | LangChain 的 Agent 构建在 LangGraph 之上,因此可以利用 LangGraph 的持久化执行(durable execution)、人机协同(human-in-the-loop)支持、持久化(persistence)等能力。 |
| **使用 LangSmith 调试** | 在一处检查追踪(trace)、工具调用、状态转换和延迟。定位失败模式、评估质量,并基于执行数据改进 Agent 行为。 |
