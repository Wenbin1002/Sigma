# Realtime 模块

> Sigma 的 Realtime 实现层——Provider adapter + Middleware + Audio 流处理。
>
> 这是 [Realtime 模式](../../architecture/realtime-mode.md) 的实现细节文档。

---

## 1. 模块职责

```
src/realtime/
  ├── 接入 Realtime Provider（OpenAI / Gemini 等）
  ├── 维护 WebSocket 长连接
  ├── 在 provider 和用户之间插入 Sigma middleware
  │   ├── 会话前注入（system prompt 含 memory + RAG）
  │   ├── 会话中拦截（tool calls / signal extraction）
  │   └── 会话后沉淀（transcript → memory）
  └── 暴露 CLI / API 给 client
```

---

## 2. 内部结构

```
src/realtime/
  base.py               # RealtimeSession 数据结构
  registry.py           # provider registry
  
  providers/
    openai.py           # OpenAI Realtime API
    gemini.py           # Gemini Live (V5)
  
  middleware/
    inject.py           # 会话前注入（memory + RAG → system prompt）
    intercept.py        # 会话中拦截（tool calls / signals）
    persist.py          # 会话后沉淀（transcript / memory extraction）
  
  audio/
    buffer.py           # 音频流 buffer
    barge_in.py         # 打断处理
    metadata.py         # 音频元数据提取（沉默时长 / 语速 / 笑声）
  
  session.py            # Session 生命周期管理
  cli.py                # `sigma realtime` 命令实现
```

---

## 3. 是否抽 RealtimePort？

**V4：不抽**。原因：
- 只有 OpenAI Realtime 一个 adapter
- Provider API 形态差异极大（OpenAI 的 client_secret + WebSocket vs Gemini Live 的 BiDi gRPC vs ...）
- 强行抽出来很可能形状不对

**V5：观察后决定**。如果接 Gemini Live 时发现接口可以收敛 → 抽 `RealtimePort`；否则保留两套独立 adapter。

