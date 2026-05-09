# 架构总览

## 项目定位

Sigma 是一个可插拔的 AI Agent 框架。核心设计哲学：**Runtime 是 Graph，负责编排；执行单元是 Node，内部黑盒，对外只有 input/output；Node 通过 Port 消费可替换的能力（LLM、Memory、RAG、Tools），Port 保证换实现时类型安全。** Graph 不关心 Node 内部实现，Node 不关心 Port 背后是谁。

## 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                            App Layer                                  │
│                   CLI  /  Web UI  /  API                              │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                                                                      │
│                     Runtime = Graph（编排）                           │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                                              │    │
│  │   Input ──→ Context Builder Node ──→ Agent Node ──→ Output  │    │
│  │                                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Voice Pipeline (可选): Mic → VAD → STT → [上述 Graph] → TTS        │
│  Task Manager: 后台长任务执行                                         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

        每个 Node 内部通过 Port 消费能力（依赖注入）：

┌───────────────────────────────┐   ┌────────────────────────────────┐
│     Context Builder Node      │   │        Agent Node               │
│                               │   │                                 │
│  内部消费:                     │   │  内部消费:                       │
│    MemoryPort  → Local/Mem0   │   │    LLMPort   → OpenAI/DeepSeek │
│    RetrievalPort → LlamaIndex │   │    ToolPort  → Native/MCP      │
│    (历史压缩逻辑)              │   │    MemoryPort → (当 tool 用)    │
│                               │   │    RetrievalPort → (当 tool 用) │
│  输出: Context                │   │                                 │
│                               │   │  输出: AsyncIterator[AgentChunk]│
└───────────────────────────────┘   └────────────────────────────────┘

        Port = 可替换能力的类型契约:

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  AgentRuntimePort    ContextBuilderPort    LLMPort    STT/TTS Port  │
│  RetrievalPort       MemoryPort            ToolPort   (可选)        │
│                                                                      │
│  每个 Port 背后 = 一个 Adapter（具体实现）+ Registry（工厂）          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

        Adapter 实现:

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Agent: SimpleLoop, LangGraph       LLM: OpenAI, DeepSeek, Qwen    │
│  Context: Passthrough, Full         Memory: Local, Mem0             │
│  RAG: LlamaIndex, Qdrant           Tools: Native, MCP              │
│  Voice (可选): Whisper, CosyVoice                                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

横切: Trace（内部模块）/ Config / Registry
```

## 分层模型

### Graph / Node / Port

```
Graph  = Runtime 编排（定义 node 怎么连、数据怎么流）
Node   = 执行单元（内部黑盒，可嵌套子图，对外只有 input/output）
Port   = 可替换能力的类型契约（保证换实现不出错）
```

```
Runtime Graph
  ├── Context Builder Node （消费 MemoryPort, RetrievalPort）
  ├── Agent Node           （消费 LLMPort, ToolPort）
  └── Post-process Node
```

| 模块 | 是否 Graph Node | Port |
|------|:-:|------|
| Context Builder | ✅ | `ContextBuilderPort` |
| Agent | ✅ | `AgentRuntimePort` |
| LLM | — | `LLMPort`（被 Agent 内部消费） |
| Memory | — | `MemoryPort`（被 Context Builder / Agent 消费） |
| RAG | — | `RetrievalPort`（被 Context Builder / Agent 消费） |
| Tools | — | `ToolPort`（被 Agent 内部消费） |
| STT / TTS | — | `STTPort` / `TTSPort`（可选，Voice Pipeline 内部） |

### 层级职责

| 层 | 职责 | 关键文件 | 详见 |
|---|---|---|---|
| App | 用户界面入口 | `src/app/cli.py`, `web.py`, `api.py` | — |
| Runtime | Graph 编排 — 定义 node 连接拓扑和数据流向 | `src/runtime/` | [运行模式](runtime-modes.md) |
| Nodes | Graph 中的执行单元（Context Builder, Agent 等） | `src/agent/`, `src/context/` | 各模块文档 |
| Ports | 可替换能力的类型契约（Protocol） | `src/ports/` | [Ports & Adapters](ports-and-adapters.md) |
| Adapters | Port 的具体实现，可随时替换 | `src/voice/`, `src/rag/` 等 | 各模块文档 |
| Infrastructure | 存储、队列等基础设施 | — | — |
| Trace/Config | 横切关注点，贯穿所有层 | `src/trace/`, `config.yaml` | [Trace](../modules/trace/) |

## 后台任务

Task Manager 是 Agent 可调用的工具，不是独立的"模式"。

```
用户："帮我把所有 API 文档重新生成"
  → Agent 判断：需要长时间执行
  → 调用 create_task 工具
  → Task Manager 后台起子 Agent 执行
  → Agent 回复用户："已创建后台任务，可随时查看进度"
