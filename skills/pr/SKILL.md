---
name: pr
description: |
  Prepare and submit a pull request. Reviews all commits on the current branch for coherence, squashes if unified, or suggests splitting if divergent. Use this skill when the user says "提 PR", "submit PR", "create PR", "push and PR", or when the dev workflow reaches step 6.
---

# PR Skill

Prepare a clean, single-purpose pull request from the current feature branch.

## Workflow

### 1. 预检

```bash
git fetch origin
git log origin/main..HEAD --oneline
```

列出当前分支相对于 origin/main 的所有 commits。

### 2. 一致性审查

审查所有 commits，判断是否满足 **一个分支一个目的** 原则：

- **满足**：所有 commits 围绕同一个目的 → 进入步骤 3（Squash）
- **不满足**：commits 包含多个不相关变更 → 进入步骤 4（拆分）

判断标准：
- scope 相同或强相关（如 `ports` + `runtime` 为同一功能联动）
- type 不同但服务同一目标（如 `feat` + `test` + `docs` 都是为同一功能）
- 如果出现完全不相关的 scope/目的组合（如修 bug + 加新功能 + 改配置），判为不一致

### 3. Squash & 生成 commit message

如果一致性通过：

```bash
git rebase -r origin/main
git reset --soft origin/main
git commit
```

重新生成一个干净的 commit message，遵循 `git-commit` skill 的格式规范：

- 分析 squash 后的完整 diff
- 生成 `type(scope): subject` 格式的 message
- body 中概括所有变更要点
- 提交前让用户确认 message

### 4. 拆分建议（不一致时）

如果 commits 不满足统一目的：

```
当前分支包含多个不相关变更：

1. feat(voice): add streaming support — commits: a1b2c3, d4e5f6
2. fix(runtime): fix duplicate events — commits: g7h8i9

建议拆分为两个分支分别提 PR：
- feat/voice-streaming
- fix/pipeline-duplicate-events

需要我帮你拆分吗？
```

拆分方式：
- 用 `git cherry-pick` 将相关 commits 分到各自新分支
- 每个新分支独立走一遍 squash + PR 流程

### 5. Push & 创建 PR

一致性确认 + squash 完成后：

```bash
git push -u origin <branch-name>
gh pr create --title "<commit subject>" --body "<body>"
```

PR body 格式：

```markdown
## Summary
<从 commit message body 提炼，3 bullet points 以内>

## Test plan
- [ ] <验证清单>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### 6. 完成

输出 PR URL，流程结束。

## 规则

1. **绝不跳过一致性审查**——即使只有一个 commit 也要确认它符合分支目的
2. **Squash 前必须让用户确认 commit message**——不要静默 squash
3. **拆分是建议不是强制**——用户可以选择不拆，但要明确告知风险
4. **Force push 前警告**——squash 后需要 `--force-with-lease` push，必须提示用户
5. **不自动 merge**——PR 创建后由用户决定何时 merge
