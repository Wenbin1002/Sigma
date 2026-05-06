# Agent Runtime

语音助手的核心决策引擎。接收用户输入，通过 LLM + 工具调用生成回复，流式输出。

## 特性

- 流式输出（第一个字生成即可播放）
- 人工介入（危险操作暂停等确认）
- 可替换实现（Simple Loop / LangGraph / 自定义）
- 支持 Conversation Mode（实时对话）和 Task Mode（后台长任务）

## 接口

```python
class AgentRuntimePort(Protocol):
    def stream(self, input: str, state: ConversationState) -> AsyncIterator[AgentChunk]: ...
    def resume(self, state: ConversationState, decision: UserDecision) -> AsyncIterator[AgentChunk]: ...
```

## 可用实现

| Adapter | 说明 | 配置值 |
|---------|------|--------|
| Simple Loop | while 循环 + 工具分发，简单直接 | `simple_loop` |
| LangGraph | 状态图，支持并行/分支/multi-agent | `langgraph` |

## 扩展

实现 `AgentRuntimePort` 接口 + 注册到 `adapters/agent_runtime/REGISTRY` 即可。

详见 [architecture.md](../architecture.md) 中的扩展流程。
