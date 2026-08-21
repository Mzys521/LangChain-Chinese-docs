# 无头工具(Headless tools)

> 通过无头工具实现,在客户端运行浏览器与设备 API

> 原文:[Headless tools](https://docs.langchain.com/oss/python/langchain/frontend/headless-tools)

无头工具(Headless tools)允许你的 Agent 调用那些**真正执行必须发生在用户应用中、而非服务器上**的工具。Agent 看到的仍然是一个普通的工具 schema,但具体实现位于前端,从而可以访问 IndexedDB、地理位置、剪贴板、canvas 或文件选择器等浏览器 API。

当数据应当保留在设备本地时,这一模式尤其有用。本页面的 playground 示例使用了一个基于 IndexedDB 的小型浏览器记忆工具包,外加一个完全在客户端运行的地理位置工具。

(页面内嵌的交互式示例 demo 此处省略)

## 无头工具的工作原理

从高层次看,无头工具将**工具 schema**与**仅在浏览器中运行的实现**分离开来:

1. 在 Agent 上注册一个工具,该工具立即调用 `interrupt()` 将执行推迟到前端。
2. 在前端定义中镜像相同的工具名与参数字段。
3. 在前端用 `.implement(...)` 实现匹配的工具,并将它们传入 `useStream({ tools: [...] })`。
4. 当 Agent 调用某个匹配的工具时,客户端处理该动作,并用工具结果恢复被中断的运行。

## 在 Agent 上注册工具

playground 定义了一小组客户端工具,它们遵循相同的模式:Agent 暴露工具 schema,前端负责实际执行。

在服务器上定义普通工具,让它们立即调用 `interrupt()`,然后在前端的 `tools.ts` 文件中镜像相同的工具名与参数字段。

```python
# agent.py
from typing import Any

from langchain import create_agent
from langchain.tools import ToolRuntime, tool
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import interrupt
from pydantic import BaseModel


class MemoryPutInput(BaseModel):
    key: str
    value: Any


class MemoryGetInput(BaseModel):
    key: str


class GeolocationGetInput(BaseModel):
    save: bool = True


def _interrupt_for_client(
    tool_name: str,
    args: dict[str, Any],
    runtime: ToolRuntime,
) -> Any:
    return interrupt({
        "type": "tool",
        "tool_call": {
            "id": runtime.tool_call_id,
            "name": tool_name,
            "args": args,
        },
    })


@tool(
    "memory_put",
    description="Store a memory in the user's browser.",
    args_schema=MemoryPutInput,
)
def memory_put(key: str, value: Any, runtime: ToolRuntime) -> Any:
    return _interrupt_for_client(
        "memory_put",
        {"key": key, "value": value},
        runtime,
    )


@tool(
    "memory_get",
    description="Look up a memory stored in the user's browser.",
    args_schema=MemoryGetInput,
)
def memory_get(key: str, runtime: ToolRuntime) -> Any:
    return _interrupt_for_client("memory_get", {"key": key}, runtime)


@tool(
    "geolocation_get",
    description="Get the user's current location from the browser.",
    args_schema=GeolocationGetInput,
)
def geolocation_get(runtime: ToolRuntime, save: bool = True) -> Any:
    return _interrupt_for_client(
        "geolocation_get",
        {"save": save},
        runtime,
    )

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[memory_put, memory_get, geolocation_get],
    checkpointer=MemorySaver(),
)
```

每个工具都带着前端可以处理的结构化载荷(payload)进行中断,然后在运行恢复时返回所提供的值。在客户端镜像相同的工具名与 schema,前端就能挂载具体实现。

```ts
// tools.ts
import * as z from "zod";
import { tool } from "langchain";

// 在客户端镜像 Python 工具的名称与 schema。
export const memoryPut = tool({
  name: "memory_put",
  description: "Store a memory in the user's browser.",
  schema: z.object({
    key: z.string(),
    value: z.unknown(),
  }),
});

export const memoryGet = tool({
  name: "memory_get",
  description: "Look up a memory stored in the user's browser.",
  schema: z.object({
    key: z.string(),
  }),
});

export const geolocationGet = tool({
  name: "geolocation_get",
  description: "Get the user's current location from the browser.",
  schema: z.object({
    save: z.boolean().optional(),
  }),
});
```

## 实现浏览器行为

把仅在客户端运行的行为放到独立模块中,并用 `.implement(...)` 挂载。真实的 playground 包含一个功能更完整的 IndexedDB 存储,支持搜索、列举、过期与删除操作。下面的示例以更高层次展示了相同的结构:

```ts
// impl.ts
import {
  geolocationGet as geolocationGetDefinition,
  memoryGet as memoryGetDefinition,
  memoryPut as memoryPutDefinition,
} from "./tools";

async function saveMemory(key: string, value: unknown) {
  localStorage.setItem(`agent-memory:${key}`, JSON.stringify(value));
}

async function getMemory(key: string) {
  const value = localStorage.getItem(`agent-memory:${key}`);
  return value ? JSON.parse(value) : null;
}

export const memoryPut = memoryPutDefinition.implement(async ({ key, value }) => {
  await saveMemory(key, value);
  return { success: true, key };
});

export const memoryGet = memoryGetDefinition.implement(async ({ key }) => {
  const value = await getMemory(key);
  return value === null ? { found: false, key } : { found: true, key, value };
});

export const geolocationGet = geolocationGetDefinition.implement(
  async ({ save = true }) => {
    const position = await new Promise<GeolocationPosition>((resolve, reject) =>
      navigator.geolocation.getCurrentPosition(resolve, reject),
    );

    const location = {
      latitude: position.coords.latitude,
      longitude: position.coords.longitude,
      accuracy: position.coords.accuracy,
    };

    if (save) {
      await saveMemory("user_location", location);
    }

    return location;
  },
);
```

## 将实现接入 `useStream`

把已实现的工具传入 `useStream`。当 Agent 发出匹配的工具调用时,该 hook 会运行客户端实现并替你恢复运行。

定义一个与 Agent 状态 schema 相匹配的 TypeScript 接口,并将其作为类型参数传给 `useStream`,以获得类型安全的状态访问:

```ts
// types.ts
export interface AgentState {
  messages: BaseMessage[];
}
```

**React:**

```tsx
import { useStream } from "@langchain/react";

import { geolocationGet, memoryGet, memoryPut } from "./impl";
import type { AgentState } from "./types";

const AGENT_URL = "http://localhost:2024";

export function Chat() {
  const stream = useStream<AgentState>({
    apiUrl: AGENT_URL,
    assistantId: "headless_tools",
    tools: [memoryPut, memoryGet, geolocationGet],
  });

  return <ChatView messages={stream.messages} toolCalls={stream.toolCalls} />;
}
```

**Vue:**

```vue
<script setup lang="ts">
import { useStream } from "@langchain/vue";

import { geolocationGet, memoryGet, memoryPut } from "./impl";
import type { AgentState } from "./types";

const AGENT_URL = "http://localhost:2024";

const stream = useStream<AgentState>({
  apiUrl: AGENT_URL,
  assistantId: "headless_tools",
  tools: [memoryPut, memoryGet, geolocationGet],
});
</script>

<template>
  <ChatView
    :messages="stream.messages.value"
    :tool-calls="stream.toolCalls.value"
  />
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream } from "@langchain/svelte";

  import { geolocationGet, memoryGet, memoryPut } from "./impl";
  import type { AgentState } from "./types";

  const AGENT_URL = "http://localhost:2024";

  const { messages, toolCalls } = useStream<AgentState>({
    apiUrl: AGENT_URL,
    assistantId: "headless_tools",
    tools: [memoryPut, memoryGet, geolocationGet],
  });
</script>

<ChatView messages={$messages} toolCalls={$toolCalls} />
```

**Angular:**

```ts
import { Component } from "@angular/core";
import { useStream } from "@langchain/angular";

import { geolocationGet, memoryGet, memoryPut } from "./impl";
import type { AgentState } from "./types";

const AGENT_URL = "http://localhost:2024";

@Component({
  selector: "app-chat",
  template: `
    <app-chat-view
      [messages]="stream.messages()"
      [toolCalls]="stream.toolCalls()"
    />
  `,
})
export class ChatComponent {
  stream = useStream<AgentState>({
    apiUrl: AGENT_URL,
    assistantId: "headless_tools",
    tools: [memoryPut, memoryGet, geolocationGet],
  });
}
```

## 内联渲染工具活动

playground 将每次记忆或地理位置操作渲染为独立的卡片,并在输入框附近保留一个小型记忆统计面板。关键步骤是将 `stream.toolCalls` 中的每一项与其所触发的 AI 消息对应起来:

```tsx
import type { ToolCallWithResult, DefaultToolCall } from "@langchain/react";

function Message({ message, toolCalls }: {
  message: AIMessage,
  toolCalls: ToolCallWithResult[]
}) {
  const messageToolCalls = toolCalls.filter((tc) =>
    message.tool_calls?.some((call) => call.id === tc.call.id),
  );

  return (
    <div>
      {message.text && <p>{message.text}</p>}
      {messageToolCalls.map((tc) => (
        <HeadlessToolCard key={tc.call.id} toolCall={tc} />
      ))}
    </div>
  );
}
```

结合 [工具调用 UI](https://docs.langchain.com/oss/python/langchain/frontend/tool-calling) 中更丰富的 UI 模式,效果会更好——每个工具结果都可以渲染为专用卡片,而非原始 JSON。

## 使用场景

当任务依赖于仅存在于客户端的 API 或数据时,使用无头工具:

* IndexedDB 或 `localStorage` 中的本地记忆
* 地理位置、剪贴板、摄像头或文件选择器等设备 API
* Canvas、音频或其他仅在浏览器中可用的渲染原语
* 应保留在用户设备上的隐私敏感数据
* 需要直接访问前端内存中状态的 UI 操作

## 最佳实践

* 保持工具小而类型化。优先使用多个窄工具,而不是一个通用的"运行任意浏览器代码"工具。
* 返回可 JSON 序列化的结果。不要尝试返回 DOM 节点、文件句柄或其他不可序列化的浏览器对象。
* 共享定义,分离实现。Agent 与客户端应在工具名与 schema 上达成一致,但只有客户端才应加载浏览器 API。
* 在 UI 中呈现工具状态。使用 `stream.toolCalls` 和 `onTool` 来展示等待、成功与错误状态。
* 需要时增加审查环节。对于敏感的客户端操作,将此模式与 [人机协同(Human-in-the-loop)](https://docs.langchain.com/oss/python/langchain/frontend/human-in-the-loop) 结合使用。
