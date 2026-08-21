# Messages(消息)

> 原文:[Messages](https://docs.langchain.com/oss/python/langchain/messages)

消息(message)是 LangChain 中模型上下文的基本单位。它们表示模型的输入和输出,同时携带与 LLM 交互时表示对话状态所需的内容(content)和元数据(metadata)。

消息是包含以下内容的对象:

- [**角色(Role)**](#消息类型):标识消息类型(例如 `system`、`user`)
- [**内容(Content)**](#消息内容):表示消息的实际内容(如文本、图像、音频、文档等)
- [**元数据(Metadata)**](#消息元数据):可选字段,如响应信息、消息 ID 和 token 用量

LangChain 提供了一套适用于所有模型提供商的标准消息类型,确保无论调用哪个模型,行为都保持一致。

## 基本用法

使用消息最简单的方式是创建消息对象,并在[调用](https://docs.langchain.com/oss/python/langchain/models#调用invocation)模型时传入。

```python
from langchain.chat_models import init_chat_model
from langchain.messages import HumanMessage, AIMessage, SystemMessage

model = init_chat_model("gpt-5-nano")

system_msg = SystemMessage("You are a helpful assistant.")
human_msg = HumanMessage("Hello, how are you?")

# 与对话模型一起使用
messages = [system_msg, human_msg]
response = model.invoke(messages)  # 返回 AIMessage
```

> **提示**
>
> 多轮 [Agent](https://docs.langchain.com/oss/python/langchain/agents) 会累积很长的消息历史。[LangSmith](https://docs.langchain.com/langsmith/observability) 会记录每一轮对话、工具结果和模型响应,让你可以检查完整对话。可参考[追踪快速入门](https://docs.langchain.com/langsmith/trace-with-langchain)启用追踪。我们还建议同时配置 [LangSmith Engine](https://docs.langchain.com/langsmith/engine),它可以监控你的追踪数据、检测问题并提出修复建议。

### 文本提示(Text prompts)

文本提示就是字符串——适合不需要保留对话历史的简单生成任务。

```python
response = model.invoke("Write a haiku about spring")
```

**适合使用文本提示的场景:**

- 单次、独立的请求
- 不需要对话历史
- 想要最少的代码复杂度

### 消息提示(Message prompts)

另外,你也可以通过提供消息对象列表,向模型传入一组消息。

```python
from langchain.messages import SystemMessage, HumanMessage, AIMessage

messages = [
    SystemMessage("You are a poetry expert"),
    HumanMessage("Write a haiku about spring"),
    AIMessage("Cherry blossoms bloom...")
]
response = model.invoke(messages)
```

**适合使用消息提示的场景:**

- 管理多轮对话
- 处理多模态内容(图像、音频、文件)
- 包含系统指令

### 字典格式(Dictionary format)

你也可以直接用 OpenAI chat completions 格式指定消息。

```python
messages = [
    {"role": "system", "content": "You are a poetry expert"},
    {"role": "user", "content": "Write a haiku about spring"},
    {"role": "assistant", "content": "Cherry blossoms bloom..."}
]
response = model.invoke(messages)
```

## 消息类型

- [System message(系统消息)](#system-message系统消息):告诉模型如何行动,并为交互提供上下文
- [Human message(人类消息)](#human-message人类消息):表示用户输入以及与模型的交互
- [AI message(AI 消息)](#ai-messageai-消息):模型生成的响应,包括文本内容、工具调用和元数据
- [Tool message(工具消息)](#tool-message工具消息):表示[工具调用](https://docs.langchain.com/oss/python/langchain/models#工具调用tool-calling)的输出

### System message(系统消息)

[`SystemMessage`](https://reference.langchain.com/python/langchain-core/messages/system/SystemMessage) 表示一组初始指令,用于预先塑造模型的行为。你可以用系统消息设定语气、定义模型的角色,并为响应建立准则。

```python
# 基本指令
system_msg = SystemMessage("You are a helpful coding assistant.")

messages = [
    system_msg,
    HumanMessage("How do I create a REST API?")
]
response = model.invoke(messages)
```

```python
# 详细的人设(persona)
from langchain.messages import SystemMessage, HumanMessage

system_msg = SystemMessage("""
You are a senior Python developer with expertise in web frameworks.
Always provide code examples and explain your reasoning.
Be concise but thorough in your explanations.
""")

messages = [
    system_msg,
    HumanMessage("How do I create a REST API?")
]
response = model.invoke(messages)
```

---

### Human message(人类消息)

[`HumanMessage`](https://reference.langchain.com/python/langchain-core/messages/human/HumanMessage) 表示用户输入和交互。它可以包含文本、图像、音频、文件以及其他任意数量的多模态[内容](#消息内容)。

#### 文本内容

```python
# 消息对象
response = model.invoke([
  HumanMessage("What is machine learning?")
])
```

```python
# 字符串快捷方式:使用字符串是传入单条 HumanMessage 的快捷方式
response = model.invoke("What is machine learning?")
```

#### 消息元数据

```python
human_msg = HumanMessage(
    content="Hello!",
    name="alice",  # 可选:标识不同的用户
    id="msg_123",  # 可选:用于追踪的唯一标识符
)
```

> **注意**:`name` 字段的行为因提供商而异——有些用它来标识用户,有些则忽略它。如需确认,请参见模型提供商的[参考文档](https://reference.langchain.com/python/integrations/)。

---

### AI message(AI 消息)

[`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 表示一次模型调用的输出。它可以包含多模态数据、工具调用以及提供商特有的元数据,你可以稍后访问这些信息。

```python
response = model.invoke("Explain AI")
print(type(response))  # <class 'langchain.messages.AIMessage'>
```

调用模型时会返回 [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 对象,其中包含响应中所有相关的元数据。

提供商对不同类型消息的权重/上下文化处理方式不同,这意味着有时手动创建一个新的 [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 对象并将其插入消息历史(就像它来自模型一样)会很有帮助。

```python
from langchain.messages import AIMessage, SystemMessage, HumanMessage

# 手动创建一条 AI 消息(例如用于对话历史)
ai_msg = AIMessage("I'd be happy to help you with that question!")

# 添加到对话历史
messages = [
    SystemMessage("You are a helpful assistant"),
    HumanMessage("Can you help me?"),
    ai_msg,  # 像来自模型一样插入
    HumanMessage("Great! What's 2+2?")
]

response = model.invoke(messages)
```

#### 属性(Attributes)

| 属性 | 类型 | 说明 |
|------|------|------|
| `text` | string | 消息的文本内容。 |
| `content` | string \| dict[] | 消息的原始内容。 |
| `content_blocks` | ContentBlock[] | 消息的标准化[内容块](#消息内容)。 |
| `tool_calls` | dict[] \| None | 模型发起的工具调用。未调用工具时为空。 |
| `id` | string | 消息的唯一标识符(由 LangChain 自动生成,或在提供商响应中返回)。 |
| `usage_metadata` | dict \| None | 消息的用量元数据,可用时可包含 token 数。 |
| `response_metadata` | ResponseMetadata \| None | 消息的响应元数据。 |

#### 工具调用(Tool calls)

当模型发起[工具调用](https://docs.langchain.com/oss/python/langchain/models#工具调用tool-calling)时,它们会包含在 [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 中:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-5-nano")

def get_weather(location: str) -> str:
    """获取某地的天气。"""
    ...

model_with_tools = model.bind_tools([get_weather])
response = model_with_tools.invoke("What's the weather in Paris?")

for tool_call in response.tool_calls:
    print(f"Tool: {tool_call['name']}")
    print(f"Args: {tool_call['args']}")
    print(f"ID: {tool_call['id']}")
```

其他结构化数据(如推理或引用)也可能出现在消息[内容](#消息内容)中。

#### Token 用量(Token usage)

[`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 可以在其 [`usage_metadata`](https://reference.langchain.com/python/langchain-core/messages/ai/UsageMetadata) 字段中保存 token 数和其他用量元数据:

```python
from langchain.chat_models import init_chat_model

model = init_chat_model("gpt-5-nano")

response = model.invoke("Hello!")
response.usage_metadata
```

```python
{'input_tokens': 8,
 'output_tokens': 304,
 'total_tokens': 312,
 'input_token_details': {'audio': 0, 'cache_read': 0},
 'output_token_details': {'audio': 0, 'reasoning': 256}}
```

细节参见 [`UsageMetadata`](https://reference.langchain.com/python/langchain-core/messages/ai/UsageMetadata)。

#### 流式与块(Streaming and chunks)

流式输出期间,你会收到 [`AIMessageChunk`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessageChunk) 对象,它们可以合并成一个完整的消息对象:

```python
chunks = []
full_message = None
for chunk in model.stream("Hi"):
    chunks.append(chunk)
    print(chunk.text)
    full_message = chunk if full_message is None else full_message + chunk
```

> **了解更多**:
>
> - [从对话模型流式获取 token](https://docs.langchain.com/oss/python/langchain/models#stream)
> - [从 Agent 流式获取 token 和/或步骤](https://docs.langchain.com/oss/python/langchain/streaming)

---

### Tool message(工具消息)

对于支持[工具调用](https://docs.langchain.com/oss/python/langchain/models#工具调用tool-calling)的模型,AI 消息可以包含工具调用。工具消息用于把单次工具执行的结果传回模型。

[工具](https://docs.langchain.com/oss/python/langchain/tools)可以直接生成 [`ToolMessage`](https://reference.langchain.com/python/langchain-core/messages/tool/ToolMessage) 对象。下面展示一个简单示例,更多内容参见[工具指南](https://docs.langchain.com/oss/python/langchain/tools)。

```python
from langchain.messages import AIMessage
from langchain.messages import ToolMessage

# 在模型发起工具调用之后
# (为简洁起见,这里演示手动创建消息)
ai_message = AIMessage(
    content=[],
    tool_calls=[{
        "name": "get_weather",
        "args": {"location": "San Francisco"},
        "id": "call_123"
    }]
)

# 执行工具并创建结果消息
weather_result = "Sunny, 72°F"
tool_message = ToolMessage(
    content=weather_result,
    tool_call_id="call_123"  # 必须与调用 ID 匹配
)

# 继续对话
messages = [
    HumanMessage("What's the weather in San Francisco?"),
    ai_message,  # 模型的工具调用
    tool_message,  # 工具执行结果
]
response = model.invoke(messages)  # 模型处理结果
```

#### 属性(Attributes)

| 属性 | 类型 | 说明 |
|------|------|------|
| `content` | string(必填) | 工具调用的字符串化输出。 |
| `tool_call_id` | string(必填) | 本消息所响应的工具调用的 ID。必须与 [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 中工具调用的 ID 匹配。 |
| `name` | string(必填) | 被调用的工具名称。 |
| `artifact` | dict | 不发送给模型、但可以通过编程方式访问的附加数据。 |

> **说明**:
>
> `artifact` 字段存储不会发送给模型、但可以以编程方式访问的补充数据。这对于存储原始结果、调试信息或供下游处理使用的数据很有用,而不会污染模型的上下文。
>
> **示例:使用 artifact 存储检索元数据**
>
> 例如,[检索](https://docs.langchain.com/oss/python/deepagents/retrieval)工具可以从文档中检索一段文字供模型参考。消息的 `content` 包含模型将参考的文本,而 `artifact` 可以包含应用可以使用的文档标识符或其他元数据(例如用于渲染页面)。示例如下:
>
> ```python
> from langchain.messages import ToolMessage
>
> # 发送给模型
> message_content = "It was the best of times, it was the worst of times."
>
> # 供下游使用的 artifact
> artifact = {"document_id": "doc_123", "page": 0}
>
> tool_message = ToolMessage(
>     content=message_content,
>     tool_call_id="call_123",
>     name="search_books",
>     artifact=artifact,
> )
> ```
>
> 使用 LangChain 构建检索 [Agent](https://docs.langchain.com/oss/python/langchain/agents) 的端到端示例参见 [RAG 教程](https://docs.langchain.com/oss/python/deepagents/rag)。

---

## 消息内容

可以把消息的内容(content)理解为发送给模型的数据载荷(payload)。消息有一个 `content` 属性,它是弱类型的(loosely-typed),支持字符串和无类型对象(如字典)的列表。这使得 LangChain 对话模型可以直接支持提供商原生的结构,例如[多模态](#多模态multimodal)内容和其他数据。

此外,LangChain 为文本、推理、引用、多模态数据、服务器端工具调用以及其他消息内容提供了专门的内容类型。参见下面的[内容块](#标准内容块standard-content-blocks)。

LangChain 对话模型接受 `content` 属性中的消息内容。它可以是以下任意一种:

1. 一个字符串
2. 提供商原生格式的内容块列表
3. [LangChain 标准内容块](#标准内容块standard-content-blocks)的列表

以下是使用[多模态](#多模态multimodal)输入的示例:

```python
from langchain.messages import HumanMessage

# 字符串内容
human_message = HumanMessage("Hello, how are you?")

# 提供商原生格式(例如 OpenAI)
human_message = HumanMessage(content=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
])

# 标准内容块列表
human_message = HumanMessage(content_blocks=[
    {"type": "text", "text": "Hello, how are you?"},
    {"type": "image", "url": "https://example.com/image.jpg"},
])
```

> **提示**:初始化消息时指定 `content_blocks` 仍然会填充消息的 `content`,但提供了类型安全的接口。

### 标准内容块(Standard content blocks)

LangChain 为消息内容提供了一种跨提供商工作的标准表示。

消息对象实现了 `content_blocks` 属性,它会惰性地把 `content` 属性解析成标准的、类型安全的表示。例如,由 [`ChatAnthropic`](https://docs.langchain.com/oss/python/integrations/chat/anthropic) 或 [`ChatOpenAI`](https://docs.langchain.com/oss/python/integrations/chat/openai) 生成的消息会包含各自提供商格式的 `thinking` 或 `reasoning` 块,但可以被惰性解析成一致的 [`ReasoningContentBlock`](#内容块参考content-block-reference) 表示:

**Anthropic:**

```python
from langchain.messages import AIMessage

message = AIMessage(
    content=[
        {"type": "thinking", "thinking": "...", "signature": "WaUjzkyp..."},
        {"type": "text", "text": "..."},
    ],
    response_metadata={"model_provider": "anthropic"}
)
message.content_blocks
```

```python
[{'type': 'reasoning',
  'reasoning': '...',
  'extras': {'signature': 'WaUjzkyp...'}},
 {'type': 'text', 'text': '...'}]
```

**OpenAI:**

```python
from langchain.messages import AIMessage

message = AIMessage(
    content=[
        {
            "type": "reasoning",
            "id": "rs_abc123",
            "summary": [
                {"type": "summary_text", "text": "summary 1"},
                {"type": "summary_text", "text": "summary 2"},
            ],
        },
        {"type": "text", "text": "...", "id": "msg_abc123"},
    ],
    response_metadata={"model_provider": "openai"}
)
message.content_blocks
```

```python
[{'type': 'reasoning', 'id': 'rs_abc123', 'reasoning': 'summary 1'},
 {'type': 'reasoning', 'id': 'rs_abc123', 'reasoning': 'summary 2'},
 {'type': 'text', 'text': '...', 'id': 'msg_abc123'}]
```

要开始使用你所选择的推理提供商,参见[集成指南](https://docs.langchain.com/oss/python/integrations/providers/overview)。

> **注意:序列化标准内容**
>
> 如果 LangChain 之外的应用需要访问标准内容块表示,你可以选择把内容块存储在消息内容中。为此,可以把环境变量 `LC_OUTPUT_VERSION` 设为 `v1`;或者用 `output_version="v1"` 初始化任意对话模型:
>
> ```python
> from langchain.chat_models import init_chat_model
>
> model = init_chat_model("gpt-5-nano", output_version="v1")
> ```

### 多模态(Multimodal)

**多模态(Multimodality)**指的是处理不同形式数据的能力,如文本、音频、图像和视频。LangChain 为这些数据提供了可跨提供商使用的标准类型。

[对话模型](https://docs.langchain.com/oss/python/langchain/models)可以接受多模态数据作为输入,并生成多模态输出。下面展示包含多模态数据的输入消息的简短示例。

> **注意**:额外的键可以放在内容块的顶层,也可以嵌套在 `"extras": {"key": value}` 中。例如 [OpenAI](https://docs.langchain.com/oss/python/integrations/chat/openai) 要求 PDF 提供文件名。具体细节参见你所选模型的[提供商页面](https://docs.langchain.com/oss/python/integrations/providers/overview)。

**图像输入:**

```python
# 从 URL
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this image."},
        {"type": "image", "url": "https://example.com/path/to/image.jpg"},
    ]
}

# 从 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this image."},
        {
            "type": "image",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "image/jpeg",
        },
    ]
}

# 从提供商托管的 File ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this image."},
        {"type": "image", "file_id": "file-abc123"},
    ]
}
```

**PDF 文档输入:**

```python
# 从 URL
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this document."},
        {"type": "file", "url": "https://example.com/path/to/document.pdf"},
    ]
}

# 从 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this document."},
        {
            "type": "file",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "application/pdf",
        },
    ]
}

# 从提供商托管的 File ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this document."},
        {"type": "file", "file_id": "file-abc123"},
    ]
}
```

**音频输入:**

```python
# 从 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this audio."},
        {
            "type": "audio",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "audio/wav",
        },
    ]
}

# 从提供商托管的 File ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this audio."},
        {"type": "audio", "file_id": "file-abc123"},
    ]
}
```

**视频输入:**

```python
# 从 base64 数据
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this video."},
        {
            "type": "video",
            "base64": "AAAAIGZ0eXBtcDQyAAAAAGlzb21tcDQyAAACAGlzb2...",
            "mime_type": "video/mp4",
        },
    ]
}

