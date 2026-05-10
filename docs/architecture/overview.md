# 架构总览

> Sigma 是一个**通用 personal AI assistant**，对标 ChatGPT + Codex 集成体的本地版本。
>
> 这份文档是入口，描述产品形态、内核结构、扩展模型。细节展开请走链接。

---

## 1. Sigma 是什么

### 是

- **本地跑的通用 AI 助手**，覆盖日常对话 + 长任务执行 + 实时语音陪伴 + 代码场景
- **Chat / Task / Realtime 三种交互模式**：异步思考 / 后台执行 / 实时陪伴
- **基于 LangGraph 的 agent 内核**，自己长 trace / cost guard / streaming 协议等横切肌肉
- **三层正交扩展**：Tool（行为）+ Skill（知识）+ Agent（执行单元）
- **Self-improvement**：从用户互动中学习（V2 起，V4 跟 Realtime 一起完整闭环）
- **学习项目**，不追求生产级性能 / 不做 hosted service

### 不是

- ❌ 不是"可插拔 AI Agent 框架"（早期定位，已废弃）
- ❌ 不是 coding agent（不锁场景）
- ❌ 不是工作流编辑器（无可视化拖拽）
- ❌ 不是 LangGraph 的劣化版（差异化在 trace viewer / cost guard / 三层扩展模型 / 教学优先）

### 锚定场景（V1 必须跑通的 4 个 demo）

| # | 场景 | 模式 |
|---|---|---|
| 1 | 每天早上从社交媒体（小红书/X/知乎）筛选感兴趣内容并推送 | task（周期性） |
| 2 | 查询生猪期货数据，生成趋势图，判断供需走向 | task（一次性） |
| 3 | 多轮讨论一本书，最终总结笔记 | chat |
| 4 | Coding（写代码、改代码、跑代码） | chat + task |

场景跨度大 = Sigma 必须做"通用"，不能为单一场景优化。

---

## 2. 产品形态

### 2.1 三种交互模式

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Chat            │  │  Task            │  │  Realtime        │
│  (异步对话)       │  │  (后台执行)       │  │  (实时语音陪伴)   │
├──────────────────┤  ├──────────────────┤  ├──────────────────┤
│ · 默认入口        │  │ · 长任务          │  │ · 双向音频流      │
│ · 文本中心        │  │ · 周期任务        │  │ · 感知语气        │
│ · 可建议升级 task │  │ · 可暂停 / 恢复   │  │ · 可打断          │
│ · 同步 < 秒级     │  │ · 异步 / 不阻塞   │  │ · 同步 < 500ms   │
└────────┬─────────┘  └────────┬─────────┘  └─────────┬────────┘
         │                     │                       │
         └──── 共享 ───────────┴────── memory / RAG / agent
                                       trace / cost / improvement
