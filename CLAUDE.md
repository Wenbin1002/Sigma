# Sigma

通用 personal AI assistant — 对标 ChatGPT + Codex 集成体的本地版本。
Chat + Task + Realtime 三种交互模式 / 基于 LangGraph 的 agent 内核 / Tool · Skill · Agent 三层正交扩展 / Self-improvement 闭环。

## 硬性规则（违反即阻断）

### Git 工作流

- **禁止**在 main 上直接 commit — 所有变更必须先建分支，走 PR 合入
- 分支命名：`<type>/<short-desc>`（type: feat/fix/refactor/chore/perf/test/docs）
- 流程：需求分析 → 建分支 → 开发 → Commit → Review → 提 PR
- 严格遵守 [skills/git-workflow/SKILL.md](skills/git-workflow/SKILL.md)

### 依赖规则（精简版）

- **`core/`** 只 import 标准库，禁止任何外部依赖
- **`ports/`** 只 import `core/`，不依赖任何具体实现
- **域目录**（`llm/`, `tools/`, `trace/`, `checkpoint/`）只 import `ports/` + `core/` + 外部 SDK，**互相禁止 import**
- **内核模块**（`agent/`, `task/`, `context/`, `rag/`, `memory/`, `skill/`, `chat/`）可互相 import；调域实现走 registry
- **`server/`** 调内核模块；**`app/`** 只走 `server/` 暴露的 HTTP/SSE API

详见 [docs/architecture/dependency-rules.md](docs/architecture/dependency-rules.md)。

### 新增能力的标准步骤

