# 推理 Token(Reasoning tokens)

> 将模型的推理过程与最终答案分开渲染

> 原文:[Reasoning tokens](https://docs.langchain.com/oss/python/langchain/frontend/reasoning-tokens)

推理 token(Reasoning tokens)暴露了高级模型(如 OpenAI 的 GPT-5、具备扩展思考(extended thinking)能力的 Anthropic Claude)的内部思考过程。这些模型产生结构化的内容块(content blocks),将推理过程与最终答案区分开,让你可以构建展示模型是*如何*得出其响应的 UI。

(页面内嵌的交互式示例 demo 此处省略)

## 什么是推理 token?

当具备推理能力的模型处理提示时,它们会生成两种截然不同的内容:

1. **推理块(reasoning blocks):** 模型内部的思维链(chain-of-thought)、问题拆解与逐步分析
2. **文本块(text blocks):** 呈现给用户的最终、精炼的回答

它们作为 `AIMessage` 内的**带类型内容块**交付,可以通过 `contentBlocks` 属性访问:

```ts
// 推理块
{ type: "reasoning", reasoning: "Let me think about this step by step..." }

// 文本块
{ type: "text", text: "The answer is 42." }
```

> **注意**
>
> 并非所有模型都会产生推理 token。此模式专门适用于支持扩展思考或思维链输出的模型。标准聊天模型只返回文本块。

## 使用场景

* **透明度:** 向用户展示模型的推理过程,建立对其答案的信任
* **调试:** 检查模型的思考过程,定位出错的位置
* **教育工具:** 通过揭示 AI 如何处理问题,教学生如何解决问题
* **决策支持:** 让领域专家验证推荐背后的推理依据
* **质量保证:** 在受监管行业中审计推理链以确保合规

## 提取推理块与文本块

`AIMessage` 上的 `contentBlocks` 数组按生成顺序包含所有块。按 `type` 过滤即可把推理与文本分开:

```ts
import { AIMessage } from "langchain";

function extractBlocks(msg: AIMessage) {
  const reasoningBlocks = msg.contentBlocks
    .filter((b) => b.type === "reasoning")
    .map((b) => b.reasoning);

  const textBlocks = msg.contentBlocks
    .filter((b) => b.type === "text")
    .map((b) => b.text);

  return {
    reasoning: reasoningBlocks.join(""),
    text: textBlocks.join(""),
  };
}
```

单条消息可能包含多个推理块(例如模型暂停推理、生成部分文本、然后继续推理)。把它们拼接起来,就能得到完整的思考过程。

## 从 `useStream` 访问消息

将 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 连接到你的具备推理能力的 Agent,并在聊天 UI 中遍历 `stream.messages`。根据 `HumanMessage.isInstance` 和 `AIMessage.isInstance` 分支处理,然后把每条 assistant 消息传给一个读取 `contentBlocks` 并把推理与文本分开的组件。当 `stream.isLoading` 为 true 时,为最后一条消息设置 `isStreaming`,使思考块随 token 到达而更新。

> **说明**
>
> 代码示例使用 `useStream<typeof myAgent>` 以获得类型安全的流式状态。类型推断(Type inference)参见 [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) 或 [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference) 后端的相关章节。

**React:**

```tsx
import { useStream } from "@langchain/react";
import { AIMessage, HumanMessage } from "langchain";

function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "reasoning",
  });

  return (
    <div className="messages">
      {stream.messages.map((msg, i) => {
        if (HumanMessage.isInstance(msg)) {
          return <HumanBubble key={i} text={msg.text} />;
        }
        if (AIMessage.isInstance(msg)) {
          return (
            <AIResponse
              key={i}
              message={msg}
              isStreaming={stream.isLoading && i === stream.messages.length - 1}
            />
          );
        }
        return null;
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

const stream = useStream<typeof myAgent>({
  apiUrl: "http://localhost:2024",
  assistantId: "reasoning",
});
</script>

<template>
  <div class="messages">
    <template v-for="(msg, i) in stream.messages.value" :key="i">
      <HumanBubble v-if="HumanMessage.isInstance(msg)" :text="msg.text" />
      <AIResponse
        v-else-if="AIMessage.isInstance(msg)"
        :message="msg"
        :isStreaming="stream.isLoading.value && i === stream.messages.value.length - 1"
      />
    </template>
  </div>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream } from "@langchain/svelte";
  import { AIMessage, HumanMessage } from "langchain";

  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "reasoning",
  });
</script>

<div class="messages">
  {#each stream.messages as msg, i}
    {#if HumanMessage.isInstance(msg)}
      <HumanBubble text={msg.text} />
    {:else if AIMessage.isInstance(msg)}
      <AIResponse
        message={msg}
        isStreaming={stream.isLoading && i === stream.messages.length - 1}
      />
    {/if}
  {/each}
</div>
```

**Angular:**

```ts
import { Component } from "@angular/core";
import { injectStream } from "@langchain/angular";
import { AIMessage, HumanMessage } from "langchain";

@Component({
  selector: "app-chat",
  template: `
    <div class="messages">
      @for (msg of stream.messages(); track $index) {
        @if (isHuman(msg)) {
          <human-bubble [text]="msg.text" />
        } @else if (isAI(msg)) {
          <ai-response
            [message]="msg"
            [isStreaming]="stream.isLoading() && $index === stream.messages().length - 1"
          />
        }
      }
    </div>
  `,
})
export class ChatComponent {
  stream = injectStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "reasoning",
  });

  isHuman = HumanMessage.isInstance;
  isAI = AIMessage.isInstance;
}
```

## 构建 ThinkingBubble 组件

`ThinkingBubble` 把推理 token 呈现在一个视觉上有所区分、可折叠的容器中。用户可以展开它查看完整的思考过程,也可以折叠它以专注于最终答案。

```tsx
import { useState } from "react";

function ThinkingBubble({
  reasoning,
  isStreaming,
}: {
  reasoning: string;
  isStreaming: boolean;
}) {
  const [isExpanded, setIsExpanded] = useState(false);

  const charCount = reasoning.length;
  const previewLength = 120;
  const preview =
    reasoning.length > previewLength
      ? reasoning.slice(0, previewLength) + "..."
      : reasoning;

  return (
    <div className="thinking-bubble">
      <button
        className="thinking-header"
        onClick={() => setIsExpanded(!isExpanded)}
      >
        <span className="thinking-icon">
          {isStreaming ? (
            <span className="thinking-spinner" />
          ) : (
            "💭"
          )}
        </span>
        <span className="thinking-label">
          {isStreaming ? "Thinking..." : `Thought process (${charCount} chars)`}
        </span>
        <span className={`chevron ${isExpanded ? "expanded" : ""}`}>▶</span>
      </button>

      {isExpanded && (
        <div className="thinking-content">
          <pre>{reasoning}</pre>
        </div>
      )}

      {!isExpanded && !isStreaming && (
        <div className="thinking-preview">{preview}</div>
      )}
    </div>
  );
}
```

## 渲染完整的 AI 响应

把 `ThinkingBubble` 和标准文本气泡组合成一个 `AIResponse` 组件:

```tsx
function AIResponse({
  message,
  isStreaming,
}: {
  message: AIMessage;
  isStreaming: boolean;
}) {
  const reasoningBlocks = message.contentBlocks
    .filter((b) => b.type === "reasoning")
    .map((b) => b.reasoning)
    .join("");

  const textBlocks = message.contentBlocks
    .filter((b) => b.type === "text")
    .map((b) => b.text)
    .join("");

  const hasReasoning = reasoningBlocks.length > 0;
  const hasText = textBlocks.length > 0;

  const isReasoningPhase = isStreaming && !hasText;
  const isTextPhase = isStreaming && hasText;

  return (
    <div className="ai-response">
      {hasReasoning && (
        <ThinkingBubble
          reasoning={reasoningBlocks}
          isStreaming={isReasoningPhase}
        />
      )}
      {hasText && (
        <div className="ai-text-bubble">
          <p>{textBlocks}</p>
          {isTextPhase && <span className="cursor-blink">▊</span>}
        </div>
      )}
    </div>
  );
}
```

## 处理边界情况

### 不含推理的消息

并非每条 AI 消息都会包含推理块。当 `contentBlocks` 只有文本块时,渲染一个不带 ThinkingBubble 的标准消息气泡即可。

### 空的推理块

有些模型会产生空的推理块作为占位符。将它们过滤掉:

```ts
const meaningfulReasoning = message.contentBlocks
  .filter((b) => b.type === "reasoning" && b.reasoning.trim().length > 0);
```

### 多轮推理—文本循环

单条消息可以在推理块与文本块之间交替。如果你需要保留这种交错结构,请按顺序遍历 `contentBlocks`,而不是按类型分组:

```ts
message.contentBlocks.forEach((block) => {
  if (block.type === "reasoning") {
    // 渲染 ThinkingBubble
  } else if (block.type === "text") {
    // 渲染文本段落
  }
});
```

## 最佳实践

* **默认折叠**:按需展示推理,而非默认展开
* **显示字符数**:让用户快速了解这个回答经过了多少思考
* **视觉上区分**:使用不同的颜色、边框或背景,使推理永远不会与真实答案混淆
* **动画过渡**:平滑的展开/折叠动画能提升感知质量
* **考虑无障碍性**:在切换按钮上使用恰当的 ARIA 属性(`aria-expanded`、`aria-controls`)
* **预览时截断**:折叠时显示推理的简短预览,让用户决定是否展开
