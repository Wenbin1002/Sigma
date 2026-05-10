# RAG 模块

> Sigma 的检索增强系统——**多 index 分层管理**，按 task / session 配置不同的 index。

---

## 1. 定位

RAG 在 Sigma 中是**核心功能之一**，不是可选的 add-on（详见 [架构总览](../../architecture/overview.md#1-sigma-是什么)）。

四个核心场景对 RAG 的需求差别巨大：

| 场景 | 需要的 RAG | Index 内容 |
|---|---|---|
| 1 社媒筛选 | 不直接需要 RAG（实时拉取） | — |
| 2 期货分析 | 行业知识 RAG（可选） | 养殖业基础概念 / 历史报告 |
| 3 讨论书 | 必需 | 这本书的全文 |
| 4 Coding | 必需 | 代码库 |

**结论**：RAG 不是单一全局索引，是**多 index 系统**——按 task / session 配置不同 index，agent 按需查不同的。

---

## 2. 多 Index 模型

```
~/.sigma/rag/
  indices/
    book-xxx/                    # 某本书的索引
      vectors/                   #   向量
      metadata.json              #   chunking 配置 / 创建时间 / source
    code-myproject/              # 某个代码库的索引
    industry-pig-farming/        # 行业知识
    user-notes/                  # 用户笔记
```

每个 index 是 self-contained 的：
- 自己的向量存储
- 自己的 metadata（chunking 策略 / embedding model / 来源）
- 可独立创建、删除、更新

---

## 3. Index 生命周期

```bash
# 创建
sigma rag create <name> --source <path-or-url>
sigma rag create book-pride-and-prejudice --source ./book.epub

# 列出
sigma rag list

# 更新
sigma rag update <name>          # 重建索引

# 删除
sigma rag delete <name>

# 配置 task 用哪个 index
sigma task new "..." --rag book-xxx,user-notes
```

---

## 4. 检索流程

### 4.1 查询

```python
@sigma.tool
def search_knowledge(query: str, indices: list[str], top_k: int = 5) -> list[Citation]:
    """从指定 index 检索相关片段。"""
    ...
```

返回结构化 `Citation`（来源 / 片段 / score / 元数据），不是裸文本——方便 LLM 理解和引用。

### 4.2 进入 Context

检索结果由 [Context Engine](../context/) 决定：
- 放多少进 LLM context（按 score 截断）
- 怎么标注（prefix "[来源 X]"）
- 是否要求 LLM 在回复时带引用

---

## 5. Indexing Pipeline

```
源文件 (PDF / 代码 / markdown / web URL)
    ↓
解析（提取文本、保留结构）
    ↓
Chunking（按 size / 语义 / 结构）
    ↓
Embedding（按 index 配置选 embedding model）
    ↓
存入 vector store
```

策略可配置：
- Chunk size
- Overlap
- Embedding model
- Vector store backend（V3 选定，可能 Qdrant / Chroma / LanceDB）

---

## 6. 与 Memory 的区分

容易混淆的边界。澄清：

| | RAG | Memory |
|---|---|---|
| 数据来源 | 用户提供的文档 / 代码 | 系统自动从对话提取 |
| 更新频率 | 显式（用户运行 `rag update`） | 隐式（每次对话） |
| 内容 | "事实"（文档说什么） | "用户和对话状态"（喜好、决定、上下文） |
| 检索方式 | 语义检索 | 多层召回（global / session / task） |
| 时间敏感 | 标注源时间但内容静态 | 时间序列，新覆盖旧 |

详见 [Memory 模块](../memory/)。

---

## 7. RAG 在 multi-agent 中

不同 sub-agent 看到不同 index：

```python
class CoderAgent(sigma.Agent):
    rag_indices = ["code-myproject", "code-stdlib"]

class ResearcherAgent(sigma.Agent):
    rag_indices = ["industry-pig-farming", "user-notes"]
```

`AgentContext.rag_indices` 限制 sub-agent 只能查哪些 index——既是隔离也是性能优化。

---

## 8. Phase 0 / V3 实现路径

V3 是 RAG 真正落地的版本。之前的版本可以用极简实现（直接全文喂给 LLM）应付小规模 demo。

V3 必做：
- 至少接 1 个 vector store
- Chunking pipeline
- Multi-index 管理 CLI
- 检索时带引用

V4+ 可加：
- GraphRAG / Hybrid retrieval
- Re-ranking
- 自动 index 更新（监听文件变更）

---

## 9. 实现位置

```
src/rag/
  manager.py          # Multi-index 管理
  indexer.py          # Indexing pipeline (chunk / embed / store)
  retriever.py        # 检索接口
  citation.py         # Citation 数据结构
  
  stores/
    qdrant.py         # 可能的 backend
    chroma.py
```

---

## 10. 未决问题

| 问题 | 状态 |
|---|---|
| Vector store 主选哪个 | V3 落地时定 |
| Chunking 默认策略 | V3 |
| 是否需要 RetrievalPort（即多 RAG 实现是否真有需求） | 未决——V3 落地后看 |
| GraphRAG / Hybrid retrieval 何时引入 | V4+ |

---

## 11. 相关文档

- [Context 模块](../context/) — RAG 输出的消费者
- [Memory 模块](../memory/) — RAG 的"姐妹模块"
- [架构总览](../../architecture/overview.md)
- [设计决策日志 § 4.3](../../architecture/design-log.md)
