# 开发流程

所有代码变更必须遵循以下流程，禁止直接在 main 上提交。

## 流程步骤

```
1. 需求分析 → 2. 建分支 → 3. 开发 → 4. Commit → 5. Review → 6. 提 PR
```

| 步骤 | 动作 | 调度到 |
|------|------|--------|
| 1. 需求分析 | 理解需求，拆分任务 | — |
| 2. 建分支 | `git fetch origin` → 从 `origin/main` checkout 新分支 | `git-workflow` skill (branch) |
| 3. 开发 | 编码、测试 | 各研发 skill/agent |
| 4. Commit | 暂存 + 提交 | `git-workflow` skill (commit) |
| 5. Review | 自查代码质量和架构约束 | `review` skill |
| 6. 提 PR | 一致性审查 → squash → push → 创建 PR | `git-workflow` skill (pr) |

## 分支命名

格式：`<type>/<short-desc>`

- type 与 commit type 一致（feat, fix, refactor, chore 等）
- short-desc 用英文短横线连接，简明描述意图

示例：
- `feat/voice-streaming`
- `fix/pipeline-duplicate-events`
- `refactor/extract-port-base`
- `chore/upgrade-openai-sdk`

## 规则

1. **禁止在 main 上直接 commit**——必须先建分支
2. **一个分支一个目的**——不要在同一分支混合不相关变更
3. **Commit 前检查架构约束**——由 git-workflow skill 自动执行
4. **PR 前 Review**——确保代码质量，由 review skill 执行
5. **不自动 push**——push 和提 PR 需用户明确指示
