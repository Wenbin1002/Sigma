# Overview

## 项目是什么

一个支持语音交互的 AI 助手。能听、能说、能用工具做事、能记住你说过的话、能基于你的资料回答问题。支持实时对话，复杂任务可自动转为后台执行。

## 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                            App Layer                                 │
│                     CLI  /  Web UI  /  API                           │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                        Runtime Layer                                  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Voice Pipeline: Mic→VAD→STT→Agent→TTS（打断 + 流式播放）      │ │
│  │  Text Pipeline:  Input→Agent→Output（流式输出）                 │ │
│  └────────────────────────────────┬───────────────────────────────┘ │
│                                   │                                  │
└───────────────────────────────────┼──────────────────────────────────┘
                                    │  stream() / resume()
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Context Builder (上下文组装)                      │
│                                                                      │
│  input + state ──→ [Memory召回 | RAG检索 | 历史压缩] ──→ Context     │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │  Context
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Agent Runtime Port (纯决策图)                        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ 输出流: AsyncIterator[AgentChunk]                               │ │
│  │                                                                 │ │
│  │   text_delta ──→ "你好，我来帮你..."                              │ │
│  │   artifact ──→ [生成的文件/图片/代码]                             │ │
│  │   progress ──→ "正在读取文件..."                                  │ │
│  │   needs_approval ──→ "要执行 rm -rf 吗？"                        │ │
│  │   needs_input ──→ "你指的是哪个文件？"                            │ │
│  │   error ──→ "执行失败: ..."                                      │ │
│  │   done                                                          │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  内部实现（外部不可见）:                                                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  LLM ──→ Tool Router ──→ Loop                                │   │
│  │               │                                               │   │
│  │               ▼                                               │   │
│  │  ┌──────────┐ ┌──────────────┐                               │   │
│  │  │Tool Exec │ │Task Manager  │ (Agent 判断需要后台执行时调用)   │   │
│  │  └──────────┘ └──────────────┘                               │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│                         Ports Layer                                   │
│                                                                      │
│  ┌───────┐ ┌───────┐ ┌───────┐ ┌────────┐ ┌────────┐ ┌─────────┐  │
│  │  STT  │ │  TTS  │ │  LLM  │ │Retrieval│ │ Memory │ │  Tools  │  │
│  │ Port  │ │ Port  │ │ Port  │ │  Port  │ │  Port  │ │  Port   │  │
│  └───┬───┘ └───┬───┘ └───┬───┘ └───┬────┘ └───┬────┘ └────┬────┘  │
└──────┼─────────┼─────────┼─────────┼──────────┼───────────┼────────┘
       │         │         │         │          │           │
┌──────▼─────────▼─────────▼─────────▼──────────▼───────────▼────────┐
│                       Adapters Layer                                  │
│                                                                      │
│  Whisper   Edge     OpenAI   LlamaIndex  Local     Native           │
│  FunASR    Cosy     DeepSeek  Qdrant     Mem0      MCP              │
│  Deepgram  Fish     Qwen     LightRAG    Zep       Browser          │
└─────────────────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────────┐
│                        Infrastructure                                │
│                                                                      │
│     Postgres / SQLite    Vector DB    Object Storage    Redis        │
└─────────────────────────────────────────────────────────────────────┘

横切关注点（贯穿所有层）:
┌─────────────────────────────────────────────────────────────────────┐
│  Trace / Observability          Config / Registry                    │
└─────────────────────────────────────────────────────────────────────┘
```

## 分层说明

| 层 | 职责 | 详见 |
|---|---|---|
| App | 用户界面入口（CLI/Web/API） | — |
| Runtime | Voice/Text Pipeline，流式交互 | — |
| Context Builder | 每轮调用前组装上下文（Memory/RAG/历史压缩） | [context/](context/) |
| Agent Runtime | 纯决策图，流式输入输出，内部完全自治 | [agent/](agent/) |
| Ports | 模块接口定义，不含任何实现 | 各模块文档 |
| Adapters | 具体实现，挂在 Port 下面，可随时替换 | 各模块文档 |
| Infrastructure | 存储、队列等基础设施 | — |
| Trace/Config | 横切关注点，贯穿所有层 | [observability/](observability/) |

## 后台任务

Task Manager 是 Agent 可调用的工具/基础设施，不是独立的"模式"。

```
用户："帮我把所有 API 文档重新生成"
  → Agent 判断：需要长时间执行
  → 调用 create_task 工具
  → Task Manager 后台起子 Agent 执行
  → Agent 回复用户："已创建后台任务，可随时查看进度"
