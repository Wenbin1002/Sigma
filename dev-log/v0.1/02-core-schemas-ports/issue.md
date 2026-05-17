# Issue 02 — core/schemas + ports 接口

**Milestone**：v0.1
**Type**：`feat(core,ports)`
**依赖**：issue 01
**预计 PR 大小**：M

---

## 背景

后续所有 issue（LLM / Tools / Trace / Agent / Chat）都要 import `core/` 的数据结构和 `ports/` 的 Protocol。这一步把 0.1 用到的契约一次性定下来——**只放真用到的字段**，避免为想象中的需求超前抽象。

参考：

- [Ports & Adapters § 3-4](../../../docs/architecture/ports-and-adapters.md#3-核心-port-接口)
- [LLM 模块 § 2](../../../docs/modules/llm/README.md#2-llmport)
- [Tools 模块 § 3](../../../docs/modules/tools/README.md#3-toolport)
- [Trace 模块 § 6](../../../docs/modules/trace/README.md#6-tracerport)

## 范围

### 1. `src/sigma/core/schemas.py`

只放 0.1 用到的（不要超前定义 Task / Artifact / NeededInput / ProxyAudit 等 0.2+ 才用的字段）：

```python
@dataclass
class Message:
    role: Literal["system", "user", "assistant", "tool"]
    content: str | None
    tool_calls: list[ToolCall] | None = None
    tool_call_id: str | None = None     # role=tool 时填

@dataclass
class ToolCall:
    id: str
    name: str
    arguments: dict           # 已 parse 的 JSON

@dataclass
class ToolResult:
    success: bool
    data: Any
    error: str | None = None
    metadata: dict = field(default_factory=dict)

@dataclass
class ToolSchema:
    name: str
    description: str
    parameters: dict          # JSON Schema

@dataclass
class TokenUsage:
    prompt_tokens: int
    completion_tokens: int
    total_tokens: int
    cost_usd: float | None = None

@dataclass
class LLMChunk:
    type: Literal["text", "tool_call_delta", "usage", "done"]
    content: str | None = None
    tool_call: ToolCall | None = None
    usage: TokenUsage | None = None

@dataclass
class AgentChunk:
    # 0.1 只用到 5 个 type；其余（agent_handoff / artifact / progress /
    # needs_approval / needs_input / proxy_decision）留给后续 milestone 加
    type: Literal["text_delta", "tool_call", "tool_result", "done", "error"]
    content: str | None = None
    tool_call: ToolCall | None = None
    tool_result: ToolResult | None = None
    metadata: dict = field(default_factory=dict)

@dataclass
class TraceEvent:
    ts: str                   # ISO 8601
    type: str                 # agent_start / llm_call / tool_call / ...
    trace_id: str
    span_id: str
    parent_span_id: str | None = None
    payload: dict = field(default_factory=dict)
```

**`BlockedException`**：0.1 暂不使用（没有 sub-agent / task pause）。文档已声明它的位置，但**本 issue 先不实现**——等 0.2 加 task pause 时一起做。在 README 留个 TODO。

### 2. `src/sigma/ports/llm.py`

```python
class LLMPort(Protocol):
    async def stream(
        self,
        messages: list[Message],
        tools: list[ToolSchema] | None = None,
        **kwargs,
    ) -> AsyncIterator[LLMChunk]: ...

    async def generate(
        self,
        messages: list[Message],
        tools: list[ToolSchema] | None = None,
        **kwargs,
    ) -> LLMChunk: ...   # 收尾 chunk（含 usage）
```

### 3. `src/sigma/ports/tools.py`

```python
class ToolPort(Protocol):
    name: str
    description: str
    parameters_schema: dict

    async def execute(self, **kwargs) -> ToolResult: ...
```

### 4. `src/sigma/ports/tracer.py`

```python
class TracerPort(Protocol):
    async def record_event(self, event: TraceEvent) -> None: ...
    async def flush(self) -> None: ...
```

### 5. `src/sigma/ports/checkpointer.py`

直接复用 LangGraph 的 `BaseCheckpointSaver`，本 port 文件**只是一个再导出**+ 工厂签名注释：

```python
from langgraph.checkpoint.base import BaseCheckpointSaver
CheckpointerPort = BaseCheckpointSaver   # 显式 alias，便于内核 import
```

理由见 [ports-and-adapters § 3.4](../../../docs/architecture/ports-and-adapters.md#34-checkpointerport)。

## 设计要点

- **`core/` 只 import 标准库**——红线，pre-commit hook 会查
- **dataclass 而非 pydantic**——code-style 已规定跨模块通信用 dataclass，序列化用 `dataclasses.asdict`
- **流式协议双层**：LLM adapter 出 `LLMChunk` → Master Agent 翻译成 `AgentChunk` 给上层。两层不要混
- **不要在 schema 里塞业务方法**——`AgentChunk` / `LLMChunk` 是纯数据
- **`tool_call_delta` 暂不细化**——0.1 的 OpenAI 兼容 adapter 可能在流式中拆 tool call 字段；adapter 自己内部累积，对外只在 args 完整时发一个 `tool_call` chunk

## 验收标准

- [ ] `python -c "from sigma.core.schemas import Message, AgentChunk; from sigma.ports.llm import LLMPort"` 通过
- [ ] `grep -rE "^import |^from " src/sigma/core/` 只匹配标准库（`typing` / `dataclasses` / `enum`）
- [ ] `grep -rE "^from sigma\.(llm|tools|trace|checkpoint|agent|chat|server|app)" src/sigma/ports/` 必须为空
- [ ] 给 `Message` / `AgentChunk` / `TraceEvent` 写一组 round-trip 测试（dataclass → dict → dataclass）
- [ ] `ruff` 通过

## 不做

- 不实现 `BlockedException` / `Task` / `Artifact` / `NeededInput` / `ProxyAudit`（0.2+ 用到时再加）
- 不实现 `EmbeddingPort` / `STTPort` / `TTSPort`（[ports-and-adapters § 1.3](../../../docs/architecture/ports-and-adapters.md#13-已废弃的-port演化历史)）
- 不为序列化做"为 gRPC 拆分预留"的过度设计

## 相关文档

- [Ports & Adapters](../../../docs/architecture/ports-and-adapters.md)
- [代码规范 § 数据结构](../../../docs/guides/code-style.md#数据结构)
