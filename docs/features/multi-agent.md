# Multi-Agent

> Sigma 的 multi-agent 设计：**Supervisor 自动路由 + @-mention 显式召唤** 双入口，**三级回退** 解决 sub-agent 沟通问题。

---

## 1. 为什么要做 multi-agent

Sigma 的扩展模型有 Tool / Skill / Agent 三层：

| | 解决什么 | 限制 |
|---|---|---|
| Tool | 单个动作（"发邮件"、"抓网页"） | 不会推理 |
| Skill | 给 agent 注入"怎么做"的方法论 | 不会拆任务 / 不维护状态 |
| **Agent** | **完整的 reasoning loop**——能拆任务、调多个 tool、自己推进 | **复杂任务的真正承载** |

Skill 一次 LLM 调用做不了复杂任务（比如"分析生猪期货走势"——要拉数据、清洗、推理、画图、综合判断）。这种场景**必须**走 sub-agent。

---

## 2. 双入口路由

### 2.1 入口 A：Supervisor 自动（默认）

```
User: 帮我分析一下生猪期货走势
  ↓
Master Agent 收到，进入 Supervisor 节点
  ↓ 看 user input + 所有可用 sub-agent 的 description
  ↓ 决定路由
  ↓ 派给 Analyst Agent
```

**Supervisor 的职责**：
- 输入：user input + 历史 context
- 输出：选哪个 sub-agent + 给它什么 task description
- 实现：一次专门优化过的 LLM 调用，prompt 专注于"理解意图 + 选 agent"

**Supervisor ≠ Master Agent**。两者职责不同：
- Master Agent：负责跟用户对话、整合 sub-agent 结果、最终回复
- Supervisor：负责"派给谁"这一个决策

实际可能合并实现（同一个 LLM 调用做两件事）也可能拆开（两次 LLM 调用，各自专注）。这是实现细节，未来根据效果决定。

### 2.2 入口 B：@-mention 显式（用户主动）

```
User: @researcher 帮我查一下数据
  ↓
解析 @researcher → 跳过 Supervisor
  ↓ 直接派给 Researcher Agent
```

**适合**：
- 用户已经知道要调谁（信任感强、避免误路由）
- 多个 agent 协作时精确控制顺序：
  ```
  User: @researcher 收集数据
  ...
  User: @analyst 基于刚才的数据分析趋势
  ```

### 2.3 两条路的关系

```
              User Input
                  │
                  ▼
        ┌─────────────────────┐
        │ Parse @-mention?    │
        └────┬────────────┬───┘
             │ no         │ yes
             ▼            ▼
        ┌─────────┐   ┌──────────────┐
        │Supervisor│   │ Direct Route │
        └────┬────┘   └──────┬───────┘
             │               │
             └───────┬───────┘
                     ▼
             派给 sub-agent 执行
```

---

## 3. Sub-agent 的契约

### 3.1 输入 / 输出

```python
class Agent(Protocol):
    name: str
    description: str           # Supervisor 看这个做路由
    tools: list[Tool]          # 这个 agent 能调的 tool 子集
    
    async def run(
        self,
        task: str,             # 主 agent 派下来的 task description
        context: AgentContext, # chat 历史 / artifact / parent state
    ) -> AsyncIterator[AgentChunk]:
        ...
```

**Agent 内部完全黑盒**——自己的 prompt、graph、state、memory scope。外部只看 input / output。

### 3.2 Agent metadata

每个 sub-agent 必须声明：

| 字段 | 用途 |
|---|---|
| `name` | @-mention 用 / trace 标识 |
| `description` | Supervisor 路由用（一段简短描述，写清楚"擅长什么 / 不擅长什么"） |
| `tools` | 能调用的 tool 子集 |
| `triggers`（可选） | 显式触发词 / 关键词，提高路由准确度 |

---

## 4. 三级回退：Sub-agent 沟通问题

**这是 multi-agent 设计里最难的一点。** Sub-agent 跑过程中需要追问怎么办？

### 4.1 三级回退总览

