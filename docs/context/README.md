# Context Builder

每轮 Agent 调用前，组装 LLM 应该看到的完整上下文。独立于 Agent，策略可插拔。

## 职责

决定"LLM 看到什么"：

- System prompt 组装
- 对话历史压缩 / 截断
- Memory 召回注入
- RAG 检索结果注入
- 用户/会话元信息

**不做**：决策、工具调用、循环控制（这些是 Agent 的事）。

## 与 Agent 的关系

```
input + state
     │
     ▼
Context Builder ──→ Context（组装好的上下文）
     │                    │
     │ Memory/RAG/        ▼
     │ History/Prompt     Agent（纯决策图）
     │                    │
                          ▼
                       output
```

- Context Builder = 前置，被动，每轮开始前调用
- Agent 运行中主动查资料 = Tool 调用 RAG/Memory，走工具链路

## 接口

```python
class ContextBuilderPort(Protocol):
    async def build(self, input: str, state: ConversationState) -> Context: ...

@dataclass
class Context:
    system_prompt: str
    messages: list[dict]
    memories: list[MemoryItem]
    references: list[Chunk]
```

## 可用实现

| Adapter | 说明 | 配置值 |
|---------|------|--------|
| Passthrough | 透传，不做额外处理 | `passthrough` |

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 透传（input 包成 messages） | Passthrough |
| V1 | 历史滑动窗口 + token 截断 | 简单截断 |
| V2 | Memory 召回 + RAG 注入 + 历史摘要压缩 | LLM 摘要 |
| V3 | 动态策略（按任务类型决定注入内容）、多 Agent 共享 | — |

## 可扩展能力

- 压缩策略可插拔（截断 / 摘要 / 混合）
- 注入优先级（当 token 预算有限时，先保证什么）
- 多 Agent 场景下 context 隔离与共享
- Prompt 模板管理（按场景切换 system prompt）

## 扩展方式

实现 `ContextBuilderPort` 接口 + 注册即可。
