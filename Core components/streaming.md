# Streaming(流式输出)

> 从 Agent 运行中流式获取实时更新

> 原文:[Streaming](https://docs.langchain.com/oss/python/langchain/streaming)

> **提示**
>
> 对于新应用,我们推荐使用[事件流(event streaming)](https://docs.langchain.com/oss/python/langchain/event-streaming)——这是 LangChain v1.3 引入的类型化投影(typed-projection)API。事件流为每个投影(messages、values、tool calls、subgraphs)提供独立的迭代器,让你可以独立消费它们,而不必根据 `stream_mode` 块进行分支处理。

LangChain 实现了一套流式系统来呈现实时更新。

流式输出对于提升基于 LLM 构建的应用的响应能力至关重要。通过渐进式显示输出——即使在完整响应尚未就绪时——流式输出显著改善了用户体验(UX),尤其是在面对 LLM 延迟时。

## 概述

LangChain 的流式系统让你可以把 Agent 运行中的实时反馈呈现给你的应用。

LangChain 流式能做到的事情:

- [**流式 Agent 进度**](#agent-进度agent-progress)——在每个 Agent 步骤后获取状态更新。
- [**流式 LLM token**](#llm-token)——在语言模型 token 生成时进行流式输出。
- [**流式思考/推理 token**](#流式思考--推理-tokenstreaming-thinking--reasoning-tokens)——在模型推理生成时呈现。
- [**流式自定义更新**](#自定义更新custom-updates)——发出用户定义的信号(例如 `"Fetched 10/100 records"`)。
- [**流式多种模式**](#流式多种模式stream-multiple-modes)——从 `updates`(Agent 进度)、`messages`(LLM token + 元数据)或 `custom`(任意用户数据)中选择。

更多端到端示例参见下面的[常见模式](#常见模式common-patterns)部分。

## 支持的流模式(Supported stream modes)

将以下一种或多种流模式以列表形式传给 [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) 或 [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream) 方法:

| 模式 | 说明 |
| ---------- | ------------------- |
| `updates`  | 在每个 Agent 步骤后流式输出状态更新。如果同一步骤中有多个更新(例如运行了多个节点),这些更新会分别流式输出。 |
| `messages` | 从任何调用了 LLM 的图节点中流式输出 `(token, metadata)` 元组。 |
| `custom`   | 使用 stream writer 从图节点内部流式输出自定义数据。 |

## Agent 进度(Agent progress)

要流式获取 Agent 进度,请使用 `stream_mode="updates"` 调用 [`stream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.stream) 或 [`astream`](https://reference.langchain.com/python/langgraph/graphs/#langgraph.graph.state.CompiledStateGraph.astream) 方法。这会在每个 Agent 步骤后发出一个事件。

例如,如果你有一个调用一次工具的 Agent,你应该会看到如下更新:

- **LLM 节点**:带工具调用请求的 [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage)
- **工具节点**:带执行结果的 [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage)
- **LLM 节点**:最终的 AI 响应

通过 `config` 传入 `thread_id`,让对话被检查点化(checkpointed),后续轮次可以恢复同一历史。`thread_id` 与 `stream_mode` 相互独立;你还可以同时传入 `context`,承载工具从 `runtime.context` 读取的每次运行数据。

```python
from langchain.agents import create_agent
from langchain_core.utils.uuid import uuid7
from langgraph.checkpoint.memory import InMemorySaver

def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_weather],
    checkpointer=InMemorySaver()
)
config = {"configurable": {"thread_id": str(uuid7())}}
stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    config=config,
    version="v3",
)
for kind, item in stream.interleave("messages", "tool_calls"):
    if kind == "messages":
        for token in item.text:
            print(token, end="", flush=True)
    elif kind == "tool_calls":
        print(f"\nTool call: {item.tool_name}({item.input})")
        for delta in item.output_deltas:
            print(delta, end="", flush=True)
        print(f"\nTool result: {item.output}")

final_state = stream.output
```

输出示例:

```shell
step: model
content: [{'type': 'tool_call', 'name': 'get_weather', 'args': {'city': 'San Francisco'}, 'id': 'call_9lBtsDbmmobzyA8xc4I4Ctne'}]
step: tools
content: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]
step: model
content: [{'type': 'text', 'text': "San Francisco weather: It's always sunny in San Francisco!\n\n..."}]
```

> **注意**:使用 `thread_id` 持久化对话历史,需要为 Agent 配置一个 [checkpointer](https://docs.langchain.com/oss/python/langchain/long-term-memory)。在 [LangSmith 部署](https://docs.langchain.com/langsmith/deployment)上 checkpointer 会自动配置好;在本地需要显式传入,例如 `create_agent(..., checkpointer=InMemorySaver())`。本页后续代码片段为简洁起见省略了 `thread_id`,但生产环境中你应该传入。

## LLM token(LLM tokens)

要在 LLM 生成 token 时进行流式输出,使用 `stream_mode="messages"`。下面可以看到 Agent 流式输出工具调用和最终响应的效果。

```python
from langchain.agents import create_agent


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"It's always sunny in {city}!"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode="messages",
    version="v2",
):
    if chunk["type"] == "messages":
        token, metadata = chunk["data"]
        print(f"node: {metadata['langgraph_node']}")
        print(f"content: {token.content_blocks}")
        print("\n")
```

输出(节选):

```shell
node: model
content: [{'type': 'tool_call_chunk', 'id': 'call_vbCyBcP8VuneUzyYlSBZZsVa', 'name': 'get_weather', 'args': '', 'index': 0}]

node: model
content: [{'type': 'tool_call_chunk', 'id': None, 'name': None, 'args': '{"', 'index': 0}]

...

node: tools
content: [{'type': 'text', 'text': "It's always sunny in San Francisco!"}]

node: model
content: [{'type': 'text', 'text': 'Here'}]
node: model
content: [{'type': 'text', 'text': "'s"}]
...
```

> **注意**:
> **把 Agent 作为节点包装进父级 `StateGraph`?** [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) 返回的是一个编译后的图,因此把它用作节点会使其成为子图。除非传入 `subgraphs=True`,否则父图上的 `stream_mode="messages"` 不会输出内部 Agent LLM 调用的 token 块。参见[子图输出](https://docs.langchain.com/oss/python/langgraph/streaming#subgraph-outputs)。

## 自定义更新(Custom updates)

要在工具执行期间流式输出更新,可以使用 [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer)。

```python
from langchain.agents import create_agent
from langgraph.config import get_stream_writer


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    writer = get_stream_writer()
    # 流式发出任意数据
    writer(f"Looking up data for city: {city}")
    writer(f"Acquired data for city: {city}")
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[get_weather],
)

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode="custom",
    version="v2",
):
    if chunk["type"] == "custom":
        print(chunk["data"])
```

输出:

```shell
Looking up data for city: San Francisco
Acquired data for city: San Francisco
```

> **注意**:如果你在工具中加入了 [`get_stream_writer`](https://reference.langchain.com/python/langgraph/config/get_stream_writer),该工具就无法在 LangGraph 执行上下文之外被调用。

## 流式多种模式(Stream multiple modes)

把流模式以列表形式传入即可指定多个流模式:`stream_mode=["updates", "custom"]`。

每个流式块都是一个带 `type`、`ns` 和 `data` 键的 `StreamPart` 字典。使用 `chunk["type"]` 判断流模式,使用 `chunk["data"]` 访问载荷。

```python
from langchain.agents import create_agent
from langgraph.config import get_stream_writer


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    writer = get_stream_writer()
    writer(f"Looking up data for city: {city}")
    writer(f"Acquired data for city: {city}")
    return f"It's always sunny in {city}!"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode=["updates", "custom"],
    version="v2",
):
    print(f"stream_mode: {chunk['type']}")
    print(f"content: {chunk['data']}")
    print("\n")
```

输出(节选):

```shell
stream_mode: updates
content: {'model': {'messages': [AIMessage(content='', ..., tool_calls=[{'name': 'get_weather', 'args': {'city': 'San Francisco'}, ...}], ...)]}}

stream_mode: custom
content: Looking up data for city: San Francisco

stream_mode: custom
content: Acquired data for city: San Francisco

stream_mode: updates
content: {'tools': {'messages': [ToolMessage(content="It's always sunny in San Francisco!", name='get_weather', tool_call_id='call_KTNQIftMrl9vgNwEfAJMVu7r')]}}

stream_mode: updates
content: {'model': {'messages': [AIMessage(content='San Francisco weather: It's always sunny in San Francisco!...', ...)]}}
```

## 常见模式(Common patterns)

下面是流式输出常见使用场景的示例。

### 流式思考 / 推理 token(Streaming thinking / reasoning tokens)

部分模型在给出最终答案前会进行内部推理。你可以通过过滤[标准内容块](https://docs.langchain.com/oss/python/langchain/messages#standard-content-blocks)中 `type` 为 `"reasoning"` 的块,流式输出这些思考/推理 token。

> **注意**:必须在模型上启用推理输出。配置细节参见[推理部分](https://docs.langchain.com/oss/python/langchain/models#reasoning)和你的[提供商集成页面](https://docs.langchain.com/oss/python/integrations/providers/overview)。要快速检查模型的推理支持情况,参见 [models.dev](https://models.dev)。

要从 Agent 流式输出思考 token,使用 `stream_mode="messages"` 并过滤推理内容块:

```python
from langchain.agents import create_agent
from langchain_anthropic import ChatAnthropic
from langchain_core.runnables import Runnable


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"


model = ChatAnthropic(
    model_name="claude-sonnet-4-6",
    timeout=None,
    stop=None,
    thinking={"type": "enabled", "budget_tokens": 5000},
)
agent: Runnable = create_agent(
    model=model,
    tools=[get_weather],
)

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    version="v3",
)
for message in stream.messages:
    for token in message.reasoning:
        print(f"[thinking] {token}", end="")
    for token in message.text:
        print(token, end="", flush=True)
```

输出:

```shell
[thinking] The user is asking about the weather in San Francisco. I have a tool
[thinking]  available to get this information. Let me call the get_weather tool
[thinking]  with "San Francisco" as the city parameter.
The weather in San Francisco is: It's always sunny in San Francisco!
```

无论使用哪个模型提供商,其工作方式都相同——LangChain 通过 [`content_blocks`](https://docs.langchain.com/oss/python/langchain/messages#standard-content-blocks) 属性,把提供商特有的格式(Anthropic 的 `thinking` 块、OpenAI 的 `reasoning` 摘要等)规范化为标准的 `"reasoning"` 内容块类型。

要直接从对话模型流式输出推理 token(不经过 Agent),参见[对话模型的流式输出](https://docs.langchain.com/oss/python/langchain/models#reasoning)。

### 流式工具调用(Streaming tool calls)

你可能想同时流式输出:

1. 在生成[工具调用](https://docs.langchain.com/oss/python/langchain/models#工具调用tool-calling)时的部分 JSON
2. 被执行时已完成、已解析的工具调用

指定 [`stream_mode="messages"`](#llm-tokenllm-tokens) 会流式输出 Agent 中所有 LLM 调用产生的增量[消息块](https://docs.langchain.com/oss/python/langchain/messages#streaming-and-chunks)。要访问带已解析工具调用的完整消息:

1. 如果这些消息被跟踪在 [state](https://docs.langchain.com/oss/python/langchain/short-term-memory) 中(如 [`create_agent`](https://docs.langchain.com/oss/python/langchain/agents) 的 model 节点),使用 `stream_mode=["messages", "updates"]`,通过[状态更新](#agent-进度agent-progress)访问完整消息(下面演示)。
2. 如果这些消息没有被跟踪在 state 中,使用[自定义更新](#自定义更新custom-updates),或在流式循环中聚合消息块(见[下一节](#访问完整消息accessing-completed-messages))。

> **注意**:如果你的 Agent 包含多个 LLM,请参考下面关于[从子 Agent 流式输出](#从子-agent-流式输出streaming-from-sub-agents)的部分。

```python
from typing import Any

from langchain.agents import create_agent
from langchain.messages import AIMessage, AIMessageChunk, AnyMessage, ToolMessage


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"It's always sunny in {city}!"


agent = create_agent("openai:gpt-5.5", tools=[get_weather])


def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)
    # 注意:所有内容都可以通过 token.content_blocks 获取


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"Tool calls: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"Tool response: {message.content_blocks}")


input_message = {"role": "user", "content": "What is the weather in Boston?"}
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],
    version="v2",
):
    if chunk["type"] == "messages":
        token, metadata = chunk["data"]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":
        for source, update in chunk["data"].items():
            if source in ("model", "tools"):  # `source` 捕获节点名称
                _render_completed_message(update["messages"][-1])
```

输出(节选):

```shell
[{'name': 'get_weather', 'args': '', 'id': 'call_D3Orjr89KgsLTZ9hTzYv7Hpf', 'index': 0, 'type': 'tool_call_chunk'}]
[{'name': None, 'args': '{"', 'id': None, 'index': 0, 'type': 'tool_call_chunk'}]
...
Tool calls: [{'name': 'get_weather', 'args': {'city': 'Boston'}, 'id': 'call_D3Orjr89KgsLTZ9hTzYv7Hpf', 'type': 'tool_call'}]
Tool response: [{'type': 'text', 'text': "It's always sunny in Boston!"}]
The| weather| in| Boston| is| **|sun|ny|**|.|
```

#### 访问完整消息(Accessing completed messages)

> **注意**:如果完整消息被跟踪在 Agent 的 [state](https://docs.langchain.com/oss/python/langchain/short-term-memory) 中,你可以像[流式工具调用](#流式工具调用streaming-tool-calls)部分演示的那样使用 `stream_mode=["messages", "updates"]`,在流式输出期间访问完整消息。

某些情况下,完整消息不会反映在[状态更新](#agent-进度agent-progress)中。如果你能访问 Agent 内部,可以使用[自定义更新](#自定义更新custom-updates)在流式输出期间访问这些消息。否则,你可以在流式循环中聚合消息块(见下文)。

考虑下面的示例:我们把一个 [stream writer](#自定义更新custom-updates) 整合进一个简化的[护栏中间件](https://docs.langchain.com/oss/python/langchain/guardrails#after-agent-guardrails)。该中间件演示了用工具调用生成结构化的"安全/不安全"评估(也可以为此使用[结构化输出](https://docs.langchain.com/oss/python/langchain/models#structured-output)):

```python
from typing import Any, Literal

from langchain.agents.middleware import after_agent, AgentState
from langgraph.runtime import Runtime
from langchain.messages import AIMessage
from langchain.chat_models import init_chat_model
from langgraph.config import get_stream_writer
from pydantic import BaseModel


class ResponseSafety(BaseModel):
    """把响应评估为 safe 或 unsafe。"""
    evaluation: Literal["safe", "unsafe"]


safety_model = init_chat_model("openai:gpt-5.5")

@after_agent(can_jump_to=["end"])
def safety_guardrail(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """基于模型的护栏:使用 LLM 评估响应安全性。"""
    stream_writer = get_stream_writer()
    # 获取模型响应
    if not state["messages"]:
        return None

    last_message = state["messages"][-1]
    if not isinstance(last_message, AIMessage):
        return None

    # 使用另一个模型评估安全性
    model_with_tools = safety_model.bind_tools([ResponseSafety], tool_choice="any")
    result = model_with_tools.invoke(
        [
            {
                "role": "system",
                "content": "Evaluate this AI response as generally safe or unsafe."
            },
            {
                "role": "user",
                "content": f"AI response: {last_message.text}"
            }
        ]
    )
    stream_writer(result)

    tool_call = result.tool_calls[0]
    if tool_call["args"]["evaluation"] == "unsafe":
        last_message.content = "I cannot provide that response. Please rephrase your request."

    return None
```

然后我们可以把这个中间件整合进 Agent,并包含它的自定义流式事件:

```python
agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_weather],
    middleware=[safety_guardrail],
)

