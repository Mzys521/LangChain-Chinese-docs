# 人机协同(Human-in-the-loop)

> 在执行不可逆操作前,让人类审查并批准 Agent 的动作

> 原文:[Human-in-the-loop](https://docs.langchain.com/oss/python/langchain/frontend/human-in-the-loop)

并非所有 Agent 动作都应该在无人监督的情况下运行。当 Agent 即将发送邮件、删除记录、执行金融交易,或进行任何不可逆操作时,你需要人类先审查并批准该动作。**人机协同(Human-in-the-Loop,HITL)**模式允许你的 Agent 暂停执行、将待处理动作呈现给用户,并仅在获得明确批准后才恢复。

由于 HITL 建立在 LangGraph 的中断(interrupts)与检查点(checkpoints)之上,因此暂停是**持久的(durable)**。用户可以刷新页面,审查者可以从不同的组件作出回应,而 Agent 依然能从执行停止的确切位置恢复,而无需重放整个运行。

(页面内嵌的交互式示例 demo 此处省略)

## 中断的工作原理

LangGraph Agent 支持**中断(interrupts)**——显式的暂停点,Agent 在此将控制权交还给客户端。当 Agent 命中一个中断时:

1. Agent 停止执行并发出一个中断载荷(interrupt payload)
2. [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) hook 通过 `stream.interrupt` 暴露该中断
3. 你的 UI 渲染一张带批准/拒绝/编辑选项的审查卡片
4. 用户作出决定
5. 你的代码用恢复命令调用 `stream.submit()`
6. Agent 从中断处继续执行

前端 SDK 会把中断与线程状态的其余部分一起保存,因此你的 UI 可以在任何合适的地方渲染它:内联在对话记录中、放在审查队列里、放在管理员仪表盘中,或放在一个模态框中、在作出决定前阻止用户的下一步操作。

## 配置 `useStream`

将 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 连接到你的人机协同 Agent。当图命中一个中断时,该 hook 会在 `stream.interrupt` 上暴露待处理的载荷。在该值存在期间渲染一张批准卡片,然后在用户批准、拒绝或编辑动作后,用 `stream.submit(null, { command: { resume: response } })` 恢复运行。

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
    assistantId: "human_in_the_loop",
  });

  const interrupt = stream.interrupt;

  return (
    <div>
      {stream.messages.map((msg) => (
        <Message key={msg.id} message={msg} />
      ))}
      {interrupt && (
        <ApprovalCard
          interrupt={interrupt}
          onRespond={(response) =>
            stream.submit(null, { command: { resume: response } })
          }
        />
      )}
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
  assistantId: "human_in_the_loop",
});

function handleRespond(response: HITLResponse) {
  stream.submit(null, { command: { resume: response } });
}
</script>

<template>
  <div>
    <Message
      v-for="msg in stream.messages.value"
      :key="msg.id"
      :message="msg"
    />
    <ApprovalCard
      v-if="stream.interrupt.value"
      :interrupt="stream.interrupt.value"
      @respond="handleRespond"
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
    assistantId: "human_in_the_loop",
  });

  function handleRespond(response: HITLResponse) {
    stream.submit(null, { command: { resume: response } });
  }
</script>

<div>
  {#each stream.messages as msg (msg.id)}
    <Message message={msg} />
  {/each}

  {#if stream.interrupt}
    <ApprovalCard interrupt={stream.interrupt} onRespond={handleRespond} />
  {/if}
</div>
```

**Angular:**

```ts
import { Component } from "@angular/core";
import { injectStream } from "@langchain/angular";
import type { HITLResponse } from "langchain";

const AGENT_URL = "http://localhost:2024";

@Component({
  selector: "app-chat",
  template: `
    @for (msg of stream.messages(); track msg.id) {
      <app-message [message]="msg" />
    }
    @if (stream.interrupt()) {
      <app-approval-card
        [interrupt]="stream.interrupt()"
        (respond)="handleRespond($event)"
      />
    }
  `,
})
export class ChatComponent {
  stream = injectStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "human_in_the_loop",
  });

  handleRespond(response: HITLResponse) {
    this.stream.submit(null, { command: { resume: response } });
  }
}
```

## 中断载荷

当 Agent 暂停时,`stream.interrupt` 包含一个 [HITLRequest](https://reference.langchain.com/javascript/langchain/index/HITLRequest),其结构如下:

```ts
interface HITLRequest {
  actionRequests: ActionRequest[];
  reviewConfigs: ReviewConfig[];
}

