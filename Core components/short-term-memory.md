# Short-term memory(短期记忆)

> 原文:[Short-term memory](https://docs.langchain.com/oss/python/langchain/short-term-memory)

## 概述

记忆(memory)是一个记住先前交互信息的系统。对 AI Agent 而言,记忆至关重要,因为它让 Agent 能够记住之前的交互、从反馈中学习,并适应用户偏好。随着 Agent 承担越来越复杂的任务、面对大量用户交互,这一能力对效率和用户满意度都变得必不可少。

短期记忆让你的应用能够记住单个线程(thread)或对话中之前的交互。

> **注意**:thread 把一个会话中的多次交互组织在一起,类似于电子邮件把多条消息归入同一个会话。

对话历史是最常见的短期记忆形式。长对话对今天的 LLM 是一个挑战:完整历史可能无法放进 LLM 的上下文窗口,从而导致上下文丢失或错误。

即使你的模型支持完整的上下文长度,大多数 LLM 在长上下文上的表现仍然不佳。它们会被陈旧或偏题的内容"分心",同时还伴随更慢的响应速度和更高的成本。

对话模型使用[消息](https://docs.langchain.com/oss/python/langchain/messages)接受上下文,消息包括指令(system message)和输入(human message)。在聊天应用中,消息在人类输入和模型响应之间交替,形成一个不断增长的消息列表。由于上下文窗口有限,许多应用都能从使用技术移除或"遗忘"陈旧信息中受益。

> **提示**
>
> 需要**跨**对话记住信息?使用[长期记忆](https://docs.langchain.com/oss/python/langchain/long-term-memory)在不同 thread 和会话间存储并召回用户特定或应用级别的数据。

## 用法

要为 Agent 添加短期记忆(线程级持久化),需要在创建 Agent 时指定一个 `checkpointer`。

> **说明**
>
> LangChain 的 Agent 把短期记忆作为 Agent 状态(state)的一部分来管理。
>
> 通过把这些内容存储在图(graph)的状态中,Agent 可以访问给定对话的完整上下文,同时保持不同 thread 之间的隔离。
>
> 状态通过 checkpointer 持久化到数据库(或内存),因此 thread 可以随时恢复。
>
> 短期记忆在 Agent 被调用或某个步骤(如工具调用)完成时更新,状态在每个步骤开始时被读取。

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver


def get_user_info() -> str:
    """查找当前用户的信息。"""
    return "No user profile on file."


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_user_info],
    checkpointer=InMemorySaver(),
)

thread_config = {"configurable": {"thread_id": "1"}}
response = agent.invoke(
    {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
    thread_config,
)["messages"][-1].content

print(response)  # "Hi Bob! Nice to see you here. How are you doing?"

response = agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    thread_config,
)["messages"][-1].content

print(response)  # "You are Bob!"
```

### 生产环境

在生产环境中,请使用由数据库支撑的 checkpointer:

```bash
pip install -U langgraph-checkpoint-postgres "psycopg[binary]"
# 或使用 uv:uv add langgraph-checkpoint-postgres "psycopg[binary]"
```

> **注意**:默认情况下,`langgraph-checkpoint-postgres` 安装的是不带 extras 的 `psycopg`(Psycopg 3)。上面的安装命令添加了 `psycopg[binary]`,推荐给大多数用户。其他选项参见 [Psycopg 安装文档](https://www.psycopg.org/psycopg3/docs/basic/install.html)。

```python
from langchain.agents import create_agent
from langgraph.checkpoint.postgres import PostgresSaver

def get_user_info() -> str:
    """查找当前用户的信息。"""
    return "No user profile on file."

DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup() # 自动在 PostgreSQL 中创建表
    agent = create_agent(
        "gpt-5.5",
        tools=[get_user_info],
        checkpointer=checkpointer,
    )
```

> **注意**:更多 checkpointer 选项(包括 SQLite、Postgres 和 Azure Cosmos DB)参见持久化文档中的 [checkpointer 库列表](https://docs.langchain.com/oss/python/langgraph/checkpointers#checkpointer-libraries)。

## 自定义 Agent 记忆

默认情况下,Agent 使用 [`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState) 管理短期记忆,具体来说是通过 `messages` 键管理对话历史。

