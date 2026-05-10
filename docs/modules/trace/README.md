# Trace 模块

> Sigma 的可观察性系统——**JSONL 默认 + 本地 HTML viewer + replay 能力**。

> Trace 是 Sigma 真正的差异化能力之一（详见 [架构总览 § 1](../../architecture/overview.md#1-sigma-是什么)）。

---

## 1. 定位

LangSmith / Langfuse 是云端付费的 trace 平台。Sigma 的差异化方向：

- **本地优先**：trace 落到 `~/.sigma/traces/`，不上传任何地方
- **零依赖**：JSONL 文件 + 一个本地 HTML viewer，开箱即用
- **Replay**：基于 trace 重跑某次执行（debug 利器）
- **教学优先**：trace 内容自包含，新手能看懂"agent 这次做了什么决策"

---

## 2. 捕获什么

每个 task / session 一个 trace 文件，记录所有有意义的事件：

| 事件类型 | 内容 |
|---|---|
| `agent_start` / `agent_end` | 哪个 agent 开始 / 结束 |
| `node_start` / `node_end` | LangGraph node 边界（input / output / duration） |
| `llm_call` | model / messages / response / tokens / cost / duration |
| `tool_call` | tool name / args / result / duration |
| `agent_handoff` | 主 agent 派给 sub-agent（task description / context） |
| `proxy_decision` | L2 主 agent 代答（reason / question / answer / 推理） |
| `user_input` | L3 用户回复（chat 追问 / task resume） |
| `state_transition` | task 状态变更（queued → running → paused → ...） |
| `checkpoint` | LangGraph checkpoint 写入 |
| `error` | 任何异常 |

---

## 3. JSONL 格式

每行一个 JSON event：

```jsonl
{"ts":"2026-05-09T09:30:00Z","type":"agent_start","agent":"master","task_id":"t-123","trace_id":"tr-001"}
{"ts":"2026-05-09T09:30:01Z","type":"llm_call","model":"gpt-4o","prompt_tokens":1500,"completion_tokens":200,"cost_usd":0.012,"duration_ms":850,"trace_id":"tr-001","span_id":"sp-002"}
{"ts":"2026-05-09T09:30:02Z","type":"tool_call","tool":"search_web","args":{"q":"..."},"duration_ms":1200,"trace_id":"tr-001","span_id":"sp-003"}
```

字段约定：
- `ts`：ISO 8601 时间戳
- `type`：事件类型
- `trace_id`：整个 task / session 的 trace ID
- `span_id`：单个事件的 ID
- `parent_span_id`：父 span（构成树状结构）
- 各类型自己的 payload

---

## 4. 本地 HTML Viewer

启动方式：

```bash
sigma trace view <trace_id>           # 打开浏览器看某次执行
sigma trace list                      # 列出最近 trace
sigma trace replay <trace_id>         # 基于 trace 重跑
```

Viewer 形态：
- **时间轴**：每个事件按时间排列
- **嵌套树**：agent / sub-agent / tool call 的层级
- **Cost 面板**：累计 token / 费用 / 各 model 占比
- **Diff 视图**：前后两次 trace 对比（用于 debug 行为变化）
- **过滤**：按 event type / agent / time range

---

## 5. Replay

基于 trace 重跑：

```bash
sigma trace replay <trace_id> --modify-prompt "..."
sigma trace replay <trace_id> --skip-tools     # 干跑，不真的调 tool
sigma trace replay <trace_id> --diff           # 对比新旧
```

**用途**：
- Debug：某次决策错了，改 prompt 重跑看是否修复
- 优化：调整 cost-aware 路由策略，重跑历史 trace 看成本变化
- 测试：把好的 trace 当 regression case

实现基础：LangGraph 的 checkpoint + 事件回放。

---

## 6. TracerPort

```python
class TracerPort(Protocol):
    async def record_event(self, event: TraceEvent) -> None: ...
    async def flush(self) -> None: ...
```

Adapter：
- `JSONLTracer`（默认）：写本地 `~/.sigma/traces/<trace_id>.jsonl`
- `OTelTracer`（V5+）：导出到 OpenTelemetry collector
- `LangfuseTracer` / `LangSmithTracer`（V5+，可选接入云端）

---

## 7. Trace 在 multi-agent 中

嵌套 agent 的 trace 是树状的：

```
trace_id: tr-001
└── agent_start (master)
    ├── llm_call
    ├── agent_handoff → researcher
    │   └── agent_start (researcher) [parent: master]
    │       ├── llm_call
    │       ├── tool_call (search_web)
    │       └── agent_end
    ├── agent_handoff → coder
    │   └── agent_start (coder) [parent: master]
    │       └── ...
    └── agent_end
```

Viewer 里能折叠/展开每一层。

---

## 8. 隐私

Trace 默认存本地，不上传任何地方。但 LLM messages 可能含敏感信息（用户私信、API key）。

**安全机制**：
- 默认开启敏感信息掩码（按 regex 匹配 API key / token / email）
- 用户可在配置里调整 / 关闭
- `sigma trace clean` 删除指定 trace

---

## 9. 实现位置

```
src/trace/
  base.py             # TracerPort
  schema.py           # TraceEvent 数据结构
  registry.py         # tracer registry
  
  jsonl.py            # JSONLTracer（默认）
  otel.py             # OTelTracer (V5+)
  
  viewer/             # 本地 HTML viewer
    server.py         # 启动 viewer 服务
    static/           # 前端
  
  replay/             # Replay 引擎
    engine.py
    diff.py
```

---

## 10. 未决问题

| 问题 | 状态 |
|---|---|
| Trace 文件大小管理（rotation / 压缩 / 删除策略） | V2 |
| Viewer 前端选型（vanilla JS / React / Svelte） | V2 |
| Replay 的精确语义（输入完全相同时 LLM 输出不一致怎么办） | V3 |
| 跟 LangSmith / Langfuse 的兼容（导入导出） | V5+ |

---

## 11. 相关文档

- [架构总览 § 3.2](../../architecture/overview.md#32-sigma-自己造-vs-直接用-langgraph) — Trace 是 Sigma 自己长的肌肉
- [Multi-agent](../../architecture/multi-agent.md) — Multi-agent trace 嵌套
- [Ports & Adapters § 3.3](../../architecture/ports-and-adapters.md#33-tracerport)
- [设计决策日志](../../architecture/design-log.md)
