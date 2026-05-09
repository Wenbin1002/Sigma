# Agent Runtime

> 核心认知引擎。接收组装好的上下文，通过 LLM + 工具调用生成回复，流式输出。

## 在架构中的位置

- **所属层**: Nodes（Runtime Graph 中的核心节点）
- **Port 接口**: `src/ports/agent_runtime.py` → `AgentRuntimePort`
- **实现目录**: `src/agent/`
- **被谁调用**: Runtime（Pipeline）
- **依赖**: LLMPort, ToolPort（通过 ports）

## Port 接口定义

```python
class AgentRuntimePort(Protocol):
    def stream(self, context: Context, input: str) -> AsyncIterator[AgentChunk]: ...
    def resume(self, state: ConversationState, decision: UserDecision) -> AsyncIterator[AgentChunk]: ...
```

Agent 不负责组装上下文——接收的 `Context` 已由 [Context Builder](../context/) 处理好。

## 定位：Graph 中的 Node

```
Runtime (Graph) 负责：            Agent (Node) 负责：
  node 之间怎么连                   拿到上下文后怎么决策
  数据怎么流                        该不该调工具？调哪个？
  走哪条边（确定性条件）              结果不对要不要重试？
  开发者定义                         LLM 运行时动态决定
```

对 Runtime Graph 来说，Agent 等价于一次异步 API 调用：

```python
# Runtime 的视角
chunks = agent.stream(context, input)
async for chunk in chunks:
    yield chunk  # 透传给用户
```

内部是 while loop、LangGraph 子图、还是多 agent 协作，调用方不关心。

## 可用实现

| 实现 | 类名 | 文件 | 配置值 | 状态 |
|------|------|------|--------|------|
| Simple Loop | `SimpleLoopAgent` | `src/agent/simple_loop.py` | `simple_loop` | V0 |
| LangGraph | `LangGraphAgent` | `src/agent/langgraph.py` | `langgraph` | V1 planned |

## 工作原理

Agent 接收 Context 后进入决策循环：调用 LLM → 判断是否需要工具 → 调用工具 → 把结果给 LLM → 循环直到 LLM 决定回复。全程流式输出 AgentChunk。

当遇到高风险操作时，输出 `needs_approval` chunk，暂停等待用户决策。用户确认后通过 `resume()` 继续执行。

## 配置示例

```yaml
agent_runtime:
  provider: simple_loop
```

## Multi-Agent 协作

多个 Agent 可来自不同来源，通过统一的 `AgentRuntimePort` 递归调用：

```
┌─────────────────────────────────────────────────────────┐
│              Orchestrator Agent (LangGraph)               │
│                                                          │
│   ┌──────┐     ┌──────────┐     ┌───────────┐          │
│   │ Plan │────→│ Dispatch │────→│ Aggregate │          │
│   └──────┘     └────┬─────┘     └───────────┘          │
│                     │ │ │                                │
└─────────────────────┼─┼─┼───────────────────────────────┘
                      │ │ │
         ┌────────────┘ │ └────────────┐
         ▼              ▼              ▼
   AgentRuntimePort  AgentRuntimePort  AgentRuntimePort
         │              │              │
    本地 Agent      远程 Agent      容器 Agent
```

| 协作模式 | 说明 |
|---------|------|
| 顺序流水线 | 依次调不同 Agent（生成 → 审查 → 修改） |
| 并行扇出 | 同时调多个 Agent，汇总结果 |
| 动态路由 | 按意图/任务类型选 Agent |
| 反思循环 | 执行 + 审查，循环直到通过 |

## Agent 可插拔接入

外部 Agent 接入方式：

| 方式 | 安装体验 | 适用场景 |
|------|---------|---------|
| Python 包 | `pip install xxx`，entry_points 自动发现 | Python 实现 |
| 远程服务 | 配置 gRPC endpoint | 其他语言/独立部署 |
| Container | 配置镜像地址，系统自动拉起 | 需要隔离环境 |

Agent Manifest：

```yaml
name: "code-reviewer"
description: "审查代码变更，给出改进建议"
capabilities: [code_review, refactoring_suggestion]
input:
  accepts: ["text", "artifact"]
output:
  produces: ["text_delta", "artifact", "needs_approval"]
runtime:
  type: "local"                    # local | remote | container
  entrypoint: "myagent:create"     # Python 入口 | gRPC endpoint
```

## 扩展方式

### 添加新 Agent 实现

1. 创建文件 `src/agent/<impl>.py`
2. 实现 `AgentRuntimePort` Protocol
3. 注册到 `src/agent/registry.py`
4. 添加配置项

### 约束

- 输出必须是 `AsyncIterator[AgentChunk]`
- 不负责上下文组装（那是 Context Builder 的事）
- Port 边界隔离框架选择——换框架 = 重写 `agent/` 内部，上层零改动

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 单 LLM 调用，流式输出 | Simple Loop |
| V1 | Tool-use loop，单步规划 | LangGraph |
| V2 | 多步规划、并行工具、checkpoint | LangGraph + persistence |
| V3 | Multi-agent 协作、外部 Agent 接入 | Agent Manifest |

## 相关文档

- [Context Builder](../context/) — 上下文组装
- [Tools 模块](../tools/) — Agent 可调用的工具
- [运行模式](../../architecture/runtime-modes.md) — Agent 在 Pipeline 中的位置
- [Ports & Adapters](../../architecture/ports-and-adapters.md) — AgentChunk 数据结构