详见 [Ports & Adapters § 1](../../architecture/ports-and-adapters.md#1-哪些能力是-port为什么) 的 Port 设计原则。

---

## 4. RealtimeSession 数据结构

```python
@dataclass
class RealtimeSession:
    id: str
    user_id: str
    provider: str                       # openai / gemini / ...
    model: str                          # e.g. gpt-4o-realtime-preview
    
    # 配置
    system_prompt: str                  # 会话前注入的（含 memory）
    available_tools: list[str]          # 可调用的轻量 tool 子集
    rag_indices: list[str]              # 可查的 RAG index
    voice_config: dict                  # 语速 / 音色 / 语气
    
    # 状态
    status: Literal["connecting", "active", "paused", "ended", "error"]
    started_at: datetime
    ended_at: datetime | None
    
    # 数据
    transcript: list[TranscriptEntry]   # 实时记录
    audio_signals: list[AudioSignal]    # 打断 / 沉默 / 笑声等
    tool_calls: list[ToolCall]          # 会话中调用的 tool
    artifacts: list[Artifact]           # 会话产生的产物（笔记 / 报告等）
```

---

## 5. Middleware 三层

### 5.1 会话前注入

```python
async def prepare_session(session: RealtimeSession, user: User) -> SessionConfig:
    # 1. 召回 memory
    global_mem = await memory.recall(scope="global", user_id=user.id)
    related_mem = await memory.recall(
        scope="all",
        user_id=user.id,
        query=session.intent_hint,  # 如果有的话
    )
    
    # 2. 可选 RAG（如果 session 配置了 rag_indices）
    rag_context = ""
    if session.rag_indices:
        rag_context = await rag.preload(session.rag_indices)
    
    # 3. 拼装 system prompt
    system_prompt = compose_system_prompt(
        base=BASE_REALTIME_PROMPT,
        memory=global_mem + related_mem,
        rag=rag_context,
        agent_persona=session.agent_persona,
    )
    
    # 4. 配置 tool 清单
    tools = [TOOL_DEFS[name] for name in session.available_tools]
    
    return SessionConfig(system_prompt=system_prompt, tools=tools, ...)
```

### 5.2 会话中拦截

```python
async def on_event(event: RealtimeEvent):
    if event.type == "tool_call":
        # 拦截 tool call，调 Sigma 的 tool 系统执行
        result = await tools.execute(event.tool_name, event.args)
        # 注入回 session
        await session.inject_tool_result(event.id, result)
    
    elif event.type == "transcript_delta":
        session.transcript.append(event.text)
    
    elif event.type == "audio_signal":
        # 打断 / 沉默 / 语速变化等
        signal = parse_audio_signal(event)
        session.audio_signals.append(signal)
        # 可能立即触发某些反应（如长时间沉默）
```

### 5.3 会话后沉淀

```python
async def finalize_session(session: RealtimeSession):
    # 1. Transcript 持久化
    await store.save_session(session)
    
    # 2. Memory 提取（异步）
    asyncio.create_task(extract_memories(session))
    
    # 3. Self-improvement signals（异步）
    asyncio.create_task(process_signals(session))
    
    # 4. 可选：生成会话报告
    if session.config.generate_report:
        report = await llm.generate_session_report(session)
        await store.save_artifact(session.id, report)
```

---

## 6. Audio 元数据提取

Realtime API 通常提供这些 audio metadata（OpenAI Realtime 已有；Gemini 类似）：

| 信号 | 来源 | 提取方式 |
|---|---|---|
| 打断 | `input_audio_buffer.committed` 在 model 输出期间 | 直接从事件流标记 |
| 沉默时长 | 用户停止说话到下一次说话的间隔 | 计算事件时间戳差 |
| 语速 | transcribed 字数 / 时长 | 简单 div |
| 笑声 / 叹息 | 部分 provider 提供（Gemini 更强） | 看 provider 支持 |

这些 signal 进入 [Self-improvement](../../architecture/self-improvement.md) 的 signal pipeline。

---

## 7. 跟其他模块的接口

```
Realtime
  ├── 启动时 ─→ Memory.recall（召回 global + session memory）
  ├── 启动时 ─→ RAG.preload（如果配置了 index）
  ├── 会话中 ─→ Tools.execute（拦截到 tool call）
  ├── 会话中 ─→ Memory.recall（user 显式问"我之前说过什么"）
  ├── 会话中 ─→ Task.create（用户起后台 task）
  ├── 会话后 ─→ Memory.extract（transcript 提取）
  ├── 会话后 ─→ Improvement.process（signal 处理）
  └── 全程 ──→ Trace.record（每个事件进 trace）
```

---

## 8. CLI 实现

```python
# src/realtime/cli.py
@cli.command("realtime")
def realtime_group():
    pass

@realtime_group.command("start")
@click.option("--agent", default=None)
@click.option("--rag", default=None, multiple=True)
@click.option("--provider", default=None)
def start(agent, rag, provider):
    """启动 realtime 会话。"""
    ...

@realtime_group.command("list")
def list_sessions():
    """列出历史 realtime session。"""
    ...
```

---

## 9. 性能要求

Realtime 模式延迟敏感。Sigma middleware 的延迟预算：

| 阶段 | 预算 | 实现要点 |
|---|---|---|
| 会话前注入 | < 1s | 可阻塞，会话还没开始 |
| 会话中 tool 执行 | **< 200ms** | 不能阻塞实时对话；超时降级 |
| Memory recall（同步路径） | < 50ms | 必须有缓存 |
| RAG query（同步路径） | < 100ms | 限制 top_k；预热 index |
| 会话后沉淀 | 后台异步 | 不影响下一次会话启动 |

**超时降级策略**：tool 调用超过 200ms → 返回"现在查不到，稍后我会补上"，会话后用 task 补查。

---

## 10. 已废弃的 Voice Pipeline

之前文档里 voice 模块支持 STT/TTS/VAD 的级联模式。**Realtime 模块取代了它的 V0 范围**：

- Voice 级联模式不在 Sigma 主线
- STT/TTS Port 暂不实现
- 如果将来真有用户场景需要 STT → LLM → TTS（比如电话客服），再单独评估

---

## 11. 未决问题

| 问题 | 状态 |
|---|---|
| 是否抽 RealtimePort | V5 接 Gemini 时定 |
| 多 provider 的 voice config schema 统一 | V5 |
| Realtime session 跟 Chat session 的切换体验 | V5+ |
| 用户 audio 隐私（本地存还是丢弃） | V4 默认开本地存，可关 |

---

## 12. 相关文档

- [Realtime 模式](../../architecture/realtime-mode.md) — 产品形态和场景
- [Self-improvement](../../architecture/self-improvement.md) — Realtime 的杀手伙伴
- [Memory 模块](../memory/) — Realtime 的核心依赖
- [Tools 模块](../tools/) — 会话中可调用的 tool
- [Trace 模块](../trace/) — Realtime trace
