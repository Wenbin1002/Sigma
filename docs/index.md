# 文档导航

## 新人入门

| 文档 | 用途 |
|------|------|
| [README.md](../README.md) | Sigma 是什么、不是什么、快速上手 |
| [贡献指南](../CONTRIBUTING.md) | 从零开始参与贡献 |
| [新手指南](guides/getting-started.md) | 环境搭建、跑通 chat / task |
| [添加扩展](guides/adding-an-adapter.md) | 写 Tool / Skill / Sub-agent / LLM adapter |
| [代码规范](guides/code-style.md) | 命名、风格、约定 |

## 架构设计

| 文档 | 内容 |
|------|------|
| [架构总览](architecture/overview.md) | 定位、产品形态、内核结构、扩展模型 |
| [Chat / Task 模式](architecture/chat-task-modes.md) | 异步双视图、状态机、pause/resume |
| [Realtime 模式](architecture/realtime-mode.md) | 实时语音 + memory + RAG + self-improvement |
| [Multi-agent](architecture/multi-agent.md) | Supervisor + @-mention、三级回退 |
| [Self-improvement](architecture/self-improvement.md) | Signal 提取 / 反馈路由 / 审计 |
| [Ports & Adapters](architecture/ports-and-adapters.md) | 哪些能力做 Port、为什么；数据结构 |
| [依赖规则](architecture/dependency-rules.md) | Import 约束、违规修复 |
| [设计决策日志](architecture/design-log.md) | 每个设计选择的演化和理由 |

## 模块参考

| 模块 | 角色 | 文档 |
|------|------|------|
| Agent | Master / Supervisor / Sub-agent 框架 | [agent/](modules/agent/) |
| Skill | 纯 markdown 扩展、渐进式加载 | [skill/](modules/skill/) |
| Task | 状态机、queue、pause/resume、周期调度 | [task/](modules/task/) |
| Chat | 对话流 / session / 升级建议 / 回流 | [chat/](modules/chat/) |
| Realtime | 实时语音 middleware（V4 起） | [realtime/](modules/realtime/) |
| Improvement | Self-improvement 实现层 | [improvement/](modules/improvement/) |
| Context | Context engineering：多源拼装 | [context/](modules/context/) |
| RAG | 多 index 分层管理 | [rag/](modules/rag/) |
| Memory | 全局 / session / task 三层 | [memory/](modules/memory/) |
| Tools | Python 函数 + MCP 接入 | [tools/](modules/tools/) |
| LLM | 多 provider + cost-aware 路由 | [llm/](modules/llm/) |
| Trace | JSONL + 本地 HTML viewer + replay | [trace/](modules/trace/) |

## 项目规划

| 文档 | 内容 |
|------|------|
| [路线图](roadmap.md) | V0~V5+ 版本规划、退出标准 |
| [CLAUDE.md](../CLAUDE.md) | 项目规则总纲（AI agent 首选入口） |