input_message = {"role": "user", "content": "What is the weather in Boston?"}
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates", "custom"],
    version="v2",
):
    if chunk["type"] == "messages":
        token, metadata = chunk["data"]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":
        for source, update in chunk["data"].items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
    elif chunk["type"] == "custom":
        # 在流中访问完整消息
        print(f"Tool calls: {chunk['data'].tool_calls}")
```

如果你无法向流中添加自定义事件,也可以在流式循环中聚合消息块:

```python
input_message = {"role": "user", "content": "What is the weather in Boston?"}
full_message = None
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],
    version="v2",
):
    if chunk["type"] == "messages":
        token, metadata = chunk["data"]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
            full_message = token if full_message is None else full_message + token
            if token.chunk_position == "last":
                if full_message.tool_calls:
                    print(f"Tool calls: {full_message.tool_calls}")
                full_message = None
    elif chunk["type"] == "updates":
        for source, update in chunk["data"].items():
            if source == "tools":
                _render_completed_message(update["messages"][-1])
```

### 流式人机协同(Streaming with human-in-the-loop)

要处理人机协同(human-in-the-loop)[中断](https://docs.langchain.com/oss/python/langchain/human-in-the-loop),我们在[上面的示例](#流式工具调用streaming-tool-calls)基础上构建:

1. 为 Agent 配置[人机协同中间件和 checkpointer](https://docs.langchain.com/oss/python/langchain/human-in-the-loop#configuring-interrupts)
2. 收集在 `"updates"` 流模式期间生成的中断(interrupts)
3. 用一个 [command](https://docs.langchain.com/oss/python/langchain/human-in-the-loop#responding-to-interrupts) 响应这些中断

```python
from typing import Any

