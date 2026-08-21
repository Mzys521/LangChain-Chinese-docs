# 预构建中间件(Prebuilt middleware)

> 面向常见 Agent 使用场景的预构建中间件

> 原文:[Prebuilt middleware](https://docs.langchain.com/oss/python/langchain/middleware/built-in)

LangChain 和 [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) 为常见使用场景提供了预构建中间件。每个中间件都可用于生产环境,并可根据你的具体需求进行配置。

## 与提供商无关的中间件(Provider-agnostic middleware)

以下中间件适用于任何 LLM 提供商:

| 中间件 | 说明 |
| --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [Tool error(工具错误)](#tool-error工具错误) | 捕获工具执行异常,并将其转换为模型可见的错误消息。 |
| [Tool retry(工具重试)](#tool-retry工具重试) | 以指数退避自动重试失败的工具调用。 |
| [Model retry(模型重试)](#model-retry模型重试) | 以指数退避自动重试失败的模型调用。 |
| [Model fallback(模型回退)](#model-fallback模型回退) | 主模型失败时自动回退到备选模型。 |
| [Summarization(摘要)](#summarization摘要) | 接近 token 上限时自动摘要对话历史。 |
| [Human-in-the-loop(人机协同)](#human-in-the-loop人机协同) | 暂停执行,等待人工审批工具调用。 |
| [Model call limit(模型调用限制)](#model-call-limit模型调用限制) | 限制模型调用次数,防止成本过高。 |
| [Tool call limit(工具调用限制)](#tool-call-limit工具调用限制) | 通过限制调用次数来控制工具执行。 |
| [PII detection(PII 检测)](#pii-detectionpii-检测) | 检测并处理个人敏感信息(PII)。 |
| [To-do list(待办清单)](#to-do-list待办清单) | 为 Agent 配备任务规划与跟踪能力。 |
| [LLM tool selector(LLM 工具选择器)](#llm-tool-selectorllm-工具选择器) | 在调用主模型之前,用 LLM 选择相关工具。 |
| [Provider tool search(提供商工具搜索)](#provider-tool-search提供商工具搜索) | 把工具交给提供商的服务器端工具搜索,按需呈现。 |
| [Shell tool(Shell 工具)](#shell-toolshell-工具) | 向 Agent 暴露持久化 shell 会话以执行命令。 |
| [Filesystem(文件系统)](#filesystem-middleware文件系统中间件) | 为 Agent 提供文件系统,用于存储上下文和长期记忆。 |
| [Subagent(子 Agent)](#subagent子-agent) | 添加派生子 Agent 的能力。 |
| [Rubric grading(评分准则,Beta)](#rubric-grading评分准则) | 应用 LLM 评审式打分,让 Agent 自评并迭代,直到满足准则。 |
| [File search(文件搜索)](#file-search文件搜索) | 提供针对文件系统文件的 Glob 和 Grep 搜索工具。 |
| [Context editing(上下文编辑)](#context-editing上下文编辑) | 通过裁剪或清除工具使用记录来管理对话上下文。 |
| [LLM tool emulator(LLM 工具模拟器)](#llm-tool-emulatorllm-工具模拟器) | 用 LLM 模拟工具执行,用于测试目的。 |

### Tool error(工具错误)

捕获工具执行期间抛出的异常,将其转换为模型可见并可从中恢复的错误 `ToolMessage`,而不是中止 Agent 运行。工具错误中间件在以下场景中很有用:

- 让模型用修正后的参数重试失败的工具调用。
- 呈现受控的、经过清洗的错误消息,而不是原始异常细节。
- 防止意外的工具异常导致 Agent 崩溃。

> **注意**:工具错误中间件不会自动重试失败的调用。如需重试,请与 [Tool retry](#tool-retry工具重试) 中间件组合使用,将其放在*内层*(`middleware` 列表中更靠前的位置),并配置 `on_failure="error"`,以便异常能传递到工具错误中间件。参见下面的[完整示例](#tool-error-完整示例)。

**API 参考:** [`ToolErrorMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_error/ToolErrorMiddleware)

> **注意**:`ToolErrorMiddleware` 需要 `langchain>=1.3.14`。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolErrorMiddleware


def on_error(exc: Exception, request: ToolCallRequest) -> str | None:
    if isinstance(exc, ValueError):
        return f"`{request.tool_call['name']}` failed with {type(exc).__name__}."
    # 其余异常全部向上传播


agent = create_agent(
    model="gpt-5.5",
    tools=[your_tools],
    middleware=[ToolErrorMiddleware(on_error)],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `on_error` | `Callable[[Exception, ToolCallRequest], str \| list[ContentBlock] \| None]` | 针对工具执行抛出的每个异常调用的同步处理器。返回内容(`str` 或内容块列表)会把异常转换为 `ToolMessage(status="error")`。返回 `None` 或省略 return 语句则让异常向上传播。用于同步路径,在未提供 `aon_error` 时也用于异步路径。 |
| `aon_error` | `Callable[[Exception, ToolCallRequest], Awaitable[str \| list[ContentBlock] \| None]]` | 可选的异步处理器,用于异步执行路径。未提供时回退到 `on_error`。 |
| `tools` | `list[BaseTool \| str]` | 可选的应用错误处理的工具或工具名列表。为 `None` 时应用于所有工具。 |

#### Tool error 完整示例

`on_error` 处理器接收异常和 `ToolCallRequest`(包含带 name、args 和 call ID 的工具调用字典)。对你不想处理的异常返回 `None`,它们会正常向上传播。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolErrorMiddleware, ToolRetryMiddleware


def on_error(exc: Exception, request: ToolCallRequest) -> str | None:
    # 把 ValueError 呈现给模型,让它修正输入
    if isinstance(exc, ValueError):
        return f"`{request.tool_call['name']}` failed: {type(exc).__name__}. Fix the input and retry."
    # 让所有其他异常向上传播(中止运行)
    return None


# 仅异步用法
async def aon_error(exc: Exception, request: ToolCallRequest) -> str | None:
    if isinstance(exc, ConnectionError):
        return f"Tool `{request.tool_call['name']}` encountered a connection error."
    return None


agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool],
    middleware=[
        # 把 retry 放在内层,使异常在重试耗尽后到达 ToolErrorMiddleware
        ToolRetryMiddleware(max_retries=3, on_failure="error"),
        ToolErrorMiddleware(on_error=on_error, tools=["search_tool"]),
    ],
)

# 仅异步:只传 aon_error(不要传 on_error)
async_agent = create_agent(
    model="gpt-5.5",
    tools=[api_tool],
    middleware=[ToolErrorMiddleware(aon_error=aon_error)],
)
```

> **注意**:建议返回指明异常类型的内容,而不是原始异常消息——后者可能携带敏感或内部细节。`on_error` 处理器控制信息披露:除非你选择包含,原始异常消息永远不会发送给模型。

### Tool retry(工具重试)

以可配置的指数退避自动重试失败的工具调用。工具重试在以下场景中很有用:

- 处理外部 API 调用的瞬时故障。
- 提升依赖网络的工具的可靠性。
- 构建能优雅处理临时错误的韧性 Agent。

**API 参考:** [`ToolRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_retry/ToolRetryMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolRetryMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_retries` | number | 2 | 首次调用之后的最大重试次数(默认共 3 次尝试) |
| `tools` | `list[BaseTool \| str]` | — | 可选的应用重试逻辑的工具或工具名列表。为 `None` 时应用于所有工具。 |
| `retry_on` | `tuple[type[Exception], ...] \| callable` | default_retry_on | 可以是要重试的异常类型元组,也可以是接收异常并返回 `True` 表示应重试的可调用对象。在 `langchain>=1.3.16` 中,默认重试可重试的[模型错误](https://docs.langchain.com/oss/python/langchain/models#model-exceptions)和所有未分类异常,不再重试标记为不可重试的模型错误。 |
| `on_failure` | `string \| callable` | `continue` | 重试耗尽时的行为:`'continue'`(默认)——返回带错误详情的 `ToolMessage`,让 LLM 处理失败;`'error'`——重新抛出异常,停止 Agent 执行;自定义可调用对象——接收异常并返回 `ToolMessage` 内容字符串的函数。**已弃用的值:** `'return_message'`(请改用 `'continue'`)和 `'raise'`(请改用 `'error'`)。 |
| `backoff_factor` | number | 2.0 | 指数退避乘数。每次重试等待 `initial_delay * (backoff_factor ** retry_number)` 秒。设为 `0.0` 则为恒定延迟。 |
| `initial_delay` | number | 1.0 | 首次重试前的初始延迟(秒) |
| `max_delay` | number | 60.0 | 重试之间的最大延迟(秒)(限制指数退避的增长) |
| `jitter` | boolean | true | 是否给延迟添加随机抖动(`±25%`)以避免惊群效应 |

**完整示例:**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolRetryMiddleware


agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool, api_tool],
    middleware=[
        ToolRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
            max_delay=60.0,
            jitter=True,
            tools=["api_tool"],
            retry_on=(ConnectionError, TimeoutError),
            on_failure="continue",
        ),
    ],
)
```

### Model retry(模型重试)

以可配置的指数退避自动重试失败的模型调用。模型重试在以下场景中很有用:

- 处理模型 API 调用的瞬时故障。
- 提升依赖网络的模型请求的可靠性。
- 构建能优雅处理临时模型错误的韧性 Agent。

**API 参考:** [`ModelRetryMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_retry/ModelRetryMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelRetryMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool],
    middleware=[
        ModelRetryMiddleware(
            max_retries=3,
            backoff_factor=2.0,
            initial_delay=1.0,
        ),
    ],
)
```

**配置选项:**与 Tool retry 类似:

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `max_retries` | number | 2 | 首次调用之后的最大重试次数 |
| `retry_on` | `tuple \| callable` | default_retry_on | 要重试的异常类型元组,或判断是否重试的可调用对象 |
| `on_failure` | `string \| callable` | `continue` | `'continue'`(默认)——返回带错误详情的 `AIMessage`,让 Agent 有机会优雅处理失败;`'error'`——重新抛出异常;自定义可调用对象——接收异常并返回 `AIMessage` 内容字符串的函数 |
| `backoff_factor` | number | 2.0 | 指数退避乘数。设为 `0.0` 则为恒定延迟 |
| `initial_delay` | number | 1.0 | 首次重试前的初始延迟(秒) |
| `max_delay` | number | 60.0 | 重试之间的最大延迟(秒) |
| `jitter` | boolean | true | 是否添加随机抖动(`±25%`)以避免惊群效应 |

**完整示例:**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelRetryMiddleware


# 使用默认设置的基本用法(2 次重试,指数退避)
agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool],
    middleware=[ModelRetryMiddleware()],
)

# 只重试特定异常
retry = ModelRetryMiddleware(
    max_retries=4,
    retry_on=(TimeoutError, ConnectionError),
    backoff_factor=1.5,
)


def should_retry(error: Exception) -> bool:
    # 只在速率限制错误时重试
    if isinstance(error, TimeoutError):
        return True
    # 或者检查特定 HTTP 状态码
    if hasattr(error, "status_code"):
        return error.status_code in (429, 503)
    return False

retry_with_filter = ModelRetryMiddleware(
    max_retries=3,
    retry_on=should_retry,
)

# 返回错误消息而不是抛出异常
retry_continue = ModelRetryMiddleware(
    max_retries=4,
    on_failure="continue",  # 返回带错误的 AIMessage 而不是抛出
)

# 自定义错误消息格式
def format_error(error: Exception) -> str:
    return f"Model call failed: {error}. Please try again later."

retry_with_formatter = ModelRetryMiddleware(
    max_retries=4,
    on_failure=format_error,
)

# 恒定退避(无指数增长)
constant_backoff = ModelRetryMiddleware(
    max_retries=5,
    backoff_factor=0.0,  # 无指数增长
    initial_delay=2.0,  # 始终等待 2 秒
)

# 失败时抛出异常
strict_retry = ModelRetryMiddleware(
    max_retries=2,
    on_failure="error",  # 重新抛出异常而不是返回消息
)
```

### Model fallback(模型回退)

主模型失败时自动回退到备选模型。模型回退在以下场景中很有用:

- 构建能应对模型宕机的韧性 Agent。
- 通过回退到更便宜的模型进行成本优化。
- 跨 OpenAI、Anthropic 等的提供商冗余。

**API 参考:** [`ModelFallbackMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_fallback/ModelFallbackMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelFallbackMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        ModelFallbackMiddleware(
            "gpt-5.4-mini",
            "claude-3-5-sonnet-20241022",
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `first_model`(必填) | `string \| BaseChatModel` | 主模型失败时尝试的第一个回退模型。可以是模型标识符字符串(如 `'openai:gpt-5.4-mini'`)或 `BaseChatModel` 实例。 |
| `*additional_models` | `string \| BaseChatModel` | 前面的模型失败时按顺序尝试的额外回退模型 |

### Summarization(摘要)

接近 token 上限时自动摘要对话历史,在压缩较早上下文的同时保留近期消息。摘要在以下场景中很有用:

- 超出上下文窗口的长时间对话。
- 有大量历史的多轮对话。
- 需要保留完整对话上下文的应用。

> **注意**:摘要是面向文本的上下文压缩。它不会调整大小、下采样或以其他方式压缩图像/音频/视频载荷。`keep` 保留的近期消息仍包含其原始多模态块,而被摘要的较早多模态消息仅由生成的文本摘要表示。对于图像密集型应用,请把媒体存储在文件系统或对象存储中,通过消息历史传递 URL 或文件引用。

**API 参考:** [`SummarizationMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/summarization/SummarizationMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20),
        ),
    ],
)
```

**配置选项:**

> **提示**:如果使用 `langchain>=1.1`,`trigger` 和 `keep` 的 `fraction` 条件依赖对话模型的[画像数据](https://docs.langchain.com/oss/python/langchain/models#模型画像model-profiles)。如果数据不可用,请使用其他条件或手动指定:
>
> ```python
> from langchain.chat_models import init_chat_model
>
> custom_profile = {
>     "max_input_tokens": 100_000,
>     # ...
> }
> model = init_chat_model("gpt-5.5", profile=custom_profile)
> ```

| 参数 | 类型 | 说明 |
|------|------|------|
| `model`(必填) | `string \| BaseChatModel` | 生成摘要的模型。可以是模型标识符字符串或 `BaseChatModel` 实例。 |
| `trigger` | `ContextSize \| TriggerClause \| list[...] \| None` | 触发摘要的条件。可以是:单个 `ContextSize` 元组(必须满足指定阈值);单个 `TriggerClause` 字典(所有指定阈值都必须满足——AND 逻辑);混合两种形式的列表(任一项满足即可——OR 逻辑)。支持的阈值:`fraction`(float,模型上下文大小的比例,0-1)、`tokens`(int,绝对 token 数)、`messages`(int,消息数)。未提供 `trigger` 时,摘要不会自动触发。 |
| `keep` | `ContextSize`(默认 `('messages', 20)`) | 摘要后保留多少上下文。只能指定其一:`fraction`——保留模型上下文大小的比例(0-1);`tokens`——保留的绝对 token 数;`messages`——保留的近期消息数。 |
| `token_counter` | function | 自定义 token 计数函数。默认基于字符计数。 |
| `summary_prompt` | string | 自定义摘要提示词模板。未指定时使用内置模板。模板应包含 `{messages}` 占位符,对话历史将插入此处。 |
| `trim_tokens_to_summarize` | number(默认 4000) | 生成摘要时包含的最大 token 数。摘要前消息会被裁剪以适应此限制。 |
| `summary_prefix` | string(**已弃用**) | 请改用 `summary_prompt` 提供完整提示词。 |
| `max_tokens_before_summary` | number(**已弃用**) | 请改用 `trigger: ("tokens", value)`。 |
| `messages_to_keep` | number(**已弃用**) | 请改用 `keep: ("messages", value)`。 |

**完整示例:**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware


# 单条件:tokens >= 4000 时触发
agent = create_agent(
    model="gpt-5.5",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=("tokens", 4000),
            keep=("messages", 20),
        ),
    ],
)

# 多条件:tokens >= 3000 或 messages >= 6 时触发
agent2 = create_agent(
    model="gpt-5.5",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=[
                ("tokens", 3000),
                ("messages", 6),
            ],
            keep=("messages", 20),
        ),
    ],
)

# AND 逻辑:仅当 tokens >= 4000 且 messages >= 10 时触发
agent3 = create_agent(
    model="gpt-5.5",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger={"tokens": 4000, "messages": 10},
            keep=("messages", 20),
        ),
    ],
)

# 组合 AND 与 OR:(tokens >= 5000 且 messages >= 3)
# 或 (tokens >= 3000 且 messages >= 6) 时触发
agent4 = create_agent(
    model="gpt-5.5",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=[
                {"tokens": 5000, "messages": 3},
                {"tokens": 3000, "messages": 6},
            ],
            keep=("messages", 20),
        ),
    ],
)

# 使用比例限制
agent5 = create_agent(
    model="gpt-5.5",
    tools=[your_weather_tool, your_calculator_tool],
    middleware=[
        SummarizationMiddleware(
            model="gpt-5.4-mini",
            trigger=("fraction", 0.8),
            keep=("fraction", 0.3),
        ),
    ],
)
```

### Human-in-the-loop(人机协同)

在工具调用执行前暂停 Agent 执行,等待人工批准、编辑或拒绝。[人机协同](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)在以下场景中很有用:

- 需要人工批准的高风险操作(如数据库写入、金融交易)。
- 人工监督是强制要求的合规工作流。
- 人工反馈引导 Agent 的长时间对话。

**API 参考:** [`HumanInTheLoopMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/human_in_the_loop/HumanInTheLoopMiddleware)

> **警告**:人机协同中间件需要 [checkpointer](https://docs.langchain.com/oss/python/langgraph/checkpointers#checkpoints) 以在中断之间维持状态。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.checkpoint.memory import InMemorySaver


def your_read_email_tool(email_id: str) -> str:
    """根据 ID 读取邮件的模拟函数。"""
    return f"Email content for ID: {email_id}"

def your_send_email_tool(recipient: str, subject: str, body: str) -> str:
    """发送邮件的模拟函数。"""
    return f"Email sent to {recipient} with subject '{subject}'"

agent = create_agent(
    model="gpt-5.5",
    tools=[your_read_email_tool, your_send_email_tool],
    checkpointer=InMemorySaver(),
    middleware=[
        HumanInTheLoopMiddleware(
            interrupt_on={
                "your_send_email_tool": {
                    "allowed_decisions": ["approve", "edit", "reject"],
                },
                "your_read_email_tool": False,
            }
        ),
    ],
)
```

> **提示**:完整示例、配置选项和集成模式参见[人机协同文档](https://docs.langchain.com/oss/python/langchain/human-in-the-loop)。

### Model call limit(模型调用限制)

限制模型调用次数,以防止无限循环或成本过高。模型调用限制在以下场景中很有用:

- 防止失控的 Agent 发起过多 API 调用。
- 在生产部署中强制执行成本控制。
- 在特定调用预算内测试 Agent 行为。

**API 参考:** [`ModelCallLimitMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/model_call_limit/ModelCallLimitMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ModelCallLimitMiddleware
from langgraph.checkpoint.memory import InMemorySaver

agent = create_agent(
    model="gpt-5.5",
    checkpointer=InMemorySaver(),  # 线程级限制所必需
    tools=[],
    middleware=[
        ModelCallLimitMiddleware(
            thread_limit=10,
            run_limit=5,
            exit_behavior="end",
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `thread_limit` | number | 一个 thread 中所有运行的最大模型调用数。默认无限制。 |
| `run_limit` | number | 单次调用的最大模型调用数。默认无限制。 |
| `exit_behavior` | string(默认 `end`) | 达到限制时的行为:`'end'`(优雅终止)或 `'error'`(抛出异常) |

### Tool call limit(工具调用限制)

通过限制工具调用次数来控制 Agent 执行,可以全局限制所有工具,也可以针对特定工具。工具调用限制在以下场景中很有用:

- 防止对昂贵外部 API 的过多调用。
- 限制网页搜索或数据库查询。
- 对特定工具的使用强制执行速率限制。
- 防止失控的 Agent 循环。

**API 参考:** [`ToolCallLimitMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_call_limit/ToolCallLimitMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool],
    middleware=[
        # 全局限制
        ToolCallLimitMiddleware(thread_limit=20, run_limit=10),
        # 特定工具限制
        ToolCallLimitMiddleware(
            tool_name="search",
            thread_limit=5,
            run_limit=3,
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `tool_name` | string | 要限制的特定工具名称。未提供时,限制**全局应用于所有工具**。 |
| `thread_limit` | number | 一个 thread(对话)中所有运行的最大工具调用数。在同一 thread ID 的多次调用之间持续累计。需要 checkpointer 维持状态。`None` 表示无线程限制。 |
| `run_limit` | number | 单次调用(一个用户消息 → 响应周期)的最大工具调用数。每条新用户消息时重置。`None` 表示无运行限制。**注意:** `thread_limit` 和 `run_limit` 至少需指定一个。 |
| `exit_behavior` | string(默认 `continue`) | 达到限制时的行为:`'continue'`(默认)——用错误消息阻止超限的工具调用,其他工具和模型继续运行,由模型根据错误消息决定何时结束;`'error'`——抛出 `ToolCallLimitExceededError` 异常,立即停止执行;`'end'`——立即停止执行,为超限的工具调用附加 `ToolMessage` 和 AI 消息。仅在限制单个工具时有效;如果其他工具还有待处理的调用,会抛出 `NotImplementedError`。 |

**完整示例:**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ToolCallLimitMiddleware


global_limiter = ToolCallLimitMiddleware(thread_limit=20, run_limit=10)
search_limiter = ToolCallLimitMiddleware(tool_name="search", thread_limit=5, run_limit=3)
database_limiter = ToolCallLimitMiddleware(tool_name="query_database", thread_limit=10)
strict_limiter = ToolCallLimitMiddleware(tool_name="scrape_webpage", run_limit=2, exit_behavior="error")

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, database_tool, scraper_tool],
    middleware=[global_limiter, search_limiter, database_limiter, strict_limiter],
)
```

### PII detection(PII 检测)

使用可配置的策略检测并处理对话中的个人敏感信息(PII)。PII 检测在以下场景中很有用:

- 有合规要求的医疗和金融应用。
- 需要清洗日志的客服 Agent。
- 任何处理敏感用户数据的应用。

> **注意**:设置 `apply_to_output=True` 时,`PIIMiddleware` 还会通过注册的流转换器编辑流式线路输出——文本增量、工具调用参数、工具输出和状态快照。需要 `langchain>=1.3.2`。参见[在中间件上注册转换器](https://docs.langchain.com/oss/python/langchain/event-streaming#在中间件上注册转换器)。

**API 参考:** [`PIIMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/pii/PIIMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware("email", strategy="redact", apply_to_input=True),
        PIIMiddleware("credit_card", strategy="mask", apply_to_input=True),
    ],
)
```

#### 自定义 PII 类型

你可以通过提供 `detector` 参数创建自定义 PII 类型。这让你能够检测内置类型之外的、符合你使用场景的特定模式。

**创建自定义检测器的三种方式:**

1. **正则表达式模式字符串** —— 简单的模式匹配
2. **自定义函数** —— 带校验的复杂检测逻辑

```python
from langchain.agents import create_agent
from langchain.agents.middleware import PIIMiddleware
import re


# 方式一:正则表达式模式字符串
agent1 = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware(
            "api_key",
            detector=r"sk-[a-zA-Z0-9]{32}",
            strategy="block",
        ),
    ],
)

# 方式二:编译后的正则表达式模式
agent2 = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware(
            "phone_number",
            detector=re.compile(r"\+?\d{1,3}[\s.-]?\d{3,4}[\s.-]?\d{4}"),
            strategy="mask",
        ),
    ],
)

# 方式三:自定义检测器函数
def detect_ssn(content: str) -> list[dict[str, str | int]]:
    """带校验地检测 SSN(社会安全号码)。

    返回包含 'text'、'start' 和 'end' 键的字典列表。
    """
    import re
    matches = []
    pattern = r"\d{3}-\d{2}-\d{4}"
    for match in re.finditer(pattern, content):
        ssn = match.group(0)
        # 校验:前 3 位不应是 000、666 或 900-999
        first_three = int(ssn[:3])
        if first_three not in [0, 666] and not (900 <= first_three <= 999):
            matches.append({
                "text": ssn,
                "start": match.start(),
                "end": match.end(),
            })
    return matches

agent3 = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        PIIMiddleware(
            "ssn",
            detector=detect_ssn,
            strategy="hash",
        ),
    ],
)
```

**自定义检测器函数签名:**检测器函数必须接受一个字符串(content)并返回匹配结果,即包含 `text`、`start` 和 `end` 键的字典列表:

```python
def detector(content: str) -> list[dict[str, str | int]]:
    return [
        {"text": "matched_text", "start": 0, "end": 12},
        # ... 更多匹配
    ]
```

> **提示**:对于自定义检测器:
>
> - 简单模式使用正则字符串
> - 需要标志位(如不区分大小写匹配)时使用编译后的正则对象
> - 需要模式匹配之外的校验逻辑时使用自定义函数
> - 自定义函数让你完全控制检测逻辑,可以实现复杂的校验规则

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `pii_type`(必填) | string | 要检测的 PII 类型。可以是内置类型(`email`、`credit_card`、`ip`、`mac_address`、`url`)或自定义类型名。 |
| `strategy` | string(默认 `redact`) | 如何处理检测到的 PII:`'block'`——检测到时抛出异常;`'redact'`——替换为 `[REDACTED_{PII_TYPE}]`;`'mask'`——部分掩码(如 `****-****-****-1234`);`'hash'`——替换为确定性哈希 |
| `detector` | `function \| regex` | 自定义检测器函数或正则模式。未提供时使用该 PII 类型的内置检测器。 |
| `apply_to_input` | boolean(默认 True) | 在模型调用前检查用户消息 |
| `apply_to_output` | boolean(默认 False) | 在模型调用后检查 AI 消息。在 `langchain>=1.3.2` 中,还会通过注册的流转换器编辑流式线路输出(文本增量、工具调用参数、工具输出、状态快照)。参见[事件流](https://docs.langchain.com/oss/python/langchain/event-streaming)。 |
| `apply_to_tool_results` | boolean(默认 False) | 在执行后检查工具结果消息 |

### To-do list(待办清单)

为 Agent 配备任务规划与跟踪能力,以处理复杂的多步骤任务。待办清单在以下场景中很有用:

- 需要跨多个工具协调的复杂多步骤任务。
- 进度可见性很重要的长时间运行操作。

> **注意**:此中间件会自动为 Agent 提供 `write_todos` 工具和引导有效任务规划的系统提示词。

**API 参考:** [`TodoListMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/todo/TodoListMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[read_file, write_file, run_tests],
    middleware=[TodoListMiddleware()],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `system_prompt` | string | 引导待办使用的自定义系统提示词。未指定时使用内置提示词。 |
| `tool_description` | string | `write_todos` 工具的自定义描述。未指定时使用内置描述。 |

### LLM tool selector(LLM 工具选择器)

在调用主模型之前,用 LLM 智能地选择相关工具。LLM 工具选择器在以下场景中很有用:

- 工具很多(10 个以上)且每个查询大多不相关的 Agent。
- 通过过滤不相关的工具减少 token 使用。
- 提升模型的专注度和准确性。

该中间件使用结构化输出询问 LLM 哪些工具与当前查询最相关。结构化输出的 schema 定义了可用的工具名称和描述。模型提供商通常会在幕后把这些结构化输出信息加入系统提示词。

**API 参考:** [`LLMToolSelectorMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/tool_selection/LLMToolSelectorMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolSelectorMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[tool1, tool2, tool3, tool4, tool5, ...],
    middleware=[
        LLMToolSelectorMiddleware(
            model="gpt-5.4-mini",
            max_tools=3,
            always_include=["search"],
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | `string \| BaseChatModel` | 用于工具选择的模型。默认为 Agent 的主模型。 |
| `system_prompt` | string | 给选择模型的指令。未指定时使用内置提示词。 |
| `max_tools` | number | 最多选择的工具数。如果模型选择了更多,只会使用前 max_tools 个。未指定则无限制。 |
| `always_include` | `list[string]` | 无论选择结果如何都始终包含的工具名。这些不计入 max_tools 限制。 |

### Provider tool search(提供商工具搜索)

把选定工具交给模型提供商的服务器端工具搜索,让模型按需发现它们,而不是一开始就接收所有工具的 schema。提供商工具搜索在以下场景中很有用:

- 使用大量工具时减少上下文膨胀。
- 只呈现相关工具,提升工具选择的准确性。

> **注意**:需要支持服务器端工具搜索的模型:Anthropic(Claude Sonnet 4+/Opus 4+/Haiku 4.5+)或 OpenAI(gpt-5.5+)。其他提供商会抛出 `ValueError`。

**API 参考:** [`ProviderToolSearchMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/provider_tool_search/ProviderToolSearchMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ProviderToolSearchMiddleware

agent = create_agent(
    model="anthropic:claude-opus-4-8",
    tools=[get_weather, lookup_order],
    middleware=[
        ProviderToolSearchMiddleware(searchable_tools=["lookup_order"]),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `searchable_tools` | `list[str \| BaseTool]` | 通过名称或实例给出的、要交给提供商工具搜索延迟加载的工具。被延迟的工具会被对模型隐藏,直到其搜索将其呈现出来。用 `extras={"defer_loading": True}` 构造的工具无论此选项如何都会被延迟;如果省略 `searchable_tools`,只有这些预先标记的工具会被延迟。 |

**完整示例:**工具也可以在构造时通过设置 `extras={"defer_loading": True}` 选择延迟加载:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ProviderToolSearchMiddleware
from langchain.tools import tool


# 构造时标记了 `defer_loading`,所以它自己就会被延迟——
# 无需把它列进 `searchable_tools`。
@tool(extras={"defer_loading": True})
def send_email(to: str) -> str:
    """发送邮件。"""
    return "sent"


agent = create_agent(
    model="anthropic:claude-opus-4-8",
    tools=[send_email],
    middleware=[ProviderToolSearchMiddleware()],
)
```

### Shell tool(Shell 工具)

向 Agent 暴露持久化 shell 会话以执行命令。Shell 工具中间件在以下场景中很有用:

- 需要执行系统命令的 Agent
- 开发和部署自动化任务
- 测试与验证工作流
- 文件系统操作和脚本执行

> **警告:安全考量**:请使用合适的执行策略(`HostExecutionPolicy`、`DockerExecutionPolicy` 或 `CodexSandboxExecutionPolicy`)以匹配你部署的安全要求。

> **注意:限制**:持久化 shell 会话目前不能与中断(human-in-the-loop)一起工作。我们预计未来会添加支持。

**API 参考:** [`ShellToolMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/shell_tool/ShellToolMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ShellToolMiddleware,
    HostExecutionPolicy,
)

agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            execution_policy=HostExecutionPolicy(),
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `workspace_root` | `str \| Path \| None` | shell 会话的基础目录。省略时在 Agent 启动时创建临时目录,结束时删除。 |
| `startup_commands` | `tuple[str, ...] \| list[str] \| str \| None` | 会话启动后按顺序执行的可选命令 |
| `shutdown_commands` | `tuple[str, ...] \| list[str] \| str \| None` | 会话关闭前执行的可选命令 |
| `execution_policy` | `BaseExecutionPolicy \| None` | 控制超时、输出限制和资源配置的执行策略:`HostExecutionPolicy`——完全主机访问(默认),适合 Agent 已经运行在容器或 VM 内的可信环境;`DockerExecutionPolicy`——为每次 Agent 运行启动独立的 Docker 容器,提供更强的隔离;`CodexSandboxExecutionPolicy`——复用 Codex CLI 沙箱,提供额外的系统调用/文件系统限制 |
| `redaction_rules` | `tuple[RedactionRule, ...] \| list[RedactionRule] \| None` | 可选的编辑规则,在命令输出返回给模型之前进行清洗。**警告**:编辑规则在执行之后应用,使用 `HostExecutionPolicy` 时不能阻止秘密或敏感数据的外泄。 |
| `tool_description` | `str \| None` | 可选的已注册 shell 工具描述覆盖 |
| `shell_command` | `Sequence[str] \| str \| None` | 用于启动持久会话的可选 shell 可执行文件(字符串)或参数序列。默认为 `/bin/bash`。 |
| `env` | `Mapping[str, Any] \| None` | 提供给 shell 会话的可选环境变量。值在命令执行前会被强制转换为字符串。 |

**完整示例:**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import (
    ShellToolMiddleware,
    HostExecutionPolicy,
    DockerExecutionPolicy,
    RedactionRule,
)


# 带主机执行的基本 shell 工具
agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            execution_policy=HostExecutionPolicy(),
        ),
    ],
)

# 带启动命令的 Docker 隔离
agent_docker = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            startup_commands=["pip install requests", "export PYTHONPATH=/workspace"],
            execution_policy=DockerExecutionPolicy(
                image="python:3.11-slim",
                command_timeout=60.0,
            ),
        ),
    ],
)

# 带输出编辑(执行后应用)
agent_redacted = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            redaction_rules=[
                RedactionRule(pii_type="api_key", detector=r"sk-[a-zA-Z0-9]{32}"),
            ],
        ),
    ],
)
```

### Filesystem middleware(文件系统中间件)

上下文工程是构建有效 Agent 的主要挑战。当使用返回可变长度结果的工具(例如 `web_search` 和 RAG)时尤其困难,因为长工具结果会迅速填满你的上下文窗口。

来自 [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) 的 `FilesystemMiddleware` 提供了四个用于与短期和长期记忆交互的工具:

- `ls`:列出文件系统中的文件
- `read_file`:读取整个文件或文件中的特定行数
- `write_file`:向文件系统写入新文件
- `edit_file`:编辑文件系统中的现有文件

```python
from langchain.agents import create_agent
from deepagents.middleware.filesystem import FilesystemMiddleware

# FilesystemMiddleware 默认包含在 create_deep_agent 中
# 构建自定义 Agent 时你可以自定义它
agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        FilesystemMiddleware(
            backend=None,  # 可选:自定义后端(默认为 StateBackend)
            system_prompt="Write to the filesystem when...",  # 可选的对系统提示词的自定义补充
            custom_tool_descriptions={
                "ls": "Use the ls tool when...",
                "read_file": "Use the read_file tool to..."
            },  # 可选:文件系统工具的自定义描述
            tools=["read_file", "ls", "glob", "grep"],  # 可选:限制暴露哪些文件系统工具的允许列表
        ),
    ],
)
```

#### 短期 vs. 长期文件系统

默认情况下,这些工具写入图状态中的本地"文件系统"。要启用跨线程的持久化存储,请配置一个 `CompositeBackend`,把特定路径(如 `/memories/`)路由到 `StoreBackend`。

```python
from langchain.agents import create_agent
from deepagents.middleware import FilesystemMiddleware
from deepagents.backends import CompositeBackend, StateBackend, StoreBackend
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

agent = create_agent(
    model="claude-sonnet-4-6",
    store=store,
    middleware=[
        FilesystemMiddleware(
            backend=CompositeBackend(
                default=StateBackend(),
                routes={"/memories/": StoreBackend()}
            ),
            custom_tool_descriptions={
                "ls": "Use the ls tool when...",
                "read_file": "Use the read_file tool to..."
            }  # 可选:文件系统工具的自定义描述
        ),
    ],
)
```

当你配置了带 `/memories/` 的 `StoreBackend` 的 `CompositeBackend` 时,任何以 **/memories/** 为前缀的文件都会保存到持久化存储中,并在不同线程之间存活。没有该前缀的文件仍留在临时的状态存储中。

### Subagent(子 Agent)

把任务交给子 Agent 可以隔离上下文,让主(监督者)Agent 的上下文窗口保持干净,同时仍能在任务上深入。

来自 [Deep Agents](https://docs.langchain.com/oss/python/deepagents/overview) 的 subagents 中间件让你可以通过 `task` 工具提供子 Agent。

```python
from langchain.tools import tool
from langchain.agents import create_agent
from deepagents.middleware.subagents import SubAgentMiddleware


@tool
def get_weather(city: str) -> str:
    """获取某城市的天气。"""
    return f"The weather in {city} is sunny."

agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        SubAgentMiddleware(
            default_model="claude-sonnet-4-6",
            default_tools=[],
            subagents=[
                {
                    "name": "weather",
                    "description": "This subagent can get weather in cities.",
                    "system_prompt": "Use the get_weather tool to get the weather in a city.",
                    "tools": [get_weather],
                    "model": "gpt-5.5",
                    "middleware": [],
                }
            ],
        )
    ],
)
```

子 Agent 由**名称**、**描述**、**系统提示词**和**工具**定义。你也可以为子 Agent 提供自定义**模型**或额外的**中间件**。当你想给子 Agent 一个与主 Agent 共享的额外 state 键时,这特别有用。

对于更复杂的使用场景,你也可以提供自己预构建的 LangGraph 图作为子 Agent。

```python
from langchain.agents import create_agent
from deepagents.middleware.subagents import SubAgentMiddleware
from deepagents import CompiledSubAgent
from langgraph.graph import StateGraph

