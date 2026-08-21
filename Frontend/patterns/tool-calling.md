# 工具调用 UI(Tool calling)

> 以类型安全的富 UI 卡片展示 Agent 的工具调用

> 原文:[Tool calling](https://docs.langchain.com/oss/python/langchain/frontend/tool-calling)

(页面内嵌的交互式示例 demo 此处省略)

## 工具调用的工作原理

当 LangGraph Agent 判断需要外部数据时,它会作为 AI 消息的一部分发出一个或多个**工具调用(tool calls)**。每个工具调用包含:

* **name**:被调用的工具(例如 `"get_weather"`、`"calculator"`)
* **args**:传递给工具的结构化参数
* **id**:将调用与其结果关联起来的唯一标识符

Agent 运行时(runtime)负责执行工具,执行结果以 `ToolMessage` 的形式返回。[`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) hook 将所有这些统一为一个 `toolCalls` 数组,你可以直接渲染它。

## 配置 `useStream`

第一步是将 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 与你的 Agent 后端连接起来。该 hook 返回响应式状态,其中包含一个 `toolCalls` 数组,会随着 Agent 的流式输出实时更新。

> **说明**
>
> 代码示例使用 `useStream<typeof myAgent>` 以获得类型安全的流式状态。类型推断(Type inference)参见 [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) 或 [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference) 后端的相关章节。

**React:**

```tsx
import { useStream } from "@langchain/react";

const AGENT_URL = "http://localhost:2024";

export function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "tool_calling",
  });

  return (
    <div>
      {stream.messages.map((msg) => (
        <Message key={msg.id} message={msg} toolCalls={stream.toolCalls} />
      ))}
    </div>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
import { useStream } from "@langchain/vue";

const AGENT_URL = "http://localhost:2024";

const stream = useStream<typeof myAgent>({
  apiUrl: AGENT_URL,
  assistantId: "tool_calling",
});
</script>

<template>
  <div>
    <Message
      v-for="msg in stream.messages.value"
      :key="msg.id"
      :message="msg"
      :tool-calls="stream.toolCalls.value"
    />
  </div>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream } from "@langchain/svelte";

  const AGENT_URL = "http://localhost:2024";

  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "tool_calling",
  });
</script>

<div>
  {#each stream.messages as msg (msg.id)}
    <Message message={msg} toolCalls={stream.toolCalls} />
  {/each}
</div>
```

**Angular:**

```ts
import { Component } from "@angular/core";
import { injectStream } from "@langchain/angular";

const AGENT_URL = "http://localhost:2024";

@Component({
  selector: "app-chat",
  template: `
    @for (msg of stream.messages(); track msg.id) {
      <app-message [message]="msg" [toolCalls]="stream.toolCalls()" />
    }
  `,
})
export class ChatComponent {
  stream = injectStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "tool_calling",
  });
}
```

## AssembledToolCall 类型

`toolCalls` 数组中的每一项都是一个 `AssembledToolCall` 对象:

```ts
interface AssembledToolCall<
  TName extends string = string,
  TInput = unknown,
  TOutput = unknown,
> {
  name: TName;
  callId: string;
  id: string;
  namespace: string[];
  input: TInput;
  args: TInput;
  output: TOutput | null;
  status: "running" | "finished" | "error";
  error: string | undefined;
}
```

| 属性 | 说明 |
| ----------- | ------------------------------------------------------------------------------ |
| `name`      | 工具名称(例如 `"get_weather"`) |
| `callId`    | 与 AI 消息中 `tool_calls` 条目相匹配的唯一 ID |
| `id`        | `callId` 的别名,与消息级工具调用保持一致 |
| `namespace` | 发出该工具调用所在的命名空间 |
| `input`     | Agent 传递给工具的结构化参数 |
| `args`      | `input` 的别名,与消息级工具调用保持一致 |
| `output`    | 调用成功后的工具输出;运行中或出错后为 `null` |
| `status`    | 生命周期状态:`"running"`(运行中)、`"finished"`(已完成)或 `"error"`(出错) |
| `error`     | 工具调用失败时的错误详情 |

## 按消息过滤工具调用

一条 AI 消息可能触发多个工具调用,而你的聊天中可能包含多条 AI 消息。要在每条消息下方渲染正确的工具卡片,需将 `callId` 与该消息的 `tool_calls` 数组进行匹配过滤:

```tsx
function Message({
  message,
  toolCalls,
}: {
  message: AIMessage;
  toolCalls: AssembledToolCall[];
}) {
  const messageToolCalls = toolCalls.filter((tc) =>
    message.tool_calls?.find((t) => t.id === tc.callId)
  );

  return (
    <div>
      <p>{message.text}</p>
      {messageToolCalls.map((tc) => (
        <ToolCard key={tc.callId} toolCall={tc} />
      ))}
    </div>
  );
}
```

## 构建专用的工具卡片

不要把原始 JSON 直接倒给用户,而应为每个工具构建专用的 UI 组件。使用 `name` 来选择正确的卡片:

```tsx
function ToolCard({ toolCall }: { toolCall: AssembledToolCall }) {
  if (toolCall.status === "running") {
    return <LoadingCard name={toolCall.name} />;
  }

  if (toolCall.status === "error") {
    return <ErrorCard name={toolCall.name} error={toolCall.error} />;
  }

  switch (toolCall.name) {
    case "get_weather":
      return <WeatherCard input={toolCall.input} output={toolCall.output} />;
    case "calculator":
      return (
        <CalculatorCard input={toolCall.input} output={toolCall.output} />
      );
    case "web_search":
      return <SearchCard input={toolCall.input} output={toolCall.output} />;
    default:
      return <GenericToolCard toolCall={toolCall} />;
  }
}
```

### 天气卡片示例

```tsx
function WeatherCard({
  input,
  output,
}: {
  input: { location: string };
  output: { temperature: number; condition: string };
}) {
  return (
    <div className="rounded-lg border p-4">
      <div className="flex items-center gap-2">
        <CloudIcon />
        <h3 className="font-semibold">{input.location}</h3>
      </div>
      <div className="mt-2 text-3xl font-bold">{output.temperature}°F</div>
      <p className="text-muted-foreground">{output.condition}</p>
    </div>
  );
}
```

### 加载与错误状态

始终处理等待(pending)和错误(error)状态,以给用户清晰的反馈:

```tsx
function LoadingCard({ name }: { name: string }) {
  return (
    <div className="flex items-center gap-2 rounded-lg border p-4 animate-pulse">
      <Spinner />
      <span>Running {name}...</span>
    </div>
  );
}

