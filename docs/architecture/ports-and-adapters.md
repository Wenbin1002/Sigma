# Ports & Adapters

本文档定义 Sigma 中可替换能力的类型契约、核心数据结构和 Registry 机制。

## 模式概述

```
Port     = 可替换能力的类型契约（Python Protocol）  定义在 src/ports/
Adapter  = Port 的具体实现                        位于域目录 src/<domain>/
Registry = 工厂注册表                             每个域的 src/<domain>/registry.py
```

**Port 解决什么问题：** Node 内部需要消费各种能力（LLM、Memory、RAG、Tools），Port 保证这些能力可以随时替换，且替换时有类型安全保障。

**什么有 Port：**
- Graph Node（Context Builder, Agent）——在图里有位置，需要 Port 保证 Runtime 不关心内部实现
- 被 Node 消费的能力（LLM, Memory, RAG, Tools）——不在图里，但需要 Port 保证可替换

**什么没有 Port：**
- Trace——内部基础设施，定了就不换，不需要类型契约

## Port 设计约束

为保证未来可拆为跨进程服务，所有 Port 必须遵守：

| 约束 | 原因 |
|------|------|
| 入参出参可序列化 — 只用 `str / bytes / int / float / bool / list / dict / dataclass` | 可映射 gRPC / JSON |
| 流式用 `AsyncIterator` — 不用 callback | 可映射 gRPC stream |
| 依赖注入，不持有具体引用 — 只 import `ports/` | 可替换实现 |
| 不暴露批处理优化 — Port 保持单次调用语义 | 批处理在 adapter 内部实现 |

**禁止**在 Port 接口中传递：callback 函数、文件句柄、socket、Python 特有对象（如 generator function）。

## 完整 Port 清单

| Port | 文件 | 核心方法 | 消费方 |
|------|------|---------|--------|
| `AgentRuntimePort` | `src/ports/agent_runtime.py` | `stream()`, `resume()` | Runtime |
| `ContextBuilderPort` | `src/ports/context_builder.py` | `build()` | Runtime |
| `LLMPort` | `src/ports/llm.py` | `generate()`, `stream()` | Agent |
| `STTPort` | `src/ports/stt.py` | `transcribe()` | Voice Pipeline |
| `TTSPort` | `src/ports/tts.py` | `synthesize()` | Voice Pipeline |
| `RetrievalPort` | `src/ports/retrieval.py` | `search()` | Context Builder, Agent |
| `MemoryPort` | `src/ports/memory.py` | `recall()`, `store()` | Context Builder, Agent |
| `ToolPort` | `src/ports/tools.py` | `execute()`, `list_tools()` | Agent |

> **注意**：Trace 不走 Port 模式。它是内部基础设施模块（`src/trace/`），提供固定 API，不需要 Registry 和配置切换。

## 核心数据结构

### AgentChunk（输出流协议）

```python
@dataclass
class AgentChunk:
    type: Literal["text_delta", "artifact", "progress", "needs_approval", "needs_input", "done", "error"]
    content: str | None = None
    artifact: Artifact | None = None
    pending_action: ToolCall | None = None
```

### Context（上下文对象）

```python
@dataclass
class Context:
    system_prompt: str
    messages: list[dict]
    memories: list[MemoryItem]
    references: list[Chunk]
```

### Artifact（生成物）

```python
@dataclass
class Artifact:
    type: Literal["file", "code", "image", "table", "diff"]
    data: bytes | str
    filename: str | None = None
    mime_type: str | None = None
```

### UserDecision（用户决策）

```python
@dataclass
class UserDecision:
    approved: bool | None = None
    input: str | None = None
```

### ConversationState（会话状态）

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

### Task（后台任务）

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

## 核心 Port 接口

### AgentRuntimePort

```python
class AgentRuntimePort(Protocol):
    def stream(self, context: Context, input: str) -> AsyncIterator[AgentChunk]: ...
    def resume(self, state: ConversationState, decision: UserDecision) -> AsyncIterator[AgentChunk]: ...
```

### ContextBuilderPort

```python
class ContextBuilderPort(Protocol):
    async def build(self, input: str, state: ConversationState) -> Context: ...
```

## Registry 模式

每个域目录遵循相同模式：

```python
# src/<domain>/registry.py

from ports.<port_name> import XxxPort

REGISTRY: dict[str, type] = {
    "impl_a": ImplA,
    "impl_b": ImplB,
}

def create(config) -> XxxPort:
    """工厂函数：根据 config 创建对应实现"""
    return REGISTRY[config.provider](config)
```

调用方（Runtime）只需：

```python
from <domain>.registry import create

adapter = create(config.voice.stt)  # 返回 STTPort 实例
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

## 添加新 Port（清单）

当需要定义全新的能力接口时：

1. 在 `src/ports/` 创建 `<port_name>.py`，定义 Protocol class
2. 确保入参出参满足序列化约束
3. 在相关域目录创建首个实现
4. 创建 `src/<domain>/registry.py`
5. 更新 `config.yaml` 添加配置节
6. 更新 [CLAUDE.md](../../CLAUDE.md) 的项目结构和文档索引

## 相关文档

- [架构总览](overview.md)
- [依赖规则](dependency-rules.md)
- [添加 Adapter 指南](../guides/adding-an-adapter.md)
