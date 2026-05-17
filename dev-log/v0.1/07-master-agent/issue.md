# Issue 07 — Master Agent（单节点 graph）

**Milestone**：v0.1
**Type**：`feat(agent)`
**依赖**：issue 03 + 04 + 05 + 06
**预计 PR 大小**：M-L

---

## 背景

这是 0.1 的会合点：把 LLM / Tools / Trace / Checkpoint 串起来，用 LangGraph 实现"读消息 → 调 LLM → 必要时调 tool → 回写消息 → 循环"的最简 agent。

不引入 sub-agent / supervisor / @-mention / @-skill / 三级回退——这些都是 0.5。本 issue 的 graph **只有一个 reasoning node + 一个 tools node**（标准 ReAct 形态）。

参考：[Agent 模块](../../../docs/modules/agent/README.md)、[架构总览 § 3](../../../docs/architecture/overview.md#3-内核架构)。

## 范围

### 1. `src/sigma/agent/state.py`

LangGraph state（pydantic / TypedDict 都可，建议 `MessagesState` 子类）：

```python
class AgentState(TypedDict):
    messages: list[Message]      # 含 system / user / assistant / tool
    trace_id: str
    cumulative_usage: TokenUsage  # 简单累加，不做 cost guard 拦截
```

### 2. `src/sigma/agent/master.py`

`MasterAgent` 类：

```python
class MasterAgent:
    def __init__(self, llm: LLMPort, tools: list[ToolPort],
                 tracer: TracerPort, checkpointer: BaseCheckpointSaver,
                 system_prompt: str): ...

    def build_graph(self) -> CompiledStateGraph: ...
    async def stream(self, session_id: str, user_message: str) -> AsyncIterator[AgentChunk]: ...
```

#### Graph 结构

```
        ┌─────────┐
        │ reason  │  调 LLM；返回 assistant message
        └────┬────┘
             │ has_tool_calls?
        ┌────┴────┐
   yes  │         │  no
        ▼         ▼
   ┌────────┐  ┌──────┐
   │ tools  │──┤ END  │
   └───┬────┘  └──────┘
       │ append tool messages
       └──> reason   (回到 reason)
```

最简的 LangGraph 实现：用 `StateGraph` + `add_conditional_edges`。可以参考 LangGraph 官方 `prebuilt.create_react_agent`，但**自己实现一遍**——0.1 目的就是吃透内核，不直接用 prebuilt。

### 3. Trace 集成

在 `reason` / `tools` 两个 node 入口出口手动 emit trace event：

- `agent_start`（stream 入口） / `agent_end`（终止前）
- `llm_call`（reason node 内 LLM 调用前后包一层）
- `tool_call`（tools node 内每个 tool 执行前后）

每个 event 用同一 `trace_id`，按节点层级串 `parent_span_id`。

### 4. Streaming：LLM 流 → AgentChunk 流

`stream()` 返回 `AsyncIterator[AgentChunk]`：

- LLM 流式 `LLMChunk(type="text")` → `AgentChunk(type="text_delta")`
- LLM `LLMChunk(type="tool_call")` → `AgentChunk(type="tool_call")`
- tool 执行完成 → `AgentChunk(type="tool_result")`
- 全流程结束 → `AgentChunk(type="done")`
- 异常 → `AgentChunk(type="error")` 后终止

LangGraph 提供 `astream(..., stream_mode="messages")` 等模式；选一个能拿到 LLM 流式 chunk 的模式。

### 5. System prompt

0.1 用一个最简版（写在 `src/sigma/agent/prompts.py`）：

> 你是 Sigma，一个本地运行的通用 AI 助手。回答用户问题；需要时调用提供的工具。当前可用工具：{tool_list}。

不在 0.1 做：skill 注入 / memory 注入 / context engine。

### 6. 入口工厂

`src/sigma/agent/registry.py`（保持习惯，即使现在只有一个 agent）：

```python
def create_master(llm, tools, tracer, checkpointer, system_prompt) -> MasterAgent: ...
```

## 设计要点

- **graph 在构造时 compile + bind checkpointer**，每次 `stream()` 用同一个 compiled graph，只是不同 `thread_id`
- **不要让 agent 直接 import `llm.openai_compat`**——通过依赖注入拿 `LLMPort`（构造方传入）。同理 tools / tracer / checkpointer。这是依赖规则的硬性要求（[依赖规则 § 4](../../../docs/architecture/dependency-rules.md#4-常见违规及修复)）
- **`tools` 列表**：构造方按 config 过滤后传进来；agent 不感知 config
- **错误处理**：
  - tool 执行失败 → 把 `ToolResult(success=False, error=...)` 当成 tool message 喂回 LLM；让 LLM 决定是否重试
  - LLM 调用失败 → bubble 到 `stream()` 调用方，发 `AgentChunk(type="error")`
- **不要在 0.1 做**：
  - cost guard interrupt（0.5）
  - sub-agent spawn（0.5）
  - supervisor 路由（0.5）
  - skill 渐进式加载（0.5）
  - memory / RAG 注入（0.3 / 0.4）

## 验收标准

- [ ] `tests/agent/test_master.py`：
  - mock 一个 `LLMPort` 返回固定 chunk 序列；mock 一组 tool；驱动 `stream()` 流出预期 `AgentChunk` 序列
  - 工具调用 → 工具结果 → 二次 LLM 调用闭环正确
  - LLM 失败时输出 `error` chunk 并终止
- [ ] 集成测试（用真 LLM + 真 tool）：
  - 输入「读 README.md 第一行回答我」，agent 能：调 read_file、读到内容、给出回答
  - 同一 session_id 多轮对话，第二轮能引用第一轮上下文（验证 checkpointer 工作）
- [ ] 跑完后 `~/.sigma/traces/<trace_id>.jsonl` 至少包含 `agent_start` / `llm_call` / `tool_call` / `agent_end`
- [ ] `grep -rE "^from sigma\.(llm|tools|trace|checkpoint)\.\w+\." src/sigma/agent/` 必须为空（只 import `ports` 和 `core`，具体 adapter 不直引）

## 不做

- 不做 supervisor / sub-agent（0.5）
- 不做三级回退 / `BlockedException`（0.2 + 0.5）
- 不做 cost guard 拦截 / interrupt（0.5）
- 不做 skill 注入 / `load_skill`（0.5）
- 不做 memory / RAG 注入（0.3 / 0.4）
- 不做 chat ↔ task 升级建议（0.2）
- 不做 `@step` 装饰器抽象（V2+）

## 相关文档

- [Agent 模块](../../../docs/modules/agent/README.md)
- [架构总览 § 3](../../../docs/architecture/overview.md#3-内核架构)
