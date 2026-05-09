# Memory

> 跨会话记忆。让 AI 记住用户偏好、背景信息和关键决策。

## 在架构中的位置

- **所属层**: Adapters
- **Port 接口**: `src/ports/memory.py` → `MemoryPort`
- **实现目录**: `src/memory/`
- **被谁调用**: Context Builder（前置召回）、Agent（运行时 Tool 调用）
- **依赖**: 存储后端（SQLite / 向量索引）

## Port 接口定义

```python
class MemoryPort(Protocol):
    async def recall(self, query: str, user_id: str) -> list[MemoryItem]: ...
    async def store(self, item: MemoryItem) -> None: ...
    async def delete(self, memory_id: str) -> None: ...
```

## 可用实现

| 实现 | 类名 | 文件 | 配置值 | 状态 |
|------|------|------|--------|------|
| Local | `LocalMemory` | `src/memory/local.py` | `local` | V2 planned |
| Mem0 | `Mem0Memory` | `src/memory/mem0.py` | `mem0` | V3 planned |

## 工作原理

Memory 模块管理跨会话的持久化记忆。工作流程：

1. **提取**：每轮对话结束后，从对话内容中自动提取值得记住的信息（偏好、事实、决策）
2. **存储**：提取的记忆经过向量化后存入持久化存储
3. **召回**：下次对话时，Context Builder 根据当前输入语义检索相关记忆，注入 Context

用户可以查看、编辑、删除自己的记忆——这是可控性的核心。

## 配置示例

```yaml
memory:
  provider: local
```

## 扩展方式

### 添加新记忆后端

1. 创建文件 `src/memory/<backend>.py`
2. 实现 `MemoryPort` Protocol
3. 注册到 `src/memory/registry.py`
4. 添加配置项

### 约束

- 记忆提取是异步的，不阻塞对话
- 用户必须能控制自己的记忆（查看/删除）
- 记忆召回结果需要有相关性分数，供 Context Builder 做优先级判断

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 不做 | — |
| V1 | 对话内上下文压缩（不超 token limit） | 滑动窗口 + 摘要 |
| V2 | 跨会话记忆提取和召回 | Local SQLite + embedding |
| V3 | 实体关系图、用户画像自动构建 | Mem0, Zep |

## 可扩展能力

- 提取策略可插拔（什么信息值得记）
- 召回策略可插拔（什么时候把记忆注入 context）
- 遗忘机制（过时信息衰减）
- 用户可控（查看、编辑、删除自己的记忆）

## 相关文档

- [Context Builder](../context/) — 记忆的注入方
- [RAG 模块](../rag/) — 另一个知识来源
- [Ports & Adapters](../../architecture/ports-and-adapters.md)
