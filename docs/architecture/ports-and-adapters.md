# Ports & Adapters

> 这份文档定义 Sigma 中**真有多实现需求的能力契约**——也就是 Port。
>
> **设计原则**：Port 不是"所有模块的接口层"。一个能力值不值得做 Port，取决于"换实现的频率 × 换实现的收益"是否大于"维护抽象的固定代价"。

---

## 1. 哪些能力是 Port，为什么

### 1.1 当前 Port 清单

| Port | 文件 | 多实现的真实理由 |
|---|---|---|
| `LLMPort` | `src/ports/llm.py` | OpenAI / DeepSeek / Anthropic / Qwen 等多 provider 是真实需求（成本、合规、性能差异显著） |
| `ToolPort` | `src/ports/tools.py` | 内置 Python 函数 vs MCP server vs 远程 HTTP tool，调用语义不同但要统一 |
| `TracerPort` | `src/ports/tracer.py` | JSONL（默认）/ OTel / 第三方平台都可能有人接入 |
| `CheckpointerPort` | `src/ports/checkpointer.py` | LangGraph 自带的 SqliteSaver / PostgresSaver / 自定义后端 |

**Phase 后期可能加入**：
- `STTPort` / `TTSPort`：如果 Voice 重新启用
- `EmbeddingPort`：如果 RAG 系统有跨 provider 的 embedding 需求

### 1.2 哪些**不是** Port，为什么

这一节是为了防止过度抽象的反思笔记。

| 模块 | 为什么不是 Port |
|---|---|
| **Agent / Master / Supervisor** | 这是 Sigma 内核结构，不是"可替换能力"。换实现 = 换框架 |
| **Skill** | Skill 是 markdown 文件，不需要 Python 接口 |
| **RAG** | 不是单一可替换接口，是"多 index 管理 + retrieval 策略"的内部组件。是否暴露 Port 待具体落地再定 |
| **Memory** | 同 RAG。分层（global / session / task）的语义不在 Port 层抽象 |
| **Context Engine** | Context 拼装是 Sigma 的核心算法，不开放替换 |
| **Task Engine** | Sigma 的内置实现，无第二个版本的需求 |

**关键判据**：
- ✅ "已经有 ≥2 个真实需求"——做 Port
- ❌ "未来可能要换"——不做 Port，等真撞墙再抽象

### 1.3 已废弃的 Port（演化历史）

之前文档定义过的 Port，现在已废弃：

| 已废弃 | 废弃原因 |
|---|---|
| `ContextBuilderPort` | Context engine 是 Sigma 核心算法，不该开放替换 |
| `AgentRuntimePort` | Sigma 用 LangGraph 做 runtime，没有第二个 runtime 选项 |
| `RetrievalPort` | RAG 不是单 endpoint 可替换，是多 index 管理 |
| `MemoryPort` | Memory 分层语义不应在单一 Port 抽象 |
| `STTPort` / `TTSPort` | Voice 在 Phase 1 不做，恢复时再加 |

---

## 2. Port 设计约束

| 约束 | 原因 |
|---|---|
| **入参出参可序列化**（`str / bytes / int / float / bool / list / dict / dataclass`） | 便于跨进程传输、便于 trace 序列化、便于 replay |
| **流式用 `AsyncIterator`** | 不传 callback、不传 generator function |
| **单一职责** | 一个 Port 只解决一类能力，避免巨型接口 |
| **依赖注入** | adapter 不持有外部状态引用，由 registry 工厂构造 |

> **不再有的约束**：之前文档里的"为未来可拆 gRPC 而约束"已废弃——单人项目 + 学习目标，gRPC 拆分是不存在的需求。等真要拆时再设计 wire protocol，比从 Python Protocol 倒推更清晰。

---

## 3. 核心 Port 接口

### 3.1 LLMPort

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
    ) -> LLMResponse: ...
```

**关键约束**：
- `LLMChunk` / `LLMResponse` **必须包含 token usage 信息**——Sigma 的 cost guard 依赖这个
- `tools` 参数遵循 OpenAI tool calling schema 事实标准

### 3.2 ToolPort

```python
class ToolPort(Protocol):
    name: str
    description: str
    parameters_schema: dict   # JSON Schema
    
    async def execute(self, **kwargs) -> ToolResult: ...
```

**多种 adapter**：
- `PythonTool`：装饰 Python 函数
- `MCPTool`：包装 MCP server 暴露的 tool
- `HTTPTool`：远程 HTTP API（Phase 后期）

### 3.3 TracerPort

```python
class TracerPort(Protocol):
    async def record_event(self, event: TraceEvent) -> None: ...
    async def flush(self) -> None: ...
