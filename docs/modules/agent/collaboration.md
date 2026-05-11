# Agent 协作模式

> Agent 之间如何互相调用、并行、嵌套。

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