interface ActionRequest {
  name: string;
  args: Record<string, unknown>;
  description?: string;
}

interface ReviewConfig {
  allowedDecisions: ("approve" | "reject" | "edit" | "respond")[];
}
```

| 属性 | 说明 |
| ---------------------------------- | --------------------------------------------------------------------- |
| `actionRequests`                   | Agent 想要执行的待处理动作数组 |
| `actionRequests[].name`            | 动作名称(例如 `"send_email"`、`"delete_record"`) |
| `actionRequests[].args`            | 该动作的结构化参数 |
| `actionRequests[].description`     | 可选的、供人类阅读的该动作说明 |
| `reviewConfigs`                    | 逐动作的配置,控制允许哪些决定 |
| `reviewConfigs[].allowedDecisions` | 要展示哪些按钮:`"approve"`(批准)、`"reject"`(拒绝)、`"edit"`(编辑)、`"respond"`(回应) |

## 决定类型

HITL 模式支持四种决定类型:

### 批准(Approve)

用户确认动作应按原样执行:

```ts
const response: HITLResponse = {
  decisions: [{ type: "approve" }],
};

stream.submit(null, { command: { resume: response } });
```

### 拒绝(Reject)

用户拒绝该动作,并可附上可选的理由。工具不会被执行:

```ts
const response: HITLResponse = {
  decisions: [
    {
      type: "reject",
      message: "The email tone is too aggressive. Do not send it.",
    },
  ],
};

stream.submit(null, { command: { resume: response } });
```

> **注意**
>
> 当动作被拒绝时,Agent 会收到拒绝理由,并可自行决定如何继续。如果你省略 `message`,后端会使用一条默认消息,告知模型该工具未被执行,除非用户要求,否则不要重试相同的工具调用。对于会产生副作用的工具,请传入一条清晰的消息,告诉 Agent 是放弃该动作、提出后续问题,还是尝试更安全的替代方案。

### 编辑(Edit)

用户在批准前修改动作的参数:

```ts
const response: HITLResponse = {
  decisions: [
    {
      type: "edit",
      editedAction: {
        name: actionRequest.name,
        args: {
          ...actionRequest.args,
          subject: "Updated subject line",
          body: "Revised email body with softer language.",
        },
      },
    },
  ],
};

stream.submit(null, { command: { resume: response } });
```

### 回应(Respond)

用户为"询问用户"类型的工具提供直接回复。`message` 会成为工具结果,而工具本身不会被执行:

```ts
const response: HITLResponse = {
  decisions: [{ type: "respond", message: "Blue." }],
};

stream.submit(null, { command: { resume: response } });
```

> **注意**
>
> 当工具有意作为人类输入的占位符时使用 `respond`,例如一个提示 Agent 向用户收集信息的 `ask_user` 工具。不要用 `respond` 来否决一个被提议的动作,因为它会作为一次成功的工具结果返回给模型。

## 构建 ApprovalCard

下面是批准卡片所使用的决定逻辑。UI 可以把每个动作拆成各自的卡片,但恢复载荷是单个 `HITLResponse`,其中每个待处理动作对应一个决定:

```tsx
async function approveAll() {
  const resume: HITLResponse = {
    decisions: actionRequests.map(() => ({ type: "approve" })),
  };
  await stream.submit(null, { command: { resume } });
}

async function rejectOne(index: number, message: string) {
  const resume: HITLResponse = {
    decisions: actionRequests.map((_, i) =>
      i === index
        ? { type: "reject", message }
        : { type: "reject", message: "Rejected along with other actions" },
    ),
  };
  await stream.submit(null, { command: { resume } });
}

