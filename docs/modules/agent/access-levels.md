# 三档接入模型

> 用户根据需求复杂度选择接入层级，不被强制面对 LangGraph。

---

## 总览

```
┌─────────────────────────────────────────────────┐
│  Level 0: Function Agent  （入门）               │  最简单，大多数场景够用
├─────────────────────────────────────────────────┤
│  Level 1: Stepped Agent   （结构化）             │  享受 checkpoint / resume / 可视化
├─────────────────────────────────────────────────┤
│  Level 2: Graph Agent     （完全灵活）           │  LangGraph 全部能力
└─────────────────────────────────────────────────┘
```

---

## Level 0: Function Agent

对"我就想写个异步函数"的场景。零学习成本，纯 Python：

```python
from sigma.agent import Agent, Context, AgentChunk

class StockAnalyst(Agent):
    name = "stock-analyst"
    description = "分析期货数据，产出趋势判断"
    triggers = ["期货", "行情", "K线"]
    tools = ["search_web", "fetch_url", "python_exec"]

    async def run(self, task: str, ctx: Context):
        data = await ctx.use_tool("fetch_url", url="...")
        result = await ctx.llm.chat(f"分析以下数据: {data}")
        yield AgentChunk(text=result)
```

**Sigma 内部**：把 `run()` 包成一个单 node graph，享受 agent 框架的调度/trace/cost，但不享受 node 级 checkpoint。

**适用**：脚本式任务、简单对话 agent、快速实验。

---

## Level 1: Stepped Agent

需要多步骤、断点恢复、trace 可视化时：

```python
from sigma.agent import Agent, Context, AgentChunk, step

class StockAnalyst(Agent):
    name = "stock-analyst"
    description = "分析期货数据，产出趋势判断"
    triggers = ["期货", "行情", "K线"]
    tools = ["search_web", "fetch_url", "python_exec"]

    @step
    async def gather(self, ctx: Context):
        """收集近 30 天数据"""
        ctx.state["data"] = await ctx.use_tool("fetch_url", url="...")

    @step
    async def analyze(self, ctx: Context):
        """技术面 + 基本面分析"""
        methodology = await ctx.load_skill("data-analysis")
        result = await ctx.llm.chat(f"{methodology}\n\n分析: {ctx.state['data']}")
        ctx.state["report"] = result

    @step
    async def report(self, ctx: Context):
        """输出报告"""
        yield AgentChunk(text=ctx.state["report"])

    # 默认按声明顺序线性执行
    # 需要条件分支时 override：
    def edges(self):
        return {
            "gather": "analyze",
            "analyze": lambda state: "report" if state.get("data") else "gather",
        }
```

**Sigma 内部**：把 `@step` 翻译成 LangGraph nodes，用户不需要知道。

**用户享受**：
- 每个 step 自动 checkpoint（中断后从上次的 step 恢复）
- Trace viewer 里看到 `gather → analyze → report` 的结构
- 可以从某个 step 重跑（debug 友好）

**适用**：多步骤数据处理、需要断点恢复的长任务、需要进度可视化。

---

## Level 2: Graph Agent

需要复杂拓扑（并行 fan-out、动态子图、conditional routing）：

```python
from sigma.agent import Agent, Context
from langgraph.graph import StateGraph, START, END

class ComplexResearcher(Agent):
    name = "complex-researcher"
    description = "多源并行研究，交叉验证后综合"
    triggers = ["深度研究", "多源对比"]
    tools = ["search_web", "fetch_url", "python_exec"]

    def build_graph(self, ctx: Context) -> StateGraph:
        g = StateGraph(ResearchState)
        g.add_node("plan", self.plan)
        g.add_node("search_academic", self.search_academic)
        g.add_node("search_news", self.search_news)
        g.add_node("search_social", self.search_social)
        g.add_node("synthesize", self.synthesize)
        
        g.add_edge(START, "plan")
        # fan-out: 并行搜索三个源
        g.add_edge("plan", ["search_academic", "search_news", "search_social"])
        # fan-in: 全部完成后综合
        g.add_edge(["search_academic", "search_news", "search_social"], "synthesize")
        g.add_edge("synthesize", END)
        return g
```

**明确告知用户**：这层直接依赖 LangGraph API。如果 Sigma 未来换 runtime，Level 2 agent 需要迁移。对 power user 来说这是可接受的 tradeoff——他们需要的是 LangGraph 的完整灵活性。

**适用**：并行 fan-out/fan-in、动态路由、复杂状态机、嵌套子图。

---

## 对比

| | Level 0 | Level 1 | Level 2 |
|---|---|---|---|
| 学习成本 | 零（纯 async 函数） | 低（`@step` 装饰器） | 中（需懂 LangGraph） |
| Node 级 checkpoint | ❌ | ✅ | ✅ |
| Trace 结构化 | 单块 | step 粒度 | node 粒度 |
| 并行 fan-out | 手写 asyncio | ❌ | ✅ |
| 条件路由 | 手写 if/else | `edges()` 支持 | 完全灵活 |
| 可从某 step 重跑 | ❌ | ✅ | ✅ |
| LangGraph 耦合 | 无 | 无 | 有 |
| 适合场景 | 简单脚本、快速实验 | 多步骤任务 | 复杂拓扑 |

---

## 选择指南

```
你的 agent 需要……
  │
  ├─ 只是调几个 API / LLM，返回结果？
  │   → Level 0
  │
  ├─ 多步骤，需要断点恢复、trace 里看到每步进度？
  │   → Level 1
  │
  └─ 并行执行、动态路由、复杂状态机？
      → Level 2
```

**不确定选哪个？从 Level 0 开始。** 需要更多能力时，由 Agent Builder 帮你升级。
