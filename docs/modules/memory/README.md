# Memory 模块

> Sigma 的记忆系统——**agent 从交互中积累的一切有价值的认知**。
>
> 按**认知类型**（Semantic / Episodic / Procedural）和**作用域**（global / domain）两个正交维度组织。

---

## 1. 定位

Memory 是 Sigma 的**核心差异化能力之一**——决定 agent 是不是"越用越聪明"。

传统 RAG 的问题是**知识不复利**：agent 每次推理的结论用完就丢，下次同样的问题从头来。Memory 解决这个问题——把有价值的认知持久化，让 agent 的使用价值随时间复合增长。

### Memory 不只是"关于用户"的

行业共识（Mem0 / Zep / LangGraph / Claude Code）：Memory = agent 积累的一切有价值的认知，包括对用户的理解、对领域的理解、对做事方式的理解。

详见 [设计决策日志 § D-21](../../architecture/design-log.md)。

---

## 2. 三种认知类型

借鉴认知科学的经典分类：

### 2.1 Semantic Memory（语义记忆）— 事实与知识

提炼出的稳定认知。包括关于用户的，也包括关于领域的。

| 来源 | 例子 |
|------|------|
| 关于用户 | "喜欢简洁回答"、"金融行业"、"中文为主" |
| 关于领域 | "这本书的核心主题是循环与孤独"、"这个项目禁止域模块互相 import"、"母猪存栏下降 → 6 个月后猪价上涨" |

特点：相对稳定，但可能过时（尤其领域知识）。需要标注来源和置信度。

### 2.2 Episodic Memory（情景记忆）— 过去的经历

具体事件的记录，带时间戳和上下文。

| 例子 |
|------|
| "上次改 context 模块漏了更新 memory 的调用" |
| "三周前分析过猪价趋势，判断是涨，实际确实涨了" |
| "昨天讨论人物关系时，用户觉得类比卡夫卡的角度很好" |

特点：可回溯——不是抽象结论，是"那次具体发生了什么"。对 agent 避免重复犯错、复用历史经验至关重要。

### 2.3 Procedural Memory（程序性记忆）— 做事方式

从实践中学到的技能和流程。

| 例子 |
|------|
| "在这个项目里改 API 要同时改 3 个消费者" |
| "分析期货数据的步骤：拉存栏数据 → 计算同比 → 判断趋势" |
| "用户喜欢先看结论再看推理过程" |

特点：跟 Skill 有交集，但 Skill 是静态预写的方法论，Procedural Memory 是从实践中学到的。两者可以互相转化（高频 procedural memory 可以固化为 skill）。

---

## 3. 两个正交维度

### 3.1 维度一：作用域（在哪些场景可见）

```
┌──────────────────────────────────────────────────────────────┐
│  Global（跨域）                                               │
│  极少数真正跨所有场景的认知                                      │
│  例："语言偏好=中文"、"回复风格=简洁"                            │
└──────────────────────────────────────────────────────────────┘

┌─────────────────────┐  ┌─────────────────────┐  ┌──────────┐
│  Domain: book-百年孤独 │  │  Domain: code-sigma  │  │  Domain: │
│                       │  │                       │  │  pig-ind │
│  Semantic:            │  │  Semantic:            │  │          │
│   核心主题/人物关系    │  │   架构理解/依赖规则    │  │  因果模型 │
│  Episodic:            │  │  Episodic:            │  │  历史判断 │
│   讨论历史/分析记录    │  │   debugging 经验      │  │  数据周期 │
│  Procedural:          │  │  Procedural:          │  │  分析流程 │
│   文学分析方法         │  │   改代码的流程         │  │          │
└─────────────────────┘  └─────────────────────┘  └──────────┘
```

**为什么要隔离**：不同场景的 memory 可能直接冲突。coding 要求客观可靠，讨论书要求感性发散。混在一起不仅无用，还有害——检索时互相干扰，甚至产生矛盾指令。

