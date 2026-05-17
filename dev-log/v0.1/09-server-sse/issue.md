# Issue 09 — Server（HTTP/SSE chat API）

**Milestone**：v0.1
**Type**：`feat(server)`
**依赖**：issue 08
**预计 PR 大小**：S-M

---

## 背景

依赖规则的红线之一：**`app/` 不直接调内核——必须走 `server/`**（[依赖规则 § 3](../../../docs/architecture/dependency-rules.md#3-红线违反必拒)）。所以 CLI 不能直接 import `chat/`，要通过 HTTP 跟一个本地 server 说话。

0.1 的 server 只暴露 chat 相关 API；task / trace viewer 等 endpoint 留给后续。

参考：[架构总览 § 2.2](../../../docs/architecture/overview.md#22-交互形态分阶段)、[依赖规则](../../../docs/architecture/dependency-rules.md)。

## 范围

### 1. `src/sigma/server/app.py`

- 框架：FastAPI
- 启动入口：`sigma serve` 命令（CLI 加进 issue 10）
- 默认 host/port：`127.0.0.1:7777`（issue 01 的 config 已声明）
- 启动时读 `config.yaml`、构造 `LLMPort` / tools list / `TracerPort` / `CheckpointerPort` / `MasterAgent` / `ChatEngine`，挂到 app state

### 2. Endpoints

| Method | Path | Body / Query | Response |
|---|---|---|---|
| `POST` | `/chat/sessions` | `{}` | `{"id": "...", "created_at": "..."}` |
| `GET` | `/chat/sessions` | `?limit=20` | `{"sessions": [...]}` |
| `GET` | `/chat/sessions/{id}` | — | `ChatSession` 元信息 |
| `POST` | `/chat/sessions/{id}/messages` | `{"content": "..."}` | **SSE** stream of `AgentChunk` |
| `GET` | `/healthz` | — | `{"ok": true}` |

### 3. SSE 协议

每个 `AgentChunk` 序列化为 SSE event：

```
event: chunk
data: {"type":"text_delta","content":"你好"}

event: chunk
data: {"type":"tool_call","tool_call":{"id":"call_1","name":"read_file","arguments":{"path":"README.md"}}}

event: chunk
data: {"type":"done"}
```

约定：

- `event: chunk` 是常规事件
- `event: error` 是异常（SSE 流终止）
- 每个 `data` 是合法 JSON
- client 收到 `done` 后主动断开

### 4. 错误处理

- session 不存在 → 404
- LLM / tool 异常 → SSE `event: error` + 关闭流
- 端口占用 / 启动失败 → CLI 友好报错（不要让 uvicorn 的 traceback 直接糊用户脸）

### 5. 不做认证

0.1 单人本地、监听 `127.0.0.1`，不引入 auth。Web UI / 多用户上来再加（V3+）。

## 设计要点

- **不要在 server/ 直接 import 域目录**——LLM / tools / tracer / checkpointer 都通过 registry 工厂实例化（[依赖规则 § 4](../../../docs/architecture/dependency-rules.md#4-常见违规及修复)）
- **SSE 用 `EventSourceResponse`（sse-starlette）或 FastAPI 原生 streaming**——避免自己维护 keep-alive。0.1 直接用原生 streaming + 手写 SSE 帧（不引入 sse-starlette，少一个依赖）
- **API 用 OpenAPI**：FastAPI 自动出 schema，方便 Web UI / 第三方 client 接（不主动跑 docs server，但 schema 文件能 dump）
- **`/chat/sessions/{id}/messages` 不返回完整 history**——history 是 LangGraph checkpoint 的事，client 自己维护本地显示状态。0.1 不实现 "GET messages" endpoint
- **跨进程：CLI ↔ server**：CLI 启动时检测 server 是否在跑，没跑就 spawn 一个后台 server（或提示用户先 `sigma serve`）。具体策略见 issue 10

## 验收标准

- [ ] `tests/server/test_chat_api.py`（用 `httpx.AsyncClient` + `ASGITransport`）：
  - `POST /chat/sessions` 创建并返回 id
  - `POST /chat/sessions/{id}/messages` 返回 SSE 流，能解析出多条 chunk + 末尾 `done`
  - 不存在 session → 404
  - mock agent 抛异常 → SSE 出 `error` event 后断开
- [ ] OpenAPI 能 dump：`python -c "import json; from sigma.server.app import app; json.dumps(app.openapi())"` 不报错
- [ ] `grep -rE "^from sigma\.(llm|tools|trace|checkpoint)\.\w+\." src/sigma/server/` 必须为空（只允许 import 内核模块、ports、core；具体 adapter 走 registry）

## 不做

- 不做 auth（V3+）
- 不做 task / trace / memory / rag 等 endpoint（对应 milestone 加）
- 不做 WebSocket（Realtime 才需要，V4）
- 不做 CORS / 跨域（Web UI 阶段再加）
- 不做 prompt-level 速率限制（V2）

## 相关文档

- [架构总览 § 2.2](../../../docs/architecture/overview.md#22-交互形态分阶段)
- [依赖规则](../../../docs/architecture/dependency-rules.md)
