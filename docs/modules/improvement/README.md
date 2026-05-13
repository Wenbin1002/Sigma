# Improvement 模块

> Self-improvement 的实现层。
>
> 这是 [Self-improvement](../../features/self-improvement.md) 的实现细节文档。

---

## 1. 模块职责

从用户互动中提取 signal，转换为 memory / agent 行为调整。两个维度：

```
学用户（原有）                               学知识（D-24 新增）
─────────────                               ─────────────
Signal sources        Extraction            Signal sources           Extraction
─────────────         ─────────             ─────────────            ─────────
显式用户声明   ─┐                            交互中的深度分析  ─┐
重复/转话题等  ─┼→ Extractors → Memory      session 对话历史  ─┼→ Extractors → Domain Memory
Audio metadata ─┤    (global)               原始材料变更     ─┤    (domain-scoped)
Task 反馈      ─┘                                           ─┘
                └→ Agent meta                                └→ 矛盾/过时检测
                └→ 推送过滤
```

---

## 2. 内部结构

```
src/improvement/
  base.py               # Signal 数据结构
  registry.py           # extractor / router 注册
  
  signals/              # Signal 定义
    types.py            #   显式偏好 / 隐式行为 / audio / task 反馈 / 领域认知
  
  extractors/
    explicit.py         #   从对话中提取显式声明
    pattern.py          #   从行为模式中提取（重复 / 转话题）
    audio.py            #   从 realtime audio metadata 提取（V4）
    task.py             #   从 task 完成 / 取消中提取
    knowledge.py        #   从交互中提取领域认知（D-24 新增）
  
  pipeline.py           # Signal → 反馈路由
  apply/
    memory_writer.py    # 写入 Memory（global + domain）
    agent_tuner.py      # 调整 Agent metadata
    filter_updater.py   # 推送过滤更新
  
  lint/                 # Domain memory 自检（D-23 Lint 循环）
    consistency.py      #   矛盾检测
    staleness.py        #   过时检测（原始材料变更后标记受影响结论）
    cleanup.py          #   低质量条目清理
  
  audit.py              # 审计 / 可解释性 / 用户可见
```

---

## 3. Signal 数据结构

```python
@dataclass
class Signal:
    type: SignalType                    # 见 § 4
    source: SignalSource                # chat / realtime / task / explicit
    confidence: float                   # 0.0-1.0
    payload: dict                       # type-specific
    extracted_from: str                 # session_id / task_id / message_id
    extracted_at: datetime
```

---

## 4. SignalType 清单

| Type | 来源 | 反馈 |
|---|---|---|
| `preference_explicit` | 用户明说 ("用更专业的术语") | Memory（高置信度直接写） |
| `correction` | 用户纠正 ("我是金融业不是 IT") | Memory（覆盖旧条目） |
| `dissatisfaction_repeat` | 用户重复问同一问题 | Agent metadata：当前 prompt 不够 |
| `topic_avoidance` | 用户多次跳过某话题 | Memory：兴趣过滤 |
| `interrupt` | Realtime：用户打断模型输出 | Agent metadata：调短 |
| `redo_request` | Realtime/chat："再来一次" | Agent metadata：上次回答质量低 |
| `silence_long` | Realtime：长时间沉默 | Audio signal: 思考 / 走神 |
| `engagement_high` | Realtime：笑声 / 主动延续 | 强化：当前互动模式有效 |
| `task_cancel` | 用户取消 task | 从 task description 学：不该这样理解 |
| `task_modify` | 用户多次改 task description | 第一次 prompt 没理解清楚 |
| `feedback_positive` / `_negative` | 用户对 artifact 反馈 | Artifact 质量 signal |

---

## 5. 提取流程

### 5.1 显式 signal（同步）

```python
# 在 Chat / Realtime 收到每条 user message 后
async def on_user_message(msg: Message, session: Session):
    signals = await explicit_extractor.extract(msg, session.context)
    for s in signals:
        if s.confidence > THRESHOLD_HIGH:
            await pipeline.apply(s)
```

### 5.2 隐式 signal（批处理）

```python
# 每次 session 结束 / 每隔 N 轮触发
async def extract_implicit(session: Session):
    pattern_signals = await pattern_extractor.extract(session)
    audio_signals = await audio_extractor.extract(session) if session.is_realtime else []
    
    all_signals = pattern_signals + audio_signals
    for s in all_signals:
        await pipeline.queue(s)   # 异步处理
```

---

## 6. Pipeline & Apply

```python
async def apply(signal: Signal):
    # 1. Audit 记录（必须，可解释）
    await audit.record(signal)
    
    # 2. 分类路由
    if signal.type in MEMORY_TARGETS:
        await memory_writer.write(signal)
    
    if signal.type in AGENT_TUNING_TARGETS:
        await agent_tuner.adjust(signal)
    
    if signal.type in FILTER_TARGETS:
        await filter_updater.update(signal)
    
    # 3. Trace
    await tracer.record_event(SignalAppliedEvent(signal=signal))
```

**关键**：每个 apply 都进 trace 和 audit，**永远可回溯**。

---

## 7. 用户审计接口

CLI / Web UI 提供：

```bash
# 看 self-improvement 学到了什么
sigma improvement list                    # 最近写入的 memory / 调整
sigma improvement why <memory_id>         # 这条 memory 是因为哪个 signal 写入的
sigma improvement undo <signal_id>        # 撤销某次 signal 应用
sigma improvement disable <category>      # 关闭某类 signal 提取
sigma improvement enable <category>
```

---

## 8. 实现路径

详见 [Self-improvement § 8](../../architecture/self-improvement.md#8-实现路径v2-v5)：

- V2：MVP（显式 + 简单 pipeline + 用户可见）
- V3：隐式 signal + 周期 task 反馈
- V4：Realtime audio signal
- V5：跨 session 学习

---

## 9. 跟其他模块关系

| 模块 | 关系 |
|---|---|
| [Memory](../memory/) | 主要写入目标 |
| [Realtime](../realtime/) | 主要 signal 来源（V4 起） |
| [Trace](../trace/) | 所有 signal 进 trace |
| [Agent](../agent/) | Agent metadata 可被调整 |
| [Task](../task/) | Task 反馈是 signal 来源；周期 task 是受益者 |
| [Chat](../chat/) | Chat 内的隐式 signal 提取 |

---

## 10. 未决问题

详见 [Self-improvement § 10](../../architecture/self-improvement.md#10-未决问题)。

---

## 11. 相关文档

- [Self-improvement 设计](../../architecture/self-improvement.md)
- [Memory 模块](../memory/)
- [Realtime 模块](../realtime/)
- [Trace 模块](../trace/)