# 从提供商托管的 File ID
message = {
    "role": "user",
    "content": [
        {"type": "text", "text": "Describe the content of this video."},
        {"type": "video", "file_id": "file-abc123"},
    ]
}
```

> **警告**:并非所有模型都支持所有文件类型。支持的格式和大小限制请参见模型提供商的[参考文档](https://reference.langchain.com/python/integrations/)。

### 内容块参考(Content block reference)

内容块(无论是创建消息时还是访问 `content_blocks` 属性时)都表示为带类型字典的列表。列表中的每一项都必须遵循以下某种块类型:

#### 核心(Core)

**TextContentBlock** —— 用途:标准文本输出

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"text"` |
| `text` | string(必填) | 文本内容 |
| `annotations` | object[] | 文本的注解列表 |
| `extras` | object | 额外的提供商特有数据 |

```python
{
    "type": "text",
    "text": "Hello world",
    "annotations": []
}
```

**ReasoningContentBlock** —— 用途:模型推理步骤

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"reasoning"` |
| `reasoning` | string | 推理内容 |
| `extras` | object | 额外的提供商特有数据 |

```python
{
    "type": "reasoning",
    "reasoning": "The user is asking about...",
    "extras": {"signature": "abc123"},
}
```

#### 多模态(Multimodal)

**ImageContentBlock** —— 用途:图像数据

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"image"` |
| `url` | string | 指向图像位置的 URL |
| `base64` | string | Base64 编码的图像数据 |
| `id` | string | 本内容块的唯一标识符(由提供商或 LangChain 生成) |
| `mime_type` | string | 图像 [MIME 类型](https://www.iana.org/assignments/media-types/media-types.xhtml#image)(如 `image/jpeg`、`image/png`)。base64 数据必填。 |

**AudioContentBlock** —— 用途:音频数据

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"audio"` |
| `url` | string | 指向音频位置的 URL |
| `base64` | string | Base64 编码的音频数据 |
| `id` | string | 本内容块的唯一标识符(由提供商或 LangChain 生成) |
| `mime_type` | string | 音频 [MIME 类型](https://www.iana.org/assignments/media-types/media-types.xhtml#audio)(如 `audio/mpeg`、`audio/wav`)。base64 数据必填。 |

**VideoContentBlock** —— 用途:视频数据

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"video"` |
| `url` | string | 指向视频位置的 URL |
| `base64` | string | Base64 编码的视频数据 |
| `id` | string | 本内容块的唯一标识符(由提供商或 LangChain 生成) |
| `mime_type` | string | 视频 [MIME 类型](https://www.iana.org/assignments/media-types/media-types.xhtml#video)(如 `video/mp4`、`video/webm`)。base64 数据必填。 |

**FileContentBlock** —— 用途:通用文件(PDF 等)

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"file"` |
| `url` | string | 指向文件位置的 URL |
| `base64` | string | Base64 编码的文件数据 |
| `id` | string | 本内容块的唯一标识符(由提供商或 LangChain 生成) |
| `mime_type` | string | 文件 [MIME 类型](https://www.iana.org/assignments/media-types/media-types.xhtml)(如 `application/pdf`)。base64 数据必填。 |

**PlainTextContentBlock** —— 用途:文档文本(`.txt`、`.md`)

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"text-plain"` |
| `text` | string | 文本内容 |
| `mime_type` | string | 文本的 [MIME 类型](https://www.iana.org/assignments/media-types/media-types.xhtml)(如 `text/plain`、`text/markdown`) |

#### 工具调用(Tool Calling)

**ToolCall** —— 用途:函数调用

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"tool_call"` |
| `name` | string(必填) | 要调用的工具名称 |
| `args` | object(必填) | 传给工具的参数 |
| `id` | string(必填) | 本次工具调用的唯一标识符 |

```python
{
    "type": "tool_call",
    "name": "search",
    "args": {"query": "weather"},
    "id": "call_123"
}
```

**ToolCallChunk** —— 用途:流式工具调用片段

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"tool_call_chunk"` |
| `name` | string | 被调用工具的名称 |
| `args` | string | 部分工具参数(可能是不完整的 JSON) |
| `id` | string | 工具调用标识符 |
| `index` | number \| string | 本块在流中的位置 |

**InvalidToolCall** —— 用途:格式错误的调用,旨在捕获 JSON 解析错误

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"invalid_tool_call"` |
| `name` | string | 调用失败的工具名称 |
| `args` | object | 传给工具的参数 |
| `error` | string | 出错原因的描述 |

#### 服务器端工具执行(Server-Side Tool Execution)

**ServerToolCall** —— 用途:在服务器端执行的工具调用

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"server_tool_call"` |
| `id` | string(必填) | 与工具调用关联的标识符 |
| `name` | string(必填) | 要调用的工具名称 |
| `args` | string(必填) | 部分工具参数(可能是不完整的 JSON) |

**ServerToolCallChunk** —— 用途:流式服务器端工具调用片段

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"server_tool_call_chunk"` |
| `id` | string | 与工具调用关联的标识符 |
| `name` | string | 被调用工具的名称 |
| `args` | string | 部分工具参数(可能是不完整的 JSON) |
| `index` | number \| string | 本块在流中的位置 |

**ServerToolResult** —— 用途:搜索结果

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"server_tool_result"` |
| `tool_call_id` | string(必填) | 对应服务器端工具调用的标识符 |
| `id` | string | 与服务器端工具结果关联的标识符 |
| `status` | string(必填) | 服务器端工具的执行状态:`"success"` 或 `"error"` |
| `output` | — | 所执行工具的输出 |

#### 提供商特有块(Provider-Specific Blocks)

**NonStandardContentBlock** —— 用途:提供商特有的逃生舱(escape hatch)

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string(必填) | 恒为 `"non_standard"` |
| `value` | object(必填) | 提供商特有的数据结构 |

**用法:**用于实验性特性或提供商独有的特性。更多提供商特有的内容类型可在各模型提供商的[参考文档](https://docs.langchain.com/oss/python/integrations/providers/overview)中找到。

> **提示**:规范的类型定义参见 [API 参考](https://reference.langchain.com/python/langchain/messages)。

> **说明**:内容块是 LangChain v1 中作为消息的新属性引入的,目的是在保持与现有代码向后兼容的同时,跨提供商标准化内容格式。内容块并不是 [`content`](https://reference.langchain.com/python/langchain-core/messages/base/BaseMessage) 属性的替代品,而是一个新属性,可用于以标准化格式访问消息内容。

## 序列化(Serialization)

你可以把消息序列化为普通对象以便存储,也可以反序列化回消息实例。这对于持久化对话历史和恢复会话很有用。

```python
from langchain.messages import HumanMessage
from langchain_core.load import dumpd, load

message = HumanMessage("What is the capital of France?")

# 序列化为普通 dict
serialized = dumpd(message)

# 反序列化回消息对象
restored = load(serialized)
```

> **警告**:
>
> **`load()` 会实例化 Python 对象,并可能在反序列化期间触发副作用。绝不要对来自不可信或未经验证来源的数据调用 `load()`。**

## 与对话模型配合使用

[对话模型](https://docs.langchain.com/oss/python/langchain/models)接受一个消息对象序列作为输入,并返回一个 [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 作为输出。交互通常是无状态的,因此简单的对话循环就是用不断增长的消息列表来调用模型。

参考以下指南了解更多:

- [持久化与管理对话历史](https://docs.langchain.com/oss/python/langchain/short-term-memory)的内置特性
- 管理上下文窗口的策略,包括[裁剪与摘要消息](https://docs.langchain.com/oss/python/langchain/short-term-memory#common-patterns)
