# Sigma

> 通用 personal AI assistant — 对标 ChatGPT + Codex 集成体的本地版本。
>
> 学习项目，不追求生产级性能；架构透明，每个设计决策都有 [演化记录](docs/architecture/design-log.md)。

## 是什么

- **本地跑的 AI 助手**，覆盖日常对话 + 长任务执行 + 实时语音陪伴 + 代码场景
- **Chat / Task / Realtime 三种交互模式**：异步思考 / 后台执行 / 实时陪伴
- **基于 LangGraph 的 agent 内核**，自己长 trace / cost guard / streaming 等横切肌肉
- **三层正交扩展**：
  - **Tool**（行为：能做什么）— Python 函数 / MCP
  - **Skill**（知识：怎么做）— 纯 markdown，0 Python
  - **Agent**（执行单元：谁来做）— Python 类继承 + LangGraph
- **Self-improvement**：从用户互动中学习（V2 起，V4 跟 Realtime 完整闭环）

## 核心场景（V1 跑通的 4 个 demo）

| # | 场景 | 模式 |
|---|---|---|
| 1 | 每天早上从社媒（小红书/X/知乎）筛选感兴趣内容并推送 | task（周期） |
| 2 | 查询生猪期货数据，生成趋势图，判断供需走向 | task（一次） |
| 3 | 多轮讨论一本书，最终总结笔记 | chat |
| 4 | Coding（写代码、改代码、跑代码） | chat + task |

## 不是什么

- ❌ 不是"可插拔 AI Agent 框架"（早期定位，已废弃）
- ❌ 不是 coding agent（不锁场景）
- ❌ 不是工作流编辑器（无可视化拖拽）
- ❌ 不是 LangGraph 的劣化版（差异化在 trace viewer / cost guard / 三层扩展模型 / 教学优先）
- ❌ 不做 hosted service / 不做企业版

## 架构（极简版）

```
Clients (CLI Phase 1 / Web UI Phase 2 / Realtime Phase 3)
    ↓ HTTP / SSE / WebSocket
Sigma Core Server
    ├── Chat Engine          ← 用户随便聊
    ├── Task Engine          ← 长任务、周期任务、pause/resume
    ├── Realtime Middleware  ← 实时语音（V4）：前注入 / 中拦截 / 后沉淀
    └── Master Agent
         └── Supervisor 路由 / @-mention 显式
              └── Sub-agents（用户可写）
                   ↓ 内部消费
                  Tools / Skills / Memory / RAG
                   ↓
                  LLM (多 provider)

横切：Trace / Cost Guard / Streaming / Checkpoint / Self-improvement
```

详见 [架构总览](docs/architecture/overview.md)。

## Multi-agent 设计

**双入口路由**（详见 [Multi-agent](docs/architecture/multi-agent.md)）：
- Supervisor 自动判断意图，派给 sub-agent
- 用户也可 `@agent_name` 显式召唤

**Sub-agent 三级回退**：
1. Sub-agent 自己尽力（合理默认）
2. Bubble up 到主 agent，用历史 context 代答
3. Bubble up 到用户（chat 追问 / task pause）

资源性卡住（缺 API key 等）直接到 L3。代答必须审计。

## 项目状态

> 🚧 **早期设计阶段** — 架构设计基本定型，尚未开始编码

详见 [路线图](docs/roadmap.md)。

## Quick Start（待实现）

```bash
git clone <repo-url>
cd Sigma

pip install -e ".[dev]"
cp config.example.yaml config.yaml
# 填入 LLM API key

# Phase 1: CLI
sigma chat                          # 进入对话流
sigma task new "..."                # 新建一次性 task
sigma task new "..." --daily 09:00  # 每天 9 点跑
sigma task list                     # 任务列表

# Phase 2: Web UI（暂未实现）
sigma serve                         # http://localhost:7777
```

## 配置示例

```yaml
llm:
  default:
    provider: openai
    model: gpt-4o
  routing:                  # 可选：cost-aware 路由
    simple_tasks:    { provider: openai, model: gpt-4o-mini }
    complex_tasks:   { provider: anthropic, model: claude-opus-4 }

tools:
  builtin: [read_file, write_file, shell, search_web]
  mcp_servers:
    - { name: github, command: npx, args: ["@anthropic/mcp-github"] }

trace:
  provider: jsonl
  output_dir: ~/.sigma/traces/

cost_guard:
  per_task_budget_usd: 1.0
```

## 模块

| 模块 | 角色 |
|------|------|
| [Agent](docs/modules/agent/) | Master / Supervisor / Sub-agent 框架；三级回退 |
| [Skill](docs/modules/skill/) | 纯 markdown 扩展；渐进式加载 |
| [Task](docs/modules/task/) | 状态机 / queue / pause-resume / 周期调度 |
| [Chat](docs/modules/chat/) | 对话流 / session / 升级建议 / 回流 |
| [Realtime](docs/modules/realtime/) | 实时语音 middleware（V4 起） |
| [Improvement](docs/modules/improvement/) | Self-improvement 实现层 |
| [Context](docs/modules/context/) | Context engineering：多源拼装 |
| [RAG](docs/modules/rag/) | 多 index 分层 |
| [Memory](docs/modules/memory/) | 全局 / session / task 三层 |
| [Tools](docs/modules/tools/) | Python 函数 + MCP |
| [LLM](docs/modules/llm/) | 多 provider + cost-aware 路由 |
| [Trace](docs/modules/trace/) | JSONL + 本地 HTML viewer + replay |

## 文档

完整文档入口：[docs/index.md](docs/index.md)

| 主题 | 链接 |
|------|------|
| 架构总览 | [docs/architecture/overview.md](docs/architecture/overview.md) |
| Chat / Task 模式 | [docs/architecture/chat-task-modes.md](docs/architecture/chat-task-modes.md) |
| Realtime 模式 | [docs/architecture/realtime-mode.md](docs/architecture/realtime-mode.md) |
| Multi-agent | [docs/architecture/multi-agent.md](docs/architecture/multi-agent.md) |
| Self-improvement | [docs/architecture/self-improvement.md](docs/architecture/self-improvement.md) |
| 设计决策日志 | [docs/architecture/design-log.md](docs/architecture/design-log.md) |
| 贡献指南 | [CONTRIBUTING.md](CONTRIBUTING.md) |
| 路线图 | [docs/roadmap.md](docs/roadmap.md) |

## 参与贡献

请先阅读 [CONTRIBUTING.md](CONTRIBUTING.md)。

最容易上手的贡献方向：

- **写 skill**（最低门槛）：纯 markdown，0 Python，描述一个领域的方法论
- **接入新 LLM provider**：实现 `LLMPort` 的 adapter
- **写 sub-agent**：用 LangGraph 定义一个领域专家
- **接入 MCP server**：复用 MCP 生态

## License

MIT
