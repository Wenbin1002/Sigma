# dev-log/

> Sigma 的开发日志。每个工作单元(对应一个 PR)在这里留下"立项 → 设计 → 实施 → review"的完整轨迹。

---

## 1. 这是什么

`dev-log/` 跟 `docs/` 的边界:

| | `docs/` | `dev-log/` |
|---|---|---|
| 时态 | 描述**当前的**系统 | 记录**某次开发的**过程 |
| 受众 | 用户 / 贡献者 | 维护者 / agent |
| 生命 | 跟着代码长期维护 | 写完那一刻就冻结 |
| 例子 | `architecture/overview.md` | `v0.1/03-llm-adapter/issue.md` |

`docs/` 回答 "Sigma 是什么";`dev-log/` 回答 "Sigma 是怎么变成现在这样的"。

---

## 2. 目录结构

```
dev-log/
  README.md                 ← 本文档(体系约定)
  _template/                ← 新 unit 的拷贝起点(只含 issue.md / design.md;
                              review.html 由 /review skill 生成,无模板)
    issue.md
    design.md
  v0.1/                     ← 当前按 milestone 分组
    README.md               ← 该组的索引
    01-project-skeleton/    ← 一个 unit
      issue.md
      design.md             ← 开发前补
      review.html           ← /review skill 生成,merge 之前必看
    02-core-schemas-ports/
    ...
```

每个 unit 对应一个独立可合并的 PR。

`vX.Y/` 不是 dev-log 的固有约定,只是当前的分组方式——后续可改用 `2026-Q4/`(按时间)、`multi-agent/`(按主题)等其他维度。

---

## 3. 三份产物

### 3.1 `issue.md` — 立项

**写在什么时候**:工作单元启动、规划阶段。

**回答的问题**:做什么 / 为什么 / 边界在哪?

**内容**:
- 背景(关联 roadmap 的哪一条)
- 范围(具体要交付什么)
- 设计要点(关键约束 / 反模式)
- 验收标准(明确的 checkbox)
- 不做(显式划边界,避免 scope creep)
- 依赖(其他 unit / 外部前置)

`issue.md` 是 what + why,不是 how。how 在 `design.md`。

### 3.2 `design.md` — 设计

**写在什么时候**:该 unit 真正开发**之前**。

**回答的问题**:怎么实现?有哪些备选?选哪个?

**内容**:
- 选定方案的关键技术决策(数据结构 / 调用流程 / 接口形状)
- 备选方案 + 否决理由(没思考过的不要硬凑)
- 跟其他 unit / 模块的接口
- 风险点(已知会踩坑的地方)
- 测试策略

简单 unit 一段话也算 design.md;复杂 unit 才需要展开。

### 3.3 `review.html` — Code review 报告

**写在什么时候**:该 unit 的 PR 实施完成、merge **之前**,由 [`/review` skill](../skills/review/) 生成。

**回答的问题**:这次实施的代码质量如何?能不能合?

**内容**:
- 改动概览(文件 / 行数 / diff 摘要)
- 发现按严重度分类(blocker / major / minor / nit)
- 架构 / 依赖规则自检结果
- 测试覆盖评估
- 衍生 issue 链接(需要后续 unit 处理的)

**形态**:HTML 自包含报告。**作为 merge 决策依据**——人看完 review.html,确认没 blocker 才合 PR。retrospective 类内容(如果重做一次会怎么改)不在本文件范围,放到 milestone 完结时的 design-log 提炼。

---

## 4. 跟 `docs/architecture/design-log.md` 的边界

| | `design-log.md` | `dev-log/<group>/<unit>/design.md` |
|---|---|---|
| 粒度 | 项目级、跨 unit | unit 级、单次实施 |
| 影响范围 | 长期 / 多模块 | 局部 / 一次 PR |
| 例子 | "为什么 Port 数量从 8 个砍到 4 个" | "为什么 OpenAI adapter 选 httpx 而不是 openai SDK" |

**何时从 dev-log 提升到 design-log**:

读 `review.html` 时如果发现某条 finding 触及"影响后续多个 unit / 改了项目方向"的决策,把它提炼一句到 `design-log.md`,留链接回到 `review.html`。

一组 unit 全部完成时统一过一遍各 review.html → 提炼 → design-log。

---

## 5. Agent 工作流

每个 unit 走四个 stage,每个 stage 都有"agent 写产物 → 人 review"的对子:

```
┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: 立项                                                    │
│   人:roadmap 列出方向                                           │
│   agent:拆出 unit、写每个 unit 的 issue.md                      │
│   人:review issue.md → 调整范围 / 验收 / 依赖 → approve        │
└─────────────────────────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 2: 设计(每个 unit 开发前)                                │
│   agent:写 design.md(方案 + 备选 + 风险)                      │
│   人:review design.md → 讨论 / 选方案 → approve                │
└─────────────────────────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 3: 实施                                                    │
│   agent:按 issue.md + design.md 写代码 → 提 PR                 │
└─────────────────────────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 4: Review(merge 之前)                                     │
│   agent:跑 /review skill → 生成 review.html                    │
│   人:看 review.html → 没 blocker → merge PR                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 命名约定

| 项 | 规则 | 示例 |
|---|---|---|
| 分组目录 | 当前按 milestone:`vX.Y/` | `v0.1/`, `v0.2.5/` |
| unit 目录 | `NN-kebab-name`(NN = 两位序号) | `03-llm-adapter/` |
| 三份产物 | 固定文件名 | `issue.md` / `design.md` / `review.html` |
| 跨 unit 引用 | 相对路径 | `../03-llm-adapter/issue.md` |

unit 序号反映依赖顺序(low → high),不必等于实施顺序。

---

## 7. 不做什么

- 不在 `dev-log/` 写"使用文档" — 那是 `docs/` 的事
- 不把 `dev-log/` 当 todo list — 状态走 GitHub Issues / 本地 task 系统,`dev-log/` 只放冻结的产物
- 不在 design.md 凑字数 — 简单 unit 一段话即可
- 不重复 `docs/` 的内容 — 引用就行(`参考 [Tools 模块](../../docs/modules/tools/README.md)`)

---

## 8. 相关文档

- [roadmap.md](../docs/roadmap.md) — 项目方向规划
- [docs/architecture/design-log.md](../docs/architecture/design-log.md) — 项目级决策日志
- [CONTRIBUTING.md](../CONTRIBUTING.md) — git workflow
- [CLAUDE.md](../CLAUDE.md) — 项目规则总纲(含 agent 工作流入口)
