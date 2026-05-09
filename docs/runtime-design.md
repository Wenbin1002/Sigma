# Runtime 编排设计：LangGraph + Realtime 双模式

## 设计哲学

**Every agent is a graph.** 用户的 input → output 之间是一个黑盒，内部用 LangGraph 表达为有向图。图中每个 node 是一个能力单元（Context Builder、Agent、Post-process 等），node 内部可以嵌套子图。

两个核心洞察：

1. **实时性和复杂性天然互斥** — 需要马上回复的场景本身不复杂，复杂的场景不需要马上回复
2. **Realtime Audio 和级联 Audio 是两种编排方式，不是两套系统** — 它们共享同一套能力层（Memory/RAG/Tools）

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

整个认知流程（从拿到文本到产出回复）用 LangGraph 编排。Voice Pipeline 的流控（打断、流式播放）仍由 async 代码处理，因为实时音频对延迟的要求不适合图模型。

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
│                                                          │
└──────────────────────────────────────────────────────────┘

┌─── 会话后 ──────────────────────────────────────┐
│                                                   │
│  [Transcript] → [提取记忆点] → Memory 存储       │
│               → [生成报告/笔记] → 通知用户        │
│               → [更新用户画像]                    │
│                                                   │
└───────────────────────────────────────────────────┘
```

Sigma 在 Realtime 模式下的角色是 **Middleware**：不替代模型对话，而是包裹它 — 前面注入记忆，中间响应 tool call，后面沉淀知识。

---

## Realtime 模式的使用场景

核心判断标准：**模型需要"听到"声音本身（而不只是文字）才能做好这件事。**

### 1. 语言学习

- **为什么需要原生音频**：感知发音准确度、口音、犹豫、节奏
- **Sigma 补什么**：
  - Memory：记住用户水平、学习进度、反复犯的错
  - RAG：加载教材/词库
  - 后台任务：会话后提取错误、生成复习清单

### 2. 心理陪伴 / 情绪支持

- **为什么需要原生音频**：语气比文字重要 10 倍。"我没事" 的平静语气 vs 颤抖语气完全不同
- **Sigma 补什么**：
  - Memory：用户背景、过往关键事件、情绪趋势
  - Context 注入：根据积累的了解调整对话风格
  - 后台任务：每次会话后提取情绪标签，追踪趋势

### 3. 面试模拟 / 演讲练习

- **为什么需要原生音频**：感知语速、停顿、嗯嗯啊啊、自信程度
- **Sigma 补什么**：
  - RAG：目标公司/岗位资料、面试题库
  - Memory：过往表现记录
  - 后台任务：生成结构化反馈报告，对比上次进步

### 4. 老人 / 无障碍陪伴

- **为什么需要原生音频**：说话含糊、语速不规律、方言重，STT 丢失太多信息
- **Sigma 补什么**：
  - Memory：健康信息、家庭成员、日常习惯
  - 定时任务：主动发起对话（提醒吃药、问候）
  - RAG：健康知识、用药说明

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

Realtime API 支持 function calling — 模型判断需要查资料时触发 tool call，Sigma 执行后返回结果，模型继续对话。

暴露给 Realtime Session 的 tools：

```python
tools = [
    {
        "name": "search_knowledge",
        "description": "搜索用户的资料库（笔记、文档、教材等）",
    },
    {
        "name": "recall_memory",
        "description": "回忆之前对话中提到的信息",
    },
    {
        "name": "create_task",
        "description": "创建后台任务，不需要立即完成的事",
    },
    {
        "name": "save_note",
        "description": "记录重要信息供以后回忆",
    },
]
```

用户体验示例：
- 用户："我之前跟你说过的那个单词怎么拼来着？"
- 模型：（触发 `recall_memory`）→ Sigma 查 Memory → 模型继续说
- 用户："帮我把今天的练习整理成文档"
- 模型：（触发 `create_task`）→ Sigma 创建后台任务 → 模型："好的，完成后通知你"

---

## LangGraph Pipeline 详细设计

### 级联模式下的主图

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

Agent 自己判断：这个任务能立刻做完还是需要后台？
- 简单请求 → 直接回复
- 复杂任务 → 创建后台 Task，告诉用户"在后台处理了"

### Context Builder 子图

Context Builder 本身也是一张图，内部并行执行多个检索：

```python
context_graph = StateGraph(ContextState)

context_graph.add_node("recall_memory", memory_node)
context_graph.add_node("search_rag", rag_node)
context_graph.add_node("compress_history", history_node)
context_graph.add_node("assemble", assemble_node)

# Memory、RAG、History 并行执行，然后汇总
context_graph.add_edge(START, "recall_memory")
context_graph.add_edge(START, "search_rag")
context_graph.add_edge(START, "compress_history")
context_graph.add_edge("recall_memory", "assemble")
context_graph.add_edge("search_rag", "assemble")
context_graph.add_edge("compress_history", "assemble")
```

### 后台任务图

后台 Task 执行更复杂的图：

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

## 与现有架构的映射

| 现有概念 | LangGraph 视角 |
|---------|----------------|
| `runtime/voice_pipeline.py` | 音频流控（async 事件循环）→ 拿到文本后触发 LangGraph Pipeline |
| `runtime/text_pipeline.py` | 直接触发 LangGraph Pipeline |
| `context/` | Pipeline 中的一个 node（内部为子图） |
| `agent/` | Pipeline 中的核心 node（内部可嵌套子图） |
| `runtime/task_manager.py` | 管理后台图的执行 |
| Realtime Middleware（新增） | WebSocket 事件拦截 + tool 分发 |

### 架构约束验证

| 约束 | 是否满足 | 说明 |
|------|---------|------|
| core/ 不依赖外部框架 | ✅ | core 定义 state schema（PipelineState 等），不引用 LangGraph |
| runtime/ 只 import ports/ | ✅ | 图中 node 调用 ports 接口，不直接 import 域实现 |
| 域之间不互引 | ✅ | 域通过图的 state 流通信，不直接 import |
| 配置驱动 | ✅ | config 选择走级联还是 realtime，选择哪些 adapter |

### 共享 Ports 层

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
              │  TracePort                │
              └───────────────────────────┘
```

---

## 配置示例

```yaml
# 模式选择
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