**Domain 的载体**：跟 RAG index 对齐。每个 domain = RAG（原始材料）+ Memory（积累认知），共同进化。

详见 [设计决策日志 § D-22](../../architecture/design-log.md)。

### 3.2 维度二：生命周期（存多久）

| 层级 | 生命周期 | 内容 |
|------|---------|------|
| **长期** | 永久（除非用户删除或标记过时） | Semantic / Episodic / Procedural 中值得长期保留的 |
| **Session** | session 关闭后归档（不丢，可查询） | 当前对话的上下文、已讨论话题、session 内 artifact |
| **Task** | task 结束清理（除非显式持久化） | 当前 task 的中间产物、sub-agent 共享 state |

长期 memory 同时存在于 global 和 domain 两个作用域。Session / Task memory 天然属于某个 domain（如果有的话）。

---

## 4. Domain 模型

```
~/.sigma/
  domains/
    book-百年孤独/
      rag/              ← 原始材料索引（书的全文）
      memory/           ← 积累的认知（主题分析、人物关系、讨论经验）
    
    code-sigma/
      rag/              ← 代码索引
      memory/           ← 项目理解、debugging 经验、隐含约定
    
    industry-pig/
      rag/              ← 行业文档
      memory/           ← 因果模型、历史判断、分析流程

  global/
    memory/             ← 跨域：语言偏好、回复风格
```

每个 domain 是一个完整的**知识上下文**——agent 在这个领域里"越用越聪明"的全部积累都在这里。

---

## 5. 写入：知识怎么进来

### 5.1 Query 回写（交互中自然发生）

agent 做了一次深度分析，有价值的结论顺手沉淀到 domain memory。

```
用户：这本书的主题跟《百年孤独》有什么联系？
  ↓
agent 做深度对比分析（token 已经花了）
  ↓
回答用户
  ↓
⭐ 同时：对比分析沉淀到 domain memory（边际成本≈零）
  ↓
下次相关问题：直接召回已有分析，不用重新推理
```

这是知识**复利**的核心机制。

### 5.2 Self-improvement 沉淀（后台提取）

session 结束后 / 周期性地，Self-improvement 模块回顾历史交互：

- 提取用户偏好 → 存 global memory（原有能力）
- 提取领域认知 → 存 domain memory（**新增**）
- 检查 domain memory 内部一致性
- 标记可能过时的结论

详见 [Self-improvement](../../architecture/self-improvement.md)。

### 5.3 Ingest 预理解（添加素材时）

用户往 domain 里加新素材时，RAG 模块除了切片建索引，还提炼结构化理解（实体、关系、摘要）。这些理解写入 domain memory。

### 5.4 写入时机总览

| 时机 | 写什么类型 | 写到哪 |
|------|---------|--------|
| 深度分析后（query 回写） | Semantic / Episodic | Domain memory |
| Session 结束后（self-improvement） | 三种都可能 | Domain memory + Global memory |
| 素材加入时（ingest 预理解） | Semantic | Domain memory |
| 用户显式 `/remember ...` | Semantic | 用户指定（global 或 domain） |
| 每条 user message | Session 级 | Session memory |
| Task 关键节点 | Task 级 | Task memory |

---

## 6. 召回：知识怎么出来

每次 LLM 调用前由 Context Engine 决定：

```
recall(query, domain, scope) → list[MemoryItem]
  domain: "book-百年孤独" | "code-sigma" | ...
  scope: "global" | "domain" | "session" | "task" | "all"
```

**主动召回**（每次 LLM 调用前自动查）：
- Global：语言偏好、回复风格
- Domain（如果有）：相关的 Semantic + Episodic + Procedural memory

**懒召回**（agent 通过 tool 查）：
- agent 自己决定要不要深入查 memory（通过 `recall_memory(query, domain, scope)` tool）

---

## 7. 质量控制

Memory 写入比写错的代价高——错误 memory 会污染未来所有交互。

