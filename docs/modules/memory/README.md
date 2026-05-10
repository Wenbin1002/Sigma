# Memory 模块

> Sigma 的记忆系统——**三层分层**：global（用户偏好）/ session（对话上下文）/ task（任务临时 state）。

---

## 1. 定位

Memory 是 Sigma 的**核心差异化能力之一**——决定 agent 是不是真的"懂用户"。

跟 RAG 的区别：

| | Memory | RAG |
|---|---|---|
| 数据来源 | 系统自动从对话提取 | 用户提供文档 |
| 内容性质 | 用户偏好 / 决策 / 状态 | 事实 / 知识 |
| 时间维度 | 时间序列，新覆盖旧 | 静态，标注源时间 |
| 更新方式 | 隐式（每次对话） | 显式（用户运行 `rag update`） |

详见 [RAG 模块](../rag/)。

---

## 2. 三层模型

```
┌──────────────────────────────────────────┐
│  Global Memory                            │  跨所有 session / task 共享
│  - 用户偏好（"喜欢简洁回答"）              │  长期，慢更新
│  - 个人特征（"金融行业"、"中文为主"）      │
│  - 历史决策模式                            │
│  生命周期：永久（除非用户删除）              │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  Session Memory                           │  单个 chat session 内
│  - 对话历史 / 已讨论话题                   │  中期，每轮更新
│  - 当前关心的 topic                        │
│  - Session 内的 artifact 引用              │
│  生命周期：session 关闭后归档（不丢，可查询）│
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  Task Memory                              │  单个 task 实例内
│  - 当前 task 的中间产物                    │  短期，task 结束清理
│  - sub-agent 之间共享的 state              │
│  - 周期性 task 跨实例的累积（特殊：长期）   │
│  生命周期：task 结束清理（除非显式持久化）   │
└──────────────────────────────────────────┘
```

---

## 3. Global Memory

### 3.1 来源

- **显式**：用户主动告诉（"以后用中文回答"）
- **隐式**：从对话中自动提取（"用户提到自己在金融行业"）

### 3.2 提取策略（自动）

每次 session 结束后跑一次 extraction：

```
LLM 看 session 摘要 + 已有 global memory
→ 输出："这个 session 哪些信息值得 promote 到 global memory"
→ 写入 global memory
```

提取要保守（误存比漏存代价高——错误 memory 会污染所有未来对话）。

### 3.3 召回

每次 chat / task 启动时：
- 全量加载（global memory 应保持小，1k-5k token 量级）
- 或语义召回（如果 global memory 增长太大）

---

## 4. Session Memory

Session 数据：

```python
@dataclass
class SessionMemory:
    session_id: str
    user_id: str
    
    messages: list[Message]
    topics_discussed: list[str]      # 已讨论话题
    artifacts: list[Artifact]        # session 内产生的 artifact
    pending_decisions: list[...]     # 等用户回复的决策
    
    # Realtime session 特有（V4 起）
    is_realtime: bool = False
    transcript: list[TranscriptEntry] | None = None
    audio_signals: list[AudioSignal] | None = None
    
    created_at: datetime
    last_active_at: datetime
```

Session 持久化到本地（SQLite），关闭后归档但不删——用户可以查历史 session、续接对话。

### 4.1 Realtime Session 的特殊处理

Realtime session 是 session 的一种特殊形态，跟普通 chat session 共用同一个 SessionMemory schema，但有几个差异：

| 维度 | Chat Session | Realtime Session |
|---|---|---|
| 内容形式 | 文本 messages | Transcript（含时间戳）+ audio metadata |
| 提取频率 | session 结束后批处理 | session 结束后批处理 + 中途轻量提取 |
| 提取信号源 | 文本内容 | 文本 + 打断 / 沉默 / 笑声等 audio signal |
| 提取 prompt | 文本对话 prompt | 含 audio 元数据的 prompt（见下） |

**Realtime 的 memory 提取 prompt 跟文本不同**——要让 LLM 看 transcript + audio metadata 一起：

```
对话 transcript：...
Audio metadata：
  - 用户在 "<某句话>" 处打断了你
  - "<某段>" 后用户沉默了 8 秒
  - "<某句>" 后用户笑了

请提取值得记入长期记忆的信息（含偏好 / 兴趣 / 互动模式）。
```

详见 [Realtime 模块](../realtime/)、[Self-improvement](../../architecture/self-improvement.md)。

---

## 5. Task Memory

Task 内的临时 state：

```python
@dataclass
class TaskMemory:
    task_id: str
    
    intermediate_artifacts: list[Artifact]
    sub_agent_handoffs: list[...]    # sub-agent 之间传递的数据
    proxy_audits: list[ProxyAudit]   # L2 代答记录
    user_inputs: list[...]           # L3 用户回复记录
```

Task 完成后默认清理；周期性 task 的 task memory 可显式声明跨实例持久化（场景 1 用户偏好积累）。

---

## 6. 召回策略

每次 LLM 调用前由 Context Engine 决定：

```
recall(query, scope) → list[MemoryItem]
  scope: "global" | "session" | "task" | "all"
```

**主动召回**（每次 LLM 调用前都查）：
- Global：相关偏好（"用户喜欢什么风格"）
- Session：相关话题（"刚才聊了什么"）

**懒召回**（agent 通过 tool 查）：
- 复杂场景：agent 自己决定要不要查 memory（通过 `recall_memory(query, scope)` tool）

---

## 7. Memory 在 Sub-agent 中

`AgentContext.memory_scope` 决定 sub-agent 能看哪一层：

```python
class ResearcherAgent(sigma.Agent):
    memory_scope = MemoryScope(
        global_=True,        # 看用户偏好
        session=True,        # 看 chat 上下文
        task=True,           # 看当前 task
    )
```

部分 sub-agent 可能限制只看部分 layer（隔离）。

---

## 8. 写入策略

| 时机 | 写入哪一层 |
|---|---|
| 每条 user message | Session（自动） |
| Session 结束 | Global（提取后写入） |
| Task 关键节点 | Task（自动） |
| 用户显式 `/remember ...` | Global |
| Sub-agent 输出 artifact | Task |

---

## 9. Memory 提取的可控性

风险：自动提取可能写入错误 memory，污染未来对话。

应对：
- **提取保守**：宁可漏，不写错
- **可审计**：每个 global memory entry 记录来源（哪个 session / 哪条对话提取的）
- **用户可见**：CLI / Web UI 提供 `sigma memory list/edit/delete`
- **置信度标注**：低置信的 memory 标记，召回时降权

---

## 10. 实现位置

```
src/memory/
  manager.py          # 多层管理器
  global_.py          # Global memory（持久化 + 召回）
  session.py          # Session memory
  task.py             # Task memory
  extract.py          # 自动提取策略（LLM 驱动）
  recall.py           # 召回策略
  
  stores/
    sqlite.py         # 默认 backend
```

---

## 11. 未决问题

| 问题 | 状态 |
|---|---|
| Global memory 的 schema（是否结构化 / 自由文本） | V3 落地时定 |
| 提取频率（每 session / 每 N 轮 / 后台批处理） | V3 |
| 是否需要 MemoryPort（即多 memory 实现是否真有需求） | 未决——V3 落地后看 |
| Memory 的"过期"策略（旧偏好怎么淘汰） | V4+ |

---

## 12. 相关文档

- [Context 模块](../context/) — Memory 召回的消费者
- [RAG 模块](../rag/) — 兄弟模块
- [架构总览](../../architecture/overview.md)
- [设计决策日志 § 4.3](../../architecture/design-log.md)
