# Agent 模块

> Sigma 内核的 reasoning 单元——Master / Supervisor / Sub-agent 框架。

---

## 定位

Agent 是 Sigma 的"执行单元扩展点"。区别于 Tool（行为）和 Skill（知识）：

- **Agent 有自己的 reasoning loop**（自己的 graph、state、prompt）
- **Agent 可以调用 Tool 子集 + 引用 Skill**
- **Agent 之间可协作**（Master 调 Sub，串行/并行/嵌套）

实现基础：LangGraph `StateGraph`。

---

## Agent 层级

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

## 内置 Agent

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

## 三级回退

详见 [Multi-agent](../../features/multi-agent.md)。

要点：
1. **Sub-agent 自决**——尽力推理 / 用合理默认
2. **Bubble up 主 agent**——用历史 context 代答（带审计）
3. **Bubble up 用户**——chat 追问 / task pause

---

## 横切能力

| 能力 | 在 agent 中的行为 |
|---|---|
| **Trace** | 每个 agent 调用进入 trace 树状结构 |
| **Cost** | Sub-agent 的 LLM 调用计入 task 总预算 |
| **Cancellation** | 取消传播到所有正在跑的 sub-agent |
| **Timeout** | spawn 时声明，到时强制 cancel |

---

## 实现位置

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

## 未决问题

| 问题 | 状态 |
|---|---|
| Master 和 Supervisor 是否合并 | 未决（U-2） |
| Agent metadata 精确 schema | 未决（U-3） |
| `@step` 装饰器是否支持并行 step | 未决——倾向不支持，需要并行请用 Level 2 |
| Agent Builder 的 skill 注入策略 | V2 稳定后再定 |
| Agent 分发 registry | V5+ |

---

## 子文档

| 文档 | 内容 |
|------|------|
| [protocol.md](protocol.md) | Agent Protocol + Context 接口 |
| [access-levels.md](access-levels.md) | 三档接入模型（Level 0/1/2） |
| [user-agents.md](user-agents.md) | 用户 agent 安装 / 加载 / 分发 |
| [agent-builder.md](agent-builder.md) | Agent Builder 内置 agent |
| [collaboration.md](collaboration.md) | Agent 之间协作模式 |

---

## 相关文档

- [Multi-agent](../../features/multi-agent.md) — 路由和三级回退
- [Skill 模块](../skill/) — Agent 引用的方法论
- [Tools 模块](../tools/) — Agent 调用的行为
- [Context 模块](../context/) — Agent 看到的上下文
- [Trace 模块](../trace/) — Agent 执行的可观察性
