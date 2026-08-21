# Structured output(结构化输出)

> 原文:[Structured output](https://docs.langchain.com/oss/python/langchain/structured-output)

结构化输出让 Agent 能以特定、可预期的格式返回数据。你得到的不再是自然语言响应,而是应用可以直接使用的结构化数据——JSON 对象、[Pydantic 模型](https://docs.pydantic.dev/latest/concepts/models/#basic-model-usage)或 dataclass 的形式。

> **提示**
>
> 本页介绍使用 `create_agent` 时 Agent 的结构化输出。要在模型上直接使用结构化输出(不经过 Agent),参见 [Models - Structured output](https://docs.langchain.com/oss/python/langchain/models#structured-output)。

LangChain 的 [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) 自动处理结构化输出。用户设置期望的结构化输出 schema,当模型生成结构化数据时,它会被捕获、校验,并通过 Agent 状态的 `'structured_response'` 键返回。

```python
def create_agent(
    ...
    response_format: Union[
        ToolStrategy[StructuredResponseT],
        ProviderStrategy[StructuredResponseT],
        type[StructuredResponseT],
        None,
    ]
)
```

## 响应格式(Response format)

使用 `response_format` 控制 Agent 如何返回结构化数据:

- **`ToolStrategy[StructuredResponseT]`**:使用工具调用实现结构化输出
- **`ProviderStrategy[StructuredResponseT]`**:使用提供商原生的结构化输出
- **`type[StructuredResponseT]`**:schema 类型——根据模型能力自动选择最佳策略
- **`None`**:未显式请求结构化输出

当直接传入 schema 类型时,LangChain 会自动选择:

- 如果所选模型和提供商支持原生结构化输出(例如 [OpenAI](https://docs.langchain.com/oss/python/integrations/providers/openai)、[Anthropic (Claude)](https://docs.langchain.com/oss/python/integrations/providers/anthropic) 或 [xAI (Grok)](https://docs.langchain.com/oss/python/integrations/providers/xai)),则使用 `ProviderStrategy`。
- 其他所有模型使用 `ToolStrategy`。

> **警告**:JSON Schema 字典必须包装在显式策略(`ProviderStrategy` 或 `ToolStrategy`)中。直接传给 `response_format` 时不会被自动识别。

> **注意**:如果使用 `langchain>=1.1`,对原生结构化输出特性的支持会从模型的[画像数据(profile data)](https://docs.langchain.com/oss/python/langchain/models#模型画像model-profiles)动态读取。如果数据不可用,请使用其他条件或手动指定:
>
> ```python
> custom_profile = {
>     "structured_output": True,
>     # ...
> }
> model = init_chat_model("...", profile=custom_profile)
> ```
>
> 如果指定了工具,模型必须支持工具与结构化输出的同时使用。

结构化响应通过 Agent 最终状态的 `structured_response` 键返回。

## Provider strategy(提供商策略)

部分模型提供商通过其 API 原生支持结构化输出(例如 OpenAI、xAI (Grok)、Gemini、Anthropic (Claude))。在可用时,这是最可靠的方法。

要使用此策略,请配置 `ProviderStrategy`:

```python
class ProviderStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    strict: bool | None = None
```

> **说明**:`strict` 参数需要 `langchain>=1.2`。

**`schema`(必填)**:定义结构化输出格式的 schema。支持:

- **Pydantic 模型**:带字段校验的 `BaseModel` 子类。返回经过校验的 Pydantic 实例。
- **Dataclass**:带类型注解的 Python dataclass。返回 dict。
- **TypedDict**:带类型的字典类。返回 dict。
- **JSON Schema**:带 JSON schema 规范的字典。必须包含顶层的 `title` 和 `description` 键。返回 dict。

**`strict`**:可选的布尔参数,启用严格的 schema 遵循。部分提供商支持(例如 [OpenAI](https://docs.langchain.com/oss/python/integrations/chat/openai) 和 [xAI](https://docs.langchain.com/oss/python/integrations/chat/xai))。默认为 `None`(禁用)。

当你直接把 schema 类型传给 [`create_agent.response_format`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) 且模型支持原生结构化输出时,LangChain 会自动使用 `ProviderStrategy`:

**Pydantic 模型:**

```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent


class ContactInfo(BaseModel):
    """某个人的联系信息。"""
    name: str = Field(description="The name of the person")
    email: str = Field(description="The email address of the person")
    phone: str = Field(description="The phone number of the person")

agent = create_agent(
    model="gpt-5.5",
    response_format=ContactInfo  # 自动选择 ProviderStrategy
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

print(result["structured_response"])
# ContactInfo(name='John Doe', email='john@example.com', phone='(555) 123-4567')
```

**Dataclass:**

```python
from dataclasses import dataclass
from langchain.agents import create_agent


@dataclass
class ContactInfo:
    """某个人的联系信息。"""
    name: str # The name of the person
    email: str # The email address of the person
    phone: str # The phone number of the person

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ContactInfo  # 自动选择 ProviderStrategy
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
```

**TypedDict:**

```python
from typing_extensions import TypedDict
from langchain.agents import create_agent


class ContactInfo(TypedDict):
    """某个人的联系信息。"""
    name: str # The name of the person
    email: str # The email address of the person
    phone: str # The phone number of the person

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ContactInfo  # 自动选择 ProviderStrategy
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
```

**JSON Schema:**

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ProviderStrategy


contact_info_schema = {
    "title": "ContactInfo",
    "type": "object",
    "description": "Contact information for a person.",
    "properties": {
        "name": {"type": "string", "description": "The name of the person"},
        "email": {"type": "string", "description": "The email address of the person"},
        "phone": {"type": "string", "description": "The phone number of the person"}
    },
    "required": ["name", "email", "phone"]
}

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ProviderStrategy(contact_info_schema)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract contact info from: John Doe, john@example.com, (555) 123-4567"}]
})

result["structured_response"]
# {'name': 'John Doe', 'email': 'john@example.com', 'phone': '(555) 123-4567'}
```

提供商原生的结构化输出提供高可靠性和严格校验,因为模型提供商会强制执行 schema。可用时请优先使用。

> **注意**:如果提供商对你所选的模型原生支持结构化输出,写 `response_format=ProductReview` 与写 `response_format=ProviderStrategy(ProductReview)` 在功能上是等价的。在两种情况下,如果不支持结构化输出,Agent 都会回退到工具调用策略。

## 工具调用策略(Tool calling strategy)

对于不支持原生结构化输出的模型,LangChain 使用工具调用来实现同样的结果。这适用于所有支持工具调用的模型(大多数现代模型)。

要使用此策略,请配置 `ToolStrategy`:

```python
class ToolStrategy(Generic[SchemaT]):
    schema: type[SchemaT]
    tool_message_content: str | None
    handle_errors: Union[
        bool,
        str,
        type[Exception],
        tuple[type[Exception], ...],
        Callable[[Exception], str],
    ]
```

**`schema`(必填)**:定义结构化输出格式的 schema。支持:

- **Pydantic 模型**:带字段校验的 `BaseModel` 子类。返回经过校验的 Pydantic 实例。
- **Dataclass**:带类型注解的 Python dataclass。返回 dict。
- **TypedDict**:带类型的字典类。返回 dict。
- **JSON Schema**:带 JSON schema 规范的字典。必须包含顶层的 `title` 和 `description` 键。返回 dict。
- **Union 类型**:多个 schema 选项。模型会根据上下文选择最合适的 schema。

**`tool_message_content`**:生成结构化输出时返回的工具消息的自定义内容。未提供时,默认为显示结构化响应数据的消息。

**`handle_errors`**:结构化输出校验失败的错误处理策略。默认为 `True`。

- **`True`**:使用默认错误模板捕获所有错误
- **`str`**:使用此自定义消息捕获所有错误
- **`type[Exception]`**:仅捕获此异常类型,使用默认消息
- **`tuple[type[Exception], ...]`**:仅捕获这些异常类型,使用默认消息
- **`Callable[[Exception], str]`**:返回错误消息的自定义函数
- **`False`**:不重试,让异常向上传播

**Pydantic 模型:**

```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductReview(BaseModel):
    """产品评价的分析。"""
    rating: int | None = Field(description="The rating of the product", ge=1, le=5)
    sentiment: Literal["positive", "negative"] = Field(description="The sentiment of the review")
    key_points: list[str] = Field(description="The key points of the review. Lowercase, 1-3 words each.")

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ToolStrategy(ProductReview)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# ProductReview(rating=5, sentiment='positive', key_points=['fast shipping', 'expensive'])
```

**Dataclass:**

```python
from dataclasses import dataclass
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


@dataclass
class ProductReview:
    """产品评价的分析。"""
    rating: int | None  # 产品评分(1-5)
    sentiment: Literal["positive", "negative"]  # 评价的情感倾向
    key_points: list[str]  # 评价的要点

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ToolStrategy(ProductReview)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
```

**TypedDict:**

```python
from typing import Literal
from typing_extensions import TypedDict
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductReview(TypedDict):
    """产品评价的分析。"""
    rating: int | None  # 产品评分(1-5)
    sentiment: Literal["positive", "negative"]  # 评价的情感倾向
    key_points: list[str]  # 评价的要点

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ToolStrategy(ProductReview)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
```

**JSON Schema:**

```python
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


product_review_schema = {
    "title": "ProductReview",
    "type": "object",
    "description": "Analysis of a product review.",
    "properties": {
        "rating": {
            "type": ["integer", "null"],
            "description": "The rating of the product (1-5)",
            "minimum": 1,
            "maximum": 5
        },
        "sentiment": {
            "type": "string",
            "enum": ["positive", "negative"],
            "description": "The sentiment of the review"
        },
        "key_points": {
            "type": "array",
            "items": {"type": "string"},
            "description": "The key points of the review"
        }
    },
    "required": ["sentiment", "key_points"]
}

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ToolStrategy(product_review_schema)
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# {'rating': 5, 'sentiment': 'positive', 'key_points': ['fast shipping', 'expensive']}
```

**Union 类型:**

```python
from pydantic import BaseModel, Field
from typing import Literal, Union
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductReview(BaseModel):
    """产品评价的分析。"""
    rating: int | None = Field(description="The rating of the product", ge=1, le=5)
    sentiment: Literal["positive", "negative"] = Field(description="The sentiment of the review")
    key_points: list[str] = Field(description="The key points of the review. Lowercase, 1-3 words each.")

class CustomerComplaint(BaseModel):
    """客户对产品或服务的投诉。"""
    issue_type: Literal["product", "service", "shipping", "billing"] = Field(description="The type of issue")
    severity: Literal["low", "medium", "high"] = Field(description="The severity of the complaint")
    description: str = Field(description="Brief description of the complaint")

agent = create_agent(
    model="gpt-5.5",
    tools=tools,
    response_format=ToolStrategy(Union[ProductReview, CustomerComplaint])
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Analyze this review: 'Great product: 5 out of 5 stars. Fast shipping, but expensive'"}]
})
result["structured_response"]
# ProductReview(rating=5, sentiment='positive', key_points=['fast shipping', 'expensive'])
```

### 自定义工具消息内容(Custom tool message content)

`tool_message_content` 参数允许你自定义生成结构化输出时出现在对话历史中的消息:

```python
from pydantic import BaseModel, Field
from typing import Literal
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class MeetingAction(BaseModel):
    """从会议记录中提取的行动项。"""
    task: str = Field(description="The specific task to be completed")
    assignee: str = Field(description="Person responsible for the task")
    priority: Literal["low", "medium", "high"] = Field(description="Priority level")

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(
        schema=MeetingAction,
        tool_message_content="Action item captured and added to meeting notes!"
    )
)

agent.invoke({
    "messages": [{"role": "user", "content": "From our meeting: Sarah needs to update the project timeline as soon as possible"}]
})
```

```text
================================ Human Message =================================

From our meeting: Sarah needs to update the project timeline as soon as possible
================================== Ai Message ==================================
Tool Calls:
  MeetingAction (call_1)
 Call ID: call_1
  Args:
    task: Update the project timeline
    assignee: Sarah
    priority: high
================================= Tool Message =================================
Name: MeetingAction

Action item captured and added to meeting notes!
```

如果不设置 `tool_message_content`,最终的 [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) 将是:

```text
================================= Tool Message =================================
Name: MeetingAction

Returning structured response: {'task': 'update the project timeline', 'assignee': 'Sarah', 'priority': 'high'}
```

### 错误处理(Error handling)

模型在通过工具调用生成结构化输出时可能会出错。LangChain 提供了智能重试机制来自动处理这些错误。

#### 多个结构化输出错误(Multiple structured outputs error)

当模型错误地调用了多个结构化输出工具时,Agent 会在 [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) 中提供错误反馈,并提示模型重试:

```python
from pydantic import BaseModel, Field
from typing import Union
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ContactInfo(BaseModel):
    name: str = Field(description="Person's name")
    email: str = Field(description="Email address")

class EventDetails(BaseModel):
    event_name: str = Field(description="Name of the event")
    date: str = Field(description="Event date")

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(Union[ContactInfo, EventDetails])  # 默认:handle_errors=True
)

agent.invoke({
    "messages": [{"role": "user", "content": "Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th"}]
})
```

```text
================================ Human Message =================================

Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th
================================== Ai Message ==================================
Tool Calls:
  ContactInfo (call_1)
  Args:
    name: John Doe
    email: john@email.com
  EventDetails (call_2)
  Args:
    event_name: Tech Conference
    date: March 15th
================================= Tool Message =================================
Name: ContactInfo

Error: Model incorrectly returned multiple structured responses (ContactInfo, EventDetails) when only one is expected.
 Please fix your mistakes.
================================= Tool Message =================================
Name: EventDetails

Error: Model incorrectly returned multiple structured responses (ContactInfo, EventDetails) when only one is expected.
 Please fix your mistakes.
================================== Ai Message ==================================
Tool Calls:
  ContactInfo (call_3)
  Args:
    name: John Doe
    email: john@email.com
================================= Tool Message =================================
Name: ContactInfo

Returning structured response: {'name': 'John Doe', 'email': 'john@email.com'}
```

#### Schema 校验错误(Schema validation error)

当结构化输出与预期 schema 不匹配时,Agent 会提供具体的错误反馈:

```python
from pydantic import BaseModel, Field
from langchain.agents import create_agent
from langchain.agents.structured_output import ToolStrategy


class ProductRating(BaseModel):
    rating: int | None = Field(description="Rating from 1-5", ge=1, le=5)
    comment: str = Field(description="Review comment")

agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(ProductRating),  # 默认:handle_errors=True
    system_prompt="You are a helpful assistant that parses product reviews. Do not make any field or value up."
)

agent.invoke({
    "messages": [{"role": "user", "content": "Parse this: Amazing product, 10/10!"}]
})
```

```text
================================ Human Message =================================

Parse this: Amazing product, 10/10!
================================== Ai Message ==================================
Tool Calls:
  ProductRating (call_1)
  Args:
    rating: 10
    comment: Amazing product
================================= Tool Message =================================
Name: ProductRating

Error: Failed to parse structured output for tool 'ProductRating': 1 validation error for ProductRating.rating
  Input should be less than or equal to 5 [type=less_than_equal, input_value=10, input_type=int].
 Please fix your mistakes.
================================== Ai Message ==================================
Tool Calls:
  ProductRating (call_2)
  Args:
    rating: 5
    comment: Amazing product
================================= Tool Message =================================
Name: ProductRating

Returning structured response: {'rating': 5, 'comment': 'Amazing product'}
```

#### 错误处理策略(Error handling strategies)

你可以使用 `handle_errors` 参数自定义错误的处理方式:

**自定义错误消息:**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors="Please provide a valid rating between 1-5 and include a comment."
)
```

如果 `handle_errors` 是字符串,Agent 将*始终*用固定的工具消息提示模型重试:

```text
================================= Tool Message =================================
Name: ProductRating

Please provide a valid rating between 1-5 and include a comment.
```

**只处理特定异常:**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors=ValueError  # 仅在 ValueError 时重试,其他异常抛出
)
```

如果 `handle_errors` 是异常类型,Agent 只会在抛出的异常是指定类型时重试(使用默认错误消息)。其他情况下异常会被抛出。

**处理多种异常类型:**

```python
ToolStrategy(
    schema=ProductRating,
    handle_errors=(ValueError, TypeError)  # 在 ValueError 和 TypeError 时重试
)
```

如果 `handle_errors` 是异常元组,Agent 只会在抛出的异常是指定类型之一时重试(使用默认错误消息)。其他情况下异常会被抛出。

**自定义错误处理函数:**

```python
from langchain.agents.structured_output import StructuredOutputValidationError
from langchain.agents.structured_output import MultipleStructuredOutputsError

def custom_error_handler(error: Exception) -> str:
    if isinstance(error, StructuredOutputValidationError):
        return "There was an issue with the format. Try again."
    elif isinstance(error, MultipleStructuredOutputsError):
        return "Multiple structured outputs were returned. Pick the most relevant one."
    else:
        return f"Error: {str(error)}"


agent = create_agent(
    model="gpt-5.5",
    tools=[],
    response_format=ToolStrategy(
                        schema=Union[ContactInfo, EventDetails],
                        handle_errors=custom_error_handler
                    )  # 默认:handle_errors=True
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "Extract info: John Doe (john@email.com) is organizing Tech Conference on March 15th"}]
})

for msg in result['messages']:
    # 如果消息实际上是 ToolMessage 对象(而不是 dict),检查其类名
    if type(msg).__name__ == "ToolMessage":
        print(msg.content)
    # 如果消息是字典,或者你想要一个兜底逻辑
    elif isinstance(msg, dict) and msg.get('tool_call_id'):
        print(msg['content'])
```

遇到 `StructuredOutputValidationError` 时:

```text
================================= Tool Message =================================
Name: ToolStrategy

There was an issue with the format. Try again.
```

遇到 `MultipleStructuredOutputsError` 时:

```text
================================= Tool Message =================================
Name: ToolStrategy

Multiple structured outputs were returned. Pick the most relevant one.
```

遇到其他错误时:

```text
================================= Tool Message =================================
Name: ToolStrategy

Error: <error message>
```

**不做错误处理:**

```python
response_format = ToolStrategy(
    schema=ProductRating,
    handle_errors=False  # 所有错误都抛出
)
```