新增能力分三种：**Tool / Skill / Agent**（详见 [架构总览 § 4](docs/architecture/overview.md#4-扩展模型tool--skill--agent-三层正交)）。

**新 Tool**（Python 函数 / MCP 接入）：
```
1. 写 adapter → src/tools/<name>.py 实现 ToolPort
2. 注册 → src/tools/registry.py
3. 配置 → config.yaml 的 tools.builtin
4. 测试 → tests/tools/test_<name>.py
```

**新 Skill**（纯 markdown，**不写 Python**）：
```
1. 创建目录 → ~/.sigma/skills/<name>/
2. 写 SKILL.md（含 metadata: name / description / triggers）
3. 可选：references/ 资源文件
4. Sigma 启动后自动扫描，无需注册
```

**新 Sub-agent**（继承 Agent 类）：
```
1. 创建目录 → ~/.sigma/agents/<name>/
2. 写 agent.py，继承 sigma.Agent，定义 graph
3. 声明 metadata（name / description / triggers / tools）
4. Sigma 启动后自动扫描
```

**新 LLM provider**（多 provider 是真实需求）：
```
1. 写 adapter → src/llm/<provider>.py 实现 LLMPort
2. 注册 → src/llm/registry.py
3. 配置 → config.yaml 的 llm.default 或 llm.routing
4. 测试 → tests/llm/test_<provider>.py
```

## 项目结构

```
src/
  core/               # 稳定内核：数据结构（AgentChunk / Task / Context / BlockedException 等）
  
  ports/              # Protocol 定义层（只放真有多实现的能力）
    llm.py            #   LLMPort
    tools.py          #   ToolPort
    tracer.py         #   TracerPort
    checkpointer.py   #   CheckpointerPort
  
  llm/                # LLM 域：OpenAI / DeepSeek / Anthropic 等 adapter
  tools/              # 内置 tool + MCP 接入
  trace/              # Trace 实现（JSONL + viewer 后端）
  checkpoint/         # Checkpointer 实现（基于 LangGraph）
  
  agent/              # Master / Supervisor / Sub-agent 框架
  skill/              # Skill loader：扫描 / metadata 注入 / 渐进式加载
  task/               # Task 引擎：状态机 / queue / pause-resume
  chat/               # Chat 引擎：对话流维护 / 升级建议
  context/            # Context engine：多源拼装
  rag/                # 多 index 管理
  memory/             # 分层 memory（global / session / task）
  realtime/           # Realtime middleware（V4 起）
  improvement/        # Self-improvement（V2 起）
  
  server/             # Core HTTP/SSE server（CLI 和 Web UI 共用）
  
  app/                # 客户端
    cli.py            #   Phase 1
    web.py            #   Phase 2
  
experiments/          # 平行实验（不影响主线）
tests/                # 测试
config.yaml           # 全局配置
```

## 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| Port 接口 | `XxxPort` | `LLMPort`, `ToolPort`, `TracerPort` |
| Adapter 实现 | 描述性名称 | `OpenAILLM`, `MCPTool`, `JSONLTracer` |
| Sub-agent 类 | `XxxAgent` | `ResearcherAgent`, `CoderAgent` |
| 注册表 | `REGISTRY: dict[str, type]` | 位于 `src/<domain>/registry.py` |
| 工厂函数 | `create(config) -> XxxPort` | 位于 `src/<domain>/registry.py` |
| 配置键 | snake_case | `llm.default.provider` |

## DO / DON'T

### DO

- **真有 ≥2 个实现需求才做 Port**——不要为"未来可能"抽象
- Port 入参出参用可序列化类型：`str / bytes / int / float / bool / list / dict / dataclass`
- 流式输出用 `AsyncIterator[AgentChunk]`
- 一个 adapter 一个文件
- 每个 adapter 必须注册到 registry
- 新增 sub-agent 必须声明 `name` / `description` / `triggers`（Supervisor 路由依赖）
- Sub-agent 卡住时 raise `BlockedException` 带结构化 reason，不要 raise 通用异常

### DON'T

- 不要在 Port 接口传 callback、文件句柄、Python 特有对象
- 不要让域目录互相 import（`llm/` ↔ `tools/` 之类）
- 不要在 `core/` 放业务逻辑或外部依赖
- 不要让 `app/` 直接调内核模块（必须走 `server/`）
- 不要在 main 分支直接提交代码
- 不要把"复杂任务的 skill"写成 skill——skill 是 prompt，复杂任务该写成 agent

## 架构速览

```
Clients (CLI / Web UI)
    ↓ HTTP / SSE
Server
    ↓
┌──────────────┬──────────────┐
│  Chat Engine │  Task Engine │   ← Chat / Task 双视图入口
│              │  (queue +    │
│              │   状态机)     │
└──────┬───────┴───────┬──────┘
       └───────┬───────┘
               ▼
       Master Agent
         ↓ Supervisor 自动 / @-mention 显式
       Sub-agents (Researcher / Coder / ...)
         内部消费 ↓
       Tools / Skills（注入 prompt）
         ↓
       LLM (多 provider)

横切肌肉：Trace / Cost Guard / Streaming / Checkpoint / Cancellation
```

**三层扩展正交**（详见 [架构总览](docs/architecture/overview.md)）：
- **Tool** — 行为扩展（能做什么）
- **Skill** — 知识扩展（怎么做）
- **Agent** — 执行单元扩展（谁来做）

**Multi-agent 三级回退**（详见 [Multi-agent](docs/features/multi-agent.md)）：
1. Sub-agent 自决
2. Sub-agent → 主 agent 代答
3. 主 agent → 用户（chat 追问 / task pause）

## 文档索引

| 主题 | 路径 |
|------|------|
| 文档导航 | [docs/index.md](docs/index.md) |
| 架构总览 | [docs/architecture/overview.md](docs/architecture/overview.md) |
| Chat / Task 模式 | [docs/features/chat-task-modes.md](docs/features/chat-task-modes.md) |
| Realtime 模式 | [docs/features/realtime-mode.md](docs/features/realtime-mode.md) |
| Multi-agent | [docs/features/multi-agent.md](docs/features/multi-agent.md) |
| Self-improvement | [docs/features/self-improvement.md](docs/features/self-improvement.md) |
| Ports & Adapters | [docs/architecture/ports-and-adapters.md](docs/architecture/ports-and-adapters.md) |
| 依赖规则 | [docs/architecture/dependency-rules.md](docs/architecture/dependency-rules.md) |
| 设计决策日志 | [docs/architecture/design-log.md](docs/architecture/design-log.md) |
| 新人指南 | [docs/guides/getting-started.md](docs/guides/getting-started.md) |
| 添加扩展（Tool/Skill/Agent） | [docs/guides/adding-an-adapter.md](docs/guides/adding-an-adapter.md) |
| 代码规范 | [docs/guides/code-style.md](docs/guides/code-style.md) |
| Agent 模块 | [docs/modules/agent/](docs/modules/agent/) |
| Skill 模块 | [docs/modules/skill/](docs/modules/skill/) |
| Task 模块 | [docs/modules/task/](docs/modules/task/) |
| Chat 模块 | [docs/modules/chat/](docs/modules/chat/) |
| Realtime 模块 | [docs/modules/realtime/](docs/modules/realtime/) |
| Improvement 模块 | [docs/modules/improvement/](docs/modules/improvement/) |
| Context 模块 | [docs/modules/context/](docs/modules/context/) |
| RAG 模块 | [docs/modules/rag/](docs/modules/rag/) |
| Memory 模块 | [docs/modules/memory/](docs/modules/memory/) |
| Tools 模块 | [docs/modules/tools/](docs/modules/tools/) |
| LLM 模块 | [docs/modules/llm/](docs/modules/llm/) |
| Trace 模块 | [docs/modules/trace/](docs/modules/trace/) |
| 路线图 | [docs/roadmap.md](docs/roadmap.md) |
| 贡献指南 | [CONTRIBUTING.md](CONTRIBUTING.md) |
