# 分支对话(Branching chat)

> 基于检查点分叉对话,支持编辑消息与重新生成响应

> 原文:[Branching chat](https://docs.langchain.com/oss/python/langchain/frontend/branching-chat)

与 AI Agent 的对话很少是线性的。你可能想重新措辞一个问题、重新生成一个不满意的回答,或者在不丢失检查点历史的前提下探索不同的对话路径。分支对话(Branching chat)将 LangGraph 检查点(checkpoints)用作分叉点(fork points):每一次编辑或重新生成,都会从所选消息的父检查点提交一次新的运行。

(页面内嵌的交互式示例 demo 此处省略)

> **注意**
>
> 此功能需要 [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server)。请使用 `langgraph dev` 在本地运行你的 Agent,或[部署到 LangSmith](https://docs.langchain.com/langsmith/deployment) 以使用此模式。

## 什么是分支对话?

分支对话将会话视为一个**带检查点的时间线**,而非一个扁平的列表。每条消息都带有元数据,指向该消息创建之前的检查点。编辑一条消息或重新生成一个响应,就是从该检查点提交一次新的运行。

核心能力:

* **编辑任意用户消息:** 重写先前的提示,并从该点重新运行 Agent
* **重新生成任意 AI 响应:** 让 Agent 对同一输入给出不同的回答
* **检查历史:** 当你需要分支时间线时,使用 LangGraph 客户端加载检查点

## 配置流元数据(stream metadata)

对消息使用根流(root stream),然后在渲染每条消息的组件中读取逐消息的检查点元数据。该元数据包含用于分叉的父检查点 ID。

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
    assistantId: "simple_agent",
  });

  return (
    <div>
      {stream.messages.map((msg) => (
        <MessageWithForkControls key={msg.id} stream={stream} message={msg} />
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
  assistantId: "simple_agent",
});
</script>

<template>
  <div>
    <MessageWithForkControls
      v-for="msg in stream.messages.value"
      :key="msg.id"
      :stream="stream"
      :message="msg"
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
    assistantId: "simple_agent",
  });
</script>

<div>
  {#each stream.messages as msg (msg.id)}
    <Message
      message={msg}
      {stream}
    />
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
      <app-message
        [message]="msg"
        [stream]="stream"
      />
    }
  `,
})
export class ChatComponent {
  stream = injectStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "simple_agent",
  });
}
```

## 理解消息元数据

`useMessageMetadata(stream, messageId)` 辅助函数返回单条消息的 [MessageMetadata](https://reference.langchain.com/javascript/langchain-react/MessageMetadata)。在渲染每条消息的组件中使用它,使元数据的作用域限定在该消息 ID 内:

```tsx
import type { BaseMessage } from "langchain";
import { useState } from "react";
import { useMessageMetadata, useStream } from "@langchain/react";

function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "simple_agent",
  });

  return stream.messages.map((message) => (
    <MessageWithForkControls
      key={message.id}
      stream={stream}
      message={message}
    />
  ));
}

function MessageWithForkControls({
  stream,
  message,
}: {
  stream: ReturnType<typeof useStream>;
  message: BaseMessage;
}) {
  const metadata = useMessageMetadata(stream, message.id);
  const checkpointId = metadata?.parentCheckpointId;
  const [editedText, setEditedText] = useState(message.text);

  return (
    <form
      onSubmit={(event) => {
        event.preventDefault();
        if (!checkpointId) return;

        stream.submit(
          { messages: [{ type: "human", content: editedText }] },
          { forkFrom: { checkpointId } }
        );
      }}
    >
      <textarea
        value={editedText}
        onChange={(event) => setEditedText(event.target.value)}
      />
      <button disabled={!checkpointId || editedText === message.text}>
        Submit edited branch
      </button>
    </form>
  );
}
```

`parentCheckpointId` 是该消息之前的那个检查点。把它作为编辑与重新生成的分叉点。

## 编辑消息

要编辑一条用户消息并分叉对话:

1. 从该消息的元数据中取得 `parentCheckpointId`
2. 用 `forkFrom: { checkpointId }` 提交编辑后的消息
3. Agent 从该点重新运行

```ts
function handleEdit(
  stream: ReturnType<typeof useStream>,
  originalMsg: HumanMessage,
  metadata: MessageMetadata | undefined,
  newText: string
) {
  if (!metadata?.parentCheckpointId) return;

  stream.submit(
    {
      messages: [{ type: "human", content: newText }],
    },
    { forkFrom: { checkpointId: metadata.parentCheckpointId } }
  );
}
```

编辑之后:

* Agent 用更新后的消息从分叉点重新运行
* 原始路径仍保留在线程历史中可供访问

## 重新生成响应

要在不改变输入的情况下重新生成一个 AI 响应:

1. 从该 AI 消息的元数据中取得 `parent_checkpoint`
2. 用空输入并附带 `forkFrom: { checkpointId }` 提交
3. Agent 从该点生成一个全新的响应

```ts
function handleRegenerate(
  stream: ReturnType<typeof useStream>,
  metadata: MessageMetadata | undefined
) {
  if (!metadata?.parentCheckpointId) return;

  stream.submit(undefined, {
    forkFrom: { checkpointId: metadata.parentCheckpointId },
  });
}
```

每一次重新生成都会为该位置的 AI 消息创建一条新路径。

> **提示**
>
> 重新生成对非确定性的 Agent 很有用。由于 LLM 输出会随 temperature 变化,对同一提示重新生成往往能产生明显不同的响应。

## 分支的底层工作原理

LangGraph 将每一次状态转换都持久化为一个**检查点(checkpoint)**。当你用 `forkFrom` 提交时,后端会从该点开启一条新的执行路径,而不是追加到当前会话。其结果是一个树形结构:

```
用户: "什么是 React?"
  └─ AI: "React 是一个 JavaScript 库……"(分支 A)
  └─ AI: "React 是一个 UI 框架……"(分支 B,重新生成)

用户: "介绍一下 hooks"(分支 A)
  └─ AI: "Hooks 是一些函数……"

用户: "介绍一下 JSX"(由分支 A 编辑而来)
  └─ AI: "JSX 是一种语法扩展……"
```

每条路径都被持久化在检查点存储中。当你想构建一个跨检查点的独立时间线视图时,使用 `stream.client.threads.getHistory(threadId)`。

## 最佳实践

* **在消息附近读取元数据**:在渲染消息控件的组件中调用 `useMessageMetadata`。
* **悬停时显示分叉控件**:编辑与重新生成按钮应在悬停时出现,以保持界面整洁。
* **按需刷新历史**:仅在渲染时间线时、或在分叉落定之后才调用 `client.threads.getHistory()`。
* **流式期间禁用控件**:当 Agent 正在流式输出响应时,不要允许编辑或重新生成。在启用这些操作前检查 `stream.isLoading`。
* **取消时保留编辑文本**:如果用户开始编辑后又取消,应将文本域重置为原始消息内容。
* **用较深的检查点树做测试**:频繁编辑与重新生成的用户可能创建大量路径。确保时间线渲染依然流畅。
