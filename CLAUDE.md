# Sigma

通用 personal AI assistant — Chat + Task + Realtime 三种交互模式 / 基于 LangGraph 的 agent 内核 / Tool · Skill · Agent 三层正交扩展。

## 硬性规则

### Git 工作流

- **禁止**在 main 上直接 commit — 所有变更必须先建分支，走 PR 合入
- 分支命名：`<type>/<short-desc>`（type: feat/fix/refactor/chore/perf/test/docs）
- 流程：需求分析 → 建分支 → 开发 → Commit → Review → 提 PR
- 严格遵守 [skills/git-workflow/SKILL.md](skills/git-workflow/SKILL.md)

### 依赖规则

- **`core/`** 只 import 标准库，禁止任何外部依赖
- **`ports/`** 只 import `core/`，不依赖任何具体实现
- **域目录**（`llm/`, `tools/`, `trace/`, `checkpoint/`）只 import `ports/` + `core/` + 外部 SDK，**互相禁止 import**
- **内核模块**（`agent/`, `task/`, `context/`, `rag/`, `memory/`, `skill/`, `chat/`）可互相 import；调域实现走 registry
- **`server/`** 调内核模块；**`app/`** 只走 `server/` 暴露的 HTTP/SSE API

### DO / DON'T

**DO**：
- 真有 ≥2 个实现需求才做 Port——不要为"未来可能"抽象
- 流式输出用 `AsyncIterator[AgentChunk]`
- 一个 adapter 一个文件，注册到 registry
- Sub-agent 卡住时 raise `BlockedException`，不要 raise 通用异常

**DON'T**：
- 不要在 Port 接口传 callback、文件句柄、Python 特有对象
- 不要让域目录互相 import
- 不要在 `core/` 放业务逻辑或外部依赖
- 不要让 `app/` 直接调内核模块（必须走 `server/`）
- 不要把"复杂任务的 skill"写成 skill——skill 是 prompt，复杂任务该写成 agent

## Agent 工作流

每个工作单元(对应一个独立 PR)在 `dev-log/<group>/<unit>/` 留下完整轨迹。三层流程:

**Milestone 立项**(每个 milestone 一次):
- 分支 `docs/<vX.Y>-spec`
- 批量产出 `dev-log/<vX.Y>/README.md` + 各 unit 的 `issue.md`(复杂 unit 可加 `requirement.md`)
- commit `docs(project): introduce <vX.Y> dev-log`,人 review → merge

**单 unit 实施**(每个 unit 一个分支 / 一个 PR):

| Step | 动作 | 产物 / commit |
|------|------|--------------|
| 1 | `git checkout -b <type>/<name> origin/main`(name 取 unit 目录名去序号) | — |
| 2 | agent 写 design.md → push draft PR | `docs(<scope>): design <vX.Y>/<NN>-<name>` |
| 3 | 人 review design → approve → agent 实施 | `feat/test/fix(<scope>): <subject>` ×N |
| 4 | agent 跑 `/review` skill → 生成 review.html → push 转 ready | `docs(<scope>): review <vX.Y>/<NN>-<name>` |
| 5 | 人 看 review.html → 没 blocker → squash merge | — |

**Milestone 收尾**(每个 milestone 一次):
- 分支 `chore/<vX.Y>-cleanup`
- 跨 unit / 长期影响的 finding 从 review.html 提炼到 [`docs/architecture/design-log.md`](docs/architecture/design-log.md)
- 更新 roadmap 退出标准 → commit `docs(project): close <vX.Y> milestone`

完整规则见 [`dev-log/README.md`](dev-log/README.md);git 操作细节见 [git-workflow skill](.claude/skills/git-workflow/SKILL.md)。**新开 unit 之前必读 dev-log/README.md。**

## 参考文档

需要了解项目细节时，按需读取：

| 场景 | 文档 |
|------|------|
| 文档总导航 | [docs/index.md](docs/index.md) |
| 架构 / 项目结构 | [docs/architecture/overview.md](docs/architecture/overview.md) |
| 依赖规则详解 | [docs/architecture/dependency-rules.md](docs/architecture/dependency-rules.md) |
| 添加扩展（Tool/Skill/Agent/LLM） | [docs/guides/adding-an-adapter.md](docs/guides/adding-an-adapter.md) |
| 命名 / 代码规范 | [docs/guides/code-style.md](docs/guides/code-style.md) |
| 开发日志体系 | [dev-log/README.md](dev-log/README.md) |
| 当前 milestone 拆分 | [dev-log/v0.1/README.md](dev-log/v0.1/README.md) |
