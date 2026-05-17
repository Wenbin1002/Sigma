# Issue 08 — Chat engine + session

**Milestone**：v0.1
**Type**：`feat(chat)`
**依赖**：issue 07
**预计 PR 大小**：S-M

---

## 背景

Master Agent 是"一次推理 + 工具循环"的能力；Chat engine 是"多轮 session 的容器"。它持有 session 列表、把用户消息转给 agent、把 agent 输出转给上层（server）。

0.1 退出标准的"多轮对话"和"中断恢复"都落在这一层。

参考：[Chat 模块](../../../docs/modules/chat/README.md)。

## 范围

### 1. `src/sigma/chat/session.py`

```python
@dataclass
class ChatSession:
    id: str                       # 也是 LangGraph thread_id（issue 06 约定）
    title: str | None             # 可选，从首条消息提取
    created_at: datetime
    updated_at: datetime
```

session 的"消息历史"实际由 LangGraph checkpoint 持有（按 thread_id 加载），本 dataclass 只存元信息——避免存两份。

### 2. `src/sigma/chat/store.py`

最简的 session 元信息存储：

- 复用 issue 06 的 sqlite db（同一个文件，不同表）
- 表：`sessions(id TEXT PRIMARY KEY, title TEXT, created_at TEXT, updated_at TEXT)`
- 操作：`create / get / list / update_title / touch`
- 不做 user 隔离（单人本地，user_id 暂不引入）

### 3. `src/sigma/chat/engine.py`

```python
class ChatEngine:
    def __init__(self, agent: MasterAgent, store: SessionStore): ...

    async def new_session(self) -> ChatSession: ...
    async def list_sessions(self, limit: int = 20) -> list[ChatSession]: ...
    async def get_session(self, sid: str) -> ChatSession | None: ...

    async def send(self, sid: str, user_message: str) -> AsyncIterator[AgentChunk]:
        """主流程：loaded session → 调 agent.stream → 流出 chunk → 更新元信息"""
```

`send()` 流程：

1. 校验 session 存在（不存在则 404 让 server 返回）
2. 调 `agent.stream(session_id=sid, user_message=user_message)` 拿 chunk
3. 边消费边 yield 给上层
4. 流结束后 `store.touch(sid)` 更新 `updated_at`
5. 第一条用户消息时，用启发式从 message 截前 30 字作为 title 写回

### 4. CLI 命令面对应

issue 10 的 CLI 直接调本 engine 接口（实际通过 server，但接口形状要先固定）：

| CLI | engine method |
|---|---|
| `sigma chat`（不带参数） | `new_session` + 进入 REPL，每行调 `send` |
| `sigma chat "Q"` | `new_session` + `send("Q")` 一次 |
| `sigma chat --resume <id>` | `get_session(id)` + REPL，每行 `send` |
| `sigma chat --list` | `list_sessions` |
| `sigma chat --new` | 等价于不带参数 |

### 5. 中断恢复语义

- kill 进程 → LangGraph checkpoint 已经按 node 写入 → 下次 `agent.stream(sid, ...)` 自动从最后一个 checkpoint 续
- session 元信息表里只记 id / 时间，不记"上次中断时停在哪"——这部分是 LangGraph 的事

## 设计要点

- **不要在 chat/ 复刻 LangGraph 已经做的事**：消息历史、checkpoint 写入、状态恢复，都让 LangGraph 干。`chat/` 只管 session 的"账本"
- **session_id == thread_id**（[issue 06](../06-checkpointer/issue.md) 已约定）
- **不做"升级 task" 提示**（0.2 加 task 引擎时再做）
- **不做"task 完成回流"**（0.2）
- **不做 user_id**（0.1 单人本地）

## 验收标准

- [ ] `tests/chat/test_engine.py`（mock agent）：
  - `new_session` → 写元信息 → `list_sessions` 能查到
  - `send` 流式 yield 全部 chunk + `updated_at` 被刷新
  - 不存在的 sid 时 `get_session` 返回 None（不 raise）
- [ ] 集成测试：连续两次 `send` 同 session，第二次能引用第一次的上下文
- [ ] 集成测试：第一次 `send` 中途模拟 kill（asyncio task cancel），重启后 `send("继续")` 能基于已 checkpoint 的状态继续
- [ ] `grep -rE "^from sigma\.(llm|tools|trace|checkpoint)\.\w+\." src/sigma/chat/` 必须为空（具体 adapter 不直引；checkpointer 通过 agent 间接持有）

## 不做

- 不做 task 升级建议（0.2）
- 不做 task 完成回流（0.2）
- 不做 user_id / 多用户（V3+）
- 不做 session 删除 / 归档（V2）
- 不做 session 标题自动提炼（用截断启发式即可）

## 相关文档

- [Chat 模块](../../../docs/modules/chat/README.md)
- [Chat / Task 模式](../../../docs/features/chat-task-modes.md)