你可以扩展 [`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState) 添加额外字段。自定义 state schema 通过 [`state_schema`](https://reference.langchain.com/python/langchain/middleware/#langchain.agents.middleware.AgentMiddleware.state_schema) 参数传给 [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent)。

```python
from langchain.agents import create_agent, AgentState
from langgraph.checkpoint.memory import InMemorySaver


class CustomAgentState(AgentState):
    user_id: str
    preferences: dict

agent = create_agent(
    "gpt-5.5",
    tools=[get_user_info],
    state_schema=CustomAgentState,
    checkpointer=InMemorySaver(),
)

# 自定义 state 可以在 invoke 时传入
result = agent.invoke(
    {
        "messages": [{"role": "user", "content": "Hello"}],
        "user_id": "user_123",
        "preferences": {"theme": "dark"}
    },
    {"configurable": {"thread_id": "1"}})
```

## 常见模式

启用[短期记忆](#用法)后,长对话可能会超出 LLM 的上下文窗口。常见的解决方案有:

- **裁剪消息(Trim messages)**:移除最前或最后 N 条消息(在调用 LLM 之前)
- **删除消息(Delete messages)**:从 LangGraph state 中永久删除消息
- **摘要消息(Summarize messages)**:对历史中较早的消息进行摘要,并用摘要替换它们
- **自定义策略(Custom strategies)**:自定义策略(例如消息过滤等)

这让 Agent 能够在不超出 LLM 上下文窗口的前提下跟踪对话。

### 裁剪消息(Trim messages)

大多数 LLM 都有一个最大支持的上下文窗口(以 token 计)。

决定何时截断消息的一种方式是统计消息历史中的 token 数,并在接近上限时进行截断。如果你使用 LangChain,可以用 trim messages 工具函数,指定要从列表中保留的 token 数,以及用于处理边界的 `strategy`(例如保留最后 `max_tokens`)。

要在 Agent 中裁剪消息历史,请使用 [`@before_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_model) 中间件装饰器:

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import before_model
from langgraph.runtime import Runtime
from langchain_core.runnables import RunnableConfig
from typing import Any


@before_model
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """只保留最后几条消息以适配上下文窗口。"""
    messages = state["messages"]

    if len(messages) <= 3:
        return None  # 无需修改

    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages

    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),
            *new_messages
        ]
    }