```

- **Chat / Task 互通**：统一 queue，可建议升级，详见 [Chat / Task 模式](chat-task-modes.md)
- **Realtime 独立**：实时性要求决定它无法 multi-agent 协作，跟 Chat / Task 是平级关系，详见 [Realtime 模式](realtime-mode.md)
- **三模式共享底层**：memory / RAG / trace / cost guard / improvement 是统一的内核

### 2.2 交互形态分阶段

| Phase | 形态 | 入口 |
|---|---|---|
| **Phase 1** | **CLI** | `sigma chat` / `sigma task new "..."` / `sigma task list` |
| **Phase 2** | **Web UI** | `sigma serve` → `localhost:7777` |
| **Phase 3** | **Realtime CLI/Web** | `sigma realtime start`（V4） |

**关键设计**：从 day 1 就 client/server 分离。Core 跑成 server，CLI 是个 client；Web UI 是另一个 client；Realtime 在 server 端开 WebSocket。三端共用同一套 HTTP/SSE/WebSocket API。

明确**不做 TUI / 桌面 app / MCP server**作为主线。

---

## 3. 内核架构

### 3.1 总览

```
┌────────────────────────────────────────────────────────────────┐
│  Clients                                                        │
│   CLI（Phase 1）       Web UI（Phase 2）   Realtime（Phase 3）  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP / SSE / WebSocket
┌──────────────────────────────▼──────────────────────────────────┐
│  Sigma Core Server                                              │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐     │
│  │ Chat Engine │  │ Task Engine │  │ Realtime Middleware  │     │
│  │             │  │ (queue +    │  │ (会话前注入 / 中拦截 │     │
│  │             │  │  状态机)    │  │  / 后沉淀)           │     │
│  └──────┬──────┘  └──────┬──────┘  └─────────┬────────────┘     │
│         └─────────┬──────┘                   │                  │
│                   ▼                          │                  │
│         ┌──────────────────────┐             │                  │
│         │  Master Agent        │             │ (Realtime 走单   │
│         │  ┌────────────────┐  │             │  agent + 强      │
│         │  │  Supervisor    │ ←┼── @-mention │  tool calling)   │
│         │  └───────┬────────┘  │             │                  │
│         └─────────┼────────────┘             │                  │
│                   ▼                          ▼                  │
│            ┌──────┴──────┐         ┌──────────────────┐         │
│            │ Sub-agents   │         │ Realtime Provider│         │
│            │ (researcher  │         │ (OpenAI / Gemini)│         │
│            │  / coder /…) │         └────────┬─────────┘         │
│            └──────────────┘                  │                   │
│                   │                          │                   │
│                   └────────┬─────────────────┘                   │
│                            ▼                                     │
│                  Tools / Skills / LLM                            │
│                                                                  │
│  Cross-cutting (Sigma 自己长的肌肉):                             │
│  - Trace（每个 node IO/duration/cost，配本地 viewer）            │
│  - Cost Guard（token 累加 + budget 拦截）                        │
│  - Streaming（统一 AgentChunk 协议）                              │
│  - Checkpoint（task pause/resume）                                │
│  - Self-improvement（signal 提取 → memory / agent 调整）         │
│                                                                  │
│  Internal infrastructure:                                        │
│  - LangGraph（state machine / BSP / subgraph / interrupt）       │
│  - Context Engine（多 RAG index + 分层 memory）                   │
│  - Skill Loader（渐进式：metadata 进 prompt，全文按需加载）       │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Sigma 自己造 vs 直接用 LangGraph

| 自己长的肌肉 | 用 LangGraph 提供的 |
|---|---|
| Trace（callback 挂 LangGraph events，输出 JSONL + HTML viewer） | Graph 编排 / state machine |
| Cost guard（LLM 调用包装算 token，超预算 interrupt） | Checkpointer 接口（SqliteSaver 等） |
| 统一 Streaming 协议（map LangGraph events 到 `AgentChunk`） | BSP 并行 / fan-out / fan-in |
| 三层扩展系统（Tool / Skill / Agent） | Subgraph 嵌套 |
| 多 RAG index + 分层 memory | Cancellation / interrupt / `Command(resume=...)` |
| 本地 trace viewer | — |

**Phase 1 明确不做**：自己造 Graph engine、Hot reload、Multi-agent orchestration 内核（用 LangGraph supervisor 即可）。

---

## 4. 扩展模型：Tool / Skill / Agent 三层正交

这是 Sigma 跟其他产品的核心差异之一。

### 4.1 三者本质不同（不是粒度递进）

| | **Tool** | **Skill** | **Agent** |
|---|---|---|---|
| 扩展什么 | **行为**（能做什么） | **知识/方法论**（怎么做） | **执行单元**（谁来做） |
| LLM 调用次数 | 0 | 0（被注入到调用方 prompt） | 多次（自己有 reasoning loop） |
| 状态机 | 无 | 无 | 有 |
| 写起来 | Python 函数 | markdown 文件 | Python 类 + graph |
| 用户门槛 | 低 | 极低 | 高 |

**三者正交**：一个 agent 内部可以同时引用 N 个 tool 和 M 个 skill。

### 4.2 Tool

**形态**：Python 函数（带装饰器） / MCP server。

```python
@sigma.tool
def send_email(to: str, subject: str, body: str) -> bool:
    ...
```

也支持挂载 MCP server，复用 MCP 生态。

### 4.3 Skill

**形态**：纯文件夹 + markdown，**0 Python 接口**。

```
~/.sigma/skills/
  research/
    SKILL.md         ← 一段 markdown：方法论 / 风格 / 调用哪些 tool
    references/      ← 可选的更多详细文档
  writing/
    SKILL.md
    templates/       ← 可选的资源文件
```

