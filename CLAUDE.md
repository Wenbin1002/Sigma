# CLAUDE.md

项目：语音交互 AI 助手（支持 Conversation Mode + Task Mode）

## 代码结构

```
src/
  core/               # 稳定内核：数据结构、事件
    schemas.py
    events.py

  ports/              # 所有接口定义（Protocol），合同层
    stt.py
    tts.py
    llm.py
    agent_runtime.py
    retrieval.py
    memory.py
    tools.py
    trace.py

  voice/              # 语音域：STT + TTS + VAD 实现
    whisper.py
    cosyvoice.py
    silero_vad.py
    registry.py

  agent/              # Agent Runtime 实现
    simple_loop.py
    langgraph.py
    registry.py

  rag/                # 检索增强实现
    llamaindex.py
    qdrant.py
    registry.py

  memory/             # 记忆实现
    local.py
    mem0.py
    registry.py

  tools/              # 工具执行实现
    mcp.py
    native.py
    registry.py

  trace/              # 观测实现
    jsonl.py
    registry.py

  runtime/            # 跨域业务编排
    voice_pipeline.py
    text_pipeline.py
    task_manager.py
    conversation.py

  app/
    cli.py
    web.py
    api.py

experiments/
tests/
config.yaml
```

## 架构约束

1. `core/` 不依赖任何外部 AI 框架/SDK
2. `runtime/` 只 import `ports/`，不 import 域目录（voice/agent/rag/...）
3. 域目录只 import `ports/` + 外部 SDK，域之间不互相引用
4. 新技术接入 = 域目录下新文件 + 注册到 registry.py + 改配置
5. 禁止反向依赖

## 核心接口

### AgentRuntimePort

```python
class AgentRuntimePort(Protocol):
    def stream(self, input: str, state: ConversationState) -> AsyncIterator[AgentChunk]: ...
    def resume(self, state: ConversationState, decision: UserDecision) -> AsyncIterator[AgentChunk]: ...
```

### AgentChunk（输出流协议）

```python
@dataclass
class AgentChunk:
    type: Literal["text_delta", "artifact", "progress", "needs_approval", "needs_input", "done", "error"]
    content: str | None = None
    artifact: Artifact | None = None
    pending_action: ToolCall | None = None
```

### Artifact

```python
@dataclass
class Artifact:
    type: Literal["file", "code", "image", "table", "diff"]
    data: bytes | str
    filename: str | None = None
    mime_type: str | None = None
```

### UserDecision

```python
@dataclass
class UserDecision:
    approved: bool | None = None
    input: str | None = None
```

### ConversationState

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

### Task

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

## 配置

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

## 两种模式

- **Conversation Mode**：实时对话。Voice/Text Pipeline 直接消费 agent.stream()
- **Task Mode**：后台长任务。Task Manager 后台消费 agent.stream()，存储产物，管理生命周期

两者共享同一个 AgentRuntimePort。

## Context Builder

所有进入 LLM 的信息经过 Context Builder 统一组装：
- System Prompt
- 用户 Profile（Memory）
- 相关记忆（Memory Port）
- 检索结果（Retrieval Port）
- 对话历史
- 工具调用历史

超出 token budget 时按优先级压缩。

## 命名规范

- Port 接口：`XxxPort`（如 `STTPort`, `LLMPort`, `AgentRuntimePort`）
- 实现类：描述性名称（如 `WhisperSTT`, `OpenAILLM`, `SimpleLoopAgent`）
- 注册表：每个域目录下 `registry.py` 中的 `REGISTRY` dict
- 工厂函数：`create(config) -> Port`
