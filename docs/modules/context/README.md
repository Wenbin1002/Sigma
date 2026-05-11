# Context 模块

> Context Engine = Sigma 的核心算法之一：**决定每次 LLM 调用看到什么**。

> 这是 Sigma 真正的差异化能力之一（context engineering）。

---

## 1. 定位

LLM 一次调用能塞进去的 token 有限（4k / 8k / 32k / 128k / 200k）。**塞什么、不塞什么、怎么排序、怎么压缩**——这是 context engineering，决定 agent 行为质量的核心。

通用 agent 框架里 context 一般是固定 prompt + 对话历史。Sigma 把它抽出来作为独立模块，按需拼装：

```
最终送进 LLM 的 context =
    System prompt 骨架
  + Skill metadata 索引（按需 load_skill 全文）
  + 当前 task / chat 状态
  + 历史对话（可能压缩）
  + 召回的 memory（global / session / task）
  + RAG 检索结果（带引用）
  + 当前可用 tool 清单
  + 当前 sub-agent 选项（如果走 supervisor）
```

---

## 2. 拼装来源

### 2.1 静态来源（启动时确定）

- System prompt 骨架（Sigma 自身、当前 agent）
- 工具清单
- Sub-agent 清单
- Skill metadata 索引

### 2.2 动态来源（每轮调用前查询）

- 历史对话（按 token budget 截断 / 压缩）
- Memory 召回（多层）
- RAG 检索（多 index）
- 当前 task state / artifact 引用

详见 [Memory 模块](../memory/)、[RAG 模块](../rag/)。

---

## 3. 拼装策略

### 3.1 Token Budget

每次拼装前确定本次 budget：

```
budget = model_max_tokens
       - reserved_for_response
       - reserved_for_tool_args
```

### 3.2 优先级（核心 → 边缘）

按优先级逐项加，超 budget 时丢弃低优先级：

1. System prompt（必加，不可丢）
2. 当前 user message
3. 最近 N 轮对话
4. 主动召回的 memory（用户偏好 / 上次决策）
5. 当前 task state
6. RAG 检索结果（按 score）
7. 历史压缩摘要（前 N-X 轮的总结）
8. 工具/agent 清单（如果太长可只放 description）

### 3.3 内容压缩策略

任何内容（agent 产出 / tool 返回 / RAG 召回）进入 context 前，按 token 阈值自动判断：

```
token_count(content) <= threshold?
    YES → 直接放入 context（原文）
    NO  → summary + ref
            ├─ summary: LLM 生成的摘要，放入 context
            └─ ref: 原文存储，按需取回
```

**为什么**：避免单次 tool 返回或 agent 产出撑爆 context window。同一套机制处理所有场景：

| 场景 | 触发压缩的时机 |
|------|--------------|
| Agent 产出传递给下游 agent | Master 编排 sub-agent 时 |
| Tool 返回超长结果 | tool call 结果写入对话历史时 |
| RAG 召回大量文档 | context 拼装时 |

**按需取回原文**：下游 agent 看到摘要后，如果需要完整数据（如精确字段访问），通过 `ctx.load_ref(ref_id)` 取回原文。取回的原文不进对话历史，只在当次 LLM 调用中临时注入。

**阈值配置**：

```yaml
# config.yaml
context:
  compression_threshold: 2000   # tokens，超过此值触发 summarize + ref
```

### 3.4 历史压缩

长对话超出 budget 时：

- **滚动窗口**：保留最近 N 轮，更早的丢弃
- **递归总结**：每过一定轮数，把"更早的对话"总结成一段，留摘要不留原文
- **关键消息保留**：用户的关键决策 / 系统的关键回复永不丢

策略可配置，未来可针对场景优化。

---

## 4. 多源拼装的挑战

每个来源都可能与其他来源冲突或重复：

| 冲突类型 | 例子 | 处理 |
|---|---|---|
| 重复信息 | Memory 和历史对话都说了"用户喜欢简洁回答" | 去重，保留更具体的 |
| 时间陈旧 | RAG 检出的是 2020 年的资料，对话提到 2024 现状 | 标注时间，让 LLM 自己判断 |
| 上下文歧义 | "那个文件" 在多个 memory 里指不同文件 | 提示 LLM 警告，必要时升级到 L3 让用户澄清 |

---

## 5. Context vs Memory vs RAG

容易混淆，澄清一下：

| 模块 | 职责 | 关系 |
|---|---|---|
| **Memory** | 跨会话/会话内的状态存储 | Context 的来源之一 |
| **RAG** | 文档 / 代码 / 知识库的检索 | Context 的来源之一 |
| **Context Engine** | 把上面所有源拼成最终 LLM input | 消费者 / 编排者 |

**Memory 和 RAG 都不直接面向 LLM**——它们的输出进入 Context Engine 拼装。

---

## 6. Context 在 Sub-agent 中的传递

Master agent 派 sub-agent 时，传递的 `AgentContext`：

```python
@dataclass
class AgentContext:
    chat_history: list[Message]        # 父对话历史（可压缩）
    parent_artifacts: list[Artifact]   # 父 agent 已有的产物
    parent_state: dict                 # 父 agent 想传的额外信息
    
    # 共享资源访问
    memory_scope: MemoryScope          # 该 sub-agent 能看到哪一层 memory
    rag_indices: list[str]             # 该 sub-agent 能用哪些 RAG index
```

Sub-agent 内部可能再次组装自己的 context（比 master 更聚焦），不直接把 master 的 context 全塞 LLM。

---

## 7. 不开放替换

Context Engine 是 Sigma 的**核心算法**，**不开放 Port 替换**——

- 用户可以调整策略（通过配置）
- 用户可以扩展来源（通过新增 memory adapter / RAG index / skill）
- 但不能换整个 Context Engine 实现

详见 [Ports & Adapters § 1.2](../../architecture/ports-and-adapters.md#12-哪些不是-port为什么)。

---

## 8. 实现位置

```
src/context/
  engine.py            # Context Engine 主类
  budget.py            # Token budget 管理
  prioritize.py        # 优先级策略
  compress.py          # 历史压缩
  assemble.py          # 多源拼装
  agent_context.py     # AgentContext 数据结构
```

---

## 9. 未决问题

| 问题 | 状态 |
|---|---|
| 历史压缩的具体策略（递归总结 / 滚动窗口 / 混合） | V3 落地时定 |
| 不同 sub-agent 的 context 隔离强度 | V2 落地时定 |
| Context engineering 的可观察性（哪部分占了多少 token） | V3 |

---

## 10. 相关文档

- [架构总览](../../architecture/overview.md)
- [Memory 模块](../memory/)
- [RAG 模块](../rag/)
- [Agent 模块](../agent/) — Sub-agent context 传递
- [设计决策日志 § 4.3](../../architecture/design-log.md) — RAG 多 index、Memory 分层的洞察