**渐进式加载**：
1. 启动时把每个 skill 的 **name + description** 注入 system prompt
2. Agent 判断要用哪个 → 调内置 `load_skill(name)` tool 加载完整 SKILL.md 内容
3. 防止 context 一次性塞满所有 skill

跟 Claude Code skill 一致的形态——**用户写 skill 不用写一行 Python**。

### 4.4 Agent

**形态**：Python 类继承 `Agent`，定义自己的 graph、prompt、tool 子集。

```python
class ResearcherAgent(sigma.Agent):
    name = "researcher"
    description = "擅长资料收集和总结"
    tools = [search_web, summarize, fetch_pdf]
    
    def build_graph(self) -> StateGraph:
        ...
```

**对外契约**：
- 输入：task description
- 输出：result（artifact / 报告 / 数据）
- 内部：完全黑盒（自己的 reasoning loop / state / memory scope）

详见 [Multi-agent](multi-agent.md)。

### 4.5 对外区分 / 对内统一

```
对外（产品 / 扩展接口）：     区分 Tool / Skill / Agent
                          ↓ 各自有不同的注册方式 / 生命周期 / 语义保证
对内（runtime / 调度）：      统一抽象成 LangGraph Node
                          ↓ 统一 trace / checkpoint
```

不分会出三个问题：
- 扩展接口模糊（用户不知道写啥）
- 调用语法塌陷（LLM 看到几十个"node"会傻）
- Trace / cost / timeout 一视同仁（粒度差几个数量级混在一起）

---

## 5. Multi-agent 设计要点

### 5.1 路由：Supervisor 自动 + @-mention 显式

```
入口 A（自动）：用户自由聊天
  User: 帮我分析一下生猪期货
    ↓ Master Agent 判断意图
    ↓ Supervisor 路由
    ↓ 派给 Analyst Agent

入口 B（显式）：用户精确指定
  User: @researcher 帮我查一下数据
    ↓ 直接派给 Researcher Agent（跳过 Supervisor）
```

### 5.2 Sub-agent 沟通：三级回退

```
Sub-agent 遇到不确定 / 缺信息
   ↓
Level 1: sub-agent 自己尽力（合理默认 / 推理）
   ↓ 仍然不行
Level 2: bubble up 到主 agent，主 agent 用历史上下文代答
   ↓ 主 agent 也搞不定 / 需要外部资源（API key / 登录 / 文件）
Level 3: bubble up 到用户
         · Chat mode：直接追问
         · Task mode：task 进入 paused 状态，等用户 resume
```

**两类卡住区分**：
- 认知性卡住（决策歧义）→ L1 → L2 → L3
- 资源性卡住（API key 缺失、登录态过期、文件不存在）→ 直接 L3

**主 agent 代答审计**：必须在 trace / task summary / chat 里显式说明"我替你决定了 X"，保留用户 override 机会。

详见 [Multi-agent](multi-agent.md)。

---

## 6. Realtime 模式（V4 起）

> 第三种交互模式。详见 [Realtime 模式](realtime-mode.md)。

**形态**：在 OpenAI Realtime / Gemini Live 等原生 realtime API 之上做 middleware：

```
会话前  →  注入 memory + RAG 到 system prompt
会话中  →  拦截 tool calls / 提取 audio signals
会话后  →  transcript 沉淀 → memory + self-improvement signal
```

**Sigma 的差异化**：原生 Realtime API 没记忆、没 RAG、不学习；Sigma 补这三层。

**Killer 场景**：语言陪练 / 心理陪伴 / 面试模拟 / 老人陪伴 / 辅导孩子作业（每个都需要长期记忆 + 个性化）。

**Realtime 跟 Chat / Task 的区别**：实时性要求决定它**不能 multi-agent 协作**——每个 sub-agent spawn 引入的延迟跟实时对话矛盾。Realtime 走"单 agent + 强 tool calling + 强 memory" 模式。

---

## 7. Self-Improvement

> Sigma 真正的差异化能力之一：**让 agent 越用越懂用户**。详见 [Self-improvement](self-improvement.md)。