```

用户也可通过 App 层直接提交任务。

Task Manager 职责：
- 后台执行 Agent stream
- 进度查询 / 取消 / 追加指令
- 结果存储和通知

## 核心设计决策

### 1. Graph / Node 分离

Runtime（Graph）负责系统编排：node 之间怎么连、数据怎么流、走哪条边。开发者定义的固定拓扑。

Agent（Node）负责认知决策：拿到上下文后怎么做、调什么工具、要不要重试。LLM 运行时动态决定。

两者都可以用 LangGraph 实现，但层级不同——Graph 是确定性的 DAG，Node 内部是 LLM 驱动的动态图。

### 2. Port = 可替换能力的类型契约

Port 不是"所有模块的接口层"，而是保证能力可替换时的类型安全。

Graph Node（Context Builder, Agent）通过 Port 暴露给 Runtime。
被消费的能力（LLM, Memory, Tools）通过 Port 被 Node 内部调用。

Agent 实现可以是 while loop、LangGraph 子图、远程 gRPC 调用——对 Runtime 来说都只是 `stream(context, input) → AsyncIterator[AgentChunk]`。

### 3. Context Builder 独立于 Agent

上下文组装和决策执行是两个独立的 node。Context Builder 决定"LLM 看到什么"，Agent 专注"拿到上下文后怎么做"。两者通过 state 传递数据。

### 4. 配置驱动

切换任何 node 的实现 = 改 config 里的 provider 字段。

### 5. 接入新技术的固定流程

```
1. 写 adapter → src/<domain>/<adapter_name>.py
2. 注册到 REGISTRY → src/<domain>/registry.py
3. 改配置 → config.yaml
4. 跑测试 → tests/<domain>/test_<adapter_name>.py
```

## 模块通信

### 当前方式

所有模块使用 Python Protocol（进程内调用），单进程部署。

### 演进路径

```
runtime/ → ports/stt.py (Protocol) → voice/whisper.py       # 现在：本地
runtime/ → ports/stt.py (Protocol) → voice/grpc_client.py   # 未来：远程
```

| 阶段 | 通信方式 | 部署形态 |
|------|---------|---------|
| V0 | 进程内 Protocol | 单进程 |
| V1-V2 | Tools 模块接入 MCP，其余不变 | 单进程 + MCP Server |
| V3+ | 按需拆 gRPC / Event Bus | 微服务 |

### 延迟预算

| 路径 | 协议 | 模块 | 延迟 |
|------|------|------|------|
| 热路径 | 进程内 / gRPC streaming | Voice Pipeline ↔ Agent ↔ LLM | < 10ms |
| 温路径 | gRPC unary / MCP | STT、TTS、RAG、Memory | < 50ms |
| 冷路径 | Event Bus | 文档索引、日志归档、异步通知 | 秒级 |

## 代码结构

```
src/
  core/               # 稳定内核：数据结构、事件
    schemas.py        #   AgentChunk, Context, Artifact, Task 等
    events.py         #   事件定义

  ports/              # 接口定义（Protocol），合同层
    stt.py            #   STTPort
    tts.py            #   TTSPort
    llm.py            #   LLMPort
    context_builder.py #  ContextBuilderPort
    agent_runtime.py  #   AgentRuntimePort
    retrieval.py      #   RetrievalPort
    memory.py         #   MemoryPort
    tools.py          #   ToolPort

  voice/              # 语音域
    whisper.py        #   Whisper STT
    cosyvoice.py      #   CosyVoice TTS
    silero_vad.py     #   Silero VAD
    registry.py       #   REGISTRY + create()

  context/            # 上下文组装域
    passthrough.py    #   透传实现
    registry.py

  agent/              # Agent 域
    simple_loop.py    #   简单循环实现
    langgraph.py      #   LangGraph 实现
    registry.py

  rag/                # 检索域
    llamaindex.py
    qdrant.py
    registry.py

  memory/             # 记忆域
    local.py
    mem0.py
    registry.py

  tools/              # 工具域
    mcp.py
    native.py
    registry.py

  trace/              # 观测（内部模块，不走 Port）
    jsonl.py
    registry.py

  runtime/            # 跨域业务编排
    voice_pipeline.py #   语音流控制
    text_pipeline.py  #   文本流控制
    task_manager.py   #   后台任务管理
    conversation.py   #   会话状态持久化

  app/                # 应用入口
    cli.py
    web.py
    api.py

experiments/          # 平行实验
tests/                # 测试
config.yaml           # 全局配置
```

## 相关文档

- [运行模式详解](runtime-modes.md) — Cascade 和 Realtime 双模式
- [Ports & Adapters](ports-and-adapters.md) — 接口设计和数据结构
- [依赖规则](dependency-rules.md) — Import 约束
- [路线图](../roadmap.md) — 版本规划
