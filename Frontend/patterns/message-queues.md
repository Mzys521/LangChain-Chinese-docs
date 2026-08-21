# 消息队列(Message queues)

> 让用户快速连续发送多条消息,由 Agent 按顺序排队处理

> 原文:[Message queues](https://docs.langchain.com/oss/python/langchain/frontend/message-queues)

消息排队(Message queuing)让用户无需等待 Agent 处理完当前消息,就能快速连续发送多条消息。每条消息都被立即接收、进入当前线程的队列,并**按顺序处理**,让你对等待中的工作拥有完全的可见性与控制力。

(页面内嵌的交互式示例 demo 此处省略)

> **注意**
>
> 此功能需要 [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server)。请使用 `langgraph dev` 在本地运行你的 Agent,或[部署到 LangSmith](https://docs.langchain.com/langsmith/deployment) 以使用此模式。

## 为什么需要消息队列?

在典型的聊天界面中,用户必须等 Agent 完成回答后才能发送下一条消息。这在多种场景下都会造成摩擦:

* **批量提问**:用户想一次提出五个相关问题,而不是等待每个答案
* **追问链**:在 Agent 仍在工作时提交澄清或补充上下文
* **自动化测试序列**:以编程方式发送一系列提示来验证 Agent 行为
* **数据录入工作流**:一个接一个地送入结构化输入进行处理

消息排队通过立即接收所有提交并按顺序处理,解决了这些问题。

这是一种 Agent 的 UX 原语(primitive),而非表面的聊天功能。SDK 把队列作为流控制器(stream controller)的一部分来跟踪,因此你的 UI 可以展示待处理的工作、取消过期的请求,并在当前运行继续的同时保持输入框可用。

## 工作原理

当你希望一次提交排在当前正在运行的请求之后时,传入 `multitaskStrategy: "enqueue"`。当 Agent 正在处理时,排队中的提交会被添加到当前线程的队列中。当前运行完成后,队列中的下一条消息会自动派发。

使用与你所用框架配套的队列辅助函数读取队列状态:

| 属性 | 类型 | 说明 |
| ------------------ | ------------------------------- | ---------------------------------------- |
| `queue.entries`    | `SubmissionQueueEntry[]`        | 所有待处理队列条目的数组 |
| `queue.size`       | `number`                        | 队列当前的条目数量 |
| `queue.cancel(id)` | `(id: string) => Promise<void>` | 按 ID 取消队列中的某个条目 |
| `queue.clear()`    | `() => Promise<void>`           | 取消队列中的所有条目 |

每个 [SubmissionQueueEntry](https://reference.langchain.com/javascript/langchain-react/SubmissionQueueEntry) 对象包含:

| 字段 | 类型 | 说明 |
| ----------- | -------- | --------------------------------------------------------- |
| `id`        | `string` | 该队列条目的唯一标识符 |
| `values`    | `object` | 提交的输入值(包括消息) |
| `options`   | `object` | 随提交一并传入的任意附加选项 |
| `createdAt` | `string` | 条目创建时间的 ISO 时间戳 |

## 配置 `useStream`

将 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 连接到你的 Agent,再搭配与你框架对应的提交队列辅助函数。在一次运行进行期间,调用 `stream.submit()` 发送消息;对应当排在活动请求之后的提交传入 `multitaskStrategy: "enqueue"`。读取 `queue.entries` 和 `queue.size` 来渲染待处理的工作,并使用 `queue.cancel()` 或 `queue.clear()` 在条目开始处理前将其移除。

> **说明**
>
> 代码示例使用 `useStream<typeof myAgent>` 以获得类型安全的流式状态。类型推断(Type inference)参见 [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) 或 [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference) 后端的相关章节。

**React:**

```tsx
import { useStream, useSubmissionQueue } from "@langchain/react";

function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "simple_agent",
  });
  const queue = useSubmissionQueue(stream);

  const handleSubmit = (text: string) => {
    stream.submit({
      messages: [{ type: "human", content: text }],
    });
  };

  const pendingCount = queue.size;
  const entries = queue.entries;

  return (
    <div>
      <MessageList messages={stream.messages} />
      {pendingCount > 0 && <QueueList entries={entries} queue={queue} />}
      <ChatInput onSubmit={handleSubmit} />
    </div>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
import { useStream, useSubmissionQueue } from "@langchain/vue";
import { computed } from "vue";

const stream = useStream<typeof myAgent>({
  apiUrl: "http://localhost:2024",
  assistantId: "simple_agent",
});
const queue = useSubmissionQueue(stream);

function handleSubmit(text: string) {
  stream.submit({
    messages: [{ type: "human", content: text }],
  });
}

const pendingCount = computed(() => queue.size.value);
const entries = computed(() => queue.entries.value);
</script>

<template>
  <div>
    <MessageList :messages="stream.messages" />
    <QueueList v-if="pendingCount > 0" :entries="entries" :queue="queue" />
    <ChatInput @submit="handleSubmit" />
  </div>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream, useSubmissionQueue } from "@langchain/svelte";

  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "simple_agent",
  });
  const queue = useSubmissionQueue(stream);

  function handleSubmit(text: string) {
    stream.submit({
      messages: [{ type: "human", content: text }],
    });
  }
</script>

<div>
  <MessageList messages={stream.messages} />
  {#if queue.size > 0}
    <QueueList entries={queue.entries} {queue} />
  {/if}
  <ChatInput on:submit={(e) => handleSubmit(e.detail)} />
</div>
```

**Angular:**

```ts
import { Component } from "@angular/core";
import { injectStream, injectSubmissionQueue } from "@langchain/angular";

@Component({
  selector: "app-chat",
  template: `
    <message-list [messages]="stream.messages()" />
    @if (queue.size() > 0) {
      <queue-list [entries]="queue.entries()" [queue]="queue" />
    }
    <chat-input (onSubmit)="handleSubmit($event)" />
  `,
})
export class ChatComponent {
  stream = injectStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "simple_agent",
  });
  queue = injectSubmissionQueue(this.stream);

  handleSubmit(text: string) {
    this.stream.submit({
      messages: [{ type: "human", content: text }],
    });
  }
}
```

## 展示队列

构建一个 `QueueList` 组件,展示每条待处理消息并附带取消按钮。这让用户能看到有哪些内容在等待,并能移除不再需要的条目。

```tsx
function QueueList({ entries, queue }) {
  return (
    <div className="queue-panel">
      <div className="queue-header">
        <span>Queued messages ({entries.length})</span>
        <button onClick={() => queue.clear()}>Clear all</button>
      </div>
      <ul className="queue-entries">
        {entries.map((entry) => {
          const text = entry.values?.messages?.at(-1)?.content ?? "Pending...";
          return (
            <li key={entry.id} className="queue-entry">
              <span className="queue-text">{text}</span>
              <span className="queue-time">
                {new Date(entry.createdAt).toLocaleTimeString()}
              </span>
              <button
                className="queue-cancel"
                onClick={() => queue.cancel(entry.id)}
              >
                Cancel
              </button>
            </li>
          );
        })}
      </ul>
    </div>
  );
}
```

> **提示**
>
> 把每条排队消息的前几个字符作为预览显示出来,这样用户无需阅读完整消息就能快速判断该取消哪些条目。

## 取消排队中的消息

你有两个层级的取消能力:

### 取消单个条目

按 ID 从队列中移除某条特定消息。Agent 会跳过它并处理下一个条目。

```ts
await queue.cancel(entryId);
```

### 清空整个队列

一次性移除所有待处理消息。在用户切换上下文或想重新开始时很有用。

```ts
await queue.clear();
```

> **注意**
>
> 取消队列条目只影响**尚未开始处理**的消息。如果 Agent 已经在处理某条消息,从队列中取消它不会产生任何效果。请使用 `stream.stop()` 来中断当前运行。

## 用 `onCreated` 链式提交后续消息

`onCreated` 回调在新运行被创建时触发,为你提供一个以编程方式提交后续消息的钩子。这对于构建多步骤工作流很有用——在这些工作流中,下一个问题依赖于上一个提交已被接受。

```ts
stream.submit(
  { messages: [{ type: "human", content: "What is quantum computing?" }] },
  {
    onCreated(run) {
      console.log("Run created:", run.runId);
      // 链式提交一条后续消息
      stream.submit({
        messages: [{ type: "human", content: "Give me a simple analogy." }],
      });
    },
  }
);
```

该模式会自然地填充队列:第一条消息立即开始处理,后续消息则排在它后面。

## 开启新线程

当用户想开始一段全新的对话时,更新你传入流的响应式 `threadId`。传入 `null` 会清除当前的线程绑定;下一次提交将创建一个新线程。

**React:**

```tsx
function NewThreadButton() {
  const [threadId, setThreadId] = useState<string | null>(null);
  const stream = useStream<typeof myAgent>({ threadId, onThreadId: setThreadId });

  return (
    <button onClick={() => setThreadId(null)}>
      New conversation
    </button>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
const threadId = ref<string | null>(null);
const stream = useStream<typeof myAgent>({
  threadId,
  onThreadId: (id) => (threadId.value = id),
});
</script>

<template>
  <button @click="threadId = null">New conversation</button>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  let threadId = $state<string | null>(null);
  const stream = useStream<typeof myAgent>({
    threadId: () => threadId,
    onThreadId: (id) => (threadId = id),
  });
</script>

<button onclick={() => (threadId = null)}>New conversation</button>
```

**Angular:**

```ts
threadId = signal<string | null>(null);
stream = injectStream<typeof myAgent>({
  threadId: this.threadId,
  onThreadId: (id) => this.threadId.set(id),
});

// 在模板中:
// <button (click)="threadId.set(null)">New conversation</button>
```

## 最佳实践

* **限制队列大小**:虽然客户端对队列大小没有硬性限制,但要注意过大的队列会降低用户体验。当队列超过合理阈值(如 10 条)时,考虑显示警告。
* **显示队列位置**:为每个排队条目编号,让用户知道处理顺序。
* **保持输入焦点**:提交后保持输入框聚焦,让用户可以立即输入下一条消息。
* **动画过渡**:条目开始处理时,平滑地把它们从队列面板移入消息列表。
* **优雅处理错误**:如果某条排队消息失败,呈现错误而不阻塞队列中的后续条目。
* **对快速提交做防抖**:对于自动化或程序化的提交,在消息之间加入小的延迟,避免压垮服务器。
