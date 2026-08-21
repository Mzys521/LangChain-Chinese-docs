# 时间旅行(Time travel)

> 检查历史检查点,并从任意时间点恢复执行以探索替代路径

> 原文:[Time travel](https://docs.langchain.com/oss/python/langchain/frontend/time-travel)

LangGraph Agent 中的每一次状态变化都会创建一个**检查点(checkpoint)**——该时刻 Agent 状态的完整快照。时间旅行(Time travel)让你可以检查任何检查点、查看 Agent 当时持有的确切状态,并**从该点恢复执行**以探索替代路径。它集调试器、撤销按钮与审计日志于一体。

(页面内嵌的交互式示例 demo 此处省略)

> **注意**
>
> 此功能需要 [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server)。请使用 `langgraph dev` 在本地运行你的 Agent,或[部署到 LangSmith](https://docs.langchain.com/langsmith/deployment) 以使用此模式。

## 检查点的工作原理

LangGraph 在每次节点执行后都会持久化 Agent 状态。每个持久化的状态都是一个 [ThreadState](https://reference.langchain.com/javascript/langchain-langgraph-sdk/index/ThreadState) 对象,它捕获:

* **checkpoint**:标识这一特定快照的元数据(ID、时间戳)
* **values**:该时刻的完整 Agent 状态(消息、自定义键)
* **tasks**:接下来被调度运行的图节点
* **next**:执行计划中即将到来的节点名称

这就构成了一条线性时间线,记录了 Agent 作出的每个决定、调用的每个工具,以及产生的每个响应。你的 UI 可以渲染这条时间线,并让用户跳转到任意时间点。

## 配置 `useStream`

为你的 Agent 创建流,然后通过 LangGraph 客户端为当前线程显式拉取检查点历史。从检查点恢复使用 `forkFrom: { checkpointId }`。

> **说明**
>
> 代码示例使用 `useStream<typeof myAgent>` 以获得类型安全的流式状态。类型推断(Type inference)参见 [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) 或 [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference) 后端的相关章节。

**React:**

```tsx
import { useStream } from "@langchain/react";
import { useEffect, useState } from "react";

const AGENT_URL = "http://localhost:2024";

export function TimeTravelChat() {
  const [threadId, setThreadId] = useState<string | null>(null);
  const [history, setHistory] = useState<ThreadState[]>([]);
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "time_travel",
    threadId,
    onThreadId: setThreadId,
  });

  useEffect(() => {
    if (!threadId || stream.isLoading) return;
    stream.client.threads.getHistory(threadId).then(setHistory);
  }, [stream.client, threadId, stream.isLoading]);

  function resumeFrom(cp: ThreadState) {
    stream.submit({}, {
      forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
    });
  }

  return (
    <div className="flex h-screen">
      <ChatPanel messages={stream.messages} />
      <TimelineSidebar history={history} onSelect={resumeFrom} />
    </div>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
import { useStream } from "@langchain/vue";
import { ref, watch } from "vue";

const AGENT_URL = "http://localhost:2024";
const threadId = ref<string | null>(null);
const history = ref<ThreadState[]>([]);

const stream = useStream<typeof myAgent>({
  apiUrl: AGENT_URL,
  assistantId: "time_travel",
  threadId,
  onThreadId: (id) => (threadId.value = id),
});

watch(
  [threadId, stream.isLoading],
  async ([id, isLoading]) => {
    if (isLoading) return;
    history.value = id
      ? ((await stream.client.threads.getHistory(id)) as ThreadState[])
      : [];
  },
  { immediate: true },
);

function resumeFrom(cp: ThreadState) {
  stream.submit({}, {
    forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
  });
}
</script>

<template>
  <div class="flex h-screen">
    <ChatPanel :messages="stream.messages.value" />
    <TimelineSidebar :history="history" @select="resumeFrom" />
  </div>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream } from "@langchain/svelte";

  const AGENT_URL = "http://localhost:2024";
  let threadId = $state<string | null>(null);
  let history = $state<ThreadState[]>([]);

  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "time_travel",
    threadId: () => threadId,
    onThreadId: (id) => (threadId = id),
  });

  $effect(() => {
    if (!threadId) {
      history = [];
      return;
    }
    if (stream.isLoading) return;
    stream.client.threads.getHistory(threadId).then((states) => {
      history = states as ThreadState[];
    });
  });

  function resumeFrom(cp: ThreadState) {
    stream.submit({}, {
      forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
    });
  }
</script>

<div class="flex h-screen">
  <ChatPanel messages={stream.messages} />
  <TimelineSidebar {history} onSelect={resumeFrom} />
</div>
```

**Angular:**

```ts
import { Component, effect, signal } from "@angular/core";
import { injectStream } from "@langchain/angular";

const AGENT_URL = "http://localhost:2024";

@Component({
  selector: "app-time-travel-chat",
  template: `
    <div class="flex h-screen">
      <app-chat-panel [messages]="stream.messages()" />
      <app-timeline-sidebar
        [history]="history()"
        (select)="resumeFrom($event)"
      />
    </div>
  `,
})
export class TimeTravelChatComponent {
  threadId = signal<string | null>(null);
  history = signal<ThreadState[]>([]);

  stream = injectStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "time_travel",
    threadId: this.threadId,
    onThreadId: (id) => this.threadId.set(id),
  });

  constructor() {
    effect(() => {
      if (this.stream.isLoading()) return;
      void this.refreshHistory(this.threadId());
    });
  }

  async refreshHistory(id: string | null) {
    this.history.set(id
      ? ((await this.stream.client.threads.getHistory(id)) as ThreadState[])
      : []);
  }

  resumeFrom(cp: ThreadState) {
    this.stream.submit({}, {
      forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
    });
  }
}
```

## 构建检查点时间线

时间线侧边栏把每个检查点显示为一个可点击的条目。每个条目展示当时运行的节点以及该时刻的消息数量:

```tsx
function TimelineSidebar({
  history,
  onSelect,
}: {
  history: ThreadState[];
  onSelect: (cp: ThreadState) => void;
}) {
  return (
    <aside className="w-80 overflow-y-auto border-l bg-gray-50 p-4">
      <h2 className="mb-4 text-sm font-semibold uppercase text-gray-500">
        Checkpoint Timeline
      </h2>
      <div className="space-y-2">
        {history.map((cp, i) => {
          const taskName = cp.tasks?.[0]?.name ?? "unknown";
          const msgCount = (cp.values?.messages as unknown[])?.length ?? 0;

          return (
            <button
              key={cp.checkpoint.checkpoint_id}
              onClick={() => onSelect(cp)}
              className="w-full rounded-lg border bg-white p-3 text-left
                         hover:border-blue-400 hover:shadow-sm transition-all"
            >
              <div className="flex items-center justify-between">
                <span className="text-xs text-gray-400">#{i + 1}</span>
                <NodeBadge name={taskName} />
              </div>
              <p className="mt-1 text-sm font-medium">{taskName}</p>
              <p className="text-xs text-gray-500">
                {msgCount} message{msgCount !== 1 ? "s" : ""}
              </p>
            </button>
          );
        })}
      </div>
    </aside>
  );
}
```

## 检查检查点状态

点击一个检查点应显示该时刻的完整状态。JSON 查看器让开发者对 Agent 当时所知道的和所决定的一切拥有完全的可见性:

```tsx
function CheckpointInspector({ checkpoint }: { checkpoint: ThreadState }) {
  const [expanded, setExpanded] = useState(false);

  return (
    <div className="rounded-lg border bg-white p-4">
      <div className="flex items-center justify-between">
        <h3 className="font-semibold">
          Checkpoint {checkpoint.checkpoint.checkpoint_id.slice(0, 8)}...
        </h3>
        <button
          onClick={() => setExpanded(!expanded)}
          className="text-sm text-blue-600 hover:underline"
        >
          {expanded ? "Collapse" : "Expand"} state
        </button>
      </div>

      <div className="mt-2 space-y-1 text-sm">
        <p>
          <strong>Node:</strong>{" "}
          {checkpoint.tasks?.[0]?.name ?? "—"}
        </p>
        <p>
          <strong>Next:</strong>{" "}
          {checkpoint.next?.join(", ") || "—"}
        </p>
        <p>
          <strong>Messages:</strong>{" "}
          {(checkpoint.values?.messages as unknown[])?.length ?? 0}
        </p>
      </div>

      {expanded && (
        <div className="mt-3 max-h-96 overflow-auto rounded bg-gray-900 p-3">
          <pre className="text-xs text-gray-200">
            {JSON.stringify(checkpoint.values, null, 2)}
          </pre>
        </div>
      )}
    </div>
  );
}
```

> **提示**
>
> 对于生产环境 UI,考虑使用带可折叠节点的正规 JSON 查看器组件,而不是原始的 `JSON.stringify`。像 `react-json-view` 或 `react-json-tree` 这样的库能给用户带来好得多的探索体验。

## 从检查点恢复

时间旅行的核心能力是**从任何先前的检查点恢复执行**。当用户选中一个检查点时,用 `null` 输入调用 `submit` 并传入检查点 ID:

```ts
stream.submit({}, {
  forkFrom: { checkpointId: selectedCheckpoint.checkpoint.checkpoint_id },
});
```

这会告诉 LangGraph:

1. 回滚到所选检查点的状态
2. 从该点起重新执行图
3. 把新结果流式发送给客户端

所选检查点之后的现有消息会被新的执行路径替换。这实际上在对话时间线中创建了一个**分支**。

> **注意**
>
> 从检查点恢复并不会删除原始时间线。先前的检查点仍然保留在历史中。这意味着用户始终可以返回去尝试不同的路径,而不会丢失任何先前的工作。

## SplitView 布局

时间旅行最适合使用分栏布局,左侧为主聊天区,右侧为时间线:

```tsx
function TimeTravelLayout() {
  const [threadId, setThreadId] = useState<string | null>(null);
  const [history, setHistory] = useState<ThreadState[]>([]);
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "time_travel",
    threadId,
    onThreadId: setThreadId,
  });

  const [selectedCheckpoint, setSelectedCheckpoint] =
    useState<ThreadState | null>(null);

  useEffect(() => {
    if (!threadId || stream.isLoading) return;
    stream.client.threads.getHistory(threadId).then(setHistory);
  }, [stream.client, threadId, stream.isLoading]);

  return (
    <div className="flex h-screen">
      {/* 主聊天区 */}
      <main className="flex-1 overflow-y-auto p-6">
        <div className="mx-auto max-w-2xl space-y-4">
          {stream.messages.map((msg) => (
            <Message key={msg.id} message={msg} />
          ))}
        </div>
        <ChatInput
          onSubmit={(text) =>
            stream.submit({ messages: [{ type: "human", content: text }] })
          }
          isLoading={stream.isLoading}
        />
      </main>

      {/* 时间线侧边栏 */}
      <aside className="w-96 overflow-y-auto border-l bg-gray-50">
        <TimelineSidebar
          history={history}
          selected={selectedCheckpoint}
          onSelect={setSelectedCheckpoint}
          onResume={(cp) =>
            stream.submit({}, {
              forkFrom: { checkpointId: cp.checkpoint.checkpoint_id },
            })
          }
        />
        {selectedCheckpoint && (
          <CheckpointInspector checkpoint={selectedCheckpoint} />
        )}
      </aside>
    </div>
  );
}
```

## 提取检查点元数据

把原始检查点数据转换为时间线上便于展示的条目:

```ts
function formatCheckpoints(history: ThreadState[]) {
  return history.map((cp, index) => ({
    index,
    id: cp.checkpoint?.checkpoint_id,
    taskName: cp.tasks?.[0]?.name ?? "unknown",
    messageCount: (cp.values?.messages as unknown[])?.length ?? 0,
    hasInterrupts: cp.tasks?.some((t) => t.interrupts?.length) ?? false,
    nextNodes: cp.next ?? [],
  }));
}
```

这样就能用有意义的标签来渲染时间线条目,而不是原始的 ID。

## 使用场景

时间旅行在众多场景中都非常有价值:

* **调试 Agent 行为**:逐步走过 Agent 的决定,理解它为什么选择某条特定路径
* **撤销动作**:如果 Agent 走错了方向,从更早的检查点恢复并重新尝试
* **探索替代方案**:从对话中途的检查点分叉,观察不同的输入如何改变结果
* **审计**:审查 Agent 动作的完整历史,用于合规、质量保证或事后分析
* **教学**:逐步走过 Agent 的执行过程,解释多步推理是如何运作的

> **说明**
>
> 时间旅行与[人机协同(Human-in-the-loop)](https://docs.langchain.com/oss/python/langchain/frontend/human-in-the-loop)模式结合时尤其强大。如果人类审查者在一个中断处拒绝了 Agent 的动作,他们可以从该动作发生之前的检查点恢复,并提供纠正性的输入。

## 在时间线中处理中断

包含中断(人机协同暂停)的检查点应当获得特殊的视觉处理。它们代表了 Agent 停下并等待人类输入的时刻:

```tsx
function TimelineEntry({
  checkpoint,
  index,
}: {
  checkpoint: ThreadState;
  index: number;
}) {
  const hasInterrupt = checkpoint.tasks?.some(
    (t) => t.interrupts && t.interrupts.length > 0
  );

  return (
    <div
      className={`rounded-lg border p-3 ${
        hasInterrupt
          ? "border-amber-300 bg-amber-50"
          : "border-gray-200 bg-white"
      }`}
    >
      <div className="flex items-center gap-2">
        <span className="text-xs text-gray-400">#{index + 1}</span>
        {hasInterrupt && (
          <span className="rounded bg-amber-200 px-1.5 py-0.5 text-xs font-medium text-amber-800">
            Interrupt
          </span>
        )}
      </div>
      <p className="mt-1 text-sm font-medium">
        {checkpoint.tasks?.[0]?.name ?? "—"}
      </p>
    </div>
  );
}
```

## 最佳实践

* **懒加载历史**:对于有数百个检查点的线程,应分页或只加载最近 N 个条目,以保持 UI 响应迅速。
* **展示有意义的标签**:显示节点名称和消息数量,而不是原始检查点 ID。用户需要的是上下文,而不是 UUID。
* **恢复前先确认**:从旧检查点恢复会替换当前的执行路径。显示一个确认对话框,避免用户意外丢失当前的对话状态。
* **高亮当前检查点**:让当前对话状态对应的检查点在视觉上显而易见。
* **支持键盘导航**:高级用户会希望用方向键在检查点间逐步切换。给时间线添加键盘处理,以获得流畅的调试体验。
* **对比检查点间的状态差异**:对高级用户而言,展示两个连续检查点之间的变化,可以精确揭示 Agent 的状态在每一步是如何演进的。
