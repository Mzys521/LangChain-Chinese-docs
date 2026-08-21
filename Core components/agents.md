# Agents(Agent)

> 原文:[Agents](https://docs.langchain.com/oss/python/langchain/agents)

Agent 是一个在循环中调用工具的模型,直到完成给定任务为止。

Harness(运行框架)则是围绕该循环的一切:提示词、工具,以及任何塑造模型行为的中间件(middleware)。

> **注意:Agent = 模型 + Harness**
>
> Harness 的职责:在正确的时间,为给定任务向模型提供正确的上下文。

[`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) 是一个高度可配置的 harness。最简单的用法如下:

```python
from langchain.agents import create_agent

agent = create_agent(model="openai:gpt-5.5", tools=tools)
```

> 以上以 OpenAI 为例;你也可以传入其他提供商的模型标识符,例如 `"google_genai:gemini-3.6-flash"`、`"anthropic:claude-sonnet-4-6"`、`"openrouter:z-ai/glm-5.2"`、`"fireworks:accounts/fireworks/models/glm-5p2"`、`"baseten:zai-org/GLM-5.2"`、`"ollama:north-mini-code-1.0"` 等。

在此基础上,你可以直接通过 `model=`、`tools=` 和 `system_prompt=` 参数配置基础项。如需更高级的能力,请使用[中间件](#配置-harness)扩展 harness。

> **提示**
>
> [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) 构建在 `create_agent` 之上,并预先组装了常用能力,如规划(planning)、文件系统工具、子 Agent(subagents)和记忆(memory)。当你需要自行配置 harness 时,请使用 `create_agent`。

## 核心组件

### Model(模型)

传入模型标识符字符串(`"provider:model"` 格式)或已初始化的模型实例,即可为 Agent 选择模型。关于参数、提供商配置和动态模型选择,请参见 [Models](https://docs.langchain.com/oss/python/langchain/models)。

```python
from langchain.agents import create_agent

agent = create_agent(model="openai:gpt-5.5", tools=tools)
```

### Tools(工具)

要为 Agent 提供工具,可以传入任意 Python 可调用对象(callable)、LangChain 工具或工具字典。关于工具定义、上下文访问和动态工具选择,请参见 [Tools](https://docs.langchain.com/oss/python/langchain/tools)。

```python
from langchain.agents import create_agent
from langchain.tools import tool


@tool
def search(query: str) -> str:
    """搜索信息。"""
    return f"Results for: {query}"


agent = create_agent(model="openai:gpt-5.5", tools=[search])
```

### System prompt(系统提示词)

用于塑造 Agent 处理任务的方式。`system_prompt` 参数接受字符串或 `SystemMessage`。如需在运行时动态生成提示词,请使用[中间件](https://docs.langchain.com/oss/python/langchain/middleware)。

```python
agent = create_agent(
    model="openai:gpt-5.5",
    tools=tools,
    system_prompt="You are a helpful assistant. Be concise and accurate.",
)
```

### Structured output(结构化输出)

使用 `response_format=` 让 Agent 返回经过校验的 schema。关于策略和示例,请参见 [Structured output](https://docs.langchain.com/oss/python/langchain/structured-output)。

```python
from pydantic import BaseModel
from langchain.agents import create_agent


class Answer(BaseModel):
    summary: str
    confidence: float


agent = create_agent(model="openai:gpt-5.5", tools=tools, response_format=Answer)
result = agent.invoke({"messages": [{"role": "user", "content": "Summarize AI trends"}]})
result["structured_response"]  # Answer(summary=..., confidence=...)
```

### Agent state(Agent 状态)

每个 Agent 都通过 [`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState) 管理其执行上下文。这是一个带类型注解的字典,保存当前对话历史以及你的工具和中间件所需的任何自定义字段。

内置字段为:

| 字段 | 类型 | 说明 |
| ---------- | ------------------- | ------------------- |
| `messages` | `list[BaseMessage]` | 当前线程(thread)的完整对话历史。只追加(append-only):新消息会被添加,永远不会被替换。 |

`AgentState` 同时也是所有节点式中间件钩子(`before_model`、`after_model` 等)的类型签名。钩子接收当前状态,并可返回一个更新字典(update dict),用于合并回状态中。

要添加自定义字段(例如 `user_id` 或计数器),可以继承 `AgentState`,并通过 `state_schema=` 将子类传给 `create_agent`:

```python
from langchain.agents import AgentState, create_agent


class MyState(AgentState):
    user_id: str
    call_count: int


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[],
    state_schema=MyState,
)
```

完整细节、示例以及中间件级别的状态 schema,请参见[短期记忆](https://docs.langchain.com/oss/python/langchain/short-term-memory#customizing-agent-memory)和[自定义中间件](https://docs.langchain.com/oss/python/langchain/middleware/custom#state-updates)。

## 调用(Invocation)

> **提示**
>
> 使用 [LangSmith](https://smith.langchain.com) 追踪该循环的每一步、调试工具调用并评估 Agent 输出。可参考[追踪快速入门](https://docs.langchain.com/langsmith/trace-with-langchain)完成配置。我们还建议同时配置 [LangSmith Engine](https://docs.langchain.com/langsmith/engine),它可以监控你的追踪数据、检测问题并提出修复建议。

你可以通过一条消息来调用 Agent。在底层,这会向 Agent 的 [`State`](https://docs.langchain.com/oss/python/langgraph/graph-api#state) 传入一次更新。所有 Agent 的状态中都包含[一个消息序列](https://docs.langchain.com/oss/python/langgraph/use-graph-api#messagesstate);调用 Agent 时,传入一条新消息以及一个 `thread_id`,这样 Agent 就能持久化并恢复对话历史:

```python
from langchain.agents import create_agent
from langchain_core.utils.uuid import uuid7
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[],
    checkpointer=InMemorySaver(),
)

config = {"configurable": {"thread_id": str(uuid7())}}

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
    config=config,
)

# 同一对话中的后续轮次:复用相同的 thread_id 以保留历史记录
result = agent.invoke(
    {"messages": [{"role": "user", "content": "What about tomorrow?"}]},
    config=config,
)
```

> **注意**
>
> 使用 `thread_id` 持久化对话历史,需要为 Agent 配置一个 [checkpointer](https://docs.langchain.com/oss/python/langchain/long-term-memory)。部署在 [LangSmith](https://docs.langchain.com/langsmith/deployment) 上时,checkpointer 会自动配置好;在本地运行时需要显式传入,例如 `create_agent(..., checkpointer=InMemorySaver())`。

如果还需要向工具和中间件传递每次运行的配置(如用户 ID、API key 或功能开关(feature flags)),可以将它们作为 `context` 与 `config` 一起传入。使用 `context_schema` 定义这些数据的结构,并通过 `runtime.context` 访问:

```python
from dataclasses import dataclass

from langchain.agents import create_agent
from langchain_core.utils.uuid import uuid7
from langgraph.checkpoint.memory import InMemorySaver


@dataclass
class Context:
    user_id: str


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[],
    context_schema=Context,
    checkpointer=InMemorySaver(),
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What's the weather in San Francisco?"}]},
    config={"configurable": {"thread_id": str(uuid7())}},
    context=Context(user_id="user-123"),
)
```

`thread_id` 的作用域是*对话*(消息历史、检查点),而 `context` 携带的是工具与中间件在调用时读取的*每次运行*的数据。二者通常会一起传入。更多信息参见[工具上下文](https://docs.langchain.com/oss/python/langchain/tools#context)和 [Runtime](https://docs.langchain.com/oss/python/langchain/runtime)。

## 流式输出(Streaming)

`invoke` 会在一次运行结束时返回最终响应。如果 Agent 要执行多次工具调用,用户通常需要在完成之前看到进展。使用流式输出可以实时呈现中间消息和工具活动。

```python
from langchain.messages import AIMessage, HumanMessage


stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "Search for AI news and summarize the findings"}]},
    version="v3",
)
for snapshot in stream.values:
    # 每个快照(snapshot)包含该时刻的完整状态
    latest_message = snapshot["messages"][-1]
    if latest_message.content:
        if isinstance(latest_message, HumanMessage):
            print(f"User: {latest_message.content}")
        elif isinstance(latest_message, AIMessage):
            print(f"Agent: {latest_message.content}")
    elif latest_message.tool_calls:
        print(f"Calling tools: {[tc['name'] for tc in latest_message.tool_calls]}")
```

> **提示**
>
> 关于流式模式、事件类型和 UI 模式,请参见 [Streaming](https://docs.langchain.com/oss/python/langchain/streaming)。

## 配置 Harness

`create_agent` 具有极强的可扩展性。中间件(middleware)是用于定制的核心原语:每个中间件只负责一个关注点(concern),在正确的时机挂入 Agent 循环,并且可以与其他任意中间件自由组合。只取你的使用场景所需,其余一概省略。

常见模式已作为一等公民(first-class)中间件预先内置。其他一切能力都可以通过[自定义中间件](https://docs.langchain.com/oss/python/langchain/middleware/custom)构建。

随着 Agent 承担越来越复杂的工作,它们需要在几个关键领域获得支持。中间件生态提供了:

- **执行环境(Execution environment)**:工具、文件系统、沙箱(sandboxes)和代码执行
- **上下文管理(Context management)**:摘要(summarization)、记忆(memory)、技能(skills)和提示词缓存(prompt caching)
- **规划与委派(Planning and delegation)**:待办清单(todo list)和用于并行、隔离工作的子 Agent
- **容错(Fault tolerance)**:重试、回退(fallbacks)和调用次数限制
- **护栏(Guardrails)**:PII(个人敏感信息)检测和内容管控
- **引导(Steering)**:在高影响操作之前引入人机协同(human-in-the-loop)审批

> **提示**
>
> `create_deep_agent` 已预先组装了这套能力栈,适用于长时间运行的编码与研究任务(默认包含文件系统、摘要、子 Agent 和提示词缓存)。完整的预构建 harness 请参见 [Deep Agents](https://docs.langchain.com/oss/python/deepagents/harness)。

### 执行环境(Execution environment)

Agent 在能够采取行动而不仅仅生成文本时尤为有用。执行环境为 Agent 提供了一个工作区:可调用的工具、跨轮次读写文件的文件系统,以及运行脚本或 shell 命令的代码执行能力。

```python
from langchain.agents import create_agent
from deepagents.backends import StateBackend
from deepagents.middleware import FilesystemMiddleware

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[search],
    middleware=[FilesystemMiddleware(backend=StateBackend())],
)
```

参见 [`FilesystemMiddleware`](https://reference.langchain.com/python/deepagents/middleware/filesystem/FilesystemMiddleware)、[沙箱(Sandboxes)](https://docs.langchain.com/oss/python/deepagents/sandboxes)、[解释器(Interpreters)](https://docs.langchain.com/oss/python/deepagents/interpreters)。

> **注意**
>
> 该示例依赖 `deepagents` 包,安装方式:
>
> ```bash
> pip install deepagents
> # 或使用 uv:uv add deepagents
> ```

### 上下文管理(Context management)

每次模型调用都有固定的上下文窗口(context window)。随着 Agent 的运行,窗口会被不断累积的历史、工具结果和中间步骤填满。摘要(summarization)在溢出之前压缩历史;记忆(memory)在启动时加载持久化指令,使知识跨会话保留;技能(skills)按需呈现领域知识,而不是在启动时全部加载。

```python
from deepagents.backends import StateBackend
from deepagents.middleware import FilesystemMiddleware, MemoryMiddleware, SkillsMiddleware, SummarizationMiddleware

backend = StateBackend()
model = "openai:gpt-5.5"

agent = create_agent(
    model=model,
    tools=[search],
    middleware=[
        FilesystemMiddleware(backend=backend),
        SummarizationMiddleware(model=model, backend=backend),
        MemoryMiddleware(backend=backend, sources=["./AGENTS.md"]),
        SkillsMiddleware(backend=backend, sources=["./skills/"]),
    ],
)
```

参见 [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware)、[`MemoryMiddleware`](https://reference.langchain.com/python/deepagents/middleware/memory/MemoryMiddleware)、[技能(Skills)](https://docs.langchain.com/oss/python/langchain/multi-agent/skills)、[上下文工程(Context engineering)](https://docs.langchain.com/oss/python/deepagents/context-engineering)。

> **注意**
>
> 该示例依赖 `deepagents` 包,安装方式:
>
> ```bash
> pip install deepagents
> # 或使用 uv:uv add deepagents
> ```

### 规划与委派(Planning and delegation)

复杂任务往往超出单个上下文窗口所能处理的范围。委派(delegation)让主 Agent 把工作拆分成若干部分,交给各自在独立上下文中运行的子 Agent(subagents),从而专注于协调而非执行。工作可以并行进行;主 Agent 的上下文保持干净。

```python
from deepagents.backends import StateBackend
from deepagents.middleware import FilesystemMiddleware
from deepagents.middleware.subagents import SubAgentMiddleware
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware
from langchain.tools import tool


@tool
def search(query: str) -> str:
    """搜索查询并返回简短摘要。"""
    return f"Search results for: {query}"


backend = StateBackend()

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[search],
    middleware=[
        FilesystemMiddleware(backend=backend),
        TodoListMiddleware(),
        SubAgentMiddleware(
            backend=backend,
            subagents=[
                {
                    "name": "researcher",
                    "description": "Searches and returns a structured summary.",
                    "system_prompt": "Use the search tool to research the question and summarize key points.",
                    "tools": [search],
                    "model": "anthropic:claude-sonnet-4-6",
                    "middleware": [],
                }
            ],
        ),
    ],
)
```

参见[子 Agent(Subagents)](https://docs.langchain.com/oss/python/langchain/multi-agent/subagents)。

> **注意**
>
> 该示例依赖 `deepagents` 包,安装方式:
>
> ```bash
> pip install deepagents
> # 或使用 uv:uv add deepagents
> ```

### 为 Agent 命名

可以为 Agent 指定一个标识符。当把 Agent 作为子图(subgraph)嵌入[多 Agent](https://docs.langchain.com/oss/python/langchain/multi-agent) 系统时,这尤其有用。

```python
agent = create_agent(model="openai:gpt-5.5", tools=tools, name="research_assistant")
```

### 容错(Fault tolerance)

生产环境中的 Agent 会遇到开发中很少出现的故障:速率限制(rate limits)、模型超时、临时性 API 错误。容错中间件在基础设施层面处理这些问题,使你的工具与业务逻辑不必在每次调用上都包裹 try/catch。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelRetryMiddleware, ToolRetryMiddleware
from langchain.tools import tool


@tool
def search(query: str) -> str:
    """搜索查询并返回简短摘要。"""
    return f"Search results for: {query}"


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[search],
    middleware=[
        ModelRetryMiddleware(max_retries=3),
        ToolRetryMiddleware(max_retries=2),
    ],
)
```

参见 [`ModelRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_retry/ModelRetryMiddleware)、[`ToolRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_retry/ToolRetryMiddleware)、[预构建中间件](https://docs.langchain.com/oss/python/langchain/middleware/built-in)。

### 护栏(Guardrails)

有些策略不能只写在提示词里——无论模型做什么,它们都需要被确定性地强制执行。护栏在数据流经 Agent 循环时进行拦截,在工具结果进入模型上下文之前应用合规规则或内容策略。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
from langchain.tools import tool


@tool
def search(query: str) -> str:
    """搜索查询并返回简短摘要。"""
    return f"Search results for: {query}"


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[search],
    middleware=[PIIMiddleware("email")],
)
```

参见 [`PIIMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/pii/PIIMiddleware)、[预构建中间件](https://docs.langchain.com/oss/python/langchain/middleware/built-in)。

### 引导(Steering)

完全自主并非总是合适的。引导(steering)让你可以把人安插在特定的决策点——破坏性写入之前、昂贵的 API 调用之前,或任何需要判断的地方——而无需重构你的 Agent。Agent 会暂停并等待;人来批准、编辑或拒绝;然后执行继续。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langchain.tools import tool


@tool
def search(query: str) -> str:
    """搜索查询并返回简短摘要。"""
    return f"Search results for: {query}"


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[search],
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"write_file": True})],
)
```

参见 [`HumanInTheLoopMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/human_in_the_loop/HumanInTheLoopMiddleware)、[人机协同(Human-in-the-loop)](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)。

### 中间件资源

- [中间件概览](https://docs.langchain.com/oss/python/langchain/middleware/overview):中间件栈如何工作,钩子何时触发
- [预构建中间件](https://docs.langchain.com/oss/python/langchain/middleware/built-in):带配置示例的完整参考
- [自定义中间件](https://docs.langchain.com/oss/python/langchain/middleware/custom):编写你自己的钩子,用于业务逻辑、PII 清洗等