agent = create_agent(
    "gpt-5.5",
    tools=[...],
    middleware=[trim_messages],
    checkpointer=InMemorySaver(),
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob. You told me that earlier.
If you'd like me to call you a nickname or use a different name, just say the word.
"""
```

### 删除消息(Delete messages)

你可以从图状态中删除消息以管理消息历史。当你想移除特定消息或清空整个消息历史时,这很有用。

要从图状态中删除消息,可以使用 `RemoveMessage`。

要使 `RemoveMessage` 生效,你需要对使用 [`add_messages`](https://reference.langchain.com/python/langgraph/graph/message/add_messages) [reducer](https://docs.langchain.com/oss/python/langgraph/graph-api#reducers) 的 state 键进行操作。默认的 [`AgentState`](https://reference.langchain.com/python/langchain/agents/middleware/types/AgentState) 已经提供了这一点。

移除特定消息:

```python
from langchain.messages import RemoveMessage

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # 移除最早的两条消息
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
```

移除**所有**消息:

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES

def delete_messages(state):
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}
```

> **警告**:删除消息时,**务必确保**删除后的消息历史仍然有效。请检查你所用 LLM 提供商的限制。例如:
>
> - 有些提供商要求消息历史以 `user` 消息开头
> - 大多数提供商要求带工具调用的 `assistant` 消息之后必须跟随对应的 `tool` 结果消息。

```python
from langchain.messages import RemoveMessage
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import after_model
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.runtime import Runtime
from langchain_core.runnables import RunnableConfig


@after_model
def delete_old_messages(state: AgentState, runtime: Runtime) -> dict | None:
    """移除旧消息,让对话保持可控。"""
    messages = state["messages"]
    if len(messages) > 2:
        # 移除最早的两条消息
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
    return None


agent = create_agent(
    "gpt-5-nano",
    tools=[...],
    system_prompt="Please be concise and to the point.",
    middleware=[delete_old_messages],
    checkpointer=InMemorySaver(),
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
    config,
    version="v3",
)
for snapshot in stream.values:
    print([(message.type, message.content) for message in snapshot["messages"]])

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "write a short poem about cats"}]},
    config,
    version="v3",
)
for snapshot in stream.values:
    print([(message.type, message.content) for message in snapshot["messages"]])

stream = agent.stream_events(
    {"messages": [{"role": "user", "content": "what's my name?"}]},
    config,
    version="v3",
)
for snapshot in stream.values:
    print([(message.type, message.content) for message in snapshot["messages"]])
```

输出示例(节选):

```text
[('human', "hi! I'm bob")]
[('human', "hi! I'm bob"), ('ai', 'Hi Bob! Nice to meet you. ...')]
...
[('human', 'write a short poem about cats'), ('ai', 'There once was a cat on a wall, ...'), ('human', "what's my name?")]
[('human', 'write a short poem about cats'), ('ai', 'There once was a cat on a wall, ...'), ('human', "what's my name?"), ('ai', "I don't know your name - you haven't told me!")]
[('human', "what's my name?"), ('ai', "I don't know your name - you haven't told me!")]
```

### 摘要消息(Summarize messages)

如上所示,裁剪或移除消息的问题在于,你可能因为清理消息队列而丢失信息。因此,一些应用受益于一种更精细的方法:使用对话模型对消息历史进行摘要。

要在 Agent 中摘要消息历史,请使用内置的 [`SummarizationMiddleware`](https://docs.langchain.com/oss/python/langchain/middleware#summarization):

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware
from langgraph.checkpoint.memory import InMemorySaver
from langchain_core.runnables import RunnableConfig


checkpointer = InMemorySaver()

agent = create_agent(
    model="gpt-5.5",
    tools=[...],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20)
        )
    ],
    checkpointer=checkpointer,
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}
agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob!
"""
```

更多配置选项参见 [`SummarizationMiddleware`](https://docs.langchain.com/oss/python/langchain/middleware#summarization)。

## 访问记忆

你可以通过多种方式访问和修改 Agent 的短期记忆(state):

### 工具(Tools)

#### 在工具中读取短期记忆

使用 `runtime` 参数(类型为 `ToolRuntime`)在工具中访问短期记忆(state)。

`runtime` 参数对工具签名是隐藏的(因此模型看不到它),但工具可以通过它访问 state。

```python
from langchain.agents import create_agent, AgentState
from langchain.tools import tool, ToolRuntime


class CustomState(AgentState):
    user_id: str

@tool
def get_user_info(
    runtime: ToolRuntime
) -> str:
    """查找用户信息。"""
    user_id = runtime.state["user_id"]
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

agent = create_agent(
    model="gpt-5-nano",
    tools=[get_user_info],
    state_schema=CustomState,
)

result = agent.invoke({
    "messages": "look up user information",
    "user_id": "user_123"
})
print(result["messages"][-1].content)
# > User is John Smith.
```

#### 从工具写入短期记忆

要在执行期间修改 Agent 的短期记忆(state),可以直接从工具返回 state 更新。

这对于持久化中间结果,或让信息可被后续工具或提示词访问很有用。

```python
from langchain.tools import tool, ToolRuntime
from langchain_core.runnables import RunnableConfig
from langchain.messages import ToolMessage
from langchain.agents import create_agent, AgentState
from langgraph.types import Command
from pydantic import BaseModel


class CustomState(AgentState):
    user_name: str

class CustomContext(BaseModel):
    user_id: str

@tool
def update_user_info(
    runtime: ToolRuntime[CustomContext, CustomState],
) -> Command:
    """查找并更新用户信息。"""
    user_id = runtime.context.user_id
    name = "John Smith" if user_id == "user_123" else "Unknown user"
    return Command(update={
        "user_name": name,
        # 更新消息历史
        "messages": [
            ToolMessage(
                "Successfully looked up user information",
                tool_call_id=runtime.tool_call_id
            )
        ]
    })

@tool
def greet(
    runtime: ToolRuntime[CustomContext, CustomState]
) -> str | Command:
    """在找到用户信息后用于问候用户。"""
    user_name = runtime.state.get("user_name", None)
    if user_name is None:
       return Command(update={
            "messages": [
                ToolMessage(
                    "Please call the 'update_user_info' tool it will get and update the user's name.",
                    tool_call_id=runtime.tool_call_id
                )
            ]
        })
    return f"Hello {user_name}!"

agent = create_agent(
    model="gpt-5-nano",
    tools=[update_user_info, greet],
    state_schema=CustomState,
    context_schema=CustomContext,
)

agent.invoke(
    {"messages": [{"role": "user", "content": "greet the user"}]},
    context=CustomContext(user_id="user_123"),
)
```

### 提示词(Prompt)

在中间件中访问短期记忆(state),以基于对话历史或自定义 state 字段创建动态提示词。

```python
from langchain.agents import create_agent
from typing import TypedDict
from langchain.agents.middleware import dynamic_prompt, ModelRequest


class CustomContext(TypedDict):
    user_name: str


def get_weather(city: str) -> str:
    """获取某城市的天气。"""
    return f"The weather in {city} is always sunny!"


@dynamic_prompt
def dynamic_system_prompt(request: ModelRequest) -> str:
    user_name = request.runtime.context["user_name"]
    system_prompt = f"You are a helpful assistant. Address the user as {user_name}."
    return system_prompt


agent = create_agent(
    model="gpt-5-nano",
    tools=[get_weather],
    middleware=[dynamic_system_prompt],
    context_schema=CustomContext,
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What is the weather in SF?"}]},
    context=CustomContext(user_name="John Smith"),
)
for msg in result["messages"]:
    msg.pretty_print()
```

输出:

```shell
================================ Human Message =================================

What is the weather in SF?
================================== Ai Message ==================================
Tool Calls:
  get_weather (call_WFQlOGn4b2yoJrv7cih342FG)
 Call ID: call_WFQlOGn4b2yoJrv7cih342FG
  Args:
    city: San Francisco
================================= Tool Message =================================
Name: get_weather

The weather in San Francisco is always sunny!
================================== Ai Message ==================================

Hi John Smith, the weather in San Francisco is always sunny!
```

### Before model

在 [`@before_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_model) 中间件中访问短期记忆(state),在模型调用前处理消息。