async function editOne(index: number, editedArgs: Record<string, unknown>) {
  const originalAction = actionRequests[index];
  const resume: HITLResponse = {
    decisions: actionRequests.map((_, i) =>
      i === index
        ? {
            type: "edit",
            editedAction: { name: originalAction.name, args: editedArgs },
          }
        : { type: "approve" },
    ),
  };
  await stream.submit(null, { command: { resume } });
}
```

## 恢复流程

在用户作出决定后,完整的循环如下:

1. 调用 `stream.submit(null, { command: { resume: hitlResponse } })`
2. [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) hook 将恢复命令发送到 LangGraph 后端
3. Agent 接收 `HITLResponse` 并继续执行。`decisions` 中的每一项可能是:
   * `{ type: "approve" }`:Agent 继续执行该动作
   * `{ type: "reject", message }`:工具不被执行,Agent 在决定下一步之前收到拒绝消息
   * `{ type: "edit", editedAction }`:Agent 用编辑后的参数运行工具
   * `{ type: "respond", message }`:人类的消息直接作为工具结果返回,而不执行工具
4. 当 Agent 恢复流式输出时,`interrupt` 属性被重置为 `null`

> **提示**
>
> 你可以在单次 Agent 运行中串联多个 HITL 检查点。例如,Agent 可能先请求搜索的批准,然后在用结果发送邮件前再次请求批准。每个中断都被独立处理。

## 处理多个待处理动作

当 Agent 想要一次执行多个动作时,一个中断可以包含多个 `actionRequests`。为每个动作渲染一张卡片,并在恢复前收集所有决定:

```tsx
function MultiActionReview({
  interrupt,
  onRespond,
}: {
  interrupt: { value: HITLRequest };
  onRespond: (response: HITLResponse) => void;
}) {
  const [decisions, setDecisions] = useState<Record<number, HITLResponse["decisions"][number]>>({});
  const request = interrupt.value;

  const allDecided =
    Object.keys(decisions).length === request.actionRequests.length;

  return (
    <div className="space-y-4">
      {request.actionRequests.map((action, i) => (
        <SingleActionCard
          key={i}
          action={action}
          config={request.reviewConfigs[i]}
          onDecide={(response) =>
            setDecisions((prev) => ({ ...prev, [i]: response }))
          }
        />
      ))}
      {allDecided && (
        <button
          className="rounded bg-green-600 px-4 py-2 text-white"
          onClick={() =>
            onRespond({
              decisions: request.actionRequests.map((_, i) => decisions[i]),
            })
          }
        >
          Submit All Decisions
        </button>
      )}
    </div>
  );
}
```

## 自定义中断表单

上述[恢复流程](#恢复流程)使用 `humanInTheLoopMiddleware`,它用一个通用的批准/拒绝/编辑/回应卡片来包裹工具。有时单单一组按钮并不够:预订航班、批准退款、审查社交帖子各自需要一个*不同的*表单,带有各自的字段、校验与文案。为此,可以**在工具内部**抛出 `interrupt()`,并让载荷描述 UI 应当渲染的确切表单。每个工具都可以呈现完全不同的界面。

(页面内嵌的交互式示例 demo 此处省略)

### 在中断载荷中描述表单

`interrupt()` 接受任何可 JSON 序列化的值,这使你能够提供一个前端知道如何渲染的"卡片",例如表单类型、标题、人类正在审查的上下文以及要收集的字段。`interrupt()` 在其输入和返回类型上是泛型的(`interrupt<I, R>(value: I): R`),因此你可以同时为你发送的卡片(`InterruptCard`)和用户用以解决中断的值(`ReviewDecision`)标注类型。导出这些类型,以便前端可以导入它们并保持同步:

```ts
import { createAgent, tool } from "langchain";
import { interrupt } from "@langchain/langgraph";
import { z } from "zod";

export interface FormField {
  name: string;
  label: string;
  type: "select" | "checkbox" | "textarea" | "currency";
  options?: string[];
  default?: unknown;
}

/** 用户用以解决中断的值。 */
export interface ReviewDecision {
  approved: boolean;
  /** 工具应据以行动的、已编辑/收集的表单值。 */
  values?: Record<string, unknown>;
}

/** 中断交给前端的表单规格("卡片")。 */
export interface InterruptCard {
  formType: "flight-booking" | "refund-approval" | "content-review";
  tool: string;
  title: string;
  context: Record<string, unknown>;
  fields: FormField[];
  /** 当前端把已解决的卡片提交到状态时填充。 */
  resolved?: boolean;
  decision?: ReviewDecision;
}

