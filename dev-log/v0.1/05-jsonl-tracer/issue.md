# Issue 05 — JSONLTracer

**Milestone**：v0.1
**Type**：`feat(trace)`
**依赖**：issue 02
**预计 PR 大小**：S-M

---

## 背景

0.1 的退出标准之一：「Trace JSONL 文件能看到每次 LLM 调用 / tool call」。本 issue 实现最小可用的 `JSONLTracer`——足以让 master agent 把 LLM/tool 事件落盘，方便后续 debug 和 0.6 上 viewer。

参考：[Trace 模块](../../../docs/modules/trace/README.md)、[Ports & Adapters § 3.3](../../../docs/architecture/ports-and-adapters.md#33-tracerport)。

## 范围

### 1. `src/sigma/trace/registry.py`

```python
REGISTRY: dict[str, type[TracerPort]] = {
    "jsonl": JSONLTracer,
}

def create(config) -> TracerPort: ...
```

### 2. `src/sigma/trace/jsonl.py`

`JSONLTracer` 实现 `TracerPort`：

- 文件路径：`<output_dir>/<trace_id>.jsonl`，`output_dir` 可配（默认 `~/.sigma/traces/`）
- `record_event`：把 `TraceEvent` dump 成单行 JSON，append 写
- 写入用 `aiofiles` 还是 `asyncio.to_thread`？——本 issue 直接 `asyncio.to_thread(file.write)`，避免新增依赖
- buffered：每 N 条 flush 一次，N 默认 1（即立即 flush，0.1 用量小、不卡 IO）
- `flush()`：fsync 文件 + 关闭 handle
- 进程退出 hook：`atexit` 注册 flush，避免崩溃丢事件

### 3. `core/schemas.TraceEvent` 与本 issue

`TraceEvent` 已在 issue 02 定义。本 issue 顺手补一个轻量构造帮手：

```python
# src/sigma/trace/events.py
def llm_call(model, prompt_tokens, completion_tokens, duration_ms, cost_usd, ...): ...
def tool_call(tool, args, result, duration_ms, ...): ...
def agent_start(agent, task_id): ...
def agent_end(agent, status): ...
def state_transition(from_, to, ...): ...
```

调用方（agent / chat / tools）import 这些 helper 拼 event，不需要自己组 dict。

### 4. 隐私：API key 掩码（最小版）

config 提供：

```yaml
trace:
  provider: jsonl
  output_dir: ~/.sigma/traces
  redact_patterns:           # 在 payload 序列化时 regex 替换
    - pattern: 'sk-[A-Za-z0-9]{20,}'
      replacement: '[REDACTED]'
```

实现思路：record_event 时遍历 payload 字符串字段，按 pattern 替换。

完整的"按字段类型 redact + 用户白名单"是 V2+。

## 设计要点

- **JSONL 一定要"每行一个完整 JSON"**——崩溃恢复时半行不会破坏后续解析（避免 BOM、避免在 json 里塞换行）
- **不要在 trace 里塞 binary**：tool result 是 image / file 时，`payload` 只存元信息（filename / mime / size），原始数据另存
- **trace_id 由调用方决定**——本 issue 不发 trace_id；agent / chat 启动时生成 ULID/UUID 传进 record_event
- **不要让 trace 阻塞主流程**：record_event 异常被吞掉 + 打 warning（`logging.warning`），保证 agent 能继续跑

## 验收标准

- [ ] `tests/trace/test_jsonl.py`：
  - record N 个事件后文件能解析回来 N 行 JSON
  - 并发 record（10 个 task 同时写同一 trace）不出现错行
  - api key 在 redact pattern 下被替换
- [ ] `agent_start → llm_call → tool_call → agent_end` 一组事件能正确按 `parent_span_id` 还原成树（在测试中手动构造）
- [ ] `grep -rE "^from sigma\.(llm|tools|checkpoint|agent)" src/sigma/trace/` 必须为空

## 不做

- 不实现 OTel / Langfuse / LangSmith adapter（V5+）
- 不实现 HTML viewer（0.6）
- 不实现 replay（V3）
- 不实现按字段类型自动 redact（V2+）
- 不实现 trace 文件 rotation / 压缩（V2）
- 不在 LangGraph callback 上挂自动 hook——0.1 由 agent / tools 显式调用 record_event；自动 hook 是 0.5 multi-agent 时再加，避免重复事件

## 相关文档

- [Trace 模块](../../../docs/modules/trace/README.md)
- [Ports & Adapters § 3.3](../../../docs/architecture/ports-and-adapters.md#33-tracerport)
