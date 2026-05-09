# 运行模式：Cascade 和 Realtime

> 阅读本文档前，建议先看 [架构总览](overview.md) 了解系统分层。

## 设计哲学

**Every agent is a graph.** 用户的 input → output 之间是一个黑盒，内部用 LangGraph 表达为有向图。图中每个 node 是一个能力单元，node 内部可以嵌套子图。

两个核心洞察：

1. **实时性和复杂性天然互斥** — 需要马上回复的场景不复杂，复杂的场景不需要马上回复
2. **Realtime Audio 和级联 Audio 是两种编排方式，不是两套系统** — 共享同一套能力层

---

## 双模式架构

### 级联模式（Cascade）

传统 STT → LLM → TTS 链路。适用于文本交互和不需要感知语气的语音交互。

```
Audio → [STT] → text → [LangGraph Pipeline] → text → [TTS] → Audio
                              │
                    ┌─────────┴──────────┐
                    │                     │
             Context Builder Node    Agent Node
                    │                     │
              (sub-graph)           (sub-graph)
              Memory/RAG/...        Tools/Multi-Agent
```

整个认知流程用 LangGraph 编排。Voice Pipeline 的流控（打断、流式播放）仍由 async 代码处理——实时音频对延迟的要求不适合图模型。

### Realtime 模式（Native Audio）

模型直接消费/生成音频 token。适用于需要感知语气、口音、情绪的场景。

```
┌─── 会话前 ──────────────────────────────────────┐
│                                                   │
│  [Memory Recall] ─┐                              │
│  [RAG Search] ────┼→ [Build System Prompt]       │
│  [User Profile] ──┘         │                    │
│                             ▼                    │
│                    注入 Realtime Session config   │
└───────────────────────────────────────────────────┘

┌─── 会话中 ──────────────────────────────────────────────┐
│                                                          │
│  User ←──── Realtime API WebSocket ────→ Model          │
│                       │                                  │
│                       │ 事件流                            │
│                       ▼                                  │
│              ┌─── Sigma Middleware ────┐                 │
│              │                         │                 │
│              │  • transcript 实时记录   │                 │
│              │  • function_call 拦截   │ ──→ Ports 层    │
│              │    - rag.search()       │                 │
│              │    - memory.retrieve()  │                 │
│              │    - task.create()      │                 │
│              │  • 结果注入回 session    │                 │
│              └─────────────────────────┘                 │
└──────────────────────────────────────────────────────────┘

┌─── 会话后 ──────────────────────────────────────┐
│                                                   │
│  [Transcript] → [提取记忆点] → Memory 存储       │
│               → [生成报告/笔记] → 通知用户        │
│               → [更新用户画像]                    │
└───────────────────────────────────────────────────┘
```

Sigma 在 Realtime 模式下的角色是 **Middleware**：不替代模型对话，而是包裹它 — 前面注入记忆，中间响应 tool call，后面沉淀知识。

---

## Realtime 模式使用场景

核心判断标准：**模型需要"听到"声音本身（而不只是文字）才能做好这件事。**

| 场景 | 为什么需要原生音频 | Sigma 补什么 |
|------|-------------------|-------------|
| 语言学习 | 感知发音准确度、口音、犹豫、节奏 | Memory（进度）、RAG（教材）、后台任务（复习清单） |
| 心理陪伴 | 语气比文字重要——"我没事"的平静 vs 颤抖完全不同 | Memory（背景）、Context 注入（调整风格）、情绪追踪 |
| 面试模拟 | 感知语速、停顿、自信程度 | RAG（面试题库）、Memory（表现记录）、反馈报告 |
| 老人陪伴 | 说话含糊、方言重，STT 丢失太多信息 | Memory（健康/家庭）、定时任务（主动问候）、RAG（健康知识） |

### 共性模式

所有 Realtime 场景的需求本质一样：

```
Realtime API（原生音频能力）
     +
Sigma 三层包裹：
  1. 会话前：Memory + RAG → System Prompt 注入
  2. 会话中：Tool Execution（RAG/Memory/Task）
  3. 会话后：Transcript → 知识沉淀
```

---

## 会话中的 Tool Calling

Realtime API 支持 function calling — 模型判断需要查资料时触发 tool call，Sigma 执行后返回结果。

暴露给 Realtime Session 的 tools：

```python
tools = [
    {"name": "search_knowledge", "description": "搜索用户的资料库"},
    {"name": "recall_memory", "description": "回忆之前对话中提到的信息"},
    {"name": "create_task", "description": "创建后台任务"},
    {"name": "save_note", "description": "记录重要信息供以后回忆"},
]
```

