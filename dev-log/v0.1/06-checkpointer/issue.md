# Issue 06 — Checkpointer（LangGraph SqliteSaver）

**Milestone**：v0.1
**Type**：`feat(checkpoint)`
**依赖**：issue 02
**预计 PR 大小**：S

---

## 背景

0.1 退出标准之一：「中断恢复：kill 进程后 `sigma chat --resume <session>` 能续上」。靠的是 LangGraph 自带的 checkpoint 机制——把 graph state 落到 SQLite，重启时按 thread_id 加载续跑。

参考：[Ports & Adapters § 3.4](../../../docs/architecture/ports-and-adapters.md#34-checkpointerport)、[Chat 模块 § 3](../../../docs/modules/chat/README.md#3-session-管理)。

## 范围

### 1. `src/sigma/checkpoint/registry.py`

```python
REGISTRY: dict[str, Callable[[CheckpointConfig], BaseCheckpointSaver]] = {
    "sqlite": _create_sqlite,
}

def create(config) -> BaseCheckpointSaver: ...
```

### 2. `src/sigma/checkpoint/sqlite.py`

最薄的 wrapper：

- import `langgraph.checkpoint.sqlite.SqliteSaver`
- 解析 `~/...` 展开 + 创建父目录
- 处理同步 / 异步：LangGraph 0.2+ 提供 `AsyncSqliteSaver`——用异步版（agent 在 async 上下文跑）
- 工厂：

```python
def _create_sqlite(config) -> AsyncSqliteSaver:
    db_path = Path(config.db_path).expanduser()
    db_path.parent.mkdir(parents=True, exist_ok=True)
    return AsyncSqliteSaver.from_conn_string(str(db_path))
```

### 3. 配置

匹配 issue 01：

```yaml
checkpoint:
  provider: sqlite
  db_path: ~/.sigma/checkpoint.db
```

### 4. session ↔ thread_id 约定

LangGraph checkpoint 用 `thread_id` 作为 partition key。本项目约定：

- **chat session_id 直接用作 LangGraph thread_id**（一一对应）
- 写在本 issue 的 README 注释里，issue 08 的 chat engine 复用这个约定

## 设计要点

- **不抽象 SQLite 之上的"我们自己的 schema"**——直接用 LangGraph 原生表结构，省一层维护
- **不实现 PostgresSaver / RedisSaver**——LangGraph 生态已经有，用户想用直接换 provider 字符串就行（registry 留扩展点）
- **不在 0.1 做 checkpoint 清理 / 过期**：单人本地数据库，跑半年再清也不晚

## 验收标准

- [ ] `tests/checkpoint/test_sqlite.py`：
  - 工厂能在临时目录创建 db 文件
  - 写入一条 checkpoint 后，重新打开 saver 能读出来
  - 不存在的目录会自动创建（`~` 展开正确）
- [ ] `grep -rE "^from sigma\.(llm|tools|trace|agent)" src/sigma/checkpoint/` 必须为空

## 不做

- 不实现 PostgresSaver / RedisSaver（用户自己接，registry 留扩展点）
- 不实现 checkpoint 清理 / TTL（V2+）
- 不实现自定义 schema 演进（用 LangGraph 自带的就够）
- 不接 chat session 持久化的业务逻辑（issue 08 做）

## 相关文档

- [Ports & Adapters § 3.4](../../../docs/architecture/ports-and-adapters.md#34-checkpointerport)
- [Chat 模块](../../../docs/modules/chat/README.md)
