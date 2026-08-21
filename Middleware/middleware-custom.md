# 自定义中间件(Custom middleware)

> 通过实现在 Agent 执行流程特定点运行的钩子来构建自定义中间件

> 原文:[Custom middleware](https://docs.langchain.com/oss/python/langchain/middleware/custom)

## 钩子(Hooks)

中间件提供两种风格的钩子来拦截 Agent 执行:

- [Node-style hooks(节点式钩子)](#node-style-hooks节点式钩子):在特定执行点按顺序运行。
- [Wrap-style hooks(包裹式钩子)](#wrap-style-hooks包裹式钩子):围绕每次模型或工具调用运行。

### Node-style hooks(节点式钩子)

在特定执行点按顺序运行。用于日志、校验和状态更新。

选择你的中间件需要的钩子。你可以在节点式钩子和包裹式钩子之间选择。

**节点式钩子**在特定执行点运行:

| 钩子 | 运行时机 |
| -------------- | ------------------------------------------- |
| `before_agent` | Agent 启动前(每次调用一次) |
| `before_model` | 每次模型调用前 |
| `after_model`  | 每次模型响应后 |
| `after_agent`  | Agent 完成后(每次调用一次) |

**包裹式钩子**围绕每次调用运行,让你控制执行过程:

| 钩子 | 运行时机 |
| ----------------- | ---------------------- |
| `wrap_model_call` | 围绕每次模型调用 |
| `wrap_tool_call`  | 围绕每次工具调用 |

**示例(装饰器方式):**

```python
from langchain.agents.middleware import before_model, after_model, AgentState
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any


@before_model(can_jump_to=["end"])
def check_message_limit(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    if len(state["messages"]) >= 50:
        return {
            "messages": [AIMessage("Conversation limit reached.")],
            "jump_to": "end"
        }
    return None

@after_model
def log_response(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"Model returned: {state['messages'][-1].content}")
    return None
```

**示例(类方式):**

```python
from langchain.agents.middleware import AgentMiddleware, AgentState, hook_config
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any

class MessageLimitMiddleware(AgentMiddleware):
    def __init__(self, max_messages: int = 50):
        super().__init__()
        self.max_messages = max_messages

    @hook_config(can_jump_to=["end"])
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        if len(state["messages"]) >= self.max_messages:
            return {
                "messages": [AIMessage("Conversation limit reached.")],
                "jump_to": "end"
            }
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"Model returned: {state['messages'][-1].content}")
        return None
```

### Wrap-style hooks(包裹式钩子)

拦截执行并控制 handler 何时被调用。用于重试、缓存和转换。

你决定 handler 被调用零次(短路)、一次(正常流程)还是多次(重试逻辑)。

**可用钩子:**

- `wrap_model_call` —— 围绕每次模型调用
- `wrap_tool_call` —— 围绕每次工具调用

**示例(装饰器方式):**

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable


@wrap_model_call
def retry_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    for attempt in range(3):
        try:
            return handler(request)
        except Exception as e:
            if attempt == 2:
                raise
            print(f"Retry {attempt + 1}/3 after error: {e}")
```

**示例(类方式):**

```python
from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
from typing import Callable

class RetryMiddleware(AgentMiddleware):
    def __init__(self, max_retries: int = 3):
        super().__init__()
        self.max_retries = max_retries

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        for attempt in range(self.max_retries):
            try:
                return handler(request)
            except Exception as e:
                if attempt == self.max_retries - 1:
                    raise
                print(f"Retry {attempt + 1}/{self.max_retries} after error: {e}")
```

## 状态更新(State updates)

节点式和包裹式钩子都可以更新 Agent 状态。机制有所不同:

- **节点式钩子**(`before_agent`、`before_model`、`after_model`、`after_agent`):直接返回一个 dict。该 dict 使用图的 reducer 应用到 Agent 状态上。
- **包裹式钩子**(`wrap_model_call`、`wrap_tool_call`):对于模型调用,返回带 [`Command`](https://reference.langchain.com/python/langgraph/types/Command) 的 [`ExtendedModelResponse`](https://reference.langchain.com/python/langchain/agents/middleware/types/ExtendedModelResponse),把状态更新与模型响应一起注入。对于工具调用,直接返回 [`Command`](https://reference.langchain.com/python/langgraph/types/Command)。当你需要基于模型或工具调用期间运行的逻辑来跟踪或更新状态时使用它们,例如摘要触发点、用量元数据,或从请求/响应计算出的自定义字段。

### 节点式钩子

从节点式钩子返回一个 dict,将更新合并进 Agent 状态。dict 的键对应 state 字段。

```python
from langchain.agents.middleware import after_model, AgentState
from langgraph.runtime import Runtime
from typing import Any
from typing_extensions import NotRequired


class TrackingState(AgentState):
    model_call_count: NotRequired[int]


@after_model(state_schema=TrackingState)
def increment_after_model(state: TrackingState, runtime: Runtime) -> dict[str, Any] | None:
    return {"model_call_count": state.get("model_call_count", 0) + 1}
```

### 包裹式钩子

从 `wrap_model_call` 返回带 [`Command`](https://reference.langchain.com/python/langgraph/types/Command) 的 [`ExtendedModelResponse`](https://reference.langchain.com/python/langchain/agents/middleware/types/ExtendedModelResponse),从模型调用层注入状态更新:

```python
from typing import Callable
from langchain.agents.middleware import (
    wrap_model_call,
    ModelRequest,
    ModelResponse,
    AgentState,
    ExtendedModelResponse
)
from langgraph.types import Command
from typing_extensions import NotRequired

class UsageTrackingState(AgentState):
    """带 token 用量跟踪的 Agent 状态。"""

    last_model_call_tokens: NotRequired[int]


@wrap_model_call(state_schema=UsageTrackingState)
def track_usage(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ExtendedModelResponse:
    response = handler(request)
    return ExtendedModelResponse(
        model_response=response,
        command=Command(update={"last_model_call_tokens": 150}),
    )
```

[`Command`](https://reference.langchain.com/python/langgraph/types/Command) 会流经图的 reducer,因此更新会被正确应用,消息是追加的而不是替换现有状态。

#### 多个中间件的组合

当多个中间件层都返回 `ExtendedModelResponse` 时,它们的 command 会组合起来:

- **Command 通过 reducer 应用:**每个 `Command` 成为一个独立的状态更新。对于消息,这意味着它们是追加的。
- **冲突时外层优先:**对于非 reducer 的 state 字段,command 按先内后外的顺序应用。最外层中间件的值在冲突键上优先。
- **重试安全:**如果外层中间件实现的逻辑可能导致再次多次调用 `handler()`(例如重试逻辑),较早调用产生的 command 会被丢弃。

```python
from typing import Annotated, Callable

from langchain.agents.middleware import (
    AgentMiddleware,
    AgentState,
    ExtendedModelResponse,
    ModelRequest,
    ModelResponse,
)
from langchain.messages import SystemMessage
from langgraph.types import Command
from typing_extensions import NotRequired


def _last_wins(_a: str, b: str) -> str:
    """Reducer:最后写入者获胜(外层覆盖内层)。"""
    return b


class CustomMiddlewareState(AgentState):
    """Agent 状态:trace_layer 使用 last-wins(外层获胜),messages 使用追加 reducer。"""

    # 带 last-wins 的非 reducer 字段:两个中间件都写入;最外层的值获胜
    trace_layer: NotRequired[Annotated[str, _last_wins]]


class OuterMiddleware(AgentMiddleware):
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ExtendedModelResponse:
        response = handler(request)
        return ExtendedModelResponse(
            model_response=response,
            command=Command(update={
                "trace_layer": "outer",
                "messages": [SystemMessage(content="[Outer ran]")],
            }),
        )


class InnerMiddleware(AgentMiddleware):
    """添加 trace_layer 和消息。外层对相同键追加;trace_layer:外层获胜,messages:追加。"""

    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ):
        response = handler(request)
        return ExtendedModelResponse(
            model_response=response,
            command=Command(update={
                "trace_layer": "inner",
                "messages": [SystemMessage(content="[Inner ran]")],
            }),
        )
```

## 创建中间件

你可以通过两种方式创建中间件:

- [基于装饰器的中间件](#基于装饰器的中间件decorator-based-middleware):适合单钩子中间件,快速简单。使用装饰器包装单个函数。
- [基于类的中间件](#基于类的中间件class-based-middleware):对带多个钩子或配置的复杂中间件更强大。

### 基于装饰器的中间件(Decorator-based middleware)

适合单钩子中间件,快速简单。使用装饰器包装单个函数。

**可用装饰器:**

**节点式:**

- [`@before_agent`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_agent) —— Agent 启动前运行(每次调用一次)
- [`@before_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/before_model) —— 每次模型调用前运行
- [`@after_model`](https://reference.langchain.com/python/langchain/agents/middleware/types/after_model) —— 每次模型响应后运行
- [`@after_agent`](https://reference.langchain.com/python/langchain/agents/middleware/types/after_agent) —— Agent 完成后运行(每次调用一次)

**包裹式:**

- [`@wrap_model_call`](https://reference.langchain.com/python/langchain/agents/middleware/types/wrap_model_call) —— 用自定义逻辑包裹每次模型调用
- [`@wrap_tool_call`](https://reference.langchain.com/python/langchain/agents/middleware/types/wrap_tool_call) —— 用自定义逻辑包裹每次工具调用

**便捷装饰器:**

- [`@dynamic_prompt`](https://reference.langchain.com/python/langchain/agents/middleware/types/dynamic_prompt) —— 生成动态系统提示词

**示例:**

```python
from langchain.agents.middleware import (
    before_model,
    wrap_model_call,
    AgentState,
    ModelRequest,
    ModelResponse,
)
from langchain.agents import create_agent
from langgraph.runtime import Runtime
from typing import Any, Callable


@before_model
def log_before_model(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"About to call model with {len(state['messages'])} messages")
    return None

@wrap_model_call
def retry_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    for attempt in range(3):
        try:
            return handler(request)
        except Exception as e:
            if attempt == 2:
                raise
            print(f"Retry {attempt + 1}/3 after error: {e}")

agent = create_agent(
    model="gpt-5.5",
    middleware=[log_before_model, retry_model],
    tools=[...],
)
```

**何时使用装饰器:**

- 只需要单个钩子
- 没有复杂配置
- 快速原型验证

### 基于类的中间件(Class-based middleware)

对带多个钩子或配置的复杂中间件更强大。当你需要为同一钩子定义同步和异步两种实现,或想在单个中间件中组合多个钩子时,使用类。

`AgentMiddleware` 子类可以声明三个类属性,Agent 工厂会在编译时拾取它们:

- `state_schema` —— 用自定义字段扩展 Agent 状态。参见[自定义 state schema](#自定义-state-schemacustom-state-schema)。
- `tools` —— 注册随中间件一起提供的额外工具(例如待办清单中间件上的 `write_todos`)。
- `transformers` —— 注册作用域感知的流转换器工厂。参见[自定义流转换器](#自定义流转换器custom-stream-transformers)。

**示例:**

```python
from langchain.agents.middleware import (
    AgentMiddleware,
    AgentState,
    ModelRequest,
    ModelResponse,
)
from langgraph.runtime import Runtime
from typing import Any, Callable

class LoggingMiddleware(AgentMiddleware):
    def before_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"About to call model with {len(state['messages'])} messages")
        return None

    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        print(f"Model returned: {state['messages'][-1].content}")
        return None

    async def abefore_model(
        self, state: AgentState, runtime: Runtime
    ) -> dict[str, Any] | None:
        # before_model 的异步版本
        return None

    async def aafter_model(
        self, state: AgentState, runtime: Runtime
    ) -> dict[str, Any] | None:
        # after_model 的异步版本
        print(f"Model returned: {state['messages'][-1].content}")
        return None


agent = create_agent(
    model="gpt-5.5",
    middleware=[LoggingMiddleware()],
    tools=[...],
)
```

**何时使用类:**

- 为同一钩子定义同步和异步两种实现
- 单个中间件需要多个钩子
- 需要复杂配置(例如可配置的阈值、自定义模型)
- 带初始化时配置的跨项目复用

## 自定义 state schema(Custom state schema)

如果你的中间件需要在钩子之间跟踪状态,中间件可以用自定义属性扩展 Agent 的状态。这让中间件能够:

- **跨执行跟踪状态**:维护在 Agent 整个执行生命周期中持续存在的计数器、标志或其他值
- **在钩子之间共享数据**:把信息从 `before_model` 传给 `after_model`,或在不同中间件实例之间传递
- **实现横切关注点**:在不修改核心 Agent 逻辑的情况下添加速率限制、用量跟踪、用户上下文或审计日志等功能
- **做出条件决策**:使用累积的状态决定是继续执行、跳转到不同节点,还是动态修改行为

**示例(装饰器方式):**

```python
from langchain.agents import create_agent
from langchain.messages import HumanMessage
from langchain.agents.middleware import AgentState, before_model, after_model
from typing_extensions import NotRequired
from typing import Any
from langgraph.runtime import Runtime


class CustomState(AgentState):
    model_call_count: NotRequired[int]
    user_id: NotRequired[str]


@before_model(state_schema=CustomState, can_jump_to=["end"])
def check_call_limit(state: CustomState, runtime: Runtime) -> dict[str, Any] | None:
    count = state.get("model_call_count", 0)
    if count > 10:
        return {"jump_to": "end"}
    return None


@after_model(state_schema=CustomState)
def increment_counter(state: CustomState, runtime: Runtime) -> dict[str, Any] | None:
    return {"model_call_count": state.get("model_call_count", 0) + 1}


agent = create_agent(
    model="gpt-5.5",
    middleware=[check_call_limit, increment_counter],
    tools=[],
)

# 带自定义 state 调用
result = agent.invoke({
    "messages": [HumanMessage("Hello")],
    "model_call_count": 0,
    "user_id": "user-123",
})
```

**示例(类方式):**

```python
from langchain.agents import create_agent
from langchain.messages import HumanMessage
from langchain.agents.middleware import AgentState, AgentMiddleware
from typing_extensions import NotRequired
from typing import Any


class CustomState(AgentState):
    model_call_count: NotRequired[int]
    user_id: NotRequired[str]


class CallCounterMiddleware(AgentMiddleware[CustomState]):
    state_schema = CustomState

    def before_model(self, state: CustomState, runtime) -> dict[str, Any] | None:
        count = state.get("model_call_count", 0)
        if count > 10:
            return {"jump_to": "end"}
        return None

    def after_model(self, state: CustomState, runtime) -> dict[str, Any] | None:
        return {"model_call_count": state.get("model_call_count", 0) + 1}


agent = create_agent(
    model="gpt-5.5",
    middleware=[CallCounterMiddleware()],
    tools=[],
)

# 带自定义 state 调用
result = agent.invoke({
    "messages": [HumanMessage("Hello")],
    "model_call_count": 0,
    "user_id": "user-123",
})
```

## 自定义流转换器(Custom stream transformers)

> **注意**:中间件注册的转换器需要 `langchain>=1.3.2`。

中间件可以注册流转换器工厂,把实时 Agent 流中的事件投影到类型化的扩展通道上。这对于呈现计数器、旁路通道产物、部分输出或线路级编辑很有用,且无需与框架内置投影耦合。

在编译时,中间件注册的工厂会与调用方直接传给 Agent 工厂的内容合并。[最终排序规则](https://docs.langchain.com/oss/python/langchain/event-streaming#在中间件上注册转换器)让内置的 `ToolCallTransformer` 保持在最前面,让调用方提供的条目落在最后。

把 `transformers` 类属性设置为工厂可调用对象的元组。每个工厂的形式为 `Callable[[tuple[str, ...]], StreamTransformer]`,并以 `factory(scope)` 方式调用,其中 `scope` 是 mini-mux 作用域元组(根为 `()`,子图为非空);每次调用返回一个全新的转换器,让每个子图保持隔离。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware


class ToolActivityMiddleware(AgentMiddleware):
    transformers = (ToolActivityTransformer,)


agent = create_agent(
    model="gpt-5-nano",
    tools=[...],
    middleware=[ToolActivityMiddleware()],
)
```

完整的排序规则和 PII 编辑示例参见[在中间件上注册转换器](https://docs.langchain.com/oss/python/langchain/event-streaming#在中间件上注册转换器)。

## 执行顺序(Execution order)

使用多个中间件时,理解它们的执行方式:

```python
agent = create_agent(
    model="gpt-5.5",
    middleware=[middleware1, middleware2, middleware3],
    tools=[...],
)
```

**执行流程:**

**before 钩子按顺序运行:**

1. `middleware1.before_agent()`
2. `middleware2.before_agent()`
3. `middleware3.before_agent()`

**Agent 循环开始**

4. `middleware1.before_model()`
5. `middleware2.before_model()`
6. `middleware3.before_model()`

**wrap 钩子像函数调用一样嵌套:**

7. `middleware1.wrap_model_call()` → `middleware2.wrap_model_call()` → `middleware3.wrap_model_call()` → 模型

**after 钩子按逆序运行:**

8. `middleware3.after_model()`
9. `middleware2.after_model()`
10. `middleware1.after_model()`

**Agent 循环结束**

11. `middleware3.after_agent()`
12. `middleware2.after_agent()`
13. `middleware1.after_agent()`

**关键规则:**

- `before_*` 钩子:从前到后
- `after_*` 钩子:从后到前(逆序)
- `wrap_*` 钩子:嵌套(第一个中间件包裹所有其他中间件)

## Agent 跳转(Agent jumps)

要从中间件提前退出,返回一个带 `jump_to` 的字典:

**可用的跳转目标:**

- `'end'`:跳转到 Agent 执行的末尾(或第一个 `after_agent` 钩子)
- `'tools'`:跳转到工具节点
- `'model'`:跳转到模型节点(或第一个 `before_model` 钩子)

**示例(装饰器方式):**

```python
from langchain.agents.middleware import after_model, hook_config, AgentState
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any


@after_model
@hook_config(can_jump_to=["end"])
def check_for_blocked(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    last_message = state["messages"][-1]
    if "BLOCKED" in last_message.content:
        return {
            "messages": [AIMessage("I cannot respond to that request.")],
            "jump_to": "end"
        }
    return None
```

**示例(类方式):**

```python
from langchain.agents.middleware import AgentMiddleware, hook_config, AgentState
from langchain.messages import AIMessage
from langgraph.runtime import Runtime
from typing import Any

class BlockedContentMiddleware(AgentMiddleware):
    @hook_config(can_jump_to=["end"])
    def after_model(self, state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
        last_message = state["messages"][-1]
        if "BLOCKED" in last_message.content:
            return {
                "messages": [AIMessage("I cannot respond to that request.")],
                "jump_to": "end"
            }
        return None
```

## 配置追踪(Configure tracing)

> **注意**:需要 `langchain>=1.3.15`。

中间件钩子的 span 默认会追踪其输入和输出。设置 `trace_policy` 可以控制它们记录什么。`TracePolicy` 接受如 `process_inputs` 和 `process_outputs` 这样的可调用对象来转换被追踪的值;`omit_payload` 则完全丢弃它。当完整消息历史对中间件的功能没有信息量时,这可作为一种优化。

要从中间件的追踪中省略输入载荷:

```python
from langchain.agents.middleware import AgentMiddleware, TracePolicy, omit_payload

class MyMiddleware(AgentMiddleware):
    trace_policy = TracePolicy(process_inputs=omit_payload)
```

要对所有中间件应用策略,配置一个全局默认值:

```python
from langchain.agents.middleware import configure_trace_policy, TracePolicy, omit_payload

configure_trace_policy(TracePolicy(process_inputs=omit_payload))  # 传 None 可清除
```

中间件自己的 `trace_policy` 会覆盖全局默认值。

## 最佳实践(Best practices)

1. 保持中间件聚焦——每个中间件只把一件事做好
2. 优雅地处理错误——不要让中间件错误导致 Agent 崩溃
3. **使用合适的钩子类型**:
   - 顺序逻辑用节点式(日志、校验)
   - 控制流用包裹式(重试、回退、缓存)
4. 清晰地记录任何自定义 state 属性
5. 集成前先独立对中间件做单元测试
6. 考虑执行顺序——把关键中间件放在列表最前面
7. 尽可能使用内置中间件

## 示例(Examples)

### 动态提示词(Dynamic prompt)

在运行时动态修改系统提示词,在每次模型调用前注入上下文、用户特定指令或其他信息。这是最常见的中间件使用场景之一。

使用 `ModelRequest` 上的 `system_message` 字段读取和修改系统提示词。它包含一个 [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) 对象(即使 Agent 是用字符串 `system_prompt` 创建的)。

**示例(装饰器方式):**

```python
from collections.abc import Callable

from langchain.agents.middleware import ModelRequest, ModelResponse, wrap_model_call
from langchain.messages import SystemMessage


@wrap_model_call
def add_context(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    new_content = list(request.system_message.content_blocks) + [
        {"type": "text", "text": "Additional context."}
    ]
    new_system_message = SystemMessage(content=new_content)
    return handler(request.override(system_message=new_system_message))
```

**示例(类方式):**

```python
from collections.abc import Callable

from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse


class ContextMiddleware(AgentMiddleware):
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        new_content = list(request.system_message.content_blocks) + [
            {"type": "text", "text": "Additional context."}
        ]
        new_system_message = SystemMessage(content=new_content)
        return handler(request.override(system_message=new_system_message))
```

> **注意**:
>
> - 即使 Agent 是用 `system_prompt="string"` 创建的,`ModelRequest.system_message` 也始终是 [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) 对象
> - 无论原始内容是字符串还是列表,都使用 `SystemMessage.content_blocks` 把内容作为块列表访问
> - 修改系统消息时,使用 `content_blocks` 并追加新块,以保留现有结构
> - 对于缓存控制等高级场景,你可以直接把 [`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) 对象传给 `create_agent` 的 `system_prompt` 参数

### 动态模型选择(Dynamic model selection)

**示例(装饰器方式):**

```python
from collections.abc import Callable

from langchain.agents.middleware import ModelRequest, ModelResponse, wrap_model_call
from langchain.chat_models import init_chat_model

complex_model = init_chat_model("claude-sonnet-4-6")
simple_model = init_chat_model("claude-haiku-4-5-20251001")


@wrap_model_call
def dynamic_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    if len(request.messages) > 10:
        model = complex_model
    else:
        model = simple_model
    return handler(request.override(model=model))
```

**示例(类方式):**

```python
from collections.abc import Callable

from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
from langchain.chat_models import init_chat_model

complex_model = init_chat_model("claude-sonnet-4-6")
simple_model = init_chat_model("claude-haiku-4-5-20251001")


class DynamicModelMiddleware(AgentMiddleware):
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        if len(request.messages) > 10:
            model = complex_model
        else:
            model = simple_model
        return handler(request.override(model=model))
```

### 动态选择工具(Dynamically selecting tools)

在运行时选择相关工具以提升性能和准确性。本节介绍过滤预注册的工具。关于注册在运行时发现的工具(例如来自 MCP 服务器),参见[运行时工具注册](https://docs.langchain.com/oss/python/langchain/tools#动态工具选择dynamic-tool-selection)。

**好处:**

- **更短的提示词** —— 只暴露相关工具,降低复杂度
- **更高的准确性** —— 模型从更少的选项中做出更正确的选择
- **权限控制** —— 基于用户访问权限动态过滤工具

**示例(装饰器方式):**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable


@wrap_model_call
def select_tools(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    """根据 state/context 选择相关工具的中间件。"""
    # 根据 state/context 选择一个小的、相关的工具子集
    relevant_tools = select_relevant_tools(request.state, request.runtime)
    return handler(request.override(tools=relevant_tools))

agent = create_agent(
    model="gpt-5.5",
    tools=all_tools,  # 所有可用工具需要预先注册
    middleware=[select_tools],
)
```

**示例(类方式):**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
from typing import Callable


class ToolSelectorMiddleware(AgentMiddleware):
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        """根据 state/context 选择相关工具的中间件。"""
        # 根据 state/context 选择一个小的、相关的工具子集
        relevant_tools = select_relevant_tools(request.state, request.runtime)
        return handler(request.override(tools=relevant_tools))

agent = create_agent(
    model="gpt-5.5",
    tools=all_tools,  # 所有可用工具需要预先注册
    middleware=[ToolSelectorMiddleware()],
)
```

### 工具调用监控(Tool call monitoring)

**示例(装饰器方式):**

```python
from collections.abc import Callable

from langchain.agents.middleware import wrap_tool_call
from langchain.messages import ToolMessage
from langchain.tools.tool_node import ToolCallRequest
from langgraph.types import Command


@wrap_tool_call
def monitor_tool(
    request: ToolCallRequest,
    handler: Callable[[ToolCallRequest], ToolMessage | Command],
) -> ToolMessage | Command:
    print(f"Executing tool: {request.tool_call['name']}")
    print(f"Arguments: {request.tool_call['args']}")
    try:
        result = handler(request)
        print("Tool completed successfully")
        return result
    except Exception as e:
        print(f"Tool failed: {e}")
        raise
```

**示例(类方式):**

```python
from collections.abc import Callable

from langchain.agents.middleware import AgentMiddleware
from langchain.messages import ToolMessage
from langchain.tools.tool_node import ToolCallRequest
from langgraph.types import Command


class ToolMonitoringMiddleware(AgentMiddleware):
    def wrap_tool_call(
        self,
        request: ToolCallRequest,
        handler: Callable[[ToolCallRequest], ToolMessage | Command],
    ) -> ToolMessage | Command:
        print(f"Executing tool: {request.tool_call['name']}")
        print(f"Arguments: {request.tool_call['args']}")
        try:
            result = handler(request)
            print("Tool completed successfully")
            return result
        except Exception as e:
            print(f"Tool failed: {e}")
            raise
```

### 提示词缓存(Anthropic)

使用 Anthropic 模型时,可以使用带缓存控制指令(cache control directives)的结构化内容块来缓存大型系统提示词:

**示例(装饰器方式):**

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.messages import SystemMessage
from typing import Callable


@wrap_model_call
def add_cached_context(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    # 始终使用内容块进行操作
    new_content = list(request.system_message.content_blocks) + [
        {
            "type": "text",
            "text": "Here is a large document to analyze:\n\n<document>...</document>",
            # 到此为止的内容会被缓存
            "cache_control": {"type": "ephemeral"}
        }
    ]

    new_system_message = SystemMessage(content=new_content)
    return handler(request.override(system_message=new_system_message))
```

**示例(类方式):**

```python
from langchain.agents.middleware import AgentMiddleware, ModelRequest, ModelResponse
from langchain.messages import SystemMessage
from typing import Callable


class CachedContextMiddleware(AgentMiddleware):
    def wrap_model_call(
        self,
        request: ModelRequest,
        handler: Callable[[ModelRequest], ModelResponse],
    ) -> ModelResponse:
        # 始终使用内容块进行操作
        new_content = list(request.system_message.content_blocks) + [
            {
                "type": "text",
                "text": "Here is a large document to analyze:\n\n<document>...</document>",
                "cache_control": {"type": "ephemeral"}  # 此内容会被缓存
            }
        ]

        new_system_message = SystemMessage(content=new_content)
        return handler(request.override(system_message=new_system_message))
```

**说明:**

- 即使 Agent 是用 `system_prompt="string"` 创建的,`ModelRequest.system_message` 也始终是 `SystemMessage` 对象
- 无论原始内容是字符串还是列表,都使用 `SystemMessage.content_blocks` 把内容作为块列表访问
- 修改系统消息时,使用 `content_blocks` 并追加新块,以保留现有结构
- 对于缓存控制等高级场景,你可以直接把 `SystemMessage` 对象传给 `create_agent` 的 `system_prompt` 参数

## 更多资源

- [中间件 API 参考](https://reference.langchain.com/python/langchain/middleware/)
- [内置中间件](https://docs.langchain.com/oss/python/langchain/middleware/built-in)
- [测试 Agent](https://docs.langchain.com/oss/python/langchain/test/)