---

## LangGraph Pipeline 详细设计

### 级联模式主图

```python
graph = StateGraph(PipelineState)

# Nodes
graph.add_node("context_builder", context_builder_node)
graph.add_node("agent", agent_node)
graph.add_node("respond", respond_node)
graph.add_node("create_task", create_task_node)

# Edges
graph.add_edge(START, "context_builder")
graph.add_edge("context_builder", "agent")
graph.add_conditional_edges("agent", should_go_background, {
    "respond": "respond",
    "create_task": "create_task",
})
```

Agent 判断：这个任务能立刻做完还是需要后台？

### Context Builder 子图

内部并行执行多个检索：

```python
context_graph = StateGraph(ContextState)

context_graph.add_node("recall_memory", memory_node)
context_graph.add_node("search_rag", rag_node)
context_graph.add_node("compress_history", history_node)
context_graph.add_node("assemble", assemble_node)

# Memory、RAG、History 并行，然后汇总
context_graph.add_edge(START, "recall_memory")
context_graph.add_edge(START, "search_rag")
context_graph.add_edge(START, "compress_history")
context_graph.add_edge("recall_memory", "assemble")
context_graph.add_edge("search_rag", "assemble")
context_graph.add_edge("compress_history", "assemble")
```

### 后台任务图

```python
background_graph = StateGraph(TaskState)

background_graph.add_node("plan", plan_node)
background_graph.add_node("execute", execute_node)
background_graph.add_node("reflect", reflect_node)

background_graph.add_edge(START, "plan")
background_graph.add_edge("plan", "execute")
background_graph.add_edge("execute", "reflect")
background_graph.add_conditional_edges("reflect", should_continue, {
    "continue": "execute",
    "done": END,
})
```

支持 checkpoint，可断点续传。

---

## 概念映射表

| 架构概念 | 代码位置 | 职责 |
|---------|---------|------|
| Voice Pipeline（流控） | `src/runtime/voice_pipeline.py` | 音频流管理，拿到文本后触发 Pipeline |
| Text Pipeline | `src/runtime/text_pipeline.py` | 直接触发 LangGraph Pipeline |
| Context Builder Node | `src/context/` | Pipeline 中的前置 node（内部为子图） |
| Agent Node | `src/agent/` | Pipeline 核心 node（内部可嵌套子图） |
| Task Manager | `src/runtime/task_manager.py` | 管理后台图的执行 |
| Realtime Middleware | 新增 | WebSocket 事件拦截 + tool 分发 |

---

## 共享 Ports 层

两种模式共享同一套 Ports，区别只在编排层：

```
┌─ 级联模式 ──────────────┐     ┌─ Realtime 模式 ──────────────────┐
│  LangGraph Pipeline      │     │  WebSocket + Middleware           │
│  (图编排)                │     │  (tool executor)                  │
└───────────┬──────────────┘     └───────────┬──────────────────────┘
            │                                 │
            └──────────── 共享 ───────────────┘
                            │
                            ▼
              ┌─── Ports Layer ───────────┐
              │  MemoryPort               │
              │  RetrievalPort (RAG)      │
              │  ToolsPort                │
              │  LLMPort                  │
              └───────────────────────────┘
```

---

## 配置

```yaml
runtime:
  mode: cascade        # cascade | realtime

  cascade:
    pipeline: langgraph

  realtime:
    provider: openai   # openai | gemini
    model: gpt-4o-realtime
    middleware:
      tools: [search_knowledge, recall_memory, create_task, save_note]
      transcript: true
      post_session:
        extract_memory: true
        generate_report: false

# 共享能力配置
memory:
  provider: local
retrieval:
  provider: llamaindex
agent_runtime:
  provider: langgraph
```

---

## 架构约束验证

| 约束 | 满足 | 说明 |
|------|------|------|
| `core/` 不依赖外部框架 | ✅ | core 定义 state schema，不引用 LangGraph |
| `runtime/` 只 import `ports/` | ✅ | 图中 node 调用 ports 接口，不直接 import 域实现 |
| 域之间不互引 | ✅ | 域通过图的 state 流通信，不直接 import |
| 配置驱动 | ✅ | config 选择走级联还是 realtime，选择哪些 adapter |

## 相关文档

- [架构总览](overview.md)
- [Ports & Adapters](ports-and-adapters.md)
- [Voice 模块](../modules/voice/)
- [Agent 模块](../modules/agent/)
