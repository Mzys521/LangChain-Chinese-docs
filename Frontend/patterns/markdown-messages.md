# Markdown 消息渲染(Markdown messages)

> 将 LLM 响应渲染为格式丰富的 Markdown,并提供完善的流式支持

> 原文:[Markdown messages](https://docs.langchain.com/oss/python/langchain/frontend/markdown-messages)

(页面内嵌的交互式示例 demo 此处省略)

## Markdown 渲染的工作原理

渲染管线(rendering pipeline)分为三步:

1. **接收(Receive):** [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 将流式文本累积到每条 AI 消息的 `msg.text` 上,并在新 token 到达时进行响应式更新。
2. **解析(Parse):** Markdown 解析器将原始文本转换为 HTML(或 React 元素树)。该步骤在每次更新时都会执行,但对于聊天长度的内容而言速度足够快(5 KB 消息 \< 5ms)。
3. **渲染(Render):** 将解析后的输出渲染到 DOM 中。React 使用虚拟 DOM diff;Vue 和 Svelte 使用 `v-html` / `{@html}` 配合经过净化(sanitized)的 HTML。

## 配置 `useStream`

Markdown 模式使用一个简单的聊天 Agent,无需特殊配置。将 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 与你的 Agent URL 和 assistant ID 连接起来即可。

> **说明**
>
> 代码示例使用 `useStream<typeof myAgent>` 以获得类型安全的流式状态。类型推断(Type inference)参见 [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) 或 [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference) 后端的相关章节。

**React:**

```tsx
import { useStream } from "@langchain/react";
import { AIMessage, HumanMessage } from "langchain";

const AGENT_URL = "http://localhost:2024";

export function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "simple_agent",
  });

  return (
    <div>
      {stream.messages.map((msg) => {
        if (AIMessage.isInstance(msg)) {
          return <Markdown key={msg.id}>{msg.text}</Markdown>;
        }
        if (HumanMessage.isInstance(msg)) {
          return <p key={msg.id}>{msg.text}</p>;
        }
      })}
    </div>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
import { useStream } from "@langchain/vue";
import { AIMessage, HumanMessage } from "langchain";

const AGENT_URL = "http://localhost:2024";

const stream = useStream<typeof myAgent>({
  apiUrl: AGENT_URL,
  assistantId: "simple_agent",
});
</script>

<template>
  <div>
    <template v-for="msg in stream.messages.value" :key="msg.id">
      <Markdown v-if="AIMessage.isInstance(msg)">{{ msg.text }}</Markdown>
      <p v-else-if="HumanMessage.isInstance(msg)">{{ msg.text }}</p>
    </template>
  </div>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream } from "@langchain/svelte";
  import { AIMessage, HumanMessage } from "langchain";

  const AGENT_URL = "http://localhost:2024";

  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "simple_agent",
  });
</script>

<div>
  {#each stream.messages as msg (msg.id)}
    {#if AIMessage.isInstance(msg)}
      <Markdown content={msg.text} />
    {:else if HumanMessage.isInstance(msg)}
      <p>{msg.text}</p>
    {/if}
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
      <app-markdown [content]="msg.text" />
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

## 选择 Markdown 库

每个框架都有其自然的 Markdown 渲染选择:

| 框架 | 库 | 输出 | 原因 |
| --------- | ------------------------------- | -------------------------------- | ------------------------------------------------------------------ |
| React     | `react-markdown` + `remark-gfm` | React 元素 | 基于组件,虚拟 DOM diff,无需 `dangerouslySetInnerHTML` |
| Vue       | `marked` + `dompurify`          | 通过 `v-html` 输出净化的 HTML | 轻量、快速,内置 GFM |
| Svelte    | `marked` + `dompurify`          | 通过 `{@html}` 输出净化的 HTML | 与 Vue 相同,API 一致 |
| Angular   | `marked` + `dompurify`          | 通过 `[innerHTML]` 输出净化的 HTML | 与 Vue/Svelte 相同 |

> **提示**
>
> React 的 `react-markdown` 将 Markdown 直接转换为 React 元素,因此不需要 HTML 净化,也不涉及 `dangerouslySetInnerHTML`。而对于 Vue、Svelte 和 Angular,在渲染前务必使用 `dompurify` 对解析出的 HTML 进行净化。

## 构建 Markdown 组件

**React:**

```tsx
import ReactMarkdown from "react-markdown";
import remarkGfm from "remark-gfm";

export function Markdown({ children }: { children: string }) {
  return (
    <div className="markdown-content">
      <ReactMarkdown remarkPlugins={[remarkGfm]}>
        {children}
      </ReactMarkdown>
    </div>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
import { computed, useSlots } from "vue";
import { marked } from "marked";
import DOMPurify from "dompurify";

marked.setOptions({ gfm: true, breaks: true });

const slots = useSlots();

const html = computed(() => {
  const slot = slots.default?.();
  const text = slot
    ?.map((vnode) =>
      typeof vnode.children === "string" ? vnode.children : ""
    )
    .join("") ?? "";
  if (!text) return "";
  return DOMPurify.sanitize(marked.parse(text) as string);
});
</script>

<template>
  <div class="markdown-content" v-html="html" />
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { marked } from "marked";
  import DOMPurify from "dompurify";

  let { content }: { content: string } = $props();

  marked.setOptions({ gfm: true, breaks: true });

  let html = $derived.by(() => {
    if (!content) return "";
    return DOMPurify.sanitize(marked.parse(content) as string);
  });
</script>

<div class="markdown-content">
  {@html html}
</div>
```

**Angular:**

```ts
import { Component, Input, computed, signal } from "@angular/core";
import { marked } from "marked";
import DOMPurify from "dompurify";

marked.setOptions({ gfm: true, breaks: true });

@Component({
  selector: "app-markdown",
  template: `<div class="markdown-content" [innerHTML]="html()"></div>`,
})
export class MarkdownComponent {
  @Input() set content(value: string) {
    this._content.set(value);
  }

  private _content = signal("");

  html = computed(() => {
    const text = this._content();
    if (!text) return "";
    return DOMPurify.sanitize(marked.parse(text) as string);
  });
}
```

## 净化 HTML 输出

当以原始 HTML 方式渲染解析后的 Markdown(`v-html`、`{@html}`、`[innerHTML]`)时,必须对输出进行净化以防止跨站脚本攻击(XSS)。LLM 响应可能包含任意文本,其中可能含有 Markdown 解析器会转换为可执行 HTML 的标记。

使用 `dompurify` 剥离危险元素:

```ts
import DOMPurify from "dompurify";

const safeHtml = DOMPurify.sanitize(rawHtml);
```

DOMPurify 会移除 `<script>` 标签、`onclick` 属性、`javascript:` URL 以及其他 XSS 攻击向量,同时保留安全的 Markdown 输出,如标题、列表、代码块、表格和链接。

> **注意**
>
> React 的 `react-markdown` 不需要 `dompurify`,因为它直接生成 React 元素,不涉及任何原始 HTML 注入。

## 流式相关注意事项

`useStream` 会在每个 token 到达时响应式地更新 `msg.text`,Markdown 组件会在每次更新时重新解析。对于典型的聊天消息而言,这是有性能保障的:

* `marked` 的解析速度约为 1 MB/s,5 KB 消息耗时 \< 5ms
* `react-markdown` + remark 管线对于聊天长度的内容同样快速
* 浏览器的布局引擎能够高效地处理 DOM 更新

对于非常长的响应(> 50 KB),可以考虑以下优化:

* **节流渲染:** 使用 `requestAnimationFrame` 以 60fps 批量更新,而不是每个 token 都重新渲染
* **增量解析:** 只解析新增内容并追加到已渲染的缓冲区(进阶技巧,聊天 UI 通常不需要)

> **说明**
>
> 对于大多数聊天应用,在每个 token 到达时重新解析完整消息的简单做法就足够了。只有当你观察到超长消息导致滚动卡顿或掉帧时,才需要进行优化。

## 最佳实践

* **始终净化:** 使用 `v-html`、`{@html}` 或 `[innerHTML]` 时,务必让解析输出经过 `dompurify`。永远不要信任由 LLM 输出喂给 Markdown 解析器所产生的原始 HTML。
* **启用 GFM:** GitHub Flavored Markdown 提供了表格、删除线、任务列表和自动链接等功能,LLM 经常会用到这些特性。
* **处理空内容:** 解析前检查空字符串,避免渲染空容器。
* **使用 `breaks: true`:** 启用换行转换,使 LLM 输出中的单个换行符渲染为 `<br>` 而不是被忽略。LLM 经常使用单个换行进行视觉分隔。
* **针对聊天场景设置样式:** 使用紧凑的边距和适合聊天气泡的尺寸,而不是全宽的文章版式。
* **用富内容做测试:** 验证标题、嵌套列表、带长行的代码块、宽表格和引用块的渲染效果,以发现溢出或布局问题。
