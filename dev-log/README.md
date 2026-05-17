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
  _template/                ← 新 unit 的拷贝起点
    issue.md
    requirement.md          ← 可选,只在复杂 unit 写
    design.md
  v0.1/                     ← 当前按 milestone 分组
    README.md               ← 该组的索引
    01-project-skeleton/    ← 一个 unit
      issue.md
      design.md             ← 实施前补
      review.html           ← /review skill 生成,merge 之前必看
    02-core-schemas-ports/
    ...
```

每个 unit 对应一个独立可合并的 PR。

`vX.Y/` 不是 dev-log 的固有约定,只是当前的分组方式——后续可改用 `2026-Q4/`(按时间)、`multi-agent/`(按主题)等其他维度。

---

## 3. 产物

### 3.1 `issue.md` — 立项(必填)

**写在什么时候**:milestone 立项,跟其他 unit 一起批量产出。

**回答的问题**:做什么 / 为什么 / 边界在哪?

**内容**:
- 背景(关联 roadmap 的哪一条)
- 范围(具体要交付什么)
- 设计要点(关键约束 / 反模式)
- 验收标准(明确的 checkbox)
- 不做(显式划边界)
- 依赖(其他 unit / 外部前置)

`issue.md` 是 what + why,不是 how。how 在 `design.md`。

### 3.2 `requirement.md` — 需求(可选)

**写在什么时候**:**只在你明确要求时写**——通常是 unit 的需求空间大、要先把"做不做、做到什么程度"想清楚才能拆 issue 的场景。

**触发情形**:
- roadmap 描述只有一句话,具体边界完全没定
- 涉及用户场景驱动的功能,需要先梳理用户故事
- 跟多个其他 unit / 外部依赖耦合,需求边界本身要谈

**回答的问题**:用户/系统真正需要什么?

**内容**:
- 用户故事 / 场景
- 功能性需求(必须满足的能力)
- 非功能性需求(性能 / 安全 / 兼容性)
- 显式取舍(为什么不做 X)
- 验收边界(怎么判断需求被满足)

简单 unit(fix / refactor / 内部模块)不写。

### 3.3 `design.md` — 设计(必填)

**写在什么时候**:该 unit 实施**之前**,在自己的 unit 分支上。

**回答的问题**:怎么实现?有哪些备选?选哪个?

**内容**:
- 选定方案的关键技术决策(数据结构 / 调用流程 / 接口形状)
- 备选方案 + 否决理由(没思考过的不要硬凑)
- 跟其他 unit / 模块的接口
- 风险点(已知会踩坑的地方)
- 测试策略

简单 unit 一段话也算 design.md;复杂 unit 才需要展开。

### 3.4 `review.html` — Code review 报告(必填)

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

## 5. 完整工作流

dev-log 体系跟 git 工作流深度耦合,分三层:**milestone 立项 / 单 unit 实施 / milestone 收尾**。

### 5.1 Milestone 立项(每个 milestone 一次)

```
1. 人:在 roadmap 锁定 milestone 方向
2. agent:git checkout -b docs/<vX.Y>-spec origin/main
3. agent:批量产出
   - dev-log/<vX.Y>/README.md(索引 / 依赖图 / 退出标准映射)
   - 每个 unit 的 issue.md
   - 复杂 unit 的 requirement.md(只在你点名时)
4. agent:commit + push + 提 PR
   commit:docs(project): introduce <vX.Y> dev-log
5. 人:在 PR review issue.md → 调整范围 / 验收 / 依赖 → approve → merge
```

### 5.2 单 unit 实施(每个 unit 一次)

每个 unit 走五个 step,严格对应一个分支、一个 PR、squash merge。

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 切分支                                                    │
│   git checkout -b <type>/<name> origin/main                      │
│   - type 来自 issue.md 顶部 Type 字段                              │
│   - name 取 unit 目录名去掉序号(03-llm-adapter → llm-adapter)   │
└─────────────────────────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: 设计                                                      │
│   agent:写 design.md(方案 + 备选 + 风险)                        │
│   commit:docs(<scope>): design <vX.Y>/<NN>-<name>                │
│   push 提 draft PR                                                │
│   人:review design.md(在 PR diff 看)                            │
│        ↓ approve         ↓ 不 approve(回 agent 改)              │
└─────────────────────────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: 实施                                                      │
│   agent:按 design.md 写代码,多次 commit                          │
│   commit 形态:feat/test/fix(<scope>): <subject>                   │
│   push                                                           │
└─────────────────────────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: 自检 + 生成 review.html                                   │
│   agent:跑测试 + ruff + 依赖规则自检                              │
│   agent:跑 /review skill → 生成 review.html                      │
│   commit:docs(<scope>): review <vX.Y>/<NN>-<name>                │
│   push,把 PR 从 draft 转 ready                                   │
└─────────────────────────────────────────────────────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Merge                                                    │
│   人:看 review.html                                              │
│        ↓ 没 blocker        ↓ 有 blocker(回 Step 3 修)            │
│   GitHub Squash Merge → main                                     │
└─────────────────────────────────────────────────────────────────┘
```

