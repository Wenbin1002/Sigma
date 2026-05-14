# RAG 模块

> Sigma 的检索增强系统——**多 index 分层管理**，按 domain 配置不同的 index，配合 Ingest 预理解和 Query 回写实现知识复利。

---

## 1. 定位

RAG 在 Sigma 中是**核心功能之一**，不是可选的 add-on。

但传统 RAG 有根本缺陷——知识不复利：

| 问题 | 表现 |
|------|------|
| **推理结论不留存** | agent 花大量 token 推理出的结论用完就丢，下次从头来 |
| **全局理解缺失** | chunk 级检索只能看到局部，跨文档/跨模块的关联检索不到 |
| **知识不进化** | 索引建完就是静态的，不会随使用深入而变好 |

Sigma 的 RAG 借鉴 LLM-Wiki 的 AOT 思路，通过三循环（Ingest / Query 回写 / Lint）解决这些问题。

详见 [设计决策日志 § 4.10](../../architecture/design-log.md)。

---

## 2. RAG 在 Domain 内的角色

每个 domain = **RAG（原始材料）+ Memory（积累认知）**，共同进化。

```
Domain: book-百年孤独
├── rag/          ← 原始材料索引（书的全文，只读）
└── memory/       ← 积累的认知（主题分析、人物关系，读写）
```

**RAG 的职责**：管理原始材料的索引和检索。
**Memory 的职责**：管理 agent 推导出的认知。

两者的关键区别：**RAG 里的内容 agent 不能改（防幻觉污染），Memory 里的内容 agent 可读可写。**

详见 [Memory 模块](../memory/)、[设计决策日志 § D-22](../../architecture/design-log.md)。

---

## 3. 四个核心场景对 RAG 的需求

| 场景 | 需要的 RAG | Index 内容 | 知识复利需求 |
|---|---|---|---|
| 1 社媒筛选 | 不直接需要 RAG | — | — |
| 2 期货分析 | 行业知识 RAG | 养殖业基础 / 历史报告 | 供需模型积累、历史判断追踪 |
| 3 讨论书 | 必需 | 书的全文 | 主题深入、跨书对比、讨论积累 |
| 4 Coding | 必需 | 代码库 | 架构理解、隐含约定、debugging 经验 |

---

## 4. 多 Index 模型

```
~/.sigma/domains/
  book-百年孤独/
    rag/
      vectors/                   # 向量存储
      metadata.json              # chunking 配置 / 创建时间 / source
  code-sigma/
    rag/
      vectors/
      metadata.json
  industry-pig/
    rag/
      vectors/
      metadata.json
```

每个 index 是 self-contained 的：
- 自己的向量存储
- 自己的 metadata（chunking 策略 / embedding model / 来源）
- 可独立创建、删除、更新

---

## 5. 三循环

### 5.1 Ingest（吸收）— 用户添加素材时

```
用户添加新素材 (sigma rag add book.epub --domain book-百年孤独)
    ↓
传统流程：
  解析 → Chunking → Embedding → 存入 vector store
    ↓
⭐ 新增——预理解：
  agent 阅读素材 → 提炼结构化理解（实体、关系、摘要）
    → 写入 domain memory
```

**预理解**解决"全局理解缺失"——在 ingest 阶段就建立跨 chunk 的全局认知，而不是等到查询时再拼装。

### 5.2 Query 回写 — 交互中自然发生

```
用户提问 → 检索 RAG + 召回 Memory → agent 推理 → 回答用户
                                                      ↓
                                        ⭐ 有价值的结论沉淀到 domain memory
```

**Query 回写**解决"推理结论不留存"——每次深度分析的结论不再用完就丢。边际成本几乎为零（推理本来就做了，多一步存储）。

### 5.3 Lint（自检）— Self-improvement 模块负责

由 Self-improvement 模块在 session 后 / 周期性后台执行：

- 回顾历史交互，提取散落在对话中的领域认知
- 检查 domain memory 内部一致性（矛盾检测）
- 原始材料更新后，标记依赖它的推导结论为"可能过时"
- 清理过时 / 低质量的 memory 条目

详见 [Self-improvement](../../architecture/self-improvement.md)。

---

## 6. Index 生命周期

```bash
# 创建 domain + index
sigma domain create book-百年孤独
sigma rag add book.epub --domain book-百年孤独

# 列出
sigma rag list
sigma domain list

# 更新
sigma rag update <domain>         # 重建索引

# 删除
sigma rag delete <domain>

# 配置 task 用哪个 domain
sigma task new "..." --domain book-百年孤独
```

---

## 7. 检索流程

### 7.1 查询

```python
@sigma.tool
def search_knowledge(query: str, domains: list[str], top_k: int = 5) -> list[Citation]:
    """从指定 domain 的 RAG index 检索相关片段。"""
    ...
```

返回结构化 `Citation`（来源 / 片段 / score / 元数据），不是裸文本。

### 7.2 双路径检索

同一个 domain 内，检索同时走两条路：

1. **RAG 检索**：从原始材料的向量索引中检索相关 chunk
2. **Memory 召回**：从 domain memory 中召回相关的已有认知

Context Engine 合并两路结果，按优先级和 token budget 拼装。

### 7.3 进入 Context

检索结果由 [Context Engine](../context/) 决定：
- 放多少进 LLM context（按 score 截断）
- 怎么标注（prefix "[来源 X]"）
- Memory 召回的内容优先级高于 RAG chunk（已有理解 > 原始片段）
- 是否要求 LLM 在回复时带引用

---

## 8. Indexing Pipeline

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
- Vector store backend（0.4 落地时选定）

---

## 9. RAG 在 multi-agent 中

不同 sub-agent 看到不同 domain：

```python
class CoderAgent(sigma.Agent):
    domains = ["code-sigma"]

class ResearcherAgent(sigma.Agent):
    domains = ["industry-pig"]
```

`AgentContext.domains` 限制 sub-agent 只能查哪些 domain——既是隔离也是性能优化。

---

## 10. 原始材料只读原则

**铁律：agent 只能读 RAG 里的原始材料，绝不能改。**

这是防幻觉污染的关键闸门。如果 agent 能改原始文档，幻觉会反向污染源头，再也分不清"什么是事实、什么是推理"。

Agent 的推导结论只写入 Memory，不写入 RAG。两者物理隔离。

---

## 11. 实现位置

```
src/rag/
  manager.py          # Multi-domain RAG 管理
  indexer.py          # Indexing pipeline (parse / chunk / embed / store)
  retriever.py        # 检索接口
  ingest.py           # Ingest 预理解（调 LLM 提炼结构化理解）
  citation.py         # Citation 数据结构
  
  stores/
    qdrant.py         # 可能的 backend
    chroma.py
```

---

## 12. 未决问题

| 问题 | 状态 |
|---|---|
| Vector store 主选哪个 | 0.4 落地时定 |
| Chunking 默认策略 | 0.4 |
| Ingest 预理解的粒度（编译出什么形态的产物、成本权衡） | U-15，0.4 定 |
| 是否需要 RetrievalPort（即多 RAG 实现是否真有需求） | 0.4 落地后看 |
| GraphRAG / Hybrid retrieval 何时引入 | V2+ |

---

## 13. 相关文档

- [Context 模块](../context/) — RAG 输出的消费者
- [Memory 模块](../memory/) — Domain 内的兄弟模块
- [Self-improvement](../../architecture/self-improvement.md) — Lint 循环的驱动者
- [设计决策日志 § D-21~D-24, § 4.10](../../architecture/design-log.md) — 知识复利架构决策
- [架构总览](../../architecture/overview.md)