```
Sub-agent 遇到不确定 / 缺信息
   ↓
┌─────────────────────────────────────────────┐
│ Level 1: sub-agent 自己尽力                  │
│  - 用合理默认                                  │
│  - 多种解释下选最可能的                          │
│  - 失败的话准备 raise BlockedException          │
└─────────────────────────────────────────────┘
   ↓ 仍然不行（raise BlockedException）
┌─────────────────────────────────────────────┐
│ Level 2: 主 agent 代答                        │
│  - 主 agent 看 chat 历史 + sub-agent 的问题     │
│  - 调一次 LLM 决策                              │
│  - 如果能确信回答 → 把决策传回 sub-agent          │
│  - 否则升级到 L3                                │
└─────────────────────────────────────────────┘
   ↓ 主 agent 也搞不定
┌─────────────────────────────────────────────┐
│ Level 3: Bubble up 到用户                     │
│  - Chat mode：直接追问，等用户当下回答             │
│  - Task mode：task → paused，等用户 resume      │
└─────────────────────────────────────────────┘
```

### 4.2 两类卡住的区分

| 类型 | 例子 | 走哪条 |
|---|---|---|
| **认知性卡住** | 决策歧义（"用 A 风格还是 B 风格"）<br>缺少推理依据（"用户提到的'那个项目'是哪个"） | **L1 → L2 → L3**（先尝试自决，再尝试代答） |
| **资源性卡住** | API key 缺失<br>登录态过期<br>必需文件不存在<br>需要外部许可 | **直接 L3**（主 agent 也变不出资源） |

### 4.3 BlockedException 协议

Sub-agent 卡住时不能 raise 通用异常，必须用结构化的：

```python
class BlockedException(Exception):
    reason: Literal[
        "missing_credential",      # 资源性
        "missing_file",            # 资源性
        "external_dependency",     # 资源性
        "ambiguous_intent",        # 认知性
        "missing_information",     # 认知性
        "decision_needed",         # 认知性
    ]
    detail: str                    # 给主 agent / 用户看的人话描述
    needed_input: NeededInput      # 需要什么类型的输入（schema）
```

**主 agent 看 reason 决定走 L2 还是 L3**：
- `reason in {missing_credential, missing_file, external_dependency}` → 直接 L3
- 其他 → 尝试 L2，失败再 L3

### 4.4 L2 主 agent 代答

```python
# 简化伪代码
async def handle_blocked(blocked: BlockedException, context):
    if blocked.reason in RESOURCE_REASONS:
        # 资源性卡住，主 agent 也无能为力
        return await escalate_to_user(blocked, context)
    
    # 认知性卡住，主 agent 尝试代答
    answer = await master_llm.decide(
        prompt=f"""
        Sub-agent 在执行 task 时遇到问题：{blocked.detail}
        需要的输入：{blocked.needed_input}
        历史 context：{context.chat_history}
        
        基于 user 的意图历史，你能回答这个问题吗？
        如果能，给出答案。如果不能，回答 NEED_USER。
        """
    )
    
    if answer == "NEED_USER":
        return await escalate_to_user(blocked, context)
    
    # 代答成功，记录 trace + 标记审计
    trace.record_proxy_decision(blocked, answer)
    return answer
```

### 4.5 代答审计

主 agent 代答后**必须**显式说明，保留用户 override 机会：

```
Chat mode：
  Sigma：「(我替你决定了：使用最新版本而非之前的稳定版本) Task 已完成」

Task mode：
  Task summary 里出现：
  「⚠️ 执行中代答了 1 个决策：
    - 问题：'用 A 风格还是 B 风格' → 我选了 A（基于你之前提到偏好）
    如不符合预期，请告知修正」
```

代答的所有信息也进入 trace（reason / question / answer / 主 agent 推理依据），便于 replay 和 debug。

### 4.6 L3 升级到用户

```python
async def escalate_to_user(blocked, context):
    if context.mode == "chat":
        # Chat：直接弹追问消息
        user_input = await chat.ask(blocked.detail, blocked.needed_input)
        return user_input
    
    elif context.mode == "task":
        # Task：进入 paused
        task.status = "paused"
        task.pending_question = blocked.detail
        task.needed_input = blocked.needed_input
        notify_user(task)
        # 这里 LangGraph interrupt() 暂停执行
        # 用户提供输入后，task.resume(input) 重新进入 sub-agent
```

