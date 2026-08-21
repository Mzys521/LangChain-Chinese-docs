# Event streaming(事件流)

> 从 LangChain Agent 运行中流式获取实时更新

> 原文:[Event streaming](https://docs.langchain.com/oss/python/langchain/event-streaming)

LangChain Agent 构建在 LangGraph 之上,因此它们支持同一套流式栈(streaming stack),并针对消息、工具调用、状态和自定义更新提供了面向 Agent 的投影(projections)。

对于大多数应用和前端使用场景,请通过 `stream_events(..., version="v3")` 使用 **Event Streaming**。Event Streaming 返回一个带类型化投影的运行对象(run object),因此每个投影都可以独立消费,而不必解析 stream-mode 元组。

```python
from langchain.agents import create_agent


def get_weather(city: str) -> str:
    """获取某城市的天气。"""
    return f"It's always sunny in {city}!"


agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
)

stream = agent.stream_events({
    "messages": [{"role": "user", "content": "What is the weather in SF?"}],
}, version="v3")

for message in stream.messages:
    for delta in message.text:
        print(delta, end="", flush=True)

final_state = stream.output
```

## 可以流式获取的内容

| 投影(Projection) | 用途 |
| --------------------- | -------------------------------------------------------------------------- |
| `for event in stream` | 带完整信封(envelope)的原始协议事件,可访问每个通道(channel)。 |
| `stream.messages`     | 模型消息流,每次 LLM 调用一个。 |
| `message.text`        | 某条消息的文本增量(delta)和最终文本。 |
| `message.reasoning`   | 会暴露推理内容的模型的推理增量。 |
| `message.tool_calls`  | 工具调用参数块(argument chunks)和最终确定的工具调用。 |
| `message.output`      | 模型调用完成后的最终消息对象。 |
| `stream.values`       | Agent 状态快照(snapshot)。 |
| `stream.output`       | 最终的 Agent 状态。 |
| `stream.subgraphs`    | 嵌套图运行(子 Agent 和普通子图)。 |
| `stream.extensions`   | 自定义转换器(transformer)投影。 |
| `stream.tool_calls`   | 工具执行生命周期、输入、输出增量、最终输出和错误。 |

`stream.messages` 产出 `ChatModelStream` 对象。每个消息流都暴露 `.text`、`.reasoning`、`.tool_calls` 和 `.output`。同步投影既可迭代以获取实时增量,也可一次性取出最终值:使用 `str(message.text)` 获取最终文本,使用 `message.tool_calls.get()` 获取最终确定的工具调用。

## Agent 消息

当你想获取每次 LLM 调用的模型输出时,使用 `stream.messages`。

```python
stream = agent.stream_events(input, version="v3")

for message in stream.messages:
    print(f"[{message.node}] ", end="")
    for delta in message.text:
        print(delta, end="", flush=True)

    full_message = message.output
    usage = full_message.usage_metadata
    if usage:
        print(usage)
```

`message.output` 给你最终确定的 AI 消息,包括提供商特有的内容块。在 TypeScript 中,如果你只需要 token 数或其他用量元数据,使用 `message.usage`;在 Python 中,从 `message.output.usage_metadata` 读取用量。

## 推理内容(Reasoning content)

推理内容与文本内容的结构相同,但只在所选模型会输出推理块时才可用。

```python
stream = agent.stream_events(input, version="v3")

for message in stream.messages:
    for delta in message.reasoning:
        print(f"[thinking] {delta}", end="", flush=True)

    for delta in message.text:
        print(delta, end="", flush=True)
```

模型配置细节参见[推理指南](https://docs.langchain.com/oss/python/langchain/models#reasoning)和你所用提供商的集成页面。

## 工具调用(Tool calls)

有两个有用的工具调用投影:

- `message.tool_calls`:在模型生成工具调用的过程中,流式输出工具调用参数块。
- `stream.tool_calls`:在工具调用开始后,流式输出工具执行的生命周期。

```python
stream = agent.stream_events(input, version="v3")

for message in stream.messages:
    for chunk in message.tool_calls:
        print(f"tool call chunk: {chunk}")

    finalized = message.tool_calls.get()
    if finalized:
        print(f"finalized tool calls: {finalized}")

for call in stream.tool_calls:
    print(f"{call.tool_name}({call.input})")
    for delta in call.output_deltas:
        print(delta, end="", flush=True)
    print(call.output, call.error)
```

## 流式子 Agent(Streaming sub-agents)

当一个 `create_agent` 调用另一个命名的 `create_agent`(通常通过一个包装工具)时,内部 Agent 的事件会在嵌套的命名空间(namespace)中流动。你传给 `create_agent` 的 `name=` 在流中标识该内部 Agent,因此你可以按 Agent 过滤和打标签。

命名的子 Agent 会出现在专用的 `stream.subagents` 投影上。每个句柄(handle)暴露内部 Agent 自己的 `.messages`、`.values`、`.tool_calls` 和 `.output`,外加 `.name`(你传入的 `name=`)和 `.cause`(派发该子 Agent 的工具调用)。因为只有命名的 `create_agent` 运行才会出现在这里,你不需要过滤掉普通子图。

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model


def get_weather(city: str) -> str:
    """获取指定城市的天气。"""
    return f"It's always sunny in {city}!"


weather_agent = create_agent(
    model=init_chat_model("openai:gpt-5.5"),
    tools=[get_weather],
    name="weather_agent",
)


def call_weather(query: str) -> str:
    """查询天气 Agent。"""
    result = weather_agent.invoke({"messages": [{"role": "user", "content": query}]})
    return result["messages"][-1].text


supervisor = create_agent(
    model=init_chat_model("openai:gpt-5.5"),
    tools=[call_weather],
    name="supervisor",
)

stream = supervisor.stream_events(
    {"messages": [{"role": "user", "content": "What's the weather in Boston?"}]},
    version="v3",
)

for subagent in stream.subagents:
    print(f"{subagent.name}: ", end="")
    for message in subagent.messages:
        for token in message.text:
            print(token, end="", flush=True)
    print()
```

从工具中调用的普通 `StateGraph` 子图也会出现在 `stream.subgraphs` 上——在 `.compile(name=...)` 上设置 `name=`,即可在 `subagent.graph_name` 中获得标签。

`stream.subagents` 是命名 `create_agent` 子 Agent 的聚焦视图,而 `stream.subgraphs` 覆盖每个嵌套图。选择匹配你 UI 需求的那个即可。

## 状态与最终输出

使用 `stream.values` 获取状态快照,使用 `stream.output` 获取最终的 Agent 状态。

```python
stream = agent.stream_events(input, version="v3")

for snapshot in stream.values:
    print(snapshot)

final_state = stream.output
```

## 多个投影(Multiple projections)

在异步代码中并发消费时,使用 `astream_events` 配合 `asyncio.gather`:

```python
import asyncio

stream = await agent.astream_events(input, version="v3")

async def consume_messages():
    async for message in stream.messages:
        print(await message.text)

async def consume_tool_calls():
    async for call in stream.tool_calls:
        print(call.tool_name, call.input)

await asyncio.gather(consume_messages(), consume_tool_calls())
```

对于同步代码,改用 `stream.interleave(...)`:

```python
stream = agent.stream_events(input, version="v3")

for name, item in stream.interleave("messages", "tool_calls", "values"):
    if name == "messages":
        print(item.text)
    elif name == "tool_calls":
        print(item.tool_name, item.input)
    elif name == "values":
        print(item)
```

要访问没有作为类型化投影暴露的通道,或检查完整的事件信封,可以迭代原始协议事件:

```python
for event in stream:
    print(event["method"], event["params"]["namespace"], event["params"]["data"])
```

## 自定义更新(Custom updates)

当你的应用需要内置没有的投影(例如检索进度、产物(artifacts)或领域特定事件)时,使用自定义流转换器(stream transformers)。

```python
stream = agent.stream_events(
    input,
    version="v3",
    transformers=[ToolActivityTransformer],
)

for activity in stream.extensions["tool_activity"]:
    print(activity)
```

### 在中间件上注册转换器

> **注意**:中间件注册的转换器需要 `langchain>=1.3.2`。

中间件可以在其钩子和工具旁边声明流转换器工厂。工厂的形式因语言而异:

将 `AgentMiddleware` 子类的 `transformers` 属性设置为一个工厂序列。每个工厂的形式为 `Callable[[tuple[str, ...]], StreamTransformer]`,并以 `factory(scope)` 方式调用,其中 `scope` 是 mini-mux 作用域元组(根 mux 为 `()`,子图为非空)。每次调用返回一个全新的转换器,可以让每个子图保持隔离。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware


class ToolActivityMiddleware(AgentMiddleware):
    transformers = (ToolActivityTransformer,)


agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
    middleware=[ToolActivityMiddleware()],
)
```

在编译时,`create_agent` 会把中间件注册的工厂与传给它自己 `transformers=` 参数的内容合并。编译后图上的最终顺序为:

1. 内置的 `ToolCallTransformer`。
2. 中间件注册的工厂,按中间件顺序。
3. 调用方通过 `create_agent` 提供的 `transformers=`。

这让内置的工具调用投影位于消费者转换器之前,并让调用方提供的条目拥有最终决定权。

内置的 `PIIMiddleware` 使用这个钩子从流式线路输出(wire output)中编辑(redact)PII。当设置 `apply_to_output=True` 时,它注册的转换器会在文本增量、工具调用参数、工具输出和状态快照离开运行之前清除检测到的 PII,堵住了 `after_model` 状态级编辑原本会让原始 PII 流向 `stream_events(version="v3")` 实时读者的窗口。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="gpt-5-nano",
    tools=[],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_output=True),
    ],
)
```

完整的配置面参见 [PII 检测](https://docs.langchain.com/oss/python/langchain/middleware/built-in#pii-detection)。

转换器契约(transformer contract)参见[构建你自己的投影](https://docs.langchain.com/oss/python/langgraph/event-streaming#build-your-own-projection)。

## 相关内容

- [Streaming](https://docs.langchain.com/oss/python/langchain/streaming) 介绍底层的 Pregel 流式模式。
- [构建你自己的投影](https://docs.langchain.com/oss/python/langgraph/event-streaming#build-your-own-projection)介绍编写应用特定的投影。
- [前端流式模式](https://docs.langchain.com/oss/python/langchain/frontend/overview)展示基于流式状态构建的 UI 使用场景。