# 创建自定义 LangGraph 图
def create_weather_graph():
    workflow = StateGraph(...)
    # 构建你的自定义图
    return workflow.compile()

weather_graph = create_weather_graph()

# 包装成 CompiledSubAgent
weather_subagent = CompiledSubAgent(
    name="weather",
    description="This subagent can get weather in cities.",
    runnable=weather_graph
)

agent = create_agent(
    model="claude-sonnet-4-6",
    middleware=[
        SubAgentMiddleware(
            default_model="claude-sonnet-4-6",
            default_tools=[],
            subagents=[weather_subagent],
        )
    ],
)
```

除任何用户定义的子 Agent 外,主 Agent 始终可以访问一个 `general-purpose`(通用)子 Agent。该子 Agent 拥有与主 Agent 相同的指令及其可访问的所有工具。`general-purpose` 子 Agent 的主要用途是上下文隔离——主 Agent 可以把复杂任务委派给这个子 Agent,并获得简洁的回答,而不会被中间工具调用撑大上下文。

### Rubric grading(评分准则)

> **注意**:`RubricMiddleware` 需要 `deepagents>=0.6.5`。它处于 [**beta**](https://docs.langchain.com/oss/python/versioning) 阶段;API 未来可能变化。

有些任务对"完成"有清晰的定义,但 Agent 无法可靠地一次达成。`RubricMiddleware` 让你把*完成的样子*声明为一个准则(rubric),让 Agent 自评并迭代,直到准则满足或达到最大迭代上限。

**API 参考:** [`RubricMiddleware`](https://reference.langchain.com/python/deepagents/middleware/rubric/RubricMiddleware)

```python
from deepagents import RubricMiddleware, create_deep_agent
from langgraph.checkpoint.memory import InMemorySaver

