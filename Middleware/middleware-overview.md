# Middleware 概览(中间件概览)

> 在每一步控制和自定义 Agent 的执行

> 原文:[Middleware Overview](https://docs.langchain.com/oss/python/langchain/middleware/overview)

中间件(Middleware)提供了一种更精细地控制 Agent 内部发生的事情的方式。中间件在以下场景中很有用:

- 通过日志、分析和调试来跟踪 Agent 行为。
- 转换提示词、[工具选择](https://docs.langchain.com/oss/python/langchain/middleware/built-in#llm-tool-selector)和输出格式。
- 添加[重试](https://docs.langchain.com/oss/python/langchain/middleware/built-in#tool-retry)、[回退(fallbacks)](https://docs.langchain.com/oss/python/langchain/middleware/built-in#model-fallback)和提前终止逻辑。
- 应用[速率限制](https://docs.langchain.com/oss/python/langchain/middleware/built-in#model-call-limit)、护栏(guardrails)和 [PII 检测](https://docs.langchain.com/oss/python/langchain/middleware/built-in#pii-detection)。

把中间件传给 [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) 即可添加:

```python
from langchain.agents import create_agent
from langchain.agents.middleware import SummarizationMiddleware, HumanInTheLoopMiddleware

agent = create_agent(
    model="gpt-5.5",
    tools=[...],
    middleware=[
        SummarizationMiddleware(...),
        HumanInTheLoopMiddleware(...)
    ],
)
```

## Agent 循环(The agent loop)

核心 Agent 循环包括:调用模型、让模型选择要执行的工具,然后在它不再调用任何工具时结束。

中间件在这些步骤的每一步前后都暴露了钩子(hooks)。

## 在 LangGraph 工作流中使用中间件

中间件不是一个独立的运行时(runtime):钩子运行在 [`create_agent`](https://reference.langchain.com/python/langchain/agents/factory/create_agent) 返回的编译后 [LangGraph](https://docs.langchain.com/oss/python/langgraph/overview) 内部。你可以把整个 Agent(连同所有中间件)作为节点或子图放进更大的 [StateGraph](https://reference.langchain.com/python/langgraph/graph/state/StateGraph) 中,所有中间件钩子都会继续运行。

当周围拓扑超出了标准的"循环直到完成"模式时,就该用这种模式了:例如在路由到多个 Agent 之一之前先对输入分类、并行展开(fan out)工作,或把 Agent 调用与确定性步骤拼接在一起。

`HumanInTheLoopMiddleware` 会根据每个工具的 `.name` 进行匹配。

`@tool` 装饰的函数名取自函数本身,因此下面的键是 `"send_email"`。

```python
from langchain.agents import AgentState, create_agent
from langchain.agents.middleware import HumanInTheLoopMiddleware
from langgraph.graph import START, StateGraph

# 假定 read_email、send_email、classify_node 和 route 已在别处定义。
email_agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[read_email, send_email],
    middleware=[HumanInTheLoopMiddleware(interrupt_on={"send_email": True})],
)

graph = (
    StateGraph(AgentState)
    .add_node("classify", classify_node)
    .add_node("email_agent", email_agent)
    .add_edge(START, "classify")
    .add_conditional_edges("classify", route)
    .compile()
)
```

HITL(人机协同)中断、摘要、PII 编辑、重试以及任何自定义钩子都会随 Agent 节点一起生效。完整的组合模式集合(包括子图 checkpointer 的作用域:按调用与按线程)参见[使用子图](https://docs.langchain.com/oss/python/langgraph/use-subgraphs)。

## 更多资源

- [内置中间件](https://docs.langchain.com/oss/python/langchain/middleware/built-in):探索常见使用场景的内置中间件。
- [自定义中间件](https://docs.langchain.com/oss/python/langchain/middleware/custom):使用钩子和装饰器构建你自己的中间件。
- [中间件 API 参考](https://reference.langchain.com/python/langchain/middleware/):中间件的完整 API 参考。
- [中间件集成](https://docs.langchain.com/oss/python/integrations/middleware/):针对 Anthropic、AWS、OpenAI 等的提供商特有中间件。
- [测试 Agent](https://docs.langchain.com/oss/python/langchain/test/):使用 LangSmith 测试你的 Agent。
