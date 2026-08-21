# 安装 LangChain

> 原文:[Install LangChain](https://docs.langchain.com/oss/python/langchain/install)

安装 LangChain 包:

**使用 pip:**

```bash
pip install -U langchain
# 需要 Python 3.10+
```

**使用 uv:**

```bash
uv add langchain
# 需要 Python 3.10+
```

LangChain 提供了数百个 LLM 集成以及数千个其他集成,它们分布在各自独立的提供商(provider)包中。

**使用 pip:**

```bash
# 安装 OpenAI 集成
pip install -U langchain-openai

# 安装 Anthropic 集成
pip install -U langchain-anthropic
```

**使用 uv:**

```bash
# 安装 OpenAI 集成
uv add langchain-openai

# 安装 Anthropic 集成
uv add langchain-anthropic
```

> **提示**
>
> 完整的可用集成列表请参见 [Integrations 页签](https://docs.langchain.com/oss/python/integrations/providers/overview)。

安装完 LangChain 后,你可以按照[快速入门指南](https://docs.langchain.com/oss/python/langchain/quickstart)开始上手。

> **提示**
>
> 配置 [LangSmith](https://smith.langchain.com) 追踪(tracing)来调试你的第一个 LangChain 应用。可参考[追踪快速入门](https://docs.langchain.com/langsmith/trace-with-langchain)完成配置。我们还建议同时配置 [LangSmith Engine](https://docs.langchain.com/langsmith/engine),它可以监控你的追踪数据、检测问题并提出修复建议。
