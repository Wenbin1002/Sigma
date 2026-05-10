# 新手指南

## 前置要求

- Python 3.11+
- Git
- 一个代码编辑器（推荐 VS Code / Cursor）

## 项目当前状态

> 🚧 **早期设计阶段** — 架构基本定型，**`src/` 尚未开始编码**。

这意味着当前最需要的贡献是：

- 阅读 [设计决策日志](../architecture/design-log.md)，对设计提反馈
- 在 `experiments/` 中跑通技术验证（LangGraph 用法、Trace 协议、cost guard 实现）
- 等代码骨架建好后，按 [V0 退出标准](../roadmap.md#v0hello-agent最小可用骨架) 贡献

## 快速理解 Sigma

**三句话版本：**

1. Sigma 是**通用 personal AI 助手**，对标 ChatGPT + Codex 的本地版本
2. **Chat + Task 双视图**统一 task queue；**Multi-agent 三级回退**保证执行可控
3. **三层正交扩展**：Tool（行为）/ Skill（知识）/ Agent（执行单元）

**5 分钟版本**：读 [README.md](../../README.md) 和 [架构总览](../architecture/overview.md) 的 § 1-2。

**深入版本**：按下文"应该先读的文件"顺序读完。

## 日常使用（待 V0 实现后）

```bash
# Chat 模式
sigma chat                              # 进入对话流
sigma chat "一句话问题"                 # 单次问答
sigma chat --resume <session_id>        # 续之前的 session

# Task 模式
sigma task new "..."                    # 一次性 task
sigma task new "..." --daily 09:00      # 每天 9 点跑（V4）
sigma task list                         # 任务列表
sigma task view <id>                    # 查看进度
sigma task pause/resume/cancel <id>

# Trace
sigma trace view <trace_id>             # 浏览器看某次执行
sigma trace replay <trace_id>           # 基于历史重跑
```

## 应该先读的文件

按优先级：

1. **[README.md](../../README.md)** — Sigma 是什么 / 不是什么（3 分钟）
2. **[CLAUDE.md](../../CLAUDE.md)** — 项目规则总纲（3 分钟）
3. **[架构总览](../architecture/overview.md)** — 系统设计（10 分钟）
4. **[Chat / Task 模式](../architecture/chat-task-modes.md)** — 双视图核心（5 分钟）
5. **[Multi-agent](../architecture/multi-agent.md)** — Sub-agent 三级回退（10 分钟）
6. **[依赖规则](../architecture/dependency-rules.md)** — Import 约束（5 分钟）
7. **[设计决策日志](../architecture/design-log.md)** — 每个决定的来龙去脉（按需）
8. **你感兴趣的模块文档** — `docs/modules/<name>/`（5-10 分钟/个）
9. **[添加扩展](adding-an-adapter.md)** — 准备写代码时（5 分钟）

## 第一次贡献建议

按门槛从低到高（详见 [CONTRIBUTING.md](../../CONTRIBUTING.md)）：

| 类型 | 门槛 | 怎么开始 |
|------|------|---------|
| 写 Skill | ⭐ | 写一段 markdown，描述某个领域的方法论。**不需要 Python**。详见 [Skill 模块](../modules/skill/) |
| 接 LLM provider / Tool | ⭐⭐ | 实现 Port + 注册。详见 [添加扩展](adding-an-adapter.md) |
| 写 Sub-agent | ⭐⭐⭐ | 继承 `sigma.Agent`，定义 graph。详见 [Multi-agent](../architecture/multi-agent.md) |
| 内核改进 | ⭐⭐⭐⭐ | Trace viewer / Cost guard / Context engine。先开 issue 讨论 |
| 文档 | ⭐ | 修正、补充、翻译 |

## 开发工作流

```
1. git fetch origin
2. git checkout -b <type>/<short-desc> origin/main
3. 开发 + 测试
4. git add + git commit（遵循 commit 规范）
5. git push + 提 PR
```

详见 [贡献指南](../../CONTRIBUTING.md)。

## 相关文档

- [贡献指南](../../CONTRIBUTING.md) — 完整开发流程
- [代码规范](code-style.md) — 命名和风格
- [添加扩展](adding-an-adapter.md) — Tool / Skill / Sub-agent / LLM 添加步骤
- [路线图](../roadmap.md) — 知道项目走到哪一步了
