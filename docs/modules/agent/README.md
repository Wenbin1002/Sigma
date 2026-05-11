# Agent 模块

> Sigma 内核的 reasoning 单元——Master / Supervisor / Sub-agent 框架。

> 详见 [架构总览 § 4.4](../../architecture/overview.md#44-agent) 和 [Multi-agent](../../features/multi-agent.md)。

---

## 1. 定位

Agent 是 Sigma 的"执行单元扩展点"。区别于 Tool（行为）和 Skill（知识）：

- **Agent 有自己的 reasoning loop**（自己的 graph、state、prompt）
- **Agent 可以调用 Tool 子集 + 引用 Skill**
- **Agent 之间可协作**（Master 调 Sub，串行/并行/嵌套）

实现基础：LangGraph `StateGraph`。

---

## 2. Agent 层级

```
┌────────────────────────────────┐
│  Master Agent                  │  跟用户对话的"主入口"
│  - 维护 chat 上下文              │
│  - 决策直接做 vs 派给 sub        │
│  - 整合 sub-agent 结果           │
└──────────┬─────────────────────┘
           │
           ▼
┌────────────────────────────────┐
│  Supervisor (路由节点)           │  ← 自动路由用
│  - 看 user input + sub list     │
│  - 选 sub-agent + task desc     │
└──────────┬─────────────────────┘
           │
           ▼
┌────────────────────────────────┐
│  Sub-agents                     │  用户可写 / 内置
│  - Researcher                   │
│  - Coder                        │
│  - Analyst                      │
│  - ...                          │
└────────────────────────────────┘
```

**Master vs Supervisor**：
- Master：跟用户对话、整合结果（"前台"）
- Supervisor：决定派给谁的专门节点（"调度员"）
- 两者可能同 LLM 调用合并（简单场景）也可能拆开（复杂场景）。是 Phase 1 后期需要 benchmark 决定的事（U-2）。

---

## 3. Agent Protocol

所有 agent（内置 / 用户编写）对外遵循统一 Protocol：

```python
class Agent(Protocol):
    name: str
    description: str
    triggers: list[str]                  # @-mention / 路由关键词
    tools: list[str]                     # tool 白名单
    
    async def run(
        self,
        task: str,                       # 派下来的 task description
        ctx: Context,                    # Sigma 提供的能力窗口
    ) -> AsyncIterator[AgentChunk]:
        ...
```

**输入 / 输出**：
- 输入：task description + Context
- 输出：流式 `AgentChunk`（含 text / tool_call / artifact / proxy_decision / needs_input ...）
- 内部：完全黑盒——Sigma 不关心 agent 内部怎么实现

---

## 4. Context 接口

Context 是 Sigma 给 agent 的"能力窗口"，agent 通过它与 Sigma 生态交互：

```python
class Context:
    # LLM 调用
    llm: LLMHandle                    # ctx.llm.chat(...) / ctx.llm.stream(...)
    
    # Tool 调用
    async def use_tool(self, name: str, **kwargs) -> Any
    
    # Skill 加载
    async def load_skill(self, name: str) -> str
    
    # 状态持久化（checkpoint 会存）
    state: dict
    
    # 跟用户/上级 agent 交互
    async def ask_user(self, question: str) -> str       # 阻塞等回答
    async def report_progress(self, msg: str)            # 进度汇报（不阻塞）
    
    # 上下文
    history: list[Message]             # 对话历史（如果有）
    parent_task: str                   # 被派的任务描述
    
    # 子 agent 调用
    async def spawn_agent(self, name: str, task: str) -> AsyncIterator[AgentChunk]
    
    # 成本/限制
    budget: Budget                     # 还剩多少 token/钱
```

**设计原则**：
- Context 是 Sigma 的稳定接口，内部实现可以随时换
- Agent 不需要 import LangGraph、不需要知道 Sigma 的内部结构
- 通过 Context 就能用 Sigma 的全部能力（LLM / Tool / Skill / Checkpoint / 交互）

---

## 5. 三档接入模型

用户根据需求复杂度选择接入层级：

```
┌─────────────────────────────────────────────────┐
│  Level 0: Function Agent  （入门）               │  最简单，大多数场景够用
├─────────────────────────────────────────────────┤
│  Level 1: Stepped Agent   （结构化）             │  享受 checkpoint / resume / 可视化
├─────────────────────────────────────────────────┤
│  Level 2: Graph Agent     （完全灵活）           │  LangGraph 全部能力
└─────────────────────────────────────────────────┘
```

### 5.1 Level 0: Function Agent

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

### 5.2 Level 1: Stepped Agent

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

### 5.3 Level 2: Graph Agent

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

### 5.4 三档对比

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

## 6. Sub-agent 三级回退

详见 [Multi-agent § 4](../../features/multi-agent.md#4-三级回退sub-agent-沟通问题)。

要点：
1. **Sub-agent 自决**——尽力推理 / 用合理默认
2. **Bubble up 主 agent**——用历史 context 代答（带审计）
3. **Bubble up 用户**——chat 追问 / task pause

**两类卡住区分**：
- 资源性（缺 API key / 文件 / 登录）→ 直接 L3
- 认知性（决策歧义 / 缺信息）→ L1 → L2 → L3

**BlockedException 协议**：

```python
class BlockedException(Exception):
    reason: Literal[...]      # 见 ports-and-adapters § 4.3
    detail: str
    needed_input: NeededInput
```

---

## 7. 内置 Agent

| Agent | 角色 | 实现层级 | 状态 |
|---|---|---|---|
| `master` | 跟用户对话的主入口 | Level 2 (Graph) | V0 |
| `supervisor` | 路由节点（可能合并入 master） | Level 2 (Graph) | V2 |
| `researcher` | 资料收集和总结 | Level 2 (Graph) | V2 |
| `coder` | 代码理解 / 修改 / 跑测试 | Level 2 (Graph) | V2 |
| `analyst` | 数据分析 + 趋势判断 | Level 1 (Stepped) | V2 |
| `agent-builder` | 帮用户创建/调试/优化自定义 agent | Level 1 (Stepped) | V2 |

内置 agent 使用 Level 2（因为不需要担心 LangGraph 耦合），但对外满足同一 Agent Protocol。

---

## 8. Agent Builder（内置）

一个特殊的内置 agent，用 Sigma 自己的能力帮助用户创建 agent——**用 Sigma 扩展 Sigma**。

### 8.1 交互流程

```
用户: "我想要一个每天帮我看小红书热帖的 agent"
         ↓
@agent-builder
         ↓
  1. 你想监控哪些话题？
  2. "热帖"的标准是什么？点赞数？评论数？
  3. 结果推送到哪里？
  4. 多久跑一次？
         ↓
  判断需求复杂度 → 选择 Level 0/1/2
         ↓
  生成代码 → ~/.sigma/agents/xhs-monitor/agent.py
         ↓
  "已创建。试跑一下？"
```

### 8.2 能力范围

| 能力 | 说明 |
|---|---|
| **创建** | 引导用户梳理需求 → 选择合适的 Level → 生成 agent 代码 |
| **调试** | 用户说"我的 agent 卡了" → 查看 trace → 定位问题 |
| **升级** | Level 0 不够用了 → 帮迁移到 Level 1 或 Level 2 |
| **优化** | 分析 trace 中的 LLM 调用 → 建议合并/裁剪以降低成本 |

### 8.3 实现

```python
class AgentBuilder(Agent):
    name = "agent-builder"
    description = "帮用户创建、调试、优化自定义 agent"
    triggers = ["写个agent", "创建agent", "新agent", "build agent"]
    tools = ["file_write", "file_read", "python_exec"]
    skills = ["coding"]
```

核心能力来自 LLM 的代码生成 + Agent Protocol 文档作为 skill 注入。不需要额外基建。

### 8.4 时间线

V2 用户 agent 接入稳定后加入。前提是 Agent Protocol 和 Context 接口已冻结。

---

## 9. 用户 Agent 安装

### 9.1 文件结构

```
~/.sigma/agents/
  stock-analyst/
    agent.py                  # 必需：实现 Agent Protocol
    README.md                 # 可选：说明文档
    requirements.txt          # 可选：额外依赖（用户自行管理）
    prompts/                  # 可选：prompt 模板
    
  xhs-monitor/
    agent.py
    ...
```

### 9.2 加载机制

Sigma 启动时：
1. 扫描 `~/.sigma/agents/` 下所有子目录
2. import `agent.py`，找到 `Agent` 子类
3. 检查是否满足 Protocol（name / description / triggers / run）
4. 注册到 supervisor 的路由表

### 9.3 分发模型

```bash
# V1：本地文件夹（手动复制/git clone）
git clone https://github.com/someone/sigma-stock-analyst ~/.sigma/agents/stock-analyst

# V5+（未来）：registry
sigma install agent stock-analyst
sigma install agent gh:user/my-agent
```

V1 只做本地扫描。文件夹结构为未来 registry 留好口子。

### 9.4 依赖管理

用户 agent 如果需要额外依赖（`import pandas`），用户自己装。Sigma 不管——跟 OpenClaw / Claude Code 里用户自己装工具一样。可选放一个 `requirements.txt` 供参考。

---

## 10. Agent 之间协作

### 10.1 串行

Master agent 按顺序 spawn：

```
Master → @researcher → 拿 artifact → @writer → 拿报告
```

### 10.2 并行（fan-out / fan-in）

LangGraph BSP（仅 Level 2）：

```
Master ──┬──→ @theme_analyzer ──┐
         ├──→ @char_analyzer  ──┼──→ aggregate
         └──→ @style_analyzer ──┘
```

### 10.3 嵌套

允许（通过 `ctx.spawn_agent()`），但**不主动设计**——子嵌套容易控制不住时长和成本。Trace viewer 会展示嵌套层级。

---

## 11. 横切能力

| 能力 | 在 agent 中的行为 |
|---|---|
| **Trace** | 每个 agent 调用进入 trace 树状结构。可展开查看 |
| **Cost** | Sub-agent 的 LLM 调用计入 task 总预算 |
| **Cancellation** | 取消传播到所有正在跑的 sub-agent |
| **Timeout** | spawn 时声明，到时强制 cancel |

详见 [Trace 模块](../trace/)。

---

## 12. 实现位置

```
src/agent/
  base.py              # Agent Protocol + Context 接口定义
  context.py           # Context 实现
  master.py            # Master Agent 实现
  supervisor.py        # Supervisor 路由节点
  registry.py          # 内置 + 用户 agent 扫描注册
  spawn.py             # spawn sub-agent + 三级回退处理
  loader.py            # 用户 agent 加载（~/.sigma/agents/ 扫描）
  decorators.py        # @step 装饰器 → LangGraph node 翻译
  
  builtin/             # 内置 sub-agent
    researcher.py
    coder.py
    analyst.py
    agent_builder.py
```

---

## 13. 未决问题

| 问题 | 状态 |
|---|---|
| Master 和 Supervisor 是否合并 | 未决（U-2） |
| Agent metadata 精确 schema | 未决（U-3） |
| `@step` 装饰器是否支持并行 step | 未决——倾向不支持，需要并行请用 Level 2 |
| Agent Builder 的 skill 注入策略 | V2 稳定后再定 |
| Agent 分发 registry | V5+ |

---

## 14. 相关文档

- [Multi-agent](../../features/multi-agent.md) — 路由和三级回退
- [Skill 模块](../skill/) — Agent 引用的方法论
- [Tools 模块](../tools/) — Agent 调用的行为
- [Context 模块](../context/) — Agent 看到的上下文
- [Trace 模块](../trace/) — Agent 执行的可观察性