const bookFlight = tool(
  async ({ origin, destination, date, passengers }) => {
    // 暂停工具,把一个带类型的表单规格交给前端;带类型的返回值
    // 就是 UI 解决该中断时返回的内容。
    const decision = interrupt<InterruptCard, ReviewDecision>({
      formType: "flight-booking",
      tool: "book_flight",
      title: "Confirm flight booking",
      context: { origin, destination, date, passengers },
      fields: [
        {
          name: "seatClass",
          label: "Seat class",
          type: "select",
          options: ["Economy", "Premium Economy", "Business"],
          default: "Economy",
        },
        { name: "insurance", label: "Add trip insurance", type: "checkbox", default: false },
      ],
    });

    if (!decision.approved) {
      return `Booking cancelled. No flight from ${origin} to ${destination} was reserved.`;
    }

    // 用人类确认的值运行真实的(可能较慢的)工作。
    const seatClass = String(decision.values?.seatClass ?? "Economy");
    return `Flight booked from ${origin} to ${destination} in ${seatClass}.`;
  },
  {
    name: "book_flight",
    description: "Book a flight. Requires human confirmation of trip details.",
    schema: z.object({
      origin: z.string(),
      destination: z.string(),
      date: z.string(),
      passengers: z.number().int().min(1),
    }),
  },
);
```

为每个工具赋予一个不同的 `formType`(例如 `"refund-approval"`、`"content-review"`),这样前端就能据此切换并渲染匹配的表单。

### 为每个工具渲染不同的表单

在客户端,卡片会以 `stream.interrupt.value` 的形式到达。从你的 Agent 模块导入 `InterruptCard` 和 `ReviewDecision` 类型,使表单与载荷保持同步;根据 `formType` 切换以选择正确的表单,并把 `fields` 喂给输入控件:

```tsx
import { useStream } from "@langchain/react";
import type { InterruptCard, ReviewDecision } from "./agent";

function Chat() {
  const stream = useStream<typeof myAgent>({
    apiUrl: AGENT_URL,
    assistantId: "hitl_interrupt_forms",
  });

  const card = stream.interrupt?.value as InterruptCard | undefined;

  return (
    <div>
      {stream.messages.map((msg) => (
        <Message key={msg.id} message={msg} />
      ))}
      {card && <InterruptForm card={card} onResolve={handleResolve} />}
    </div>
  );
}

// `InterruptForm` 根据 `card.formType` 渲染航班/退款/内容卡片,
// 收集 `card.fields`,并用用户的决定与已编辑的值调用 `onResolve`。
```

### 用 `respond(decision, { update })` 让卡片留在屏幕上

当你解决一个普通中断时,卡片会在中断被清除的瞬间消失,只剩下工具结果返回。这意味着一张内容丰富的审查卡片会在运行中途消失。要让它留在屏幕上,请在*同一个*超步(superstep)中,使用 [`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 的 `respond`,**既**解决中断、**又**把一条承载该卡片的消息提交到状态:

```tsx
import { AIMessage } from "langchain";

function handleResolve(decision: ReviewDecision) {
  // 把已内嵌决定的卡片做快照,使其以只读方式渲染。
  const resolvedCard = { ...card, resolved: true, decision };
  const cardMessage = new AIMessage({
    content: `Review ${decision.approved ? "approved" : "declined"}.`,
    response_metadata: { cards: resolvedCard },
  });

  // 恢复中断并原子性地把卡片推入状态。对应
  // LangGraph 的 `Command(resume, update)`:单个检查点,无额外状态写入。
  stream.respond(decision, { update: { messages: [cardMessage] } });
}
```

`respond(response, { update })` 会**乐观地(optimistically)**应用该 `update`:卡片立即绘制,并在恢复的运行回显同一条消息后按 ID 进行协调。后端从不重新发出卡片,因此在(可能较慢的)工具运行期间,卡片无闪烁地保持渲染。通过从消息中读回已解决的卡片来渲染它:

```tsx
{stream.messages.map((msg) => {
  const card = (msg.response_metadata as { cards?: InterruptCard })?.cards;
  if (card) return <InterruptForm key={msg.id} card={card} readOnly />;
  return <Message key={msg.id} message={msg} />;
})}
```

> **提示**
>
> 由于已解决的卡片存在于消息历史中,它能在刷新后依然存在,并对每个读取该线程的组件可见——人类的决定由此成为持久对话记录的一部分,而不仅仅是转瞬即逝的 UI 状态。

## 最佳实践

实现 HITL 工作流时,请牢记以下准则:

* **展示清晰的上下文**。始终显示 Agent *想要做什么*以及*为什么*。包含动作描述和完整参数。
* **让批准成为最简单的路径**。如果动作看起来正确,批准应当只需一次点击。把多步流程留给拒绝/编辑。
* **校验已编辑的参数**。当用户编辑动作参数时,在发送前校验 JSON 结构。对格式错误的输入显示内联错误。
* **持久化中断状态**。如果用户刷新页面,中断应依然可见。[`useStream`](https://reference.langchain.com/javascript/langchain-react/index/useStream) 通过线程的检查点来处理这一点。
* **记录所有决定**。出于审计需要,记录每一次批准/拒绝/编辑决定,并附上时间戳与作出决定的用户。
* **审慎设置超时**。长时间运行的 Agent 不应无限期地阻塞在人类审查上。考虑显示 Agent 已等待了多久。