from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.messages import AIMessage, AIMessageChunk, AnyMessage, ToolMessage
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import Command, Interrupt


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"It's always sunny in {city}!"


checkpointer = InMemorySaver()

agent = create_agent(
    "openai:gpt-5.5",
    tools=[get_weather],
    middleware=[
        HumanInTheLoopMiddleware(interrupt_on={"get_weather": True}),
    ],
    checkpointer=checkpointer,
)


def _render_message_chunk(token: AIMessageChunk) -> None:
    if token.text:
        print(token.text, end="|")
    if token.tool_call_chunks:
        print(token.tool_call_chunks)


def _render_completed_message(message: AnyMessage) -> None:
    if isinstance(message, AIMessage) and message.tool_calls:
        print(f"Tool calls: {message.tool_calls}")
    if isinstance(message, ToolMessage):
        print(f"Tool response: {message.content_blocks}")


def _render_interrupt(interrupt: Interrupt) -> None:
    interrupts = interrupt.value
    for request in interrupts["action_requests"]:
        print(request["description"])


input_message = {
    "role": "user",
    "content": (
        "Can you look up the weather in Boston and San Francisco?"
    ),
}
config = {"configurable": {"thread_id": "some_id"}}
interrupts = []
for chunk in agent.stream(
    {"messages": [input_message]},
    config=config,
    stream_mode=["messages", "updates"],
    version="v2",
):
    if chunk["type"] == "messages":
        token, metadata = chunk["data"]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":
        for source, update in chunk["data"].items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
            if source == "__interrupt__":
                interrupts.extend(update)
                _render_interrupt(update[0])
