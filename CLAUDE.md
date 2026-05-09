# Sigma

可插拔 AI Agent 框架 — Graph（编排）+ Node（黑盒执行单元）+ Port（可替换能力的类型契约）。

## 硬性规则（违反即阻断）

### Git 工作流

- **禁止**在 main 上直接 commit — 所有变更必须先建分支，走 PR 合入
- 分支命名：`<type>/<short-desc>`（type: feat/fix/refactor/chore/perf/test/docs）
- 流程：需求分析 → 建分支 → 开发 → Commit → Review → 提 PR
- 严格遵守 [skills/git-workflow/SKILL.md](skills/git-workflow/SKILL.md)

### 依赖规则

```
core/     → 只 import 标准库，禁止任何外部依赖
ports/    → 只 import core/
runtime/  → 只 import ports/（禁止 import voice/, agent/, rag/ 等域目录）
域目录    → 只 import ports/ + 外部 SDK（域之间禁止互相 import）
app/      → import runtime/, ports/, core/
```

违反以上任何一条 = 架构退化，必须修复。

### 新增技术的标准步骤

```
1. 创建 adapter 文件 → src/<domain>/<adapter_name>.py
2. 实现对应 Port 接口 → src/ports/<port_name>.py 中定义的 Protocol
3. 注册到域 registry → src/<domain>/registry.py 的 REGISTRY dict
4. 添加配置项 → config.yaml
5. 写测试 → tests/<domain>/test_<adapter_name>.py
```

## 项目结构

```
src/
  core/               # 稳定内核：数据结构、事件（无外部依赖）
  ports/              # 接口定义层（Protocol），所有模块的契约
    stt.py            #   STTPort
    tts.py            #   TTSPort
    llm.py            #   LLMPort
    context_builder.py #  ContextBuilderPort
    agent_runtime.py  #   AgentRuntimePort
    retrieval.py      #   RetrievalPort
    memory.py         #   MemoryPort
    tools.py          #   ToolPort
  voice/              # 语音域：STT + TTS + VAD 实现
  context/            # 上下文组装（Memory/RAG/历史压缩 → Context）
  agent/              # Agent Runtime 实现（纯决策图）
  rag/                # 检索增强实现
  memory/             # 记忆实现
  tools/              # 工具执行实现
  trace/              # 观测（内部模块，不走 Port）
  runtime/            # 跨域业务编排（Pipeline + Task Manager）
  app/                # 应用入口（CLI/Web/API）
experiments/          # 平行实验（不影响主线）
tests/                # 测试
config.yaml           # 全局配置
```

## 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| Port 接口 | `XxxPort` | `STTPort`, `LLMPort`, `AgentRuntimePort` |
| 实现类 | 描述性名称 | `WhisperSTT`, `OpenAILLM`, `SimpleLoopAgent` |
| 注册表 | `REGISTRY: dict[str, type]` | 位于 `src/<domain>/registry.py` |
| 工厂函数 | `create(config) -> XxxPort` | 位于 `src/<domain>/registry.py` |
| 配置键 | snake_case | `agent_runtime.provider` |

## DO / DON'T

### DO

- Port 入参出参只用可序列化类型：`str / bytes / int / float / bool / list / dict / dataclass`
- 流式输出用 `AsyncIterator`（可映射 gRPC stream）
- 一个 adapter 一个文件
- 每个 adapter 必须注册到域 registry
- 新模块先定义 Port，再写实现

### DON'T

- 不要在 Port 接口传 callback、文件句柄、Python 特有对象
- 不要从 `runtime/` import 域目录的实现
- 不要从一个域 import 另一个域
- 不要在 `core/` 放业务逻辑或外部依赖
- 不要在 Port 签名暴露批处理优化（内部实现即可）
- 不要在 main 分支直接提交代码

## 架构速览

```
App Layer（CLI / Web / API）
    ↓
Runtime = Graph（编排 Node 的执行顺序和数据流）
    ↓ stream() / resume()
Context Builder Node → Agent Node → Post-process Node
    内部消费 ↓              内部消费 ↓
    Memory / RAG           LLM / Tools
    ↓
Ports（可替换能力的类型契约）
    ↓
Adapters（具体实现：OpenAI, LlamaIndex, MCP...）
```

- **Graph（Runtime）**：固定拓扑，开发者定义 node 怎么连
- **Node**：执行单元，内部黑盒，可嵌套子图
- **Port**：可替换能力的类型契约（不是所有模块都是 Node，但需要 Port 保证换实现不出错）

Node vs Port 的区分：
- **既是 Node 又有 Port**：Context Builder, Agent（在图里有位置 + 需要类型契约）
- **只有 Port 没有 Node**：LLM, Memory, RAG, Tools（不参与图编排，被 Node 内部消费）

## 文档索引

| 主题 | 路径 |
|------|------|
| 文档导航 | [docs/index.md](docs/index.md) |
| 架构总览 | [docs/architecture/overview.md](docs/architecture/overview.md) |
| 运行模式 | [docs/architecture/runtime-modes.md](docs/architecture/runtime-modes.md) |
| Ports & Adapters | [docs/architecture/ports-and-adapters.md](docs/architecture/ports-and-adapters.md) |
| 依赖规则 | [docs/architecture/dependency-rules.md](docs/architecture/dependency-rules.md) |
| 新人指南 | [docs/guides/getting-started.md](docs/guides/getting-started.md) |
| 添加 Adapter | [docs/guides/adding-an-adapter.md](docs/guides/adding-an-adapter.md) |
| 代码规范 | [docs/guides/code-style.md](docs/guides/code-style.md) |
| Voice 模块 | [docs/modules/voice/](docs/modules/voice/) |
| LLM 模块 | [docs/modules/llm/](docs/modules/llm/) |
| Context 模块 | [docs/modules/context/](docs/modules/context/) |
| Agent 模块 | [docs/modules/agent/](docs/modules/agent/) |
| RAG 模块 | [docs/modules/rag/](docs/modules/rag/) |
| Memory 模块 | [docs/modules/memory/](docs/modules/memory/) |
| Tools 模块 | [docs/modules/tools/](docs/modules/tools/) |
| Trace 模块 | [docs/modules/trace/](docs/modules/trace/) |
| 路线图 | [docs/roadmap.md](docs/roadmap.md) |
| 贡献指南 | [CONTRIBUTING.md](CONTRIBUTING.md) |