---

## 5. Sub-agent 之间的协作

### 5.1 串行（顺序调用）

```
User: 帮我研究并写报告
  ↓
Supervisor: 先派 researcher，再派 writer
  ↓
@researcher 收集 → artifact A
  ↓
@writer 基于 A 写报告 → artifact B
```

实现上：Master agent 跑一个 graph，按顺序 spawn 不同 sub-agent。

### 5.2 并行（fan-out / fan-in）

```
User: 同时分析这本书的主题、人物、风格
  ↓
Supervisor: 三个 sub-agent 并行
  ↓
@theme_analyzer ─┐
@char_analyzer  ─┼→ 汇总
@style_analyzer ─┘
```

实现上：用 LangGraph 的 BSP 并行（一个 super-step 内三个 sub-agent 同时跑，下个 super-step 汇总）。

### 5.3 嵌套（sub-agent 调 sub-sub-agent）

允许，但**不推荐主动设计**——容易控制不住执行时长和成本。如果出现，trace viewer 会展示嵌套层级。

---

## 6. Trace / Cost / Cancellation 传播

| 横切能力 | 在 multi-agent 中的行为 |
|---|---|
| **Trace** | Master / Supervisor / Sub-agent 各自的执行都进入 trace 树状结构。Viewer 里能展开看每一层 |
| **Cost** | Sub-agent 的 LLM 调用计入 task 总预算。超预算 → master agent 收到 interrupt，决定 cancel 还是降级 |
| **Cancellation** | 用户取消 task → 信号传播到所有正在跑的 sub-agent → 各 sub-agent 处理 cleanup |
| **Timeout** | 在 spawn sub-agent 时声明 timeout，到时强制 cancel |

具体实现细节走 LangGraph 的 cancel scope + interrupt 机制。

---

## 7. 用户怎么写 sub-agent

### 7.1 注册位置

```
~/.sigma/agents/
  researcher/
    agent.py          # Python 类继承 sigma.Agent
    prompts/          # 可选：prompt 模板
    tools/            # 可选：私有 tool（不暴露给其他 agent）
  
  coder/
    agent.py
```

### 7.2 最小骨架

```python
from sigma import Agent, StateGraph, START, END

class ResearcherAgent(Agent):
    name = "researcher"
    description = "擅长资料收集、网页抓取、信息总结。不擅长写代码或数据分析。"
    triggers = ["调研", "查一下", "research"]
    tools = [search_web, fetch_url, summarize]
    
    def build_graph(self) -> StateGraph:
        g = StateGraph(ResearchState)
        g.add_node("plan", self.plan)
        g.add_node("search", self.search)
        g.add_node("synthesize", self.synthesize)
        g.add_edge(START, "plan")
        g.add_edge("plan", "search")
        g.add_edge("search", "synthesize")
        g.add_edge("synthesize", END)
        return g
```

### 7.3 分发与发现（V3+）

未来支持从 registry 下载别人的 agent：

```bash
sigma agent install <name-or-url>
```

但**安全模型 / 沙箱 / 版本管理**等问题还未解决，详见 [设计决策日志](design-log.md) U-x。

---

## 8. 未决问题（明确不在本文件解决）

| 问题 | 状态 |
|---|---|
| Master Agent 和 Supervisor 是同一 LLM 调用还是独立 node | 未决（U-2） |
| Agent metadata 的精确 schema | 未决（U-3） |
| BlockedException reason 类型清单 | 未决（U-4） |
| Sub-agent 分发与发现的安全模型 | 未决（V3+） |

---

## 9. 相关文档

- [架构总览](overview.md) — 三层扩展模型
- [Chat / Task 模式](chat-task-modes.md) — Pause 在 task 中的实现
- [Agent 模块](../modules/agent/) — 内核实现细节
- [Trace 模块](../modules/trace/) — 多层 trace 展示
- [设计决策日志](design-log.md) — Multi-agent 决策历史
