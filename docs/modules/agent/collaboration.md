# Agent 协作模式

> Agent 之间如何互相调用、并行、嵌套。

---

## 数据传递

Agent 之间不需要专门的数据传递机制——这是 **Context 模块的 token 管理策略**统一处理的。

**规则**：任何内容（agent 产出 / tool 返回 / RAG 召回）进入后续 context 前，由 Context Engine 按 token 阈值自动判断：

```
token_count(content) <= threshold?
    YES → 直接放入下游 agent 的 context
    NO  → summarize + 存原文（ref），下游看摘要，按需 load 原文
```

**对写 agent 的人来说**：正常产出内容即可，不需要关心"下游怎么拿到我的数据"。框架自动处理压缩/传递。

**下游 agent 需要完整数据时**：通过 `ctx.load_ref(ref_id)` 取回原文。

这套机制同时解决了：
- Agent 之间传结构化数据（researcher → analyst）
- Tool 返回超长结果（一次 API 返回 10000 行 JSON）
- 长链协作的 context 膨胀问题

详见 [Context 模块 § 内容压缩策略](../context/README.md#33-内容压缩策略)。

---

## 串行

Master agent 按顺序 spawn sub-agent：

```
Master → @researcher → 拿 artifact → @writer → 拿报告
```

通过 `ctx.spawn_agent()` 调用：

```python
async for chunk in ctx.spawn_agent("researcher", task="收集 XX 数据"):
    # 处理 researcher 的输出
    ...
```

---

## 并行（fan-out / fan-in）

Level 2 (Graph Agent) 可声明并行分支：

```
Master ──┬──→ @theme_analyzer ──┐
         ├──→ @char_analyzer  ──┼──→ aggregate
         └──→ @style_analyzer ──┘
```

LangGraph 自动处理并行调度和结果汇聚。

Level 0/1 也可以手写 asyncio 实现并行：

```python
async def run(self, task: str, ctx: Context):
    results = await asyncio.gather(
        ctx.spawn_agent("theme_analyzer", task="..."),
        ctx.spawn_agent("char_analyzer", task="..."),
    )
    # aggregate
```

### 并行失败处理

**默认策略：best-effort**——部分 agent 失败时，其余正常完成的结果 + 失败信息一起传给下游 agent。

```
@search_academic  → ✅ 成功，数据...
@search_news      → ❌ 失败，原因: timeout after 30s
@search_social    → ✅ 成功，数据...
         ↓
synthesize 拿到部分结果 + 失败标记，自行判断结论可信度
```

下游 agent（如 synthesize）负责在不完整数据下做推理，并告知用户哪些源缺失。

**为什么 best-effort 是默认**：
- Sigma 核心场景（研究/分析）天然是多源尽力的——少一个信息源不致命
- LLM 擅长处理不完整信息，只要明确告知"哪个缺了"
- 实现简单——失败的 agent 返回 error chunk，跟成功结果一起传给下游

> **TODO**：后续需要区分场景，支持可配置的失败策略（如 `require_all=True` 强制 all-or-nothing，适用于"数据必须完整才有意义"的场景如测试全通过才能部署）。这个暂不实现，等遇到真实需求再加。

---

## 嵌套

允许 agent 内部再 spawn agent（通过 `ctx.spawn_agent()`），但**不主动设计深层嵌套**：

- 子嵌套容易控制不住时长和成本
- Trace viewer 会展示嵌套层级，便于 debug
- 建议最多 2 层（Master → Sub → Sub-sub）

---

## 协作原则

| 原则 | 说明 |
|------|------|
| **松耦合** | Agent 之间通过 task description + AgentChunk 通信，不共享内部 state |
| **成本传递** | Sub-agent 的 LLM 调用计入 parent task 总预算 |
| **取消传播** | 取消 parent 时自动 cancel 所有子 agent |
| **Timeout** | spawn 时可声明 timeout，到时强制 cancel |