```

用户也可通过 App 层直接提交任务（UI 上的"提交任务"入口）。

Task Manager 职责：
- 后台执行 Agent stream
- 进度查询 / 取消 / 追加指令
- 结果存储和通知

## 核心设计决策

### 1. Ports & Adapters

模块之间通过 Port（接口）通信，实现挂在 Adapter 里，随时可换。

```
runtime 只 import ports，永远不 import adapters
adapters 只 import ports + 外部 SDK
core 不依赖任何东西
```

### 2. Agent Runtime 是黑盒

对外只有 `stream()` 和 `resume()`。内部是 while loop、LangGraph、还是别的什么，调用方不关心。

### 3. Context Builder 独立于 Agent

上下文组装和决策执行是两件独立的事，拆开。Context Builder 决定"LLM 看到什么"，Agent 专注"拿到上下文后怎么做"。

### 4. 配置驱动

切换任何模块 = 改 config 里的 provider 字段。

### 5. 接入新技术的固定流程

```
1. 写 adapter
2. 注册到 REGISTRY
3. 改配置
4. 跑测试
```

## 模块通信策略

### 设计目标

模块完全解耦，内部黑盒，对外只有 input/output。可插拔，可独立部署，使用业界标准协议通信。

### 演进路径

现阶段全部使用 Python Protocol（进程内调用），单进程部署。未来按需将模块拆为独立服务——拆分方式为新增一个 remote client adapter，调用方代码不变。

```
runtime/ → ports/stt.py (Protocol) → voice/whisper.py       # 现在：本地
runtime/ → ports/stt.py (Protocol) → voice/grpc_client.py   # 未来：远程
```

| 阶段 | 通信方式 | 部署形态 |
|------|---------|---------|
| V0 | 进程内 Protocol | 单进程 |
| V1-V2 | Tools 模块接入 MCP，其余不变 | 单进程 + MCP Server |
| V3+ | 按需拆 gRPC / Event Bus | 微服务 |

### 通信方式选型（按延迟敏感度）

| 路径 | 协议 | 模块 | 延迟预算 |
|------|------|------|---------|
| 热路径 | 进程内 / gRPC streaming | Voice Pipeline ↔ Agent ↔ LLM | < 10ms |
| 温路径 | gRPC unary / MCP | STT、TTS、RAG、Memory | < 50ms |
| 冷路径 | Event Bus (NATS) | 文档索引、日志归档、异步通知 | 秒级 |

### Port 接口约束

为保证未来可无缝拆为跨进程服务，所有 Port 必须遵守：

1. **入参出参可序列化** — 只用 `str / bytes / int / float / bool / list / dict / dataclass`，不传回调、文件句柄、Python 特有对象
2. **流式用 AsyncIterator** — 不用 callback，可直接映射 gRPC stream
3. **依赖注入，不持有具体引用** — 只 import `ports/`，不 import 实现模块
4. **不暴露批处理优化** — Port 保持单次调用语义，批处理在 adapter 内部实现

详细示例见 [interfaces.md](interfaces.md#port-接口设计约束)。

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
    context_builder.py
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

  context/            # 上下文组装
    passthrough.py
    registry.py

  agent/              # Agent Runtime 实现（纯决策图）
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
    voice_pipeline.py # 语音流控制
    text_pipeline.py  # 文本流控制
    task_manager.py   # 后台任务管理（Agent 可调用 + App 可直接提交）
    conversation.py   # 会话状态持久化

  app/                # 应用入口
    cli.py
    web.py
    api.py

experiments/
docs/
tests/
config.yaml
```

## 文档索引

| 模块 | 文档 | 说明 |
|------|------|------|
| 总纲 | [本文](architecture.md) | 架构总览 |
| 路线 | [roadmap.md](roadmap.md) | 版本规划 |
| 语音 | [voice/](voice/) | STT/TTS/VAD、延迟优化 |
| LLM | [llm/](llm/) | 模型调用、路由、缓存 |
| Context | [context/](context/) | 上下文组装、历史压缩、注入策略 |
| Agent | [agent/](agent/) | Runtime Port 接口设计、状态管理 |
| RAG | [rag/](rag/) | 检索增强生成 |
| 记忆 | [memory/](memory/) | 长期记忆、上下文压缩 |
| 工具 | [tools/](tools/) | MCP、权限模型 |
| 观测 | [observability/](observability/) | Trace、Eval |
