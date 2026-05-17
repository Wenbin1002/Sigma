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

## 3. Unit 的来源

dev-log 的 workflow 从"已经有一个 issue.md"开始。issue 怎么来,workflow 不关心:

- 你自己写 / 改 issue.md
- agent 帮你拆某个方向(基于 roadmap / 一段需求描述)
- 从 GitHub Issues 同步过来

unit 怎么分组也跟 workflow 解耦。当前用 `vX.Y/` 是因为早期开发跟 milestone 高度对应。一组 unit 完成后是否做"运营动作"(更新 roadmap / 提炼 design-log)是项目运营的事,不属于本 workflow。

---

## 4. 三份产物

### 4.1 `issue.md` — 立项

**回答的问题**:这个 unit 是什么?边界在哪?要做到什么程度?

**内容**:
- 背景 / 用户故事(谁、在什么情境下、想完成什么)
- 范围(具体要交付的代码 / 配置 / 文档)
- 必须能力(功能性需求,可验证的事实)
- 非功能性需求(性能 / 安全 / 兼容性,没有就略)
- 显式取舍 / 不做(为什么不做 X)
- 设计要点(关键约束 / 反模式;**不展开方案**)
- 验收标准(可执行的 checkbox)

`issue.md` 是 what + why,不是 how。how 在 `design.md`。

### 4.2 `design.md` — 设计

**写在什么时候**:该 unit 实施**之前**,在自己的 unit 分支上。

**回答的问题**:怎么实现?有哪些备选?选哪个?

**内容**:
- 选定方案的关键技术决策(数据结构 / 调用流程 / 接口形状)
- 备选方案 + 否决理由(没思考过的不要硬凑)
- 跟其他 unit / 模块的接口
- 风险点(已知会踩坑的地方)
- 测试策略

简单 unit 一段话也算 design.md;复杂 unit 才需要展开。

### 4.3 `review.html` — Code review 报告

**写在什么时候**:该 unit 的 PR 实施完成、merge **之前**,由 [`/review` skill](../skills/review/) 生成。

**回答的问题**:这次实施的代码质量如何?能不能合?

**内容**:
- 改动概览(文件 / 行数 / diff 摘要)
- 发现按严重度分类(blocker / major / minor / nit)
- 架构 / 依赖规则自检结果
- 测试覆盖评估
- 衍生 issue 链接(需要后续 unit 处理的)

**形态**:HTML 自包含报告。**作为 merge 决策依据**——人看完 review.html,确认没 blocker 才合 PR。

---

## 5. Workflow(单 unit 五步)

每个 unit = 一个分支 = 一个 PR,squash merge。

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
│   commit:docs(<scope>): design <unit-path>                       │
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
│   commit:docs(<scope>): review <unit-path>                       │
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
- design.md 通过后才动代码,避免代码白写
- review.html 是 merge gate,不通过不合
- impl 阶段 commit 数量不限,squash merge 时合并

---

## 6. 命名约定

### 6.1 dev-log 内部

| 项 | 规则 | 示例 |
|---|---|---|
| 分组目录 | 跟当前组织方式走(目前是 `vX.Y/`) | `v0.1/`, `v0.2.5/` |
| unit 目录 | `NN-kebab-name`(NN = 两位序号) | `03-llm-adapter/` |
| 三份产物 | 固定文件名 | `issue.md` / `design.md` / `review.html` |
| 跨 unit 引用 | 相对路径 | `../03-llm-adapter/issue.md` |

unit 序号反映依赖顺序(low → high),不必等于实施顺序。

### 6.2 git 分支

| 项 | 规则 | 示例 |
|---|---|---|
| 单 unit 实施 | `<type>/<name>` | `feat/llm-adapter` |

`<type>` 来自该 unit 的 issue.md;`<name>` 是 unit 目录名去掉两位序号。

### 6.3 commit message

| 阶段 | 格式 | 示例 |
|---|---|---|
| design | `docs(<scope>): design <unit-path>` | `docs(llm): design v0.1/03-llm-adapter` |
| impl | 按 [git-workflow skill](../.claude/skills/git-workflow/SKILL.md) | `feat(llm): add openai-compat adapter` |
| review | `docs(<scope>): review <unit-path>` | `docs(llm): review v0.1/03-llm-adapter` |

---

## 7. 跟 `docs/architecture/design-log.md` 的边界

| | `design-log.md` | `dev-log/<group>/<unit>/design.md` |
|---|---|---|
| 粒度 | 项目级、跨 unit | unit 级、单次实施 |
| 影响范围 | 长期 / 多模块 | 局部 / 一次 PR |
| 例子 | "为什么 Port 数量从 8 个砍到 4 个" | "为什么 OpenAI adapter 选 httpx 而不是 openai SDK" |

读 `review.html` 时如果发现某条 finding 触及"影响后续多个 unit / 改了项目方向"的决策,把它提炼一句到 `design-log.md`,留链接回到 `review.html`。

---

## 8. 不做什么

- 不在 `dev-log/` 写"使用文档" — 那是 `docs/` 的事
- 不把 `dev-log/` 当 todo list — 状态走 GitHub Issues / 本地 task 系统,`dev-log/` 只放冻结的产物
- 不在 design.md 凑字数 — 简单 unit 一段话即可
- 不重复 `docs/` 的内容 — 引用就行(`参考 [Tools 模块](../../docs/modules/tools/README.md)`)
- 不跳过 design 阶段直接写代码 — 通过 design review 才动代码

---

## 9. 相关文档

- [docs/architecture/design-log.md](../docs/architecture/design-log.md) — 项目级决策日志
- [.claude/skills/git-workflow/SKILL.md](../.claude/skills/git-workflow/SKILL.md) — git 操作规范(branch / commit / PR)
- [CLAUDE.md](../CLAUDE.md) — 项目规则总纲