| 措施 | 说明 |
|------|------|
| **保守提取** | 宁可漏，不写错 |
| **来源追溯** | 每条 memory 记录来源（哪个 session / 哪次分析 / 哪份原始材料推导的） |
| **置信度** | 低置信的 memory 标记，召回时降权 |
| **过时检测** | 原始材料更新后，依赖它的推导结论标记为"可能过时" |
| **用户可见** | CLI / Web UI 提供 `sigma memory list/edit/delete` |
| **可审计** | 每条 memory 可追溯写入原因和推导依据 |
| **可回退** | Git 版本控制（domain 级） |

---

## 8. Memory 在 Sub-agent 中

`AgentContext` 限制 sub-agent 能看到哪个 domain 的 memory：

```python
class CoderAgent(sigma.Agent):
    # 只看 code-sigma 域的 memory + global
    memory_domains = ["code-sigma"]

class ResearcherAgent(sigma.Agent):
    # 可看多个域
    memory_domains = ["industry-pig", "user-notes"]
```

---

## 9. 与 RAG 的关系

| | RAG | Memory |
|---|---|---|
| 存什么 | 原始材料（用户提供的文档/代码） | Agent 积累的认知（推导出的理解） |
| 来源 | 用户显式添加 | 系统从交互中自动提取 |
| 可变性 | 原始材料只读（agent 不能改） | Agent 可读可写 |
| 检索方式 | 语义检索（向量） | 多层召回（global + domain + session + task） |

**两者在 domain 内共存、共同进化**：RAG 提供事实基础，Memory 积累理解深度。

详见 [RAG 模块](../rag/)。

---

## 10. Realtime Session 的特殊处理

Realtime session 是 session 的一种特殊形态，跟普通 chat session 共用同一个 SessionMemory schema，但有几个差异：

| 维度 | Chat Session | Realtime Session |
|---|---|---|
| 内容形式 | 文本 messages | Transcript（含时间戳）+ audio metadata |
| 提取频率 | session 结束后批处理 | session 结束后批处理 + 中途轻量提取 |
| 提取信号源 | 文本内容 | 文本 + 打断 / 沉默 / 笑声等 audio signal |

详见 [Realtime 模块](../realtime/)、[Self-improvement](../../architecture/self-improvement.md)。

---

## 11. 实现位置

```
src/memory/
  manager.py          # 多维管理器（类型 × 作用域）
  domain.py           # Domain memory 管理
  global_.py          # Global memory
  session.py          # Session memory
  task.py             # Task memory
  extract.py          # 自动提取策略（LLM 驱动）
  recall.py           # 召回策略
  quality.py          # 质量控制（置信度、过时检测）
  
  stores/
    sqlite.py         # 默认 backend
```

---

## 12. 未决问题

| 问题 | 状态 |
|---|---|
| Domain 的生命周期管理（创建 / 归档 / 跨 domain 引用） | U-13，0.3~0.4 定 |
| Query 回写的质量判断（什么结论值得沉淀、置信度标注） | U-14，0.4 定 |
| Memory 的 schema（是否结构化 / 自由文本 / 混合） | 0.3 落地时定 |
| 提取频率（每 session / 每 N 轮 / 后台批处理） | 0.3 定 |
| 是否需要 MemoryPort（即多 memory 实现是否真有需求） | 0.3 落地后看 |
| Memory 的"过期"策略（旧认知怎么淘汰） | 0.4+ |
| Procedural memory 跟 Skill 的转化机制 | 0.5+ |

---

## 13. 相关文档

- [Context 模块](../context/) — Memory 召回的消费者
- [RAG 模块](../rag/) — Domain 内的兄弟模块
- [Self-improvement](../../architecture/self-improvement.md) — Memory 提取的驱动者
- [设计决策日志 § D-21~D-24](../../architecture/design-log.md) — 知识复利架构的决策历史
- [架构总览](../../architecture/overview.md)
