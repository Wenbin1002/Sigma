# RAG — 检索增强生成

> 让 AI 基于你的文档回答问题，而不是编造答案。

## 在架构中的位置

- **所属层**: Adapters
- **Port 接口**: `src/ports/retrieval.py` → `RetrievalPort`
- **实现目录**: `src/rag/`
- **被谁调用**: Context Builder（前置检索）、Agent（运行时 Tool 调用）
- **依赖**: 向量数据库、Embedding 模型

## Port 接口定义

```python
class RetrievalPort(Protocol):
    async def search(self, query: str, top_k: int = 5) -> list[Chunk]: ...
    async def index(self, documents: list[Document]) -> None: ...
```

## 流程

```
文档 → 切块 → 向量化 → 存入向量库
用户提问 → 向量化 → 检索相似块 → 注入上下文 → LLM 回答（带引用）
```

## 可用实现

| 实现 | 类名 | 文件 | 配置值 | 状态 |
|------|------|------|--------|------|
| LlamaIndex | `LlamaIndexRetrieval` | `src/rag/llamaindex.py` | `llamaindex` | V1 planned |
| Qdrant | `QdrantRetrieval` | `src/rag/qdrant.py` | `qdrant` | V1 planned |

## 工作原理

RAG 分两个阶段：

**索引阶段**：文档被切分为 chunk，每个 chunk 经过 embedding 模型转为向量，存入向量数据库。

**检索阶段**：用户输入经过 embedding 转为向量，在向量库中找到最相似的 chunk，注入到 Context 中供 LLM 参考。LLM 基于检索到的内容回答，并附带引用来源。

检索可在两个时机触发：
1. Context Builder 前置检索（每轮自动）
2. Agent 通过 Tool 主动检索（按需）

## 配置示例

```yaml
retrieval:
  provider: llamaindex
  vector_store: qdrant
```

## 扩展方式

### 添加新检索后端

1. 创建文件 `src/rag/<backend>.py`
2. 实现 `RetrievalPort` Protocol
3. 注册到 `src/rag/registry.py`
4. 添加配置项

### 约束

- Chunking 和 Embedding 策略可独立替换
- 检索结果必须包含来源信息（用于 citation）
- 索引更新不能阻塞检索

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 不做 | — |
| V1 | 基础向量检索，文档导入，带引用回答 | LlamaIndex + Qdrant |
| V2 | 混合检索（向量+关键词）、Rerank | Cohere Rerank, BM25 |
| V3 | GraphRAG、Agentic RAG、多模态文档 | LightRAG, 知识图谱 |

## 可扩展能力

- Chunking 策略可插拔（固定大小 / 语义分割 / 文档结构感知）
- Embedding 模型可替换
- 检索策略可组合（向量 / 关键词 / 图 / 混合）
- 索引更新策略（实时 / 批量 / 增量）
- Rerank 模型可插拔

## 相关文档

- [Context Builder](../context/) — RAG 结果的注入方
- [Memory 模块](../memory/) — 另一个知识来源
- [Ports & Adapters](../../architecture/ports-and-adapters.md)
