# 断线重连(Join & rejoin)

> 在不停止 Agent 的前提下断开流连接,稍后再重新接入

> 原文:[Join & rejoin](https://docs.langchain.com/oss/python/langchain/frontend/join-rejoin)

加入与重连(Join and rejoin)让你可以**在不停止 Agent 的情况下**断开与正在运行的 Agent 流的连接,然后在稍后重新连上它。当客户端离开时,Agent 继续在服务器端执行,而你可以从离开时的确切位置继续接收流。

(页面内嵌的交互式示例 demo 此处省略)

> **注意**
>
> 此功能需要 [LangGraph Agent Server](https://docs.langchain.com/oss/python/langgraph/local-server)。请使用 `langgraph dev` 在本地运行你的 Agent,或[部署到 LangSmith](https://docs.langchain.com/langsmith/deployment) 以使用此模式。

## 为什么需要断线重连?

传统的流式 API 把客户端与服务器紧紧耦合在一起:一旦客户端断开,流就丢失了。断线重连打破这种耦合,支撑了几种重要的模式:

* **网络中断**:在基站或 Wi-Fi 网络间切换的移动用户可以无缝恢复
* **页面导航**:用户离开聊天页面后稍后返回,进度不丢失
* **移动端后台化**:被操作系统挂起的应用在前台恢复时可以重新加入流
* **长时间运行的任务**:执行数分钟级操作(研究、代码生成、数据分析)的 Agent,用户无需保持页面打开
* **多设备接力**:在手机上开始一段对话,在桌面上继续

## 核心概念

加入/重连模式涉及三个关键机制:

| 方法 / 选项 | 用途 |
| -------------------------------- | ---------------------------------------------------------------------- |
| `threadId`                       | 把流绑定到你想观察的 LangGraph 线程 |
| `onThreadId`                     | 持久化新创建的线程 ID,使重新挂载时可以重连 |
| `stream.disconnect()`            | 在客户端侧离开流,而 Agent 继续在服务器端运行 |
| 以相同的 `threadId` 重新挂载 | 重新接入该线程进行中的工作 |

> **注意**
>
> **加入/重连使用 `stream.disconnect()`,而不是 `stream.stop()`。** 默认情况下,`stream.stop()` 会**取消活动中的运行**:它既断开客户端,*又*取消服务器上的运行。对于加入/重连,请调用 `stream.disconnect()`(`stop({ cancel: false })` 的别名),这样在你离开期间 Agent 会继续处理。
>
> 要从应用代码中显式取消执行,请使用 `stream.stop()` 或 [`client.runs.cancel`](https://reference.langchain.com/javascript/langchain-langgraph-sdk/client/RunsClient/cancel)。

## 配置 `useStream`

关键的配置步骤是**持久化 `threadId`**。当组件以相同的线程 ID 重新挂载时,流会附着到该线程的当前状态以及任何进行中的运行。

> **说明**
>
> 代码示例使用 `useStream<typeof myAgent>` 以获得类型安全的流式状态。类型推断(Type inference)参见 [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) 或 [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference) 后端的相关章节。

**React:**

```tsx
import { useStream } from "@langchain/react";
import { useCallback, useState } from "react";

function Chat() {
  const [connected, setConnected] = useState(true);
  const [mountKey, setMountKey] = useState(0);
  const [threadId, setThreadId] = useState<string | null>(
    () => sessionStorage.getItem("activeThreadId"),
  );

  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "join_rejoin",
    threadId,
    onThreadId(id) {
      setThreadId(id);
      if (id) sessionStorage.setItem("activeThreadId", id);
    },
  });

  const disconnect = useCallback(() => {
    void stream.disconnect();
    setConnected(false);
  }, [stream]);

  const rejoin = useCallback(() => {
    setMountKey((key) => key + 1);
    setConnected(true);
  }, []);

  return (
    <div key={mountKey}>
      <ConnectionStatus connected={connected} />
      <MessageList messages={stream.messages} />
      <ChatControls
        stream={stream}
        threadId={threadId}
        connected={connected}
        onDisconnect={disconnect}
        onRejoin={rejoin}
      />
    </div>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
import { useStream } from "@langchain/vue";
import { ref } from "vue";

const connected = ref(true);
const mountKey = ref(0);
const threadId = ref<string | null>(sessionStorage.getItem("activeThreadId"));

const stream = useStream<typeof myAgent>({
  apiUrl: "http://localhost:2024",
  assistantId: "join_rejoin",
  threadId,
  onThreadId(id) {
    threadId.value = id;
    if (id) sessionStorage.setItem("activeThreadId", id);
  },
});

function disconnect() {
  void stream.disconnect();
  connected.value = false;
}

function rejoin() {
  mountKey.value += 1;
  connected.value = true;
}
</script>

<template>
  <div :key="mountKey">
    <ConnectionStatus :connected="connected" />
    <MessageList :messages="stream.messages" />
    <ChatControls
      :stream="stream"
      :threadId="threadId"
      :connected="connected"
      @disconnect="disconnect"
      @rejoin="rejoin"
    />
  </div>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream } from "@langchain/svelte";

  let connected = $state(true);
  let mountKey = $state(0);
  let threadId = $state<string | null>(sessionStorage.getItem("activeThreadId"));

  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "join_rejoin",
    threadId: () => threadId,
    onThreadId(id) {
      threadId = id;
      if (id) sessionStorage.setItem("activeThreadId", id);
    },
  });

  function disconnect() {
    void stream.disconnect();
    connected = false;
  }

  function rejoin() {
    mountKey += 1;
    connected = true;
  }
</script>

<div key={mountKey}>
  <ConnectionStatus {connected} />
  <MessageList messages={stream.messages} />
  <ChatControls
    {threadId}
    {connected}
    onDisconnect={disconnect}
    onRejoin={rejoin}
  />
</div>
```

**Angular:**

```ts
import { Component, signal } from "@angular/core";
import { injectStream } from "@langchain/angular";

@Component({
  selector: "app-chat",
  template: `
    <connection-status [connected]="connected()" />
    <message-list [messages]="stream.messages()" />
    <chat-controls
      [stream]="stream"
      [threadId]="threadId()"
      [connected]="connected()"
      (disconnect)="disconnect()"
      (rejoin)="rejoin()"
    />
  `,
})
export class ChatComponent {
  threadId = signal<string | null>(sessionStorage.getItem("activeThreadId"));
  connected = signal(true);
  mountKey = signal(0);

  stream = injectStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "join_rejoin",
    threadId: this.threadId,
    onThreadId: (id) => {
      this.threadId.set(id);
      if (id) sessionStorage.setItem("activeThreadId", id);
    },
  });

  disconnect() {
    void this.stream.disconnect();
    this.connected.set(false);
  }

  rejoin() {
    this.mountKey.update((key) => key + 1);
    this.connected.set(true);
  }
}
```

## 提交消息

像平常一样提交消息。正是线程 ID 的绑定,使得稍后的重新挂载能够重新连接到同一段对话:

```ts
stream.submit({ messages: [{ type: "human", content: text }] });
```

## 断开流连接

调用 `stream.disconnect()` 可以在不取消运行的情况下离开流。Agent 会继续在服务器端处理。

```ts
await stream.disconnect();
// 等价于:await stream.stop({ cancel: false })
```

这里**不要**使用 `stream.stop()`——默认情况下它会取消服务器上的运行。

调用 `disconnect()` 之后:

* `stream.isLoading` 变为 `false`
* 你自己的 `connected` 标志也应变为 `false`
* 消息列表保留到断开点为止收到的所有消息
* Agent 继续在服务器上运行
* 在你重新加入之前不会收到新消息

## 重新加入流

用保存的线程 ID 重新挂载流消费者以重新连接。在 React 中,demo 递增一个 `mountKey`;在其他框架中,使用等效的重新挂载或条件渲染模式:

```ts
setMountKey((key) => key + 1);
setConnected(true);
```

重新加入之后:

* `connected` 变为 `true`
* 断开期间生成的所有消息都会被交付
* 新的流式消息实时恢复
* 如果 Agent 仍在运行,`stream.isLoading` 变为 `true`;如果它已经完成,你会立即收到最终状态

## 最佳实践

* **加入/重连用 `disconnect()`,取消用 `stop()`**:导航离开或把应用切到后台时,应调用 `stream.disconnect()`。面向用户的"停止"或"取消"按钮应调用 `stream.stop()`(或 [`client.runs.cancel`](https://reference.langchain.com/javascript/langchain-langgraph-sdk/client/RunsClient/cancel))。
* **始终保存线程 ID**:没有它就无法重新加入。同时使用组件状态与持久化存储以增强韧性。
* **展示清晰的连接状态**:用户应始终知道自己是在接收实时更新,还是在查看快照。
* **可见性变化时自动重新加入**:使用 Page Visibility API,在用户返回标签页时自动重新加入。
* **设置合理的超时**:如果重新加入尝试耗时过长,改为回退到拉取线程历史。
* **清理过期的线程**:当用户重新开始、或后端报告线程不可用时,移除持久化的线程 ID。