**signal 来源**：
- 显式（用户明说"以后用更专业术语"）
- 隐式（行为模式：重复问 / 转话题）
- Audio metadata（Realtime 独有：打断 / 沉默 / 笑声）
- Task 反馈（取消 / 修改 description）

**反馈到**：
- Memory（主要）
- Agent metadata（次要：调短风格 / 改 prompt）
- 推送过滤（场景 1 关键）

**关键洞察**：Realtime 是 self-improvement 最容易看到效果的场景——反馈密度极高（每分钟几十次互动信号）。

---

## 8. 核心能力模块

| 模块 | 角色 | 文档 |
|---|---|---|
| **Agent** | Master agent + Sub-agent 框架；reasoning loop；三级回退 | [agent/](../modules/agent/) |
| **Skill** | 纯文件夹+markdown 扩展；渐进式加载 | [skill/](../modules/skill/) |
| **Task** | Chat 之外的一等公民模式；状态机；pause/resume；周期任务 | [task/](../modules/task/) |
| **Chat** | 对话流 / session / 升级建议 / 完成回流 | [chat/](../modules/chat/) |
| **Realtime** | 实时语音 middleware；前注入 / 中拦截 / 后沉淀 | [realtime/](../modules/realtime/) |
| **Improvement** | Signal 提取 / 反馈路由 / 审计 | [improvement/](../modules/improvement/) |
| **Context** | Context engineering：把"该看的"塞进 LLM；多源拼装 | [context/](../modules/context/) |
| **RAG** | 多 index 分层（书的全文 / 代码库 / 行业知识等） | [rag/](../modules/rag/) |
| **Memory** | 三层（全局偏好 / session 上下文 / task 临时 state） | [memory/](../modules/memory/) |
| **Tools** | Python 函数 + MCP 接入 | [tools/](../modules/tools/) |
| **LLM** | 多 provider；cost-aware 路由；streaming 统一 | [llm/](../modules/llm/) |
| **Trace** | 每个 node IO/duration/cost；JSONL + 本地 viewer；可 replay | [trace/](../modules/trace/) |

---

## 9. 项目结构

```
src/
  core/               # 稳定内核：数据结构（AgentChunk / Task / Context 等）、事件
  
  ports/              # Protocol 定义层（只放真有多实现的能力）
    llm.py            #   LLMPort（多 provider）
    tools.py          #   ToolPort（Python 函数 / MCP）
    tracer.py         #   TracerPort（JSONL / OTel / ...）
    checkpointer.py   #   CheckpointerPort（SqliteSaver / ...）
  
  llm/                # LLM 域：OpenAI / DeepSeek / Anthropic 等
  tools/              # 内置工具 + MCP 接入
  trace/              # Trace 实现（JSONL + viewer 后端）
  checkpoint/         # Checkpointer 实现
  
  agent/              # Agent 框架：Master / Supervisor / Sub-agent 基类
  skill/              # Skill loader：扫描 / metadata 注入 / 渐进式加载
  task/               # Task 引擎：状态机 / queue / pause/resume
  chat/               # Chat 引擎：session / 升级建议 / 回流
  context/            # Context engine：多源拼装
  rag/                # RAG 系统：多 index 管理
  memory/             # Memory：分层存储
  realtime/           # Realtime middleware（V4）
  improvement/        # Self-improvement（V2 起）
  
  server/             # Core HTTP/SSE/WebSocket server
  
  app/                # 客户端
    cli.py            #   Phase 1：CLI
    web.py            #   Phase 2：Web UI
  
experiments/          # 平行实验（不影响主线）
tests/                # 测试
config.yaml           # 全局配置
```

详见 [依赖规则](dependency-rules.md)、[Ports & Adapters](ports-and-adapters.md)。

---

## 10. 相关文档

- [Chat / Task 模式](chat-task-modes.md) — 异步双视图
- [Realtime 模式](realtime-mode.md) — 实时语音陪伴
- [Multi-agent](multi-agent.md) — Supervisor / @-mention / 三级回退
- [Self-improvement](self-improvement.md) — Signal 提取与反馈
- [Ports & Adapters](ports-and-adapters.md) — 哪些能力有 Port、为什么
- [依赖规则](dependency-rules.md) — Import 约束
- [设计决策日志](design-log.md) — 每个设计选择的演化和理由
- [路线图](../roadmap.md) — 版本规划
