# Agent 模块

> Sigma 内核的 reasoning 单元——Master / Supervisor / Sub-agent 框架。

> 详见 [架构总览 § 4.4](../../architecture/overview.md#44-agent) 和 [Multi-agent](../../architecture/multi-agent.md)。

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
┌────────────────────────────┐
│  Master Agent              │  跟用户对话的"主入口"
│  - 维护 chat 上下文          │
│  - 决策直接做 vs 派给 sub      │
│  - 整合 sub-agent 结果        │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│  Supervisor (路由节点)       │  ← 自动路由用
│  - 看 user input + sub list │
│  - 选 sub-agent + task desc  │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│  Sub-agents                  │  用户可写 / 内置
│  - Researcher                │
│  - Coder                     │
│  - Analyst                   │
│  - ...                       │
└────────────────────────────┘
```

**Master vs Supervisor**：
- Master：跟用户对话、整合结果（"前台"）
- Supervisor：决定派给谁的专门节点（"调度员"）
- 两者可能同 LLM 调用合并（简单场景）也可能拆开（复杂场景）。是 Phase 1 后期需要 benchmark 决定的事（U-2）。

---

## 3. Agent 契约

```python
class Agent(Protocol):
    name: str
    description: str
    triggers: list[str]                  # @-mention / 路由关键词
    tools: list[Tool]
    allowed_skills: list[str] | None      # None = 全部允许
    
    async def run(
        self,
        task: str,                       # 派下来的 task description
        context: AgentContext,           # chat 历史 / artifact / parent state
    ) -> AsyncIterator[AgentChunk]:
        ...
```

**输入 / 输出**：
- 输入：task description + context
- 输出：流式 `AgentChunk`（含 text / tool_call / artifact / proxy_decision / needs_input ...）
- 内部：完全黑盒

---

## 4. Sub-agent 三级回退

详见 [Multi-agent § 4](../../architecture/multi-agent.md#4-三级回退sub-agent-沟通问题)。

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

## 5. 内置 Agent

| Agent | 角色 | 状态 |
|---|---|---|
| `master` | 跟用户对话的主入口 | V0 |
| `supervisor` | 路由节点（可能合并入 master） | V2 |
| `researcher` | 资料收集和总结 | V2 |
| `coder` | 代码理解 / 修改 / 跑测试 | V2 |
| `analyst` | 数据分析 + 趋势判断（场景 2 关键） | V2 |

每个 sub-agent 自己定义 prompt / graph / tool 子集。

---

## 6. 用户写 Sub-agent

```
~/.sigma/agents/
  my_agent/
    agent.py                # 必需
    prompts/                # 可选
    tools/                  # 可选私有 tool
```

```python
# agent.py
from sigma import Agent, StateGraph, START, END

class MyAgent(Agent):
    name = "my_agent"
    description = "..."
    triggers = ["关键词"]
    tools = [...]
    
    def build_graph(self) -> StateGraph:
        g = StateGraph(MyState)
        g.add_node("plan", self.plan)
        g.add_node("act", self.act)
        g.add_edge(START, "plan")
        g.add_edge("plan", "act")
        g.add_edge("act", END)
        return g
```

Sigma 启动时扫描 `~/.sigma/agents/` + `sigma/agents/`，自动注册。

---

## 7. Agent 之间协作

### 7.1 串行

Master agent 一个 graph 里按顺序 spawn：

```
Master → @researcher → 拿 artifact → @writer → 拿报告
```

### 7.2 并行（fan-out / fan-in）

LangGraph BSP：

```
Master ──┬──→ @theme_analyzer ──┐
         ├──→ @char_analyzer  ──┼──→ aggregate
         └──→ @style_analyzer ──┘
```

### 7.3 嵌套

允许，但**不主动设计**——子嵌套容易控制不住时长和成本。Trace viewer 会展示嵌套层级。

---

## 8. 横切能力

| 能力 | 在 agent 中的行为 |
|---|---|
| **Trace** | 每个 agent 调用进入 trace 树状结构。可展开查看 |
| **Cost** | Sub-agent 的 LLM 调用计入 task 总预算 |
| **Cancellation** | 取消传播到所有正在跑的 sub-agent |
| **Timeout** | spawn 时声明，到时强制 cancel |

详见 [Trace 模块](../trace/)。

---

## 9. 实现位置

```
src/agent/
  base.py              # Agent Protocol / 基类
  master.py            # Master Agent 实现
  supervisor.py        # Supervisor 路由节点
  registry.py          # 内置 agent + 用户 agent 扫描
  spawn.py             # spawn sub-agent + 三级回退处理
  context.py           # AgentContext 数据结构
  
  builtin/             # 内置 sub-agent
    researcher.py
    coder.py
    analyst.py
```

---

## 10. 未决问题

| 问题 | 状态 |
|---|---|
| Master 和 Supervisor 是否合并 | 未决（U-2） |
| Agent metadata 精确 schema | 未决（U-3） |
| Agent 分发 / 安全模型 | V5+ |

---

## 11. 相关文档

- [Multi-agent](../../architecture/multi-agent.md) — 路由和三级回退
- [Skill 模块](../skill/) — Agent 引用的方法论
- [Tools 模块](../tools/) — Agent 调用的行为
- [Context 模块](../context/) — Agent 看到的上下文
- [Trace 模块](../trace/) — Agent 执行的可观察性
