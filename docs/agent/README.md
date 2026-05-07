# Agent Runtime

核心决策引擎。接收组装好的上下文，通过 LLM + 工具调用生成回复，流式输出。上层只关注 input & output，内部实现可替换。

## 接口

```python
class AgentRuntimePort(Protocol):
    def stream(self, context: Context, input: str) -> AsyncIterator[AgentChunk]: ...
    def resume(self, state: ConversationState, decision: UserDecision) -> AsyncIterator[AgentChunk]: ...
```

Agent 不负责组装上下文——接收的 `Context` 已由 [Context Builder](../context/) 处理好。

## 演进路径

| 阶段 | 能力 |
|------|------|
| V0 | 单 LLM 调用，流式输出 |
| V1 | Tool-use loop，单步规划 |
| V2 | 多步规划、并行工具、checkpoint |
| V3 | Multi-agent 协作、外部 Agent 接入、动态路由 |

## 当前实现

LangGraph（functional API）。Port 边界隔离了框架选择，换框架 = 重写 `agent/` 内部，上层零改动。

## Multi-Agent 协作

多个 Agent 可来自不同来源（本地/远程/容器），通过统一的 `AgentRuntimePort` 递归调用。

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
         ▼              ▼              ▼
  ┌────────────┐ ┌────────────┐ ┌────────────┐
  │ 本地 Agent │ │ 远程 Agent │ │ 容器 Agent │
  │ (LangGraph)│ │  (gRPC)    │ │  (Docker)  │
  └────────────┘ └────────────┘ └────────────┘
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

实现 `AgentRuntimePort` 接口 + 注册到 `agent/registry.py` 即可。