```python
from langchain.messages import RemoveMessage
from langgraph.graph.message import REMOVE_ALL_MESSAGES
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import before_model
from langchain_core.runnables import RunnableConfig
from langgraph.runtime import Runtime
from typing import Any


@before_model
def trim_messages(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    """只保留最后几条消息以适配上下文窗口。"""
    messages = state["messages"]

    if len(messages) <= 3:
        return None  # 无需修改

    first_msg = messages[0]
    recent_messages = messages[-3:] if len(messages) % 2 == 0 else messages[-4:]
    new_messages = [first_msg] + recent_messages

    return {
        "messages": [
            RemoveMessage(id=REMOVE_ALL_MESSAGES),
            *new_messages
        ]
    }


agent = create_agent(
    "gpt-5-nano",
    tools=[],
    middleware=[trim_messages],
    checkpointer=InMemorySaver()
)

config: RunnableConfig = {"configurable": {"thread_id": "1"}}

agent.invoke({"messages": "hi, my name is bob"}, config)
agent.invoke({"messages": "write a short poem about cats"}, config)
agent.invoke({"messages": "now do the same but for dogs"}, config)
final_response = agent.invoke({"messages": "what's my name?"}, config)

final_response["messages"][-1].pretty_print()
"""
================================== Ai Message ==================================

Your name is Bob. You told me that earlier.
If you'd like me to call you a nickname or use a different name, just say the word.
"""
```

### After model

在 [`@after_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/after_model) 中间件中访问短期记忆(state),在模型调用后处理消息。

```python
from langchain.messages import RemoveMessage
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent, AgentState
from langchain.agents.middleware import after_model
from langgraph.runtime import Runtime


@after_model
def validate_response(state: AgentState, runtime: Runtime) -> dict | None:
    """移除包含敏感词的消息。"""
    STOP_WORDS = ["password", "secret"]
    last_message = state["messages"][-1]
    if any(word in last_message.content for word in STOP_WORDS):
        return {"messages": [RemoveMessage(id=last_message.id)]}
    return None

agent = create_agent(
    model="gpt-5-nano",
    tools=[],
    middleware=[validate_response],
    checkpointer=InMemorySaver(),
)
```
