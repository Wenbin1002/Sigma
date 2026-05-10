# Chat 模块

> Chat 是 Sigma 的**对话流模式**——默认入口，覆盖日常问答和短任务。

> 详见 [Chat / Task 模式](../../architecture/chat-task-modes.md)。

---

## 1. 定位

| | Chat | Task |
|---|---|---|
| 角色 | 默认入口 | 显式后台任务 |
| 节奏 | 实时往返 | 后台跑，可后看 |
| 场景 | 问答、短任务、工具调用、讨论 | 长任务、周期任务 |
| 状态 | session 维护 | task 维护 |
| 升级路径 | → 可建议升级到 task | ← 完成后回流通知 |

---

## 2. Chat 引擎职责

1. 维护 session（多轮上下文、用户身份、对话历史）
2. 调用 Master Agent
3. 流式返回 `AgentChunk` 给 client（CLI / Web）
4. 检测"长任务信号" → 主动建议升级为 task
5. 接收 task 完成回流，注入对话流

---

## 3. Session 管理

```python
@dataclass
class ChatSession:
    id: str
    user_id: str
    messages: list[Message]
    created_at: datetime
    updated_at: datetime
    
    # 与 task 的关联
    spawned_task_ids: list[str]    # 这个 session 起过哪些 task
```

Session 持久化（基于 LangGraph checkpointer），重启可 resume：

```bash
sigma chat                          # 起新 session
sigma chat --resume <session_id>    # 续之前的
sigma chat --list                   # 看历史 session
```

---

## 4. "升级为 Task" 的触发

Master Agent 在每轮回复前判断：

```
判断输入：
- 当前 user message + 对话历史
- 预估执行复杂度（token / 工具调用次数 / 时长）

输出：
- 如果应该升级 → 在回复里附 "task suggestion" 卡片
  {
    "type": "task_suggestion",
    "description": "...",
    "estimated_duration": "5-10 min",
    "estimated_cost": "$0.10",
  }
- 用户确认 → 创建 task，chat 历史作为 context_seed
- 用户拒绝 → master agent 在 chat 里直接执行（即使要花一阵）
```

**判断规则不写死**——通过 LLM 自己评估。可在 system prompt 里给指南（"以下情况建议升级：1. 多文件操作 2. 多次外部 API 3. 时长 > 5 分钟..."）。

---

## 5. Task 完成回流

Task 跑完时，把结果回流到原 chat session：

```
Task #N 完成
  ↓
Sigma 在原 session 注入一条系统消息：
{
  "type": "task_completed",
  "task_id": "N",
  "summary": "...",
  "artifacts": [...],
}
  ↓
CLI / Web UI 渲染成卡片
  ↓
用户可在 chat 里追问：「Task #N 的图表第三个我想再细化一下」
  ↓
Master Agent 能引用 task artifact 继续对话
```

---

## 6. CLI 映射（Phase 1）

```bash
sigma chat                          # 进入对话流（多轮）
sigma chat "一句话问题"             # 单次问答
sigma chat --resume <session_id>    # 续 session
sigma chat --list                   # 看 session 列表
sigma chat --new                    # 强制起新 session
```

---

## 7. 实现位置

```
src/chat/
  session.py           # session 数据结构 + 持久化
  engine.py            # Chat 主循环：维护 session / 调 agent / 流式输出
  upgrade.py           # 升级为 task 的判断 + 卡片生成
  reflow.py            # task 完成回流注入
```

---

## 8. 相关文档

- [Chat / Task 模式](../../architecture/chat-task-modes.md) — 双视图整体设计
- [Task 模块](../task/) — Task 引擎
- [Agent 模块](../agent/) — Master Agent
- [设计决策日志](../../architecture/design-log.md)
