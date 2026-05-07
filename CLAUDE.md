# Sigma

语音交互 AI 助手（支持 Conversation Mode + Task Mode）

## 强制规则（违反即阻断）

1. **禁止在 main 上直接 commit**——任何变更必须先建分支，走 PR 合入
2. **分支命名**：`<type>/<short-desc>`（feat/fix/refactor/chore）
3. **流程**：需求分析 → 建分支 → 开发 → Commit → Review → 提 PR
4. 详见 [docs/development-workflow.md](docs/development-workflow.md)

## 架构约束

1. `core/` 不依赖任何外部 AI 框架/SDK
2. `runtime/` 只 import `ports/`，不 import 域目录（voice/agent/rag/...）
3. 域目录只 import `ports/` + 外部 SDK，域之间不互相引用
4. 新技术接入 = 域目录下新文件 + 注册到 registry.py + 改配置
5. 禁止反向依赖

## 代码结构概览

```
src/
  core/          # 稳定内核：数据结构、事件
  ports/         # 所有接口定义（Protocol），合同层
  voice/         # 语音域：STT + TTS + VAD 实现
  agent/         # Agent Runtime 实现
  rag/           # 检索增强实现
  memory/        # 记忆实现
  tools/         # 工具执行实现
  trace/         # 观测实现
  runtime/       # 跨域业务编排
  app/           # 应用入口（CLI/Web/API）
experiments/
tests/
config.yaml
```

## 命名规范

- Port 接口：`XxxPort`（如 `STTPort`, `LLMPort`, `AgentRuntimePort`）
- 实现类：描述性名称（如 `WhisperSTT`, `OpenAILLM`, `SimpleLoopAgent`）
- 注册表：每个域目录下 `registry.py` 中的 `REGISTRY` dict
- 工厂函数：`create(config) -> Port`

## 两种模式

- **Conversation Mode**：实时对话。Voice/Text Pipeline 直接消费 agent.stream()
- **Task Mode**：后台长任务。Task Manager 后台消费 agent.stream()，存储产物，管理生命周期

两者共享同一个 AgentRuntimePort。

## 详细文档

| 主题 | 位置 |
|------|------|
| 架构总览 | [docs/architecture.md](docs/architecture.md) |
| 核心接口定义 | [docs/interfaces.md](docs/interfaces.md) |
| 开发流程 | [docs/development-workflow.md](docs/development-workflow.md) |
| 路线图 | [docs/roadmap.md](docs/roadmap.md) |
| 语音模块 | [docs/voice/](docs/voice/) |
| Agent 模块 | [docs/agent/](docs/agent/) |
| RAG 模块 | [docs/rag/](docs/rag/) |
| 记忆模块 | [docs/memory/](docs/memory/) |
| 工具模块 | [docs/tools/](docs/tools/) |
| 观测模块 | [docs/observability/](docs/observability/) |