```

输出(节选):

```shell
[{'name': 'get_weather', 'args': '', 'id': 'call_GOwNaQHeqMixay2qy80padfE', 'index': 0, 'type': 'tool_call_chunk'}]
...
Tool calls: [{'name': 'get_weather', 'args': {'city': 'Boston'}, ...}, {'name': 'get_weather', 'args': {'city': 'San Francisco'}, ...}]
Tool execution requires approval

Tool: get_weather
Args: {'city': 'Boston'}
Tool execution requires approval

Tool: get_weather
Args: {'city': 'San Francisco'}
```

接下来我们为每个中断收集一个[决定(decision)](https://docs.langchain.com/oss/python/langchain/human-in-the-loop#interrupt-decision-types)。重要的是,决定的顺序必须与我们收集的动作顺序一致。

作为演示,我们编辑其中一个工具调用,批准另一个:

```python
def _get_interrupt_decisions(interrupt: Interrupt) -> list[dict]:
    return [
        {
            "type": "edit",
            "edited_action": {
                "name": "get_weather",
                "args": {"city": "Boston, U.K."},
            },
        }
        if "boston" in request["description"].lower()
        else {"type": "approve"}
        for request in interrupt.value["action_requests"]
    ]

decisions = {}
for interrupt in interrupts:
    decisions[interrupt.id] = {
        "decisions": _get_interrupt_decisions(interrupt)
    }