function ErrorCard({ name, error }: { name: string; error?: unknown }) {
  return (
    <div className="rounded-lg border border-red-300 bg-red-50 p-4">
      <h3 className="font-semibold text-red-700">Error in {name}</h3>
      <p className="text-sm text-red-600">
        {String(error ?? "Tool execution failed")}
      </p>
    </div>
  );
}
```

## 类型安全的工具参数

如果你的工具定义了结构化 schema,可以使用 `ToolCallFromTool` 工具类型来获得完全类型化的 `args`:

```ts
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const getWeather = tool(async ({ location }) => { /* ... */ }, {
  name: "get_weather",
  description: "Get the current weather for a location",
  schema: z.object({
    location: z.string().describe("City name"),
  }),
});

type WeatherToolCall = ToolCallFromTool<typeof getWeather>;
// WeatherToolCall.input 和 WeatherToolCall.args 现在都是 { location: string }
```

> **提示**
>
> 使用 `ToolCallFromTool` 可以获得编译期安全保障。如果工具的 schema 发生变化,你的 UI 组件会立即标记出类型错误。

## 工具调用与流式文本的内联渲染

工具调用经常与流式文本交错到达。[`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) hook 会让 `toolCalls` 与流保持同步,因此一旦 Agent 发出调用、工具还未执行完毕,等待中的卡片就会立即出现。

这意味着用户会看到:

1. AI 的文本以流式方式逐步呈现
2. 工具调用一经发出,立即出现加载卡片
3. 工具执行完成后,卡片更新以显示结果

> **注意**
>
> 工具调用是原地更新的。同一个 `callId` 会从 `"running"` 转变为 `"finished"`(或 `"error"`),因此你的 UI 会以新状态重新渲染同一个组件。

## 处理多个并发工具调用

Agent 可以并行调用多个工具。`toolCalls` 数组中会同时存在多个 `status: "running"` 的条目。每个调用独立完成,因此你的 UI 应能优雅地处理部分完成的状态:

```tsx
function ToolCallList({ toolCalls }: { toolCalls: AssembledToolCall[] }) {
  const pending = toolCalls.filter((tc) => tc.status === "running");
  const completed = toolCalls.filter((tc) => tc.status === "finished");

  return (
    <div className="space-y-2">
      {completed.map((tc) => (
        <ToolCard key={tc.callId} toolCall={tc} />
      ))}
      {pending.map((tc) => (
        <LoadingCard key={tc.callId} name={tc.name} />
      ))}
    </div>
  );
}
```

## 最佳实践

构建工具调用 UI 时遵循以下准则:

* **始终处理全部三种状态**:`running`、`finished` 和 `error`。用户不应看到空白卡片。
* **安全地校验结果**。在为特定卡片做类型收窄之前,工具输出的类型是 `unknown`。
* **提供通用的兜底卡片**。并非每个工具都需要定制卡片;对未知工具名渲染一个可折叠的 JSON 视图即可。
* **在加载期间展示工具名和参数**。用户想知道 Agent *正在做什么*,即使结果还没返回。
* **保持卡片紧凑**。工具卡片与聊天消息内联展示,避免用过大的组件喧宾夺主。
