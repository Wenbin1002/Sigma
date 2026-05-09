# 文档导航

## 新人入门

| 文档 | 用途 | 适合谁 |
|------|------|--------|
| [贡献指南](../CONTRIBUTING.md) | 从零开始参与贡献 | 所有新贡献者 |
| [新手指南](guides/getting-started.md) | 环境搭建、项目理解 | 第一次接触项目 |
| [添加 Adapter](guides/adding-an-adapter.md) | 最常见的贡献方式 | 想贡献代码的开发者 |
| [代码规范](guides/code-style.md) | 命名、风格、约定 | 写代码前必读 |

## 架构设计

| 文档 | 内容 |
|------|------|
| [架构总览](architecture/overview.md) | 系统分层、模块关系、设计决策 |
| [运行模式](architecture/runtime-modes.md) | Cascade 和 Realtime 双模式详解 |
| [Ports & Adapters](architecture/ports-and-adapters.md) | 接口设计、数据结构、Registry 模式 |
| [依赖规则](architecture/dependency-rules.md) | Import 约束、违规示例、修复方法 |

## 模块参考

| 模块 | Port 接口 | 实现目录 | 文档 |
|------|-----------|----------|------|
| Voice | `STTPort`, `TTSPort` | `src/voice/` | [voice/](modules/voice/) |
| LLM | `LLMPort` | `src/llm/` | [llm/](modules/llm/) |
| Context | `ContextBuilderPort` | `src/context/` | [context/](modules/context/) |
| Agent | `AgentRuntimePort` | `src/agent/` | [agent/](modules/agent/) |
| RAG | `RetrievalPort` | `src/rag/` | [rag/](modules/rag/) |
| Memory | `MemoryPort` | `src/memory/` | [memory/](modules/memory/) |
| Tools | `ToolPort` | `src/tools/` | [tools/](modules/tools/) |
| Trace | —（内部模块） | `src/trace/` | [trace/](modules/trace/) |

## 项目规划

| 文档 | 内容 |
|------|------|
| [路线图](roadmap.md) | V0~V3+ 版本规划、退出标准 |
| [CLAUDE.md](../CLAUDE.md) | 项目规则总纲（AI agent 首选入口） |
