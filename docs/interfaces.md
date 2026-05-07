# 核心接口定义

本文档定义项目的核心数据结构和 Port 接口签名。待 `src/` 目录建立后，这些定义将下沉到各模块的 `CLAUDE.md` 中。

## AgentRuntimePort

```python
class AgentRuntimePort(Protocol):
    def stream(self, context: Context, input: str) -> AsyncIterator[AgentChunk]: ...
    def resume(self, state: ConversationState, decision: UserDecision) -> AsyncIterator[AgentChunk]: ...
```

## ContextBuilderPort

```python
class ContextBuilderPort(Protocol):
    async def build(self, input: str, state: ConversationState) -> Context: ...

@dataclass
class Context:
    system_prompt: str
    messages: list[dict]
    memories: list[MemoryItem]
    references: list[Chunk]
```

## AgentChunk（输出流协议）

```python
@dataclass
class AgentChunk:
    type: Literal["text_delta", "artifact", "progress", "needs_approval", "needs_input", "done", "error"]
    content: str | None = None
    artifact: Artifact | None = None
    pending_action: ToolCall | None = None
```

## Artifact

```python
@dataclass
class Artifact:
    type: Literal["file", "code", "image", "table", "diff"]
    data: bytes | str
    filename: str | None = None
    mime_type: str | None = None
```

## UserDecision

```python
@dataclass
class UserDecision:
    approved: bool | None = None
    input: str | None = None
```

## ConversationState

```python
@dataclass
class ConversationState:
    id: str
    user_id: str
    messages: list[dict]
    current_task: str | None
    pending_decisions: list[ToolCall]
    metadata: dict
    created_at: datetime
    updated_at: datetime
```

## Task

```python
@dataclass
class Task:
    id: str
    user_id: str
    status: Literal["queued", "running", "paused", "completed", "failed", "cancelled"]
    input: str
    artifacts: list[Artifact]
    progress: str
    log: list[str]
    conversation_state_id: str
    created_at: datetime
    updated_at: datetime
    error: str | None = None
```

## Port 接口模式

所有 Port 遵循同一模式：

```python
# ports/xxx.py — 接口定义
class XxxPort(Protocol):
    ...

# voice/registry.py（或 agent/registry.py 等）— 注册表
REGISTRY: dict[str, type] = {
    "impl_a": ImplA,
    "impl_b": ImplB,
}

def create(config) -> "XxxPort":
    return REGISTRY[config.provider](config)
```

## 配置结构

```yaml
llm:
  provider: openai
  model: gpt-4o

voice:
  stt: { provider: whisper }
  tts: { provider: cosyvoice }
  vad: { provider: silero }

retrieval:
  provider: llamaindex
  vector_store: qdrant

memory:
  provider: local

trace:
  provider: jsonl

agent_runtime:
  provider: simple_loop

task:
  auto_approve_risk_below: read
  max_steps: 100
  timeout_minutes: 60
```

## Port 接口设计约束

所有 Port 接口遵守以下约束，保证未来可拆为跨进程服务：

1. **入参出参可序列化** — 只用 `str / bytes / int / float / bool / list / dict / dataclass`
2. **流式用 AsyncIterator** — 不用 callback，可直接映射 gRPC stream
3. **依赖注入，不持有具体引用** — 只 import `ports/`
4. **不暴露批处理优化** — Port 保持单次调用语义，批处理在 adapter 内部

详见 [architecture.md → 模块通信策略](architecture.md#模块通信策略)