decisions
```

然后,把 [command](https://docs.langchain.com/oss/python/langchain/human-in-the-loop#responding-to-interrupts) 传入同一个流式循环即可恢复执行:

```python
interrupts = []
for chunk in agent.stream(
    Command(resume=decisions),
    config=config,
    stream_mode=["messages", "updates"],
    version="v2",
):
    # 流式循环保持不变
    if chunk["type"] == "messages":
        token, metadata = chunk["data"]
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":
        for source, update in chunk["data"].items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
            if source == "__interrupt__":
                interrupts.extend(update)
                _render_interrupt(update[0])
```

### 从子 Agent 流式输出(Streaming from sub-agents)

当 Agent 中任何位置存在多个 LLM 时,通常需要在消息生成时区分消息的来源。

为此,在创建每个 Agent 时传入一个 [`name`](https://reference.langchain.com/python/langchain/agents/#langchain.agents.create_agent(name))。这个名字随后可以在 `"messages"` 模式流式输出时通过元数据中的 `lc_agent_name` 键获取。

下面,我们更新[流式工具调用](#流式工具调用streaming-tool-calls)的示例:

1. 把工具替换为一个内部调用 Agent 的 `call_weather_agent` 工具
2. 为每个 Agent 添加 `name`
3. 创建流时指定 [`subgraphs=True`](https://docs.langchain.com/oss/python/langgraph/use-subgraphs#stream-subgraph-outputs)
4. 流处理与之前相同,但增加了使用 `create_agent` 的 `name` 参数跟踪当前活跃 Agent 的逻辑

> **提示**:当你为 Agent 设置 `name` 时,该名称也会附加到该 Agent 生成的所有 `AIMessage` 上。

首先构建 Agent:

```python
from typing import Any

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.messages import AIMessage, AnyMessage


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""

    return f"It's always sunny in {city}!"


