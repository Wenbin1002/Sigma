---
name: git-workflow
description: |
  Unified git workflow skill covering branch creation, commits, and pull requests. Triggers on: "建分支", "checkout", "new branch", "commit", "提交", "save", "done", "提 PR", "submit PR", "create PR". Dispatches internally based on user intent. Enforces branch naming, commit message format, architecture checks, and PR coherence review.
---

# Git Workflow Skill

统一的 git 工作流技能，覆盖从建分支到提 PR 的完整流程。

## 共享规则

### Types

| Type | When to use |
|------|------------|
| `feat` | New functionality — a new implementation, pipeline feature, endpoint |
| `fix` | Bug fix — correcting wrong behavior |
| `refactor` | Code restructuring that doesn't change external behavior |
| `test` | Adding or updating tests |
| `docs` | Documentation changes (CLAUDE.md, architecture docs, docstrings) |
| `chore` | Dependencies, config, tooling, CI — nothing user-facing |
| `perf` | Performance improvement |
| `style` | Formatting, linting — no logic changes |

### Scopes

#### 判断逻辑（按优先级）

1. **改了 `src/xxx/` 下的文件** → scope 就是那个目录名（`voice`, `agent`, `core`, `runtime` 等）
2. **改的不在 `src/` 里，但属于特定类别** → 按性质选：`deps`（依赖文件）、`config`（运行时配置文件 config.yaml）
3. **以上都不匹配** → `project`（兜底，覆盖所有项目级基础设施：文档、skills、CI、脚手架等）
4. **跨多个目录** → 逗号连接主要 scope，或选影响最大的那个

#### 参考表

| Scope | 典型路径 |
|-------|---------|
| `voice` | `src/voice/` |
| `agent` | `src/agent/` |
| `rag` | `src/rag/` |
| `memory` | `src/memory/` |
| `tools` | `src/tools/` |
| `trace` | `src/trace/` |
| `core` | `src/core/` |
| `ports` | `src/ports/` |
| `runtime` | `src/runtime/` |
| `app` | `src/app/` |
| `deps` | `pyproject.toml`, `requirements.txt` |
| `config` | `config.yaml` |
| `project` | 其他一切（README, CLAUDE.md, skills/, CI, scripts 等） |

**Note**: `docs` 是 type 不是 scope。写文档时 scope 指向它描述的模块（如 `docs(voice)`），全局文档用 `docs(project)`。

### Commit Message 格式

```
<type>(<scope>): <subject>

[body]

[footer]
```

- `type` 和 `scope` 必填
- subject：祈使句、小写开头、无句号、50 字符内（硬限 72）
- body（可选）：解释 why，72 字符换行
- footer（可选）：`Closes #123`、`BREAKING CHANGE:`

### 架构约束检查

每次 commit 前自动验证：

| 约束 | 检查内容 | 违规提示 |
|------|---------|---------|
| core 独立 | `core/` 无外部 AI 框架 import | "core/ 引入了外部依赖，应移到域目录" |
| 无反向依赖 | `runtime/` 不 import 域目录 | "runtime/ 应只 import ports/" |
| 域隔离 | 域目录不互相引用 | "域之间不应互相引用，通过 ports/ 通信" |
| Registry 注册 | 新实现需注册到 registry.py | "新实现未注册到 registry.py" |

违规时警告但不阻塞，由用户决定是否继续。

---

## Branch — 创建分支

触发词："建分支"、"new branch"、"checkout -b"、开始新需求时

### 流程

1. `git fetch origin`
2. 根据需求确定 type 和 short-desc
3. `git checkout -b <type>/<short-desc> origin/main`

### 命名规范

格式：`<type>/<short-desc>`

- type 与 commit type 一致
- short-desc 用英文短横线连接，简明描述意图

示例：
- `feat/voice-streaming`
- `fix/pipeline-duplicate-events`
- `refactor/extract-port-base`
- `chore/upgrade-openai-sdk`

### 规则

- 必须基于 `origin/main`（先 fetch）
- 一个分支一个目的
- 分支名需跟用户确认后再创建

---

## Commit — 提交代码

触发词："commit"、"提交"、"save"、"done"、"finished"

### 流程

1. **检查当前分支** — `git branch --show-current`
   - 在 `main`/`master` 上：**STOP**，提示建分支，不继续
   - 在 feature branch 上：继续
2. **`git status`** — 查看变更
3. **`git diff --staged`**（+ `git diff`）— 理解变更内容
4. **`git log --oneline -5`** — 确认风格一致
5. **分析变更**：
   - 哪个目录？→ scope
   - 一个逻辑变更还是多个？多个则建议拆分
   - 是否违反架构约束？
6. **生成 commit message** — 按共享格式
7. **让用户确认** message
8. **执行 commit**
9. **不 push** — 永远不自动 push

### 拆分 Commit

如果 diff 包含不相关变更：

```
当前 diff 包含多个不相关变更：
1. 新的 voice 实现 (feat)
2. pipeline bug 修复 (fix)

建议拆分为独立 commit，保持历史清晰。需要我帮拆分吗？
```

---

## PR — 提交 Pull Request

触发词："提 PR"、"submit PR"、"create PR"、"push and PR"

### 流程

#### 1. 预检

```bash
git fetch origin
git log origin/main..HEAD --oneline
```

确认当前分支有 commits 待提交。

#### 2. 一致性审查

审查所有 commits，判断是否满足 **一个分支一个目的**：

- **满足**：所有 commits 围绕同一个目的 → 步骤 3
- **不满足**：commits 包含多个不相关变更 → 建议拆分

判断标准：
- scope 相同或强相关（如 `ports` + `runtime` 为同一功能联动）
- type 不同但服务同一目标（如 `feat` + `test` + `docs` 为同一功能）
- 不相关的 scope/目的组合 → 不一致

**拆分建议（不一致时）**：

```
当前分支包含多个不相关变更：

1. feat(voice): add streaming support — commits: a1b2c3, d4e5f6
2. fix(runtime): fix duplicate events — commits: g7h8i9

建议拆分为两个分支分别提 PR：
- feat/voice-streaming
- fix/pipeline-duplicate-events

需要我帮你拆分吗？
```

拆分方式：cherry-pick 到各自新分支，每个独立提 PR。

#### 3. Rebase on latest main

```bash
git rebase origin/main
```

保持与 main 最新同步。不做 squash——合并时由 GitHub 的 Squash Merge 完成。

如果 rebase 有冲突，协助用户解决后继续。

#### 4. Push

```bash
git push -u origin <branch-name>
```

普通 push。如果之前 rebase 导致需要 force push，警告用户后使用 `--force-with-lease`。

#### 5. 生成 PR title + body

- **PR title** = 最终 squash merge 后的 commit message（符合共享格式：`<type>(<scope>): <subject>`）
- 分析分支所有 commits 的完整 diff，生成准确的 title
- **让用户确认 PR title**

PR body 格式：

```markdown
## Summary
<从所有 commits 提炼，3 bullet points 以内>

## Test plan
- [ ] <验证清单>
```

#### 6. 创建 PR

```bash
gh pr create --title "<PR title>" --body "<body>"
```

#### 7. 完成

输出 PR URL，流程结束。

### PR 规则

1. **绝不跳过一致性审查** — 即使只有一个 commit
2. **PR title 即最终 commit message** — 合并时由 GitHub squash merge 使用
3. **PR title 必须让用户确认**
4. **拆分是建议不是强制** — 用户可选择不拆，但明确告知风险
5. **不自动 merge** — PR 创建后由用户决定