**关键性质**:
- 一个 unit = 一个分支 = 一个 PR(包含 design / impl / review 三类 commit)
- design.md 通过后才动代码,避免代码白写
- review.html 是 merge gate,不通过不合

### 5.3 Milestone 收尾(每个 milestone 一次)

```
1. 人:确认所有 unit 已 merge
2. agent:git checkout -b chore/<vX.Y>-cleanup origin/main
3. agent:
   - 扫所有 review.html,跨 unit / 长期影响的 finding 提炼到
     docs/architecture/design-log.md
   - 更新 roadmap.md(勾选退出标准、🚧 → ✅)
4. agent:commit + push + 提 PR
   commit:docs(project): close <vX.Y> milestone
5. 人:review → merge
```

---

## 6. 命名约定

### 6.1 dev-log 内部

| 项 | 规则 | 示例 |
|---|---|---|
| 分组目录 | 当前按 milestone:`vX.Y/` | `v0.1/`, `v0.2.5/` |
| unit 目录 | `NN-kebab-name`(NN = 两位序号) | `03-llm-adapter/` |
| 必填产物 | 固定文件名 | `issue.md` / `design.md` / `review.html` |
| 可选产物 | 固定文件名 | `requirement.md` |
| 跨 unit 引用 | 相对路径 | `../03-llm-adapter/issue.md` |

unit 序号反映依赖顺序(low → high),不必等于实施顺序。

### 6.2 git 分支

| 场景 | 命名 | 示例 |
|---|---|---|
| Milestone 立项 | `docs/<vX.Y>-spec` | `docs/v0.1-spec` |
| 单 unit 实施 | `<type>/<name>` | `feat/llm-adapter` |
| Milestone 收尾 | `chore/<vX.Y>-cleanup` | `chore/v0.1-cleanup` |

`<type>` 来自该 unit 的 issue.md;`<name>` 是 unit 目录名去掉两位序号。

### 6.3 commit message

| 场景 | 格式 | 示例 |
|---|---|---|
| Milestone 立项 | `docs(project): introduce <vX.Y> dev-log` | `docs(project): introduce v0.1 dev-log` |
| 单 unit:design | `docs(<scope>): design <vX.Y>/<NN>-<name>` | `docs(llm): design v0.1/03-llm-adapter` |
| 单 unit:impl | 按 git-workflow skill,正常 type/scope | `feat(llm): add openai-compat adapter` |
| 单 unit:review | `docs(<scope>): review <vX.Y>/<NN>-<name>` | `docs(llm): review v0.1/03-llm-adapter` |
| Milestone 收尾 | `docs(project): close <vX.Y> milestone` | `docs(project): close v0.1 milestone` |

---

## 7. 不做什么

- 不在 `dev-log/` 写"使用文档" — 那是 `docs/` 的事
- 不把 `dev-log/` 当 todo list — 状态走 GitHub Issues / 本地 task 系统,`dev-log/` 只放冻结的产物
- 不在 design.md 凑字数 — 简单 unit 一段话即可
- 不在每个 unit 都写 requirement.md — 默认不写,只在你明确点名时写
- 不重复 `docs/` 的内容 — 引用就行(`参考 [Tools 模块](../../docs/modules/tools/README.md)`)
- 不跳过 design 阶段直接写代码 — 通过 design review 才动代码

---

## 8. 相关文档

- [roadmap.md](../docs/roadmap.md) — 项目方向规划
- [docs/architecture/design-log.md](../docs/architecture/design-log.md) — 项目级决策日志
- [CONTRIBUTING.md](../CONTRIBUTING.md) — git workflow
- [.claude/skills/git-workflow/SKILL.md](../.claude/skills/git-workflow/SKILL.md) — git 操作规范(branch / commit / PR)
- [CLAUDE.md](../CLAUDE.md) — 项目规则总纲