weather_model = init_chat_model("openai:gpt-5.5")
weather_agent = create_agent(
    model=weather_model,
    tools=[get_weather],
    name="weather_agent",
)


def call_weather_agent(query: str) -> str:
    """查询天气 Agent。"""
    result = weather_agent.invoke({
        "messages": [{"role": "user", "content": query}]
    })
    return result["messages"][-1].text


supervisor_model = init_chat_model("openai:gpt-5.5")
agent = create_agent(
    model=supervisor_model,
    tools=[call_weather_agent],
    name="supervisor",
)
```

接下来,在流式循环中加入报告哪个 Agent 正在发出 token 的逻辑:

```python
input_message = {"role": "user", "content": "What is the weather in Boston?"}
current_agent = None
for chunk in agent.stream(
    {"messages": [input_message]},
    stream_mode=["messages", "updates"],
    subgraphs=True,
    version="v2",
):
    if chunk["type"] == "messages":
        token, metadata = chunk["data"]
        if agent_name := metadata.get("lc_agent_name"):
            if agent_name != current_agent:
                print(f"🤖 {agent_name}: ")
                current_agent = agent_name
        if isinstance(token, AIMessageChunk):
            _render_message_chunk(token)
    elif chunk["type"] == "updates":
        for source, update in chunk["data"].items():
            if source in ("model", "tools"):
                _render_completed_message(update["messages"][-1])
