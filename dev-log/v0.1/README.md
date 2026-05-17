# v0.1 — 骨架 + 单 Agent 对话

> 本目录是 [roadmap.md § 0.1](../../docs/roadmap.md) 的实施拆分。
> 体系约定见 [`dev-log/README.md`](../README.md)。
>
> v0.1 的每个 unit 当前都只有 `issue.md`(立项稿);开发前补 `design.md`、PR 实施完成后由 `/review` skill 生成 `review.html`,merge 之前必看。

## 目标回顾

跑通最基础的 chat、验证 LangGraph 内核选型。

退出标准(来自 roadmap):

- [ ] `sigma chat` 能跟 LLM 多轮对话
- [ ] 工具调用流跑通(agent 调 read_file 读文件回答)
- [ ] Trace JSONL 文件能看到每次 LLM 调用 / tool call
- [ ] 中断恢复:kill 进程后 `sigma chat --resume <session>` 能续上

## 不做(边界)

Task 引擎 / Skill / Sub-agent / RAG / Memory / 多 provider 路由 / Web UI / Realtime — 都留给后续 milestone。

## Unit 列表

按"功能场景"拆,11 个 unit;每个对应一个独立可合并的 PR。

| # | 主题 | 产物所在层 | 依赖 |
|---|------|-----------|------|
| [01](01-project-skeleton/issue.md) | 项目骨架 + 工具链 | repo 根 / `src/` 空目录 | — |
| [02](02-core-schemas-ports/issue.md) | core/schemas + ports 接口 | `core/` `ports/` | 01 |
| [03](03-llm-adapter/issue.md) | LLM adapter(OpenAI 兼容中转) | `llm/` | 02 |
| [04](04-builtin-tools/issue.md) | 内置 tools(file/shell/web/git) | `tools/` | 02 |
| [05](05-jsonl-tracer/issue.md) | JSONLTracer | `trace/` | 02 |
| [06](06-checkpointer/issue.md) | Checkpointer(SqliteSaver) | `checkpoint/` | 02 |
| [07](07-master-agent/issue.md) | Master Agent(单节点 graph) | `agent/` | 03 04 05 06 |
| [08](08-chat-engine/issue.md) | Chat engine + session | `chat/` | 07 |
| [09](09-server-sse/issue.md) | Server(HTTP/SSE) | `server/` | 08 |
| [10](10-cli-chat/issue.md) | CLI `sigma chat` | `app/` | 09 |
| [11](11-e2e-verification/issue.md) | E2E 验收(4 条退出标准) | `tests/` | 10 |

## 依赖图

```
                ┌──── 01 skeleton
                │
                ▼
              02 core+ports
        ┌───────┼───────┬───────────┐
        ▼       ▼       ▼           ▼
       03 llm 04 tools 05 trace  06 checkpoint
        └───────┴───────┴───────────┘
                        ▼
                   07 master agent
                        ▼
                   08 chat engine
                        ▼
                   09 server (SSE)
                        ▼
                   10 cli `sigma chat`
                        ▼
                   11 e2e 验收
```

03/04/05/06 互相独立,可并行 PR。07 是会合点。

## 退出标准 ↔ Unit 映射

| 退出标准 | 由哪些 unit 共同保证 | 由哪个 unit 做 e2e 验收 |
|---|---|---|
| `sigma chat` 能多轮对话 | 03 + 07 + 08 + 10 | 11 |
| 工具调用流跑通(read_file) | 04 + 07 | 11 |
| Trace JSONL 能看到 llm_call / tool_call | 05 + 07 | 11 |
| 中断恢复(kill + `--resume`) | 06 + 08 + 10 | 11 |

## 关键约定

所有 unit 严格遵守:

- [依赖规则](../../docs/architecture/dependency-rules.md) — 三条红线不可违反
- [代码规范](../../docs/guides/code-style.md) — 命名 / 异步 / 类型标注
- [Ports & Adapters](../../docs/architecture/ports-and-adapters.md) — Port 设计约束

## v0.1 完成时

unit 全部合入后,统一开 milestone-cleanup PR:

- 各 unit 的 `review.html` 里跨 unit / 跨 milestone 的 finding 提炼进 [design-log](../../docs/architecture/design-log.md)
- roadmap.md § 0.1 的 4 个 `[ ]` 改成 `[x]`,顶部 🚧 改成 ✅
