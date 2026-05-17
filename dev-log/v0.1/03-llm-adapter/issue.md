# Issue 03 — LLM adapter（OpenAI 兼容中转）

**Milestone**：v0.1
**Type**：`feat(llm)`
**依赖**：issue 02
**预计 PR 大小**：M

---

## 背景

0.1 验证 LangGraph 内核选型，需要一个能跑的 LLM。用户使用的是 **OpenAI 兼容中转站**——同一个 SDK 调用风格、自定义 `base_url`、可走多家底层模型。本 issue 实现 `OpenAICompatLLM` adapter（一份代码同时支持 OpenAI 官方和各类中转）。

参考：[LLM 模块](../../../docs/modules/llm/README.md)、[Ports & Adapters § 3.1](../../../docs/architecture/ports-and-adapters.md#31-llmport)。

## 范围

### 1. `src/sigma/llm/registry.py`

```python
REGISTRY: dict[str, type[LLMPort]] = {
    "openai_compat": OpenAICompatLLM,
}

def create(config: LLMConfig) -> LLMPort: ...
```

### 2. `src/sigma/llm/openai_compat.py`

实现 `LLMPort`：

- 用 `httpx.AsyncClient`（不直接依赖 `openai` SDK——一来减依赖，二来中转站偶有 SDK 不兼容的边角，自己写 SSE 解析更稳）
- 支持环境变量取 api_key（`config.api_key_env`，默认 `SIGMA_LLM_API_KEY`）
- `base_url` 可配，缺省 `https://api.openai.com/v1`
- `stream()`：调 `/chat/completions` with `stream=true`，逐 chunk 翻译成 `LLMChunk`
- `generate()`：非流式，便于内部使用；返回收尾 chunk（带 usage）

### 3. Tool calling

- 入参 `tools: list[ToolSchema]` 翻译成 OpenAI tool calling schema：
  ```json
  {"type":"function","function":{"name":"...","description":"...","parameters":{...}}}
  ```
- 流式 chunk 里 OpenAI 会拆分 `tool_calls` 字段（function name 和 arguments 分多次到达）。adapter **内部累积**，参数 JSON 完整时发 `LLMChunk(type="tool_call", tool_call=...)`；上游不需要看到中间状态。

### 4. Token usage

- OpenAI 流式默认不返回 usage；本 adapter 强制 `stream_options={"include_usage": true}`，最后一个 chunk 会带 `usage`
- 中转站不一定支持 `include_usage`——若拿不到，用 [tokenizer 兜底](../../../docs/modules/llm/README.md#6-token-计算)：本 issue **暂不实现 tokenizer**，拿不到时填 `prompt_tokens=0, completion_tokens=0` + 在 metadata 标记 `usage_missing=true`。tokenizer 等到 0.5 cost-aware 路由真正用 cost 时再补。

### 5. 错误处理

- 网络 / 4xx / 5xx 全部 catch 包成项目内统一异常 `LLMRequestError`（放在 `core/exceptions.py`，本 issue 顺手加）
- 不要 raise OpenAI SDK 的原生异常——adapter 边界要封装

### 6. 配置 schema

匹配 issue 01 的 `config.example.yaml`：

```yaml
llm:
  default:
    provider: openai_compat
    model: gpt-4o-mini
    base_url: https://YOUR_RELAY/v1
    api_key_env: SIGMA_LLM_API_KEY
    timeout_seconds: 60
    max_retries: 2
```

## 设计要点

- **一个文件一个 adapter**（[code-style § 文件组织](../../../docs/guides/code-style.md#文件组织)）。即使现在只有一个 provider，也用 registry，0.5 加 DeepSeek/Anthropic 时直接进表
- **不在 0.1 做 cost-aware 路由 / cost guard / tokenizer 兜底**——这些是 0.5 才用上，超前做容易做错
- **不在 adapter 持有外部状态**——`httpx.AsyncClient` 在 `__init__` 创建，`__aclose__` 释放；不要做单例
- **流式 SSE 解析**：用 `httpx.stream` + 自己分行；避免引入额外 SSE 库

## 验收标准

- [ ] `tests/llm/test_openai_compat.py`：用 `respx` mock 中转站响应：
  - [ ] 普通流式文本响应被翻译成 `LLMChunk(type="text")`
  - [ ] tool calling 响应被累积成完整的 `LLMChunk(type="tool_call")`，arguments 是 dict
  - [ ] usage chunk 有 `prompt_tokens` / `completion_tokens`
  - [ ] 4xx 抛 `LLMRequestError`
- [ ] 真实集成 smoke：`SIGMA_LLM_API_KEY=... python -m sigma.llm.smoke "你好"` 能流式打印一段话（这条不进 CI，写进 README）
- [ ] `grep -rE "^from sigma\.(tools|trace|checkpoint|agent)" src/sigma/llm/` 必须为空（域目录不互引）

## 不做

- 不接 DeepSeek / Anthropic / Qwen / Ollama 等其他 provider（0.5）
- 不实现 cost-aware 路由（0.5）
- 不实现 tokenizer 兜底（0.5）
- 不接 multi-modal（V4+）
- 不实现 streaming 中断 / 重连（V2）

## 相关文档

- [LLM 模块](../../../docs/modules/llm/README.md)
- [Ports & Adapters § 3.1](../../../docs/architecture/ports-and-adapters.md#31-llmport)
