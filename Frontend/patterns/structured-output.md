# 结构化输出(Structured output)

> 渲染 Agent 返回的类型化、机器可读的结构化数据

> 原文:[Structured output](https://docs.langchain.com/oss/python/langchain/frontend/structured-output)

结构化输出(Structured output)允许 Agent 返回**类型化的、机器可读的数据**,而不是纯文本。你得到的不是一个单一字符串,而是一个结构化对象,可以映射到任何 UI:卡片、表格、图表、逐步拆解,或领域专用的渲染器。

(页面内嵌的交互式示例 demo 此处省略)

## 什么是结构化输出?

Agent 不再返回自由格式的文本响应,而是通过一次工具调用来返回一个符合预定义 schema 的结构化对象。这为你带来:

* **类型安全的数据**:将响应解析为已知的 TypeScript 类型
* **精确的渲染控制**:为每个字段赋予各自的 UI 处理方式
* **一致的格式**:无论底层模型是什么,每个响应都遵循相同的结构

Agent 通过调用一个"结构化输出"工具来实现这一点,该工具的参数中包含响应数据。该工具本身不执行任何逻辑,纯粹是返回类型化数据的载体。

## 使用场景

* **产品对比**:特性表格、优缺点列表、评分
* **数据分析**:带指标、拆解和亮点的摘要
* **分步指南**:带说明和代码片段的有序指令
* **食谱**:食材、步骤、时长与营养信息
* **数学与科学**:用 LaTeX 渲染的公式、逐步推导
* **旅行规划**:带日期、地点和费用估算的行程

## 定义 schema

为 Agent 返回的结构化数据定义一个 TypeScript 类型。该 schema 的形状决定了你如何渲染 UI。

以下是内嵌 demo 使用的数学解题 schema:

```ts
interface MathSolution {
  problem: string; // 原始数学问题
  steps: {
    explanation: string;
    latex: string; // 该步骤的可选显示数学式
  }[]; // 逐步推导
  finalAnswer: string; // 纯文本的最终答案
  finalAnswerLatex: string; // 最终答案的 LaTeX 表示
}
```

你的 schema 可以是任何东西。无论形状如何,该模式的工作方式都相同。

## 从消息中提取结构化输出

结构化输出位于最后一条 `AIMessage` 的 `tool_calls` 数组中。通过找到该 AI 消息并访问第一个工具调用的参数来提取它:

```ts
import { AIMessage } from "langchain";

function extractStructuredOutput<T>(messages: any[]): T | null {
  const aiMessage = messages.find(AIMessage.isInstance);
  const toolCall = aiMessage?.tool_calls?.[0];
  if (!toolCall) return null;

  return toolCall.args as T;
}
```

> **注意**
>
> 在 Agent 完成流式输出之前,结构化输出工具调用的 `args` 可能尚未被填充。在流式期间,`args` 可能是部分填充的或未定义。渲染前务必检查其完整性。

## 配置 `useStream`

将 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 连接到你的结构化输出 Agent,然后读取 `stream.messages`,从最新的 [`AIMessage`](https://reference.langchain.com/python/langchain-core/messages/ai/AIMessage) 工具调用中提取类型化载荷。当 `args` 完整后渲染你的自定义 UI;在 `stream.isLoading` 为 true 时显示加载状态(工具参数可能会逐步流入);并使用 `stream.submit()` 发送下一条提示。

> **说明**
>
> 代码示例使用 `useStream<typeof myAgent>` 以获得类型安全的流式状态。类型推断(Type inference)参见 [Python](https://docs.langchain.com/oss/python/langchain/frontend/overview#type-inference) 或 [JavaScript](https://docs.langchain.com/oss/javascript/langchain/frontend/overview#type-inference) 后端的相关章节。

**React:**

```tsx
import { useStream } from "@langchain/react";
import { AIMessage } from "langchain";

function MathSolutionChat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "structured_output_latex",
  });

  const solution = extractStructuredOutput<MathSolution>(stream.messages);

  return (
    <div>
      {!solution && !stream.isLoading && (
        <PromptInput onSubmit={(text) =>
          stream.submit({ messages: [{ type: "human", content: text }] })
        } />
      )}
      {stream.isLoading && <LoadingIndicator />}
      {solution && <SolutionCard solution={solution} />}
    </div>
  );
}
```

**Vue:**

```vue
<script setup lang="ts">
import { useStream } from "@langchain/vue";
import { AIMessage } from "langchain";
import { computed } from "vue";

const stream = useStream<typeof myAgent>({
  apiUrl: "http://localhost:2024",
  assistantId: "structured_output_latex",
});

const solution = computed(() =>
  extractStructuredOutput<MathSolution>(stream.messages.value)
);

function handleSubmit(text: string) {
  stream.submit({ messages: [{ type: "human", content: text }] });
}
</script>

<template>
  <div>
    <PromptInput v-if="!solution && !stream.isLoading" @submit="handleSubmit" />
    <LoadingIndicator v-if="stream.isLoading" />
    <SolutionCard v-if="solution" :solution="solution" />
  </div>
</template>
```

**Svelte:**

```svelte
<script lang="ts">
  import { useStream } from "@langchain/svelte";
  import { AIMessage } from "langchain";

  const stream = useStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "structured_output_latex",
  });

  const solution = $derived(extractStructuredOutput<MathSolution>(stream.messages));

  function handleSubmit(text: string) {
    stream.submit({ messages: [{ type: "human", content: text }] });
  }
</script>

<div>
  {#if !solution && !stream.isLoading}
    <PromptInput on:submit={(e) => handleSubmit(e.detail)} />
  {/if}
  {#if stream.isLoading}
    <LoadingIndicator />
  {/if}
  {#if solution}
    <SolutionCard {solution} />
  {/if}
</div>
```

**Angular:**

```ts
import { Component, computed } from "@angular/core";
import { injectStream } from "@langchain/angular";

@Component({
  selector: "app-math-solution-chat",
  template: `
    @if (!solution() && !stream.isLoading()) {
      <prompt-input (onSubmit)="handleSubmit($event)" />
    }
    @if (stream.isLoading()) {
      <loading-indicator />
    }
    @if (solution()) {
      <solution-card [solution]="solution()" />
    }
  `,
})
export class MathSolutionChatComponent {
  stream = injectStream<typeof myAgent>({
    apiUrl: "http://localhost:2024",
    assistantId: "structured_output_latex",
  });

  solution = computed(() =>
    extractStructuredOutput<MathSolution>(this.stream.messages())
  );

  handleSubmit(text: string) {
    this.stream.submit({
      messages: [{ type: "human", content: text }],
    });
  }
}
```

## 渲染结构化数据

一旦拿到类型化对象,就构建一个把每个字段映射到合适 UI 元素的组件。这是该模式的核心:把结构化数据变成目的明确的界面。

```tsx
function LatexBlock({ latex }: { latex: string }) {
  return <div className="latex-block">{latex}</div>; // 用 KaTeX 或 MathJax 渲染。
}

function SolutionCard({ solution }: { solution: MathSolution }) {
  return (
    <div className="solution-card">
      <h3>{solution.problem}</h3>
      <ol>
        {solution.steps.map((step, i) => (
          <li key={i}>
            <span>{step.explanation}</span>
            {step.latex && <LatexBlock latex={step.latex} />}
          </li>
        ))}
      </ol>
      <strong>{solution.finalAnswer}</strong>
      {solution.finalAnswerLatex && <LatexBlock latex={solution.finalAnswerLatex} />}
    </div>
  );
}
```

## 处理部分流式数据

在流式期间,工具调用参数可能是未完成的 JSON。在你的提取逻辑中对此做好防护:

```ts
function extractStructuredOutput<T>(
  messages: any[],
  requiredFields: string[] = [],
): T | null {
  const aiMessages = messages.filter(AIMessage.isInstance);
  if (aiMessages.length === 0) return null;

  const lastAI = aiMessages[aiMessages.length - 1];
  const toolCall = lastAI.tool_calls?.[0];
  if (!toolCall?.args) return null;

  const args = toolCall.args as Record<string, unknown>;
  const hasRequired = requiredFields.every(
    (field) => args[field] !== undefined
  );

  if (requiredFields.length > 0 && !hasRequired) return null;
  return args as T;
}
```

使用 `requiredFields` 参数来等待关键字段被填充后再渲染:

```ts
const solution = extractStructuredOutput<MathSolution>(stream.messages, [
  "problem",
  "steps",
  "finalAnswer",
]);
```

## 流式期间渐进渲染

与其等待完整的结构化输出,不如在字段到达时逐一渲染。这样用户在 Agent 仍在生成时就能获得即时反馈:

```tsx
function ProgressiveSolutionCard({ messages }: { messages: any[] }) {
  const partial = extractStructuredOutput<Partial<MathSolution>>(messages);
  if (!partial) return null;

  return (
    <div className="solution-card">
      {partial.problem && <h3>{partial.problem}</h3>}

      {partial.steps && partial.steps.length > 0 && (
        <div className="solution-steps">
          <h4>Steps</h4>
          {partial.steps.map((step, i) => (
            <div key={i} className="step">
              <div className="step-number">Step {i + 1}</div>
              <p>{step.explanation}</p>
              {step.latex && <LatexBlock latex={step.latex} />}
            </div>
          ))}
        </div>
      )}

      {partial.finalAnswer && <strong>{partial.finalAnswer}</strong>}
    </div>
  );
}
```

> **提示**
>
> 当 schema 具有自然的自上而下顺序时——先是问题,再是推导步骤,最后是最终答案——渐进渲染的效果很好。Agent 通常会按 schema 顺序生成字段,因此 UI 会自然而然地逐步填充。

## 最佳实践

* **渲染前先校验**:渲染前务必检查必需字段是否存在,因为流式可能只交付部分数据
* **使用泛型提取函数**:用类型和必需字段参数化你的提取逻辑,使其适用于不同的 schema
* **渐进渲染**:字段一到达就展示,而不是等待完整对象,让用户获得即时反馈
* **提供兜底表示**:如果一个字段支持富渲染(LaTeX、Markdown、图表),在你的 schema 中同时包含一个纯文本等价字段作为兜底
* **尽可能保持 schema 扁平**:深度嵌套的 schema 更难渐进渲染,在部分流式期间也更容易出错
* **UI 与数据相匹配**:选择最能代表每种字段类型的渲染策略(数组用表格、嵌套对象用卡片、状态字段用徽章)
