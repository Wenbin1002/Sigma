# Overview

## 项目是什么

一个支持语音交互的 AI 助手。能听、能说、能用工具做事、能记住你说过的话、能基于你的资料回答问题。支持实时对话和后台长任务两种模式。

## 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                            App Layer                                 │
│                     CLI  /  Web UI  /  API                           │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                        Session Layer                                  │
│                                                                      │
│  ┌──────────────────────────┐  ┌─────────────────────────────────┐  │
│  │    Conversation Mode     │  │         Task Mode               │  │
│  │      (实时对话)           │  │        (后台任务)                │  │
│  │                          │  │                                 │  │
│  │  Voice Pipeline:         │  │  Task Manager:                  │  │
│  │  Mic→VAD→STT→Agent→TTS  │  │  submit() → 后台执行            │  │
│  │  + 打断 + 流式播放       │  │  get_status() → 查进度          │  │
│  │                          │  │  cancel() → 取消               │  │
│  │  Text Pipeline:          │  │  get_result() → 拿结果          │  │
│  │  Input→Agent→Output      │  │  subscribe() → 实时进度流       │  │
│  │  + 流式输出              │  │                                 │  │
│  └────────────┬─────────────┘  └───────────────┬─────────────────┘  │
│               │                                │                    │
│               └───────────────┬────────────────┘                    │
│                               │                                     │
└───────────────────────────────┼─────────────────────────────────────┘
                                │  stream() / resume()
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Agent Runtime Port (黑盒)                           │
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
│  │  Context Builder ──→ LLM ──→ Tool Router ──→ Loop            │   │
│  │       │                          │                            │   │
│  │       ▼                          ▼                            │   │
│  │  ┌─────────┐ ┌────────┐   ┌──────────┐                      │   │
│  │  │ Memory  │ │  RAG   │   │Tool Exec │                      │   │
│  │  └─────────┘ └────────┘   └──────────┘                      │   │
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
| Session | 管理两种交互模式：实时对话 vs 后台任务 | — |
| Agent Runtime | 核心决策黑盒，流式输入输出，内部完全自治 | [agent/](agent/) |
| Ports | 模块接口定义，不含任何实现 | 各模块文档 |
| Adapters | 具体实现，挂在 Port 下面，可随时替换 | 各模块文档 |
| Infrastructure | 存储、队列等基础设施 | — |
| Trace/Config | 横切关注点，贯穿所有层 | [trace/](trace/) |

## 两种模式

### Conversation Mode（ChatGPT-like）

实时对话。用户在线，流式交互，语音或文本。

```
用户说话/打字 → Agent 流式生成 → 实时播放/显示
                    ↕
              随时可打断/追问
```

适合：日常问答、语音聊天、短任务

### Task Mode（Codex-like）

后台长任务。用户提交后可离线，任务自主运行。

```
用户提交任务 → Task Manager 后台消费 Agent stream → 完成后通知
                    ↕
         用户可随时查看进度 / 取消 / 追加指令
```

适合：代码重构、文档生成、批量处理、需要执行几十步的复杂任务

### 两者共享同一个 Agent Runtime

关键洞察：**底层都是同一个 AgentRuntimePort.stream()**。

区别只在上层谁来消费这个流：

| | Conversation Mode | Task Mode |
|---|---|---|
| 消费者 | Voice/Text Pipeline（实时） | Task Manager（后台） |
| 用户连接 | 必须在线 | 可以离线 |
| 人工介入 | 实时确认 | 按策略自动处理 / 等用户回来 |
| 生命周期 | 一轮对话 | 分钟到小时 |
| 中间产物 | 直接播放 | 存储，最后一起给 |

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

### 3. Context Builder 是核心枢纽

所有进入 LLM 的信息——记忆、RAG 结果、工具历史、对话历史——都经过 Context Builder 统一调度。

### 4. 配置驱动

切换任何模块 = 改 config 里的 provider 字段。

### 5. 接入新技术的固定流程

```
1. 写 adapter
2. 注册到 REGISTRY
3. 改配置
4. 跑测试
```

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
    voice_pipeline.py # Conversation Mode: 语音流控制
    text_pipeline.py  # Conversation Mode: 文本流控制
    task_manager.py   # Task Mode: 任务管理
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
| Agent | [agent/](agent/) | Runtime Port 接口设计、状态管理 |
| RAG | [rag/](rag/) | 检索增强生成 |
| 记忆 | [memory/](memory/) | 长期记忆、上下文压缩 |
| 工具 | [tools/](tools/) | MCP、权限模型 |
| 观测 | [trace/](trace/) | Trace、Eval |
