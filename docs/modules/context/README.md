# Context Builder

> 每轮 Agent 调用前，组装 LLM 应该看到的完整上下文。决定"LLM 看到什么"。

## 在架构中的位置

- **所属层**: Nodes（Runtime Graph 中的前置节点）
- **Port 接口**: `src/ports/context_builder.py` → `ContextBuilderPort`
- **实现目录**: `src/context/`
- **被谁调用**: Runtime（Pipeline 中 Agent Node 之前）
- **依赖**: MemoryPort, RetrievalPort（通过 ports）

## Port 接口定义

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

## 与 Agent 的关系

```
input + state
     │
     ▼
Context Builder ──→ Context（组装好的上下文）
                          │
                          ▼
                    Agent（纯决策图）
                          │
                          ▼
                       output
```

- Context Builder = 前置，被动，每轮开始前调用
- Agent 运行中主动查资料 = Tool 调用 RAG/Memory，走工具链路

**不做**：决策、工具调用、循环控制（这些是 Agent 的事）。

## 可用实现

| 实现 | 类名 | 文件 | 配置值 | 状态 |
|------|------|------|--------|------|
| Passthrough | `PassthroughBuilder` | `src/context/passthrough.py` | `passthrough` | V0 |

## 工作原理

Context Builder 作为 Runtime Graph 中的一个 Node，内部可展开为子图（Memory / RAG / History 并行执行后汇总）。

子图结构：
```
START ─┬→ recall_memory ──┐
       ├→ search_rag ─────┼→ assemble → Context
       └→ compress_history┘
```

输出的 Context 对象包含：system prompt、压缩后的消息历史、召回的记忆、RAG 检索结果。Agent 直接消费这个 Context 做决策。

## 配置示例

```yaml
context:
  provider: passthrough
```

## 扩展方式

### 添加新策略

1. 创建文件 `src/context/<strategy>.py`
2. 实现 `ContextBuilderPort` Protocol
3. 注册到 `src/context/registry.py`
4. 添加配置项

### 约束

- Context Builder 不做决策，只做组装
- 当 token 预算有限时需要有注入优先级策略
- 输出的 Context 必须是可序列化的 dataclass

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 透传（input 直接包成 messages） | Passthrough |
| V1 | 历史滑动窗口 + token 截断 | 简单截断 |
| V2 | Memory 召回 + RAG 注入 + 历史摘要压缩 | LLM 摘要 |
| V3 | 动态策略（按任务类型决定注入内容）、多 Agent 共享 | — |

## 可扩展能力

- 压缩策略可插拔（截断 / 摘要 / 混合）
- 注入优先级（当 token 预算有限时，先保证什么）
- 多 Agent 场景下 context 隔离与共享
- Prompt 模板管理（按场景切换 system prompt）

## 相关文档

- [Agent 模块](../agent/) — Context 的消费方
- [Memory 模块](../memory/) — 记忆召回源
- [RAG 模块](../rag/) — 检索结果源
- [运行模式](../../architecture/runtime-modes.md) — Context Builder 在 Pipeline 中的位置
