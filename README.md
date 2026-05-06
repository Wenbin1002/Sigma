# Sigma

支持语音交互的 AI 助手。能听、能说、能用工具做事、能记住你说过的话、能基于你的资料回答问题。

## 特性

- **Conversation Mode** — 实时对话，语音或文本，流式输出，支持打断
- **Task Mode** — 后台长任务，提交后可离线，自主运行完成后通知
- **可插拔架构** — 所有模块通过 Port 接口解耦，切换实现只需改配置
- **配置驱动** — 一份 `config.yaml` 控制所有模块的 provider 选择

## 架构

```
App (CLI / Web / API)
        │
   Session Layer
   ┌────┴────┐
   │         │
Voice/Text  Task
Pipeline   Manager
   └────┬────┘
        │
  Agent Runtime (stream / resume)
        │
   ┌────┼────┐
   │    │    │
Memory  RAG  Tools
        │
   Ports Layer (接口定义)
        │
  Domain Implementations (voice / agent / rag / memory / tools / trace)
```

两种模式共享同一个 `AgentRuntimePort.stream()`，区别只在上层谁来消费这个流。

## 代码结构

```
src/
  core/               # 稳定内核：数据结构、事件
  ports/              # 所有接口定义（Protocol），合同层
  voice/              # 语音域：STT + TTS + VAD 实现
  agent/              # Agent Runtime 实现
  rag/                # 检索增强实现
  memory/             # 记忆实现
  tools/              # 工具执行实现
  trace/              # 观测实现
  runtime/            # 跨域业务编排
  app/                # 应用入口

experiments/          # 平行实验
tests/
config.yaml
```

详见 [docs/architecture.md](docs/architecture.md)

## Quick Start

> WIP — 项目处于早期开发阶段

```bash
# 克隆
git clone <repo-url>
cd ChatBot

# 安装依赖
pip install -e .

# 配置
cp config.example.yaml config.yaml
# 编辑 config.yaml 填入 API key

# 运行
python -m src.app.cli
```

## 配置

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
  vector_store: qdrant

memory:
  provider: local

trace:
  provider: jsonl

agent_runtime:
  provider: simple_loop
```

切换任何模块 = 改 `provider` 字段。接入新技术 = 域目录下新文件 + 注册到 `registry.py` + 改配置。

## 模块

| 模块 | 说明 | 文档 |
|------|------|------|
| Voice | STT / TTS / VAD，延迟优化 | [docs/voice/](docs/voice/) |
| Agent | Runtime Port 接口设计、状态管理 | [docs/agent/](docs/agent/) |
| RAG | 检索增强生成 | [docs/rag/](docs/rag/) |
| Memory | 长期记忆、上下文压缩 | [docs/memory/](docs/memory/) |
| Tools | MCP、权限模型 | [docs/tools/](docs/tools/) |
| Trace | 观测、Eval | [docs/trace/](docs/trace/) |

## Roadmap

| 版本 | 目标 | 周期 |
|------|------|------|
| **V0** | 能对话 — 语音/文本流式交互，延迟 < 3s | 2 周 |
| **V1** | 能做事 — 工具调用，状态持久化 | 2-3 周 |
| **V2** | 能记住 + 能查资料 — RAG + Memory | 3-4 周 |
| **V3** | 更强 — GraphRAG, MCP 生态, 多模态 | 持续迭代 |

详见 [docs/roadmap.md](docs/roadmap.md)

## License

MIT