```

输出(节选):

```shell
🤖 supervisor:
[{'name': 'call_weather_agent', 'args': '', 'id': 'call_asorzUf0mB6sb7MiKfgojp7I', 'index': 0, 'type': 'tool_call_chunk'}]
...
Tool calls: [{'name': 'call_weather_agent', 'args': {'query': "Boston weather right now and today's forecast"}, ...}]
🤖 weather_agent:
[{'name': 'get_weather', 'args': '', 'id': 'call_LZ89lT8fW6w8vqck5pZeaDIx', 'index': 0, 'type': 'tool_call_chunk'}]
...
Tool calls: [{'name': 'get_weather', 'args': {'city': 'Boston'}, ...}]
Tool response: [{'type': 'text', 'text': "It's always sunny in Boston!"}]
Boston| weather| right| now|:| **|Sunny|**|.
🤖 supervisor:
Boston| weather| right| now|:| **|Sunny|**|.
```

## 禁用流式输出(Disable streaming)

在某些应用中,你可能需要为给定模型禁用逐 token 流式输出。这在以下情况很有用:

- 在[多 Agent](https://docs.langchain.com/oss/python/langchain/multi-agent) 系统中控制哪些 Agent 流式输出其输出
- 混合使用支持流式和不支持流式的模型
- 部署到 [LangSmith](https://docs.langchain.com/langsmith/observability) 时,希望阻止某些模型输出流式传给客户端

初始化模型时设置 `streaming=False`。

```python
from langchain_openai import ChatOpenAI

model = ChatOpenAI(
    model="gpt-5.5",
    streaming=False
)
```

> **提示**:部署到 LangSmith 时,对你不想流式传给客户端的任何模型设置 `streaming=False`。这在部署前的图代码中配置。

> **注意**:并非所有对话模型集成都支持 `streaming` 参数。如果你的模型不支持,请改用 `disable_streaming=True`。该参数通过基类在所有对话模型上可用。

更多细节参见 [LangGraph 流式指南](https://docs.langchain.com/oss/python/langgraph/streaming#disable-streaming-for-specific-chat-models)。

## v2 流式格式(v2 streaming format)

> **注意**:需要 LangGraph >= 1.1。

向 `stream()` 或 `astream()` 传入 `version="v2"` 可获得统一的输出格式。每个块都是带 `type`、`ns` 和 `data` 键的 `StreamPart` 字典——无论流模式或模式数量如何,结构都相同:

```python
# v2(新)——统一格式,不再需要解包元组
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode=["updates", "custom"],
    version="v2",
):
    print(chunk["type"])  # "updates" 或 "custom"
    print(chunk["data"])  # 载荷
```

```python
# v1(当前默认)——必须解包 (mode, data) 元组
for mode, chunk in agent.stream(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    stream_mode=["updates", "custom"],
):
    print(mode)   # "updates" 或 "custom"
    print(chunk)  # 载荷
```

v2 格式还改进了 `invoke()`——它返回一个带 `.value` 和 `.interrupts` 属性的 `GraphOutput` 对象,把状态与中断元数据干净地分离:

```python
result = agent.invoke(
    {"messages": [{"role": "user", "content": "Hello"}]},
    version="v2",
)
print(result.value)       # 状态(dict、Pydantic 模型或 dataclass)
print(result.interrupts)  # Interrupt 对象的元组(没有则为空)
```

关于 v2 格式的更多细节(包括类型收窄、Pydantic/dataclass 强制转换和子图流式),参见 [LangGraph 流式文档](https://docs.langchain.com/oss/python/langgraph/streaming#stream-output-format-v2)。

## 相关内容

- [前端流式输出](https://docs.langchain.com/oss/python/langchain/frontend/overview)——使用 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 构建 React UI,实现实时 Agent 交互
- [对话模型的流式输出](https://docs.langchain.com/oss/python/langchain/models#stream)——不使用 Agent 或图,直接从对话模型流式输出 token
- [对话模型的推理](https://docs.langchain.com/oss/python/langchain/models#reasoning)——配置并访问对话模型的推理输出
- [标准内容块](https://docs.langchain.com/oss/python/langchain/messages#standard-content-blocks)——理解用于推理、文本和其他内容类型的规范化内容块格式
- [流式人机协同](https://docs.langchain.com/oss/python/langchain/human-in-the-loop#streaming-with-human-in-the-loop)——在处理人工审核中断的同时流式输出 Agent 进度
- [LangGraph 流式输出](https://docs.langchain.com/oss/python/langgraph/streaming)——高级流式选项,包括 `values`、`debug` 模式和子图流式
