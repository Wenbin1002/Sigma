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

**默认策略：retry + best-effort 降级**——失败的 agent 先重试，仍失败则降级为 best-effort。

```
@search_news 失败
    → 重试（最多 N 次，指数退避）
    → 仍失败
    → 标记 degraded，其余成功结果 + 失败信息一起传给下游

下游 synthesize 拿到：
  @search_academic  → ✅ 成功，数据...
  @search_news      → ❌ degraded，原因: timeout after 30s，已重试 2 次
  @search_social    → ✅ 成功，数据...
```

下游 agent（如 synthesize）负责在不完整数据下做推理，并告知用户哪些源缺失、结论可信度如何。

**重试策略**：
- 默认重试次数：2（可配置）
- 退避：指数退避（1s → 2s → 4s）
- 可重试条件：timeout / 网络错误 / 临时性异常。BlockedException（资源性卡住）不重试

> **TODO**：后续支持可配置的失败策略（如 `require_all=True` 强制 all-or-nothing，适用于"数据必须完整才有意义"的场景如测试全通过才能部署）。

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

---

## 外部 CLI Agent 委托（预留接口）

> **定位：逃生舱，不是主路径。** 核心场景必须自研——外包执行层会导致 Sigma 的横切能力（trace / cost / memory / checkpoint）全部失效，产品没有壁垒。此模式仅用于"确实不值得自己做"的边缘场景。

Sigma 可以将特定场景委托给外部 CLI agent 执行，而非自己做 reasoning。

### 执行模型

```
Sigma Master（before）:
  - 从 memory/context 组装偏好、规范、相关上下文
  - 注入到 prompt 中
         ↓
外部 CLI agent（黑盒执行）:
  - 子进程执行，Sigma 不干预过程
  - 内部自己 reason、调 tool
         ↓
Sigma Master（after）:
  - 收结果，沉淀到 memory/trace
  - 汇报用户
```

### 在架构里的位置

外部 CLI agent 就是一个普通的 sub-agent adapter，满足 Agent Protocol：

```python
class ExternalCLIAgent(Agent):
    name = "external-cli"
    description = "委托给外部 CLI agent 执行"
    triggers = [...]
    tools = []  # 外部 agent 自带 tool，Sigma 不管

    async def run(self, task: str, ctx: Context):
        # before: 注入 context
        prompt = f"{ctx.preferences}\n\n{task}"
        
        # 执行: 子进程，流式读 stdout
        process = await asyncio.create_subprocess_exec(
            "some-cli", "-p", prompt,
            stdout=asyncio.subprocess.PIPE
        )
        async for line in process.stdout:
            yield AgentChunk(text=line.decode())
```

### 为什么不用于核心场景

| 问题 | 说明 |
|------|------|
| 横切能力失效 | Trace 只记一次调用；Cost 黑盒；Checkpoint 无法恢复 |
| 体验割裂 | 两套行为风格/输出格式缝合 |
| 依赖风险 | 外部产品接口变动 → 核心场景挂 |
| 学不到核心难点 | 跳过了最有价值的 agent 设计挑战 |

### 适用边界

仅当同时满足以下条件时考虑：
- 场景是非核心/低频的
- 自研 ROI 极低（比如适配某个非常 niche 的 CLI 工具）
- 不需要 Sigma 的横切能力介入执行过程
