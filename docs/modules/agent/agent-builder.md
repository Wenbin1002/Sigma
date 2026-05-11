# Agent Builder

> 内置 agent，用 Sigma 自己的能力帮用户创建/调试/优化自定义 agent——**用 Sigma 扩展 Sigma**。

---

## 交互流程

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

用户全程不需要知道 Level 0/1/2 的存在——builder 根据需求自动选择。

---

## 能力范围

| 能力 | 说明 |
|---|---|
| **创建** | 引导用户梳理需求 → 选择合适的 Level → 生成 agent 代码 |
| **调试** | 用户说"我的 agent 卡了" → 查看 trace → 定位问题 |
| **升级** | Level 0 不够用了 → 帮迁移到 Level 1 或 Level 2 |
| **优化** | 分析 trace 中的 LLM 调用 → 建议合并/裁剪以降低成本 |

---

## 为什么比"写文档教用户"强

| 方式 | 问题 |
|------|------|
| 文档 | 用户得自己读、自己判断用哪层、自己写 |
| CLI scaffold (`sigma agent new`) | 生成模板但不理解需求 |
| **Agent Builder** | 理解需求 → 选模式 → 生成代码 → 还能帮调试 |

---

## 实现

```python
class AgentBuilder(Agent):
    name = "agent-builder"
    description = "帮用户创建、调试、优化自定义 agent"
    triggers = ["写个agent", "创建agent", "新agent", "build agent"]
    tools = ["file_write", "file_read", "python_exec"]
    skills = ["coding"]
```

核心能力来自 LLM 的代码生成 + Agent Protocol 文档作为 skill 注入。不需要额外基建。

---

## 时间线

V2 用户 agent 接入稳定后加入。前提：
- Agent Protocol 和 Context 接口已冻结
- 三档接入模型实现完整
- 有足够的示例 agent 作为参考
