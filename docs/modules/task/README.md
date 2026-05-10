# Task 模块

> Task 是 Sigma 的**后台任务执行系统**——Chat 之外的另一个一等公民模式。

> 详见 [Chat / Task 模式](../../architecture/chat-task-modes.md)。

---

## 1. 定位

Task 的核心特点：

- **后台执行**：用户提交后不阻塞，可以继续 chat / 关闭终端
- **持久化**：进程重启不丢，可 pause / resume
- **状态可见**：进度 / log / artifact 都有结构化记录
- **支持周期性**：cron 风格调度（每天 / 每周 / 自定义）

---

## 2. Task 的两条来源

### 2.1 Chat 升级

Master Agent 判断"这事是长任务" → 建议用户升级为 task → 用户确认 → Task 创建。

Chat 历史作为 `context_seed` 一起带进 task。

### 2.2 用户显式新建

Task 视图 / CLI 直接 `sigma task new "..."`。无 chat 历史，但可指定 schedule / target_agent。

---

## 3. 状态机

```
              ┌────────────┐
              │   queued    │
              └──────┬──────┘
                     │
                     ▼
              ┌────────────┐
              │   running   │
              └──┬───┬───┬──┘
        completed │   │   │ paused
                  ▼   ▼   ▼
            ┌──────────┐  ┌──────────┐
            │ completed │  │  paused  │ (等用户 resume)
            └──────────┘  └────┬─────┘
                               │
                               ▼ user_input
                          (回到 running)

                               failed / cancelled (终态)
```

详见 [Chat / Task 模式 § 3](../../architecture/chat-task-modes.md#3-task-状态机)。

---

## 4. Pause / Resume：multi-agent 的副产物

**Pause 的唯一来源是 multi-agent 三级回退的 L3**——sub-agent 卡住 + 主 agent 也代答不了 + 必须等用户输入。

详见 [Multi-agent § 4](../../architecture/multi-agent.md#4-三级回退sub-agent-沟通问题)。

实现基础：LangGraph 的 `interrupt()` + `Command(resume=...)`。

```python
# Sub-agent 内部
if needs_user_decision:
    raise BlockedException(
        reason="ambiguous_intent",
        detail="用 A 风格还是 B 风格?",
        needed_input=NeededInput(type="choice", options=["A", "B"]),
    )

# 主 agent 处理（伪代码）
try:
    result = await sub_agent.run(...)
except BlockedException as e:
    if e.reason in RESOURCE_REASONS:
        # 直接 L3
        await escalate_to_user_pause_task(e)
    else:
        answer = await master_proxy_decide(e)
        if answer == NEED_USER:
            await escalate_to_user_pause_task(e)
        else:
            # L2 代答成功
            sub_agent.resume_with(answer)
```

---

## 5. Task 数据结构

```python
@dataclass
class Task:
    id: str
    user_id: str
    description: str
    status: TaskStatus
    schedule: OneShot | Daily | Cron
    target_agent: str               # 默认 "master"，可指定 sub-agent
    
    # Context
    context_seed: ChatContext | None  # chat 升级时填
    parent_chat_id: str | None
    
    # Runtime
    checkpoint_id: str               # LangGraph thread_id
    artifacts: list[Artifact]
    progress: str
    log: list[LogEntry]
    
    # Pause 相关
    pending_question: str | None
    needed_input: NeededInput | None
    proxy_audits: list[ProxyAudit]   # L2 代答审计
    
    # Lifecycle
    created_at: datetime
    updated_at: datetime
    completed_at: datetime | None
    error: str | None
```

---

## 6. Task Queue

简单实现：基于 SQLite 的持久化队列。

- **入队**：`TaskQueue.enqueue(task)`
- **调度**：后台 worker 轮询 / cron 触发
- **并发**：默认串行执行（避免成本失控），可配置并发上限

V0/V1 阶段不做分布式，单进程串行就够。

---

## 7. 周期性 Task（V4）

```python
# CLI
sigma task new "..." --daily 09:00
sigma task new "..." --cron "0 9 * * 1-5"

# Schedule 类型
class OneShot:
    pass

@dataclass
class Daily:
    time: str   # "09:00"

@dataclass
class Cron:
    expression: str   # 标准 cron
```

调度器到时间 → 拉起一个新的 task 实例（每次都是独立执行，状态从干净开始）。

**周期性 task 的 memory 跨实例可见**——比如场景 1 推送，"用户上次点开了什么"要在所有实例间共享。这部分通过 [Memory 模块](../memory/) 的 task-scoped memory 实现。

---

## 8. CLI 映射（Phase 1）

```bash
sigma task new "..."                       # 一次性
sigma task new "..." --daily 09:00         # 周期
sigma task new "..." --cron "..."          # cron
sigma task new "..." --agent researcher    # 指定 sub-agent
sigma task list                            # 全部
sigma task list --status running           # 过滤
sigma task view <id>                       # 详情
sigma task pause <id>                      # 主动暂停
sigma task resume <id> --input "..."       # 恢复
sigma task cancel <id>                     # 取消
sigma task logs <id> -f                    # 实时跟随日志
```

---

## 9. 实现位置

```
src/task/
  engine.py            # 状态机、生命周期管理
  queue.py             # 队列 + worker
  scheduler.py         # 周期任务调度器（V4）
  state.py             # 状态持久化 (LangGraph checkpoint 包装)
  pause.py             # pause/resume 协议
```

---

## 10. 未决问题

| 问题 | 状态 |
|---|---|
| Task 并发模型（串行 / 受限并行 / 完全并行） | 未决（V1 默认串行） |
| 周期 task 错过执行时间的补偿策略 | 未决（V4） |
| Task 失败的自动重试策略 | 未决 |
| Task 之间能否依赖（A 完成才跑 B） | 未决（暂不做） |

---

## 11. 相关文档

- [Chat / Task 模式](../../architecture/chat-task-modes.md) — 状态机、双视图交互
- [Multi-agent § 4](../../architecture/multi-agent.md#4-三级回退sub-agent-沟通问题) — Pause 的来源
- [Trace 模块](../trace/) — Task 执行 trace
- [设计决策日志](../../architecture/design-log.md)