agent = create_deep_agent(
    model="openai:gpt-5.5",
    middleware=[
        RubricMiddleware(
            model="anthropic:claude-haiku-4-5",
            max_iterations=3,
        ),
    ],
    checkpointer=InMemorySaver(),
)
```

完整配置选项、流式事件和完整的代码生成示例参见[评分准则](https://docs.langchain.com/oss/python/deepagents/rubric)。

### File search(文件搜索)

提供针对文件系统的 Glob 和 Grep 搜索工具。文件搜索中间件在以下场景中很有用:

- 代码探索与分析
- 按名称模式查找文件
- 用正则搜索代码内容
- 需要文件发现的大型代码库

**API 参考:** [`FilesystemFileSearchMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/file_search/FilesystemFileSearchMiddleware)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import FilesystemFileSearchMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        FilesystemFileSearchMiddleware(
            root_path="/workspace",
            use_ripgrep=True,
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `root_path`(必填) | str | 要搜索的根目录。所有文件操作都相对于此路径。 |
| `use_ripgrep` | bool(默认 True) | 是否使用 ripgrep 进行搜索。ripgrep 不可用时回退到 Python 正则。 |
| `max_file_size_mb` | int(默认 10) | 可搜索的最大文件大小(MB)。超过此大小的文件会被跳过。 |

**完整示例:**该中间件为 Agent 添加两个搜索工具:

- **Glob 工具** —— 快速文件模式匹配:支持 `**/*.py`、`src/**/*.ts` 等模式;返回按修改时间排序的匹配文件路径。
- **Grep 工具** —— 带正则的内容搜索:完整正则语法支持;可用 `include` 参数按文件模式过滤;三种输出模式:`files_with_matches`、`content`、`count`。

```python
from langchain.agents import create_agent
from langchain.agents.middleware import FilesystemFileSearchMiddleware
from langchain.messages import HumanMessage


agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        FilesystemFileSearchMiddleware(
            root_path="/workspace",
            use_ripgrep=True,
            max_file_size_mb=10,
        ),
    ],
)

# Agent 现在可以使用 glob_search 和 grep_search 工具
result = agent.invoke({
    "messages": [HumanMessage("Find all Python files containing 'async def'")]
})

# Agent 会使用:
# 1. glob_search(pattern="**/*.py") 查找 Python 文件
# 2. grep_search(pattern="async def", include="*.py") 查找 async 函数
```

### Context editing(上下文编辑)

通过在达到 token 上限时清除较早的工具调用输出来管理对话上下文,同时保留近期结果。这有助于在工具调用很多的长对话中让上下文窗口保持可控。上下文编辑在以下场景中很有用:

- 工具调用很多、超出 token 限制的长对话
- 通过移除不再相关的较早工具输出来降低 token 成本
- 只在上下文中保留最近 N 个工具结果

**API 参考:** [`ContextEditingMiddleware`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ContextEditingMiddleware)、[`ClearToolUsesEdit`](https://reference.langchain.com/python/langchain/agents/middleware/context_editing/ClearToolUsesEdit)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    middleware=[
        ContextEditingMiddleware(
            edits=[
                ClearToolUsesEdit(
                    trigger=100000,
                    keep=3,
                ),
            ],
        ),
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `edits` | `list[ContextEdit]`(默认 `[ClearToolUsesEdit()]`) | 要应用的 `ContextEdit` 策略列表 |
| `token_count_method` | string(默认 `approximate`) | token 计数方法:`'approximate'` 或 `'model'` |

**`ClearToolUsesEdit` 选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `trigger` | number(默认 100000) | 触发编辑的 token 数。对话超过此 token 数时,较早的工具输出会被清除。 |
| `clear_at_least` | number(默认 0) | 编辑运行时至少回收的 token 数。设为 0 则按需清除。 |
| `keep` | number(默认 3) | 必须保留的最近工具结果数量。这些永远不会被清除。 |
| `clear_tool_inputs` | boolean(默认 False) | 是否清除 AI 消息上对应的工具调用参数。为 `True` 时,工具调用参数会被替换为空对象。 |
| `exclude_tools` | `list[string]`(默认 `()`) | 排除在清除之外的工具名列表。这些工具的输出永远不会被清除。 |
| `placeholder` | string(默认 `[cleared]`) | 为被清除的工具输出插入的占位文本,替换原始工具消息内容。 |

**完整示例:**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit


agent = create_agent(
    model="gpt-5.5",
    tools=[search_tool, your_calculator_tool, database_tool],
    middleware=[
        ContextEditingMiddleware(
            edits=[
                ClearToolUsesEdit(
                    trigger=2000,
                    keep=3,
                    clear_tool_inputs=False,
                    exclude_tools=[],
                    placeholder="[cleared]",
                ),
            ],
        ),
    ],
)
```

### LLM tool emulator(LLM 工具模拟器)

出于测试目的,用 LLM 模拟工具执行,以 AI 生成的响应替代实际工具调用。LLM 工具模拟器在以下场景中很有用:

- 在不执行真实工具的情况下测试 Agent 行为。
- 在外部工具不可用或昂贵时开发 Agent。
- 在实现实际工具之前对 Agent 工作流做原型验证。

**API 参考:** [`LLMToolEmulator`](https://reference.langchain.com/python/langchain/agents/middleware/tool_emulator/LLMToolEmulator)

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolEmulator

agent = create_agent(
    model="gpt-5.5",
    tools=[get_weather, search_database, send_email],
    middleware=[
        LLMToolEmulator(),  # 模拟所有工具
    ],
)
```

**配置选项:**

| 参数 | 类型 | 说明 |
|------|------|------|
| `tools` | `list[str \| BaseTool]` | 要模拟的工具名(str)或 BaseTool 实例列表。为 `None`(默认)时模拟**所有**工具;为空列表 `[]` 时不模拟任何工具;为带工具名/实例的数组时只模拟这些工具。 |
| `model` | `string \| BaseChatModel` | 用于生成模拟工具响应的模型。未指定时默认为 Agent 的模型。 |

**完整示例:**

```python
from langchain.agents import create_agent
from langchain.agents.middleware import LLMToolEmulator
from langchain.tools import tool


@tool
def get_weather(location: str) -> str:
    """获取某地的当前天气。"""
    return f"Weather in {location}"

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """发送邮件。"""
    return "Email sent"


# 模拟所有工具(默认行为)
agent = create_agent(
    model="gpt-5.5",
    tools=[get_weather, send_email],
    middleware=[LLMToolEmulator()],
)

# 只模拟特定工具
agent2 = create_agent(
    model="gpt-5.5",
    tools=[get_weather, send_email],
    middleware=[LLMToolEmulator(tools=["get_weather"])],
)

# 使用自定义模型进行模拟
agent4 = create_agent(
    model="gpt-5.5",
    tools=[get_weather, send_email],
    middleware=[LLMToolEmulator(model="claude-sonnet-4-6")],
)
```

## 提供商特有的中间件(Provider-specific middleware)

这些中间件针对特定 LLM 提供商进行了优化。完整细节和示例参见各提供商的文档。

- [Anthropic](https://docs.langchain.com/oss/python/integrations/middleware/anthropic):针对 Claude 模型的提示词缓存、bash 工具、文本编辑器、记忆和文件搜索中间件。
- [AWS](https://docs.langchain.com/oss/python/integrations/middleware/aws):针对 Amazon Bedrock 模型的提示词缓存中间件。
- [OpenAI](https://docs.langchain.com/oss/python/integrations/middleware/openai):针对 OpenAI 模型的内容审核中间件。