```

**TraceEvent 是统一的事件格式**——不同 adapter 决定怎么存储和呈现：
- `JSONLTracer`：默认实现，写本地 JSONL（配本地 HTML viewer）
- `OTelTracer`：导出到 OpenTelemetry collector
- `LangSmithTracer` / `LangfuseTracer`：未来如果想接

### 3.4 CheckpointerPort

直接复用 LangGraph 的 `BaseCheckpointSaver`，不另行抽象。Sigma 提供：
- `SqliteCheckpointer`（默认）
- 用户可接 LangGraph 生态的 PostgresSaver / RedisSaver 等

---

## 4. 核心数据结构

> 这些是 Sigma 内部跨模块通信的契约，**不是 Port 的入参**——但和 Port 紧密相关。

### 4.1 AgentChunk（统一流式协议）

```python
@dataclass
class AgentChunk:
    type: Literal[
        "text_delta",       # LLM 流式文本片段
        "tool_call",        # 工具调用开始
        "tool_result",      # 工具调用结果
        "agent_handoff",    # 切换到 sub-agent
        "artifact",         # 产出物
        "progress",         # 进度更新（task mode）
        "needs_approval",   # 需要用户确认
        "needs_input",      # L3 升级（pause / 追问）
        "proxy_decision",   # 主 agent 代答审计（L2）
        "done",
        "error",
    ]
    content: str | None = None
    tool_call: ToolCall | None = None
    artifact: Artifact | None = None
    pending_input: NeededInput | None = None
    proxy_audit: ProxyAudit | None = None
    metadata: dict = field(default_factory=dict)
```

### 4.2 Task

```python
@dataclass
class Task:
    id: str
    user_id: str
    description: str
    status: Literal[
        "queued", "running", "paused",
        "completed", "failed", "cancelled",
    ]
    schedule: OneShot | Daily | Cron
    target_agent: str
    
    artifacts: list[Artifact]
    progress: str
    log: list[LogEntry]
    
    # Pause 相关
    pending_question: str | None
    needed_input: NeededInput | None
    
    # 关联
    parent_chat_id: str | None       # 从 chat 升级时填
    checkpoint_id: str | None        # LangGraph checkpointer 的 thread_id
    
    created_at: datetime
    updated_at: datetime
    error: str | None = None
```

### 4.3 BlockedException

```python
class BlockedException(Exception):
    reason: Literal[
        "missing_credential",
        "missing_file",
        "external_dependency",
        "ambiguous_intent",
        "missing_information",
        "decision_needed",
    ]
    detail: str
    needed_input: NeededInput
```

### 4.4 Artifact

```python
@dataclass
class Artifact:
    type: Literal["file", "code", "image", "table", "diff", "chart", "report"]
    data: bytes | str
    filename: str | None = None
    mime_type: str | None = None
    metadata: dict = field(default_factory=dict)
```

### 4.5 其他

- `Message`：LLM 消息格式（OpenAI 风格）
- `ToolCall`、`ToolResult`、`ToolSchema`：工具调用相关
- `NeededInput`：表示"需要什么输入"的 schema（type / description / validation）
- `ProxyAudit`：L2 代答的审计记录（reason / question / answer / 推理依据）
- `TraceEvent`：trace 事件（node_id / type / timestamp / payload / cost / duration）

具体定义在 `src/core/schemas.py`。

---

## 5. Registry 模式

每个有 Port 的域目录有 registry：

```python
# src/llm/registry.py

from ports.llm import LLMPort

REGISTRY: dict[str, type[LLMPort]] = {
    "openai": OpenAILLM,
    "deepseek": DeepSeekLLM,
    "anthropic": AnthropicLLM,
}

def create(config) -> LLMPort:
    return REGISTRY[config.provider](config)
```

调用方（内核）：

```python
from llm.registry import create as create_llm

llm = create_llm(config.llm)  # 返回 LLMPort 实例
```

---

## 6. 配置结构

```yaml
llm:
  default:
    provider: openai
    model: gpt-4o
  
  # cost-aware 路由（Phase 1 后期）：
  routing:
    simple_tasks:    { provider: openai, model: gpt-4o-mini }
    complex_tasks:   { provider: anthropic, model: claude-opus-4 }
    coding:          { provider: anthropic, model: claude-sonnet-4 }

tools:
  builtin:           # 内置 tool 启用列表
    - read_file
    - write_file
    - shell
    - search_web
  mcp_servers:       # 挂载 MCP server
    - { name: "github", command: "npx", args: ["@anthropic/mcp-github"] }

trace:
  provider: jsonl
  output_dir: ~/.sigma/traces/

checkpoint:
  provider: sqlite
  db_path: ~/.sigma/checkpoint.db

cost_guard:
  per_task_budget_usd: 1.0
  daily_budget_usd: 10.0
```

---

## 7. 添加新 Port（清单）

只在以下情况成立时才考虑加 Port：

1. ✅ 已经有 ≥2 个真实需要的实现
2. ✅ 替换收益显著（成本、合规、性能、生态）
3. ✅ 接口形状能稳定（不是每次新实现都要改 Protocol）

满足后：

1. 在 `src/ports/` 创建 `<port_name>.py`
2. 定义 Protocol，遵守 § 2 设计约束
3. 在相关域目录创建首个实现
4. 创建 `src/<domain>/registry.py`
5. 更新 `config.yaml` schema
6. 更新本文档 § 1.1 表格
7. 更新 [CLAUDE.md](../../CLAUDE.md) 项目结构

---

## 8. 相关文档

- [架构总览](overview.md) — Port 在 Sigma 中的位置
- [依赖规则](dependency-rules.md) — Import 约束
- [设计决策日志](design-log.md) — Port 数量大砍的演化记录
