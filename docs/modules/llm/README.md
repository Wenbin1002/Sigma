# LLM 模块

> Sigma 的 LLM 接入层——**多 provider** + **cost-aware 路由** + **统一 streaming 协议**。

> 详见 [Ports & Adapters § 3.1](../../architecture/ports-and-adapters.md#31-llmport)。

---

## 1. 定位

LLM 是 Sigma 内**真正满足"多 provider"标准**的能力之一（详见 [Ports & Adapters § 1](../../architecture/ports-and-adapters.md#1-哪些能力是-port为什么)）：

- 成本差异显著（GPT-4 vs Claude Haiku 差几十倍）
- 性能差异显著（推理 / 代码 / 多模态各家强项不同）
- 合规需求（部分用户需要本地模型 / 国内 provider）
- 生态丰富（OpenAI / Anthropic / DeepSeek / Qwen / 各种 oss）

所以 LLM 必须有 Port，必须设计好。

---

## 2. LLMPort

```python
class LLMPort(Protocol):
    async def stream(
        self,
        messages: list[Message],
        tools: list[ToolSchema] | None = None,
        **kwargs,
    ) -> AsyncIterator[LLMChunk]: ...
    
    async def generate(
        self,
        messages: list[Message],
        tools: list[ToolSchema] | None = None,
        **kwargs,
    ) -> LLMResponse: ...
```

**关键约束**：
- 输入 `messages` 遵循 OpenAI 消息格式（`role` + `content` + `tool_calls`）——事实标准
- `tools` 遵循 OpenAI tool calling schema
- `LLMChunk` / `LLMResponse` **必须包含 token usage** —— Sigma 的 cost guard 依赖

```python
@dataclass
class LLMChunk:
    type: Literal["text", "tool_call", "usage", "done"]
    content: str | None = None
    tool_call: ToolCall | None = None
    usage: TokenUsage | None = None
    
@dataclass
class TokenUsage:
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    cost_usd: float | None = None
```

---

## 3. 内置 Adapter

| Adapter | Provider | 状态 |
|---|---|---|
| `OpenAILLM` | OpenAI | V0 |
| `DeepSeekLLM` | DeepSeek | V0 |
| `AnthropicLLM` | Anthropic | V0 |
| `QwenLLM` | 阿里 Qwen | V1 |
| `OllamaLLM` | 本地 Ollama | V2 |

---

## 4. Cost-Aware 路由（V1+）

简单任务走便宜模型，复杂任务走贵模型：

```yaml
llm:
  default:
    provider: openai
    model: gpt-4o
  
  routing:
    simple_tasks:    { provider: openai, model: gpt-4o-mini }
    complex_tasks:   { provider: anthropic, model: claude-opus-4 }
    coding:          { provider: anthropic, model: claude-sonnet-4 }
```

**判断"哪个 task 走哪个 model"**：
- 显式（agent 自己决定，比如 supervisor 判断后给 sub-agent 配 model）
- 自动（基于任务类型 / 历史成功率，长期可学）

V1 先做显式，V5+ 探索自动路由。

---

## 5. Cost Guard

详见 [架构总览 § 3.2](../../architecture/overview.md#32-sigma-自己造-vs-直接用-langgraph) 横切肌肉。

每个 LLM 调用累加 cost：

```
Task 启动时声明 budget
  ↓
每次 LLM 调用 → 累加 cost
  ↓
超 budget → LangGraph interrupt → master agent 决策（cancel / 升级到用户 / 降级走便宜 model）
```

配置：

```yaml
cost_guard:
  per_task_budget_usd: 1.0
  per_session_budget_usd: 5.0
  daily_budget_usd: 10.0
  on_exceed: "interrupt"   # interrupt / warn / continue
```

---

## 6. Token 计算

LLM provider 不一定每次都返回 token usage（流式中间 chunk 可能没有）。Sigma 维护一个本地 tokenizer 兜底：

```python
class TokenCounter:
    def count_messages(self, messages: list[Message], model: str) -> int: ...
    def count_completion(self, text: str, model: str) -> int: ...
```

实现：用 tiktoken（OpenAI / Anthropic 系列）+ 各 provider 自己的 tokenizer。

---

## 7. 统一 Streaming 协议

Sigma 内部所有 LLM 流都走 `AsyncIterator[LLMChunk]`，不管 provider 是 OpenAI 的 SSE、Anthropic 的 message stream、DeepSeek 的兼容协议——adapter 统一翻译成 `LLMChunk`。

LLMChunk 再被 agent 包成 `AgentChunk` 流给上层。

---

## 8. 实现位置

```
src/llm/
  base.py             # LLMPort / 数据结构
  registry.py         # provider registry
  
  openai.py           # OpenAILLM
  deepseek.py         # DeepSeekLLM
  anthropic.py        # AnthropicLLM
  qwen.py             # QwenLLM
  ollama.py           # OllamaLLM
  
  router.py           # cost-aware routing 决策
  cost.py             # cost 累加 + budget 拦截
  tokenizer.py        # 本地 token 计算兜底
```

---

## 9. 未决问题

| 问题 | 状态 |
|---|---|
| Cost-aware 自动路由的判断模型 | V5+ |
| 是否暴露 `EmbeddingPort`（RAG 跨 provider 需求） | V3 落地时看 |
| 多模态（image / audio）输入 | V4+ |
| Streaming 中断 / 重连 | V2 |

---

## 10. 相关文档

- [Ports & Adapters § 3.1](../../architecture/ports-and-adapters.md#31-llmport)
- [Agent 模块](../agent/) — LLM 调用方
- [Trace 模块](../trace/) — 每次 LLM 调用进 trace
- [架构总览](../../architecture/overview.md)
