# Sigma

可插拔的 AI Agent 框架。Runtime 是 Graph（编排），执行单元是 Node（黑盒），Node 通过 Port 消费可替换能力（LLM/Memory/RAG/Tools）。Graph 不关心 Node 内部，Node 不关心 Port 背后是谁。

## 核心特性

- **Ports & Adapters 架构** — 所有能力通过 Protocol 接口定义，实现可热替换，框架无锁定
- **Context 分层组装** — Memory 召回、RAG 检索、历史压缩独立为子图，按需组合注入 Agent
- **配置驱动** — 切换 LLM / Memory / RAG / Tools 的实现 = 改一行 config
- **流式 Agent Runtime** — 统一的 `AsyncIterator[AgentChunk]` 输出协议，支持中断、确认、恢复
- **语音交互（可选）** — 级联和 Realtime 双模式，作为 adapter 接入，不影响核心架构

## 架构

```
App (CLI / Web / API)
      │
 Runtime Layer（编排）
      │
Context Builder → Agent Runtime
      │                 │
Memory / RAG /       LLM + Tools
History
      │
 Ports Layer（接口契约）
      │
Domain Adapters（Whisper / OpenAI / LlamaIndex / ...）
```

所有模块通过 Port 接口通信，`runtime/` 只依赖 `ports/`，永远不直接 import 具体实现。详见 [架构文档](docs/architecture/overview.md)。

## 项目状态

> 🚧 早期开发阶段 — 架构设计已完成，正在实现 V0

当前进度参见 [路线图](docs/roadmap.md)。

## Quick Start

```bash
git clone <repo-url>
cd Sigma

# 安装依赖
pip install -e ".[dev]"

# 配置
cp config.example.yaml config.yaml
# 编辑 config.yaml 填入 API key

# 运行
python -m src.app.cli
```

## 配置示例

```yaml
llm:
  provider: openai
  model: gpt-4o

voice:
  stt: { provider: whisper }
  tts: { provider: cosyvoice }
  vad: { provider: silero }

retrieval:
  provider: llamaindex

memory:
  provider: local

agent_runtime:
  provider: simple_loop
```

切换任何模块 = 改 `provider` 字段。接入新技术 = 域目录下新文件 + 注册到 `registry.py` + 改配置。

## 模块

| 模块 | 说明 | 文档 |
|------|------|------|
| Agent | 认知引擎，决策循环 | [docs/modules/agent/](docs/modules/agent/) |
| Context | 上下文分层组装 | [docs/modules/context/](docs/modules/context/) |
| Memory | 跨会话记忆提取与召回 | [docs/modules/memory/](docs/modules/memory/) |
| RAG | 检索增强生成 | [docs/modules/rag/](docs/modules/rag/) |
| Tools | 工具调用，MCP 协议 | [docs/modules/tools/](docs/modules/tools/) |
| LLM | 多 provider 模型调用 | [docs/modules/llm/](docs/modules/llm/) |
| Voice | STT / TTS / VAD（可选） | [docs/modules/voice/](docs/modules/voice/) |
| Trace | 追踪、评估 | [docs/modules/trace/](docs/modules/trace/) |

## 文档

完整文档入口：[docs/index.md](docs/index.md)

| 主题 | 链接 |
|------|------|
| 架构总览 | [docs/architecture/overview.md](docs/architecture/overview.md) |
| 贡献指南 | [CONTRIBUTING.md](CONTRIBUTING.md) |
| 路线图 | [docs/roadmap.md](docs/roadmap.md) |

## 参与贡献

欢迎参与！请先阅读 [CONTRIBUTING.md](CONTRIBUTING.md) 了解开发流程和规范。

当前最需要的贡献方向：
- Adapter 实现（为各模块接入新的 provider）
- 文档完善
- 实验和评测（`experiments/` 目录）

## License

MIT
