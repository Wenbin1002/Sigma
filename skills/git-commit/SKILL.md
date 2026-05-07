---
name: git-commit
description: |
  Generate well-structured git commit messages following Conventional Commits format, tailored to this project's domain-based architecture. Use this skill whenever the user asks to commit changes, write a commit message, stage and commit, or asks "commit this", "save my progress", "git commit". Also trigger when you see uncommitted changes and the user says something like "done", "finished", "that looks good, save it". This skill enforces project-specific scoping rules and architectural constraints to keep the commit history clean and navigable.
---

# Commit Skill

Generate clear, consistent commit messages for this voice AI assistant project.

## Commit Message Format

```
<type>(<scope>): <subject>

[body]

[footer]
```

- `type` and `scope` are both required, never omit them
- `body` and `footer` are optional

### Types

| Type | When to use |
|------|------------|
| `feat` | New functionality — a new implementation, pipeline feature, endpoint |
| `fix` | Bug fix — correcting wrong behavior |
| `enhance` | Improvement to existing functionality (performance, UX, quality) |
| `refactor` | Code restructuring that doesn't change external behavior |
| `test` | Adding or updating tests |
| `docs` | Documentation changes (CLAUDE.md, architecture docs, docstrings) |
| `chore` | Dependencies, config, tooling, CI — nothing user-facing |
| `style` | Formatting, linting — no logic changes |

### Scopes

#### 判断逻辑（按优先级）

1. **改了 `src/xxx/` 下的文件** → scope 就是那个目录名（`voice`, `agent`, `core`, `runtime` 等）
2. **改的不在 `src/` 里，但属于特定类别** → 按性质选：`deps`（依赖文件）、`config`（运行时配置文件 config.yaml）
3. **以上都不匹配** → `project`（兜底，覆盖所有项目级基础设施：文档、skills、CI、脚手架等）
4. **跨多个目录** → 逗号连接主要 scope，或选影响最大的那个

#### 参考表

以下仅为常见映射，不是穷举。遵循上面的判断逻辑即可。

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

**Scope is mandatory** — every commit must have one.

### Subject Line Rules

1. Use imperative mood: "add", "fix", "implement" — not "added", "fixes", "implementing"
2. Lowercase first letter (no capitalization after the colon)
3. No period at the end
4. Keep under 50 characters (hard limit: 72)
5. Describe the "what", not the "how"

Good: `feat(agent): implement stream/resume lifecycle`
Bad: `feat(agent): Implemented the stream and resume methods for the agent lifecycle.`

### Body (optional but encouraged for complex changes)

Use the body to explain:
- **Why** this change was made (motivation, context)
- **What** trade-offs were considered
- **How** it relates to the architecture (especially for new domain implementations)

Wrap at 72 characters per line.

### Footer (optional)

- `Closes #123` — link to issues
- `BREAKING CHANGE: <description>` — for incompatible changes
- `Co-Authored-By: Name <email>` — for pair work

## Workflow

When the user wants to commit:

1. **Check current branch** — run `git branch --show-current`
   - If on `main` (or `master`): **STOP**.不允许直接在主分支上 commit。提示用户需要先创建新分支，询问分支名称，然后 `git checkout -b <branch>` 后再继续
   - If already on feature branch: continue
2. **Run `git status`** to see what's changed (staged and unstaged)
3. **Run `git diff --staged`** (and `git diff` if relevant) to understand the actual changes
4. **Run `git log --oneline -5`** to see recent commit style for consistency
5. **Analyze the changes** against these questions:
   - Which directory is affected? → that's your scope
   - Is this one logical change or multiple? (If multiple, suggest splitting)
   - Does this change respect architectural constraints? (see below)
6. **Draft the commit message** following the format above
7. **Present to the user** for confirmation or tweaks
8. **Execute the commit**
9. **Do NOT push** — never run `git push` automatically; let the user push manually when ready

## Architectural Constraint Checks

Before committing, verify the changes don't violate the project's architecture:

| Constraint | What to check | Warning |
|-----------|---------------|---------|
| `core/` independence | No imports of external AI frameworks in core/ | "This commit adds an external dependency to core/ — core should have zero external deps. Should this logic move to a domain directory?" |
| No reverse deps | runtime/ must not import domain directories (voice/agent/rag/...) directly | "runtime/ is importing from a domain directory — it should only import from ports/. Consider refactoring." |
| Domain isolation | Domain directories (voice/agent/rag/...) should only import ports/ + vendor SDKs, not each other | "voice/ is importing from memory/ — domains should not reference each other. Use ports/ as the contract." |
| Registry pattern | New implementations should be registered in registry.py | "New implementation detected but not added to registry.py — don't forget to register it." |

If a violation is detected, warn the user before committing. Don't block the commit — just surface the issue clearly so they can decide.

## Splitting Commits

If the diff contains changes that span multiple unrelated concerns, suggest splitting:

```
I notice this diff includes:
1. A new voice implementation (feat)
2. A bug fix in an existing port (fix)
3. Updated docs (docs)

Want me to help split these into separate commits? That keeps the history easier to navigate.
```

Use `git add -p` or file-level staging to help split.

## Examples

### Simple feature
```
feat(voice): add Whisper STT with streaming support
```

### Bug fix with context
```
fix(runtime): prevent duplicate events in voice pipeline

The voice pipeline was emitting both "text_delta" and "done" chunks
when the stream completed normally, causing the UI to render a
spurious empty message. Now "done" is only emitted once at the end.
```

### Refactor with architectural note
```
refactor(core,ports): extract common async iterator pattern

All ports that return streaming results were implementing the same
async generator boilerplate. Extracted into a shared base in
core/streaming.py to reduce duplication across STT, TTS, and Agent
ports.
```

### Dependency update
```
chore(deps): upgrade openai SDK to 1.40.0

Needed for structured outputs support in the agent domain.
```

### Multi-scope (rare, use sparingly)
```
feat(ports,runtime): add task pause/resume support

AgentRuntimePort now exposes a `pause()` method, and TaskManager
uses it to implement graceful task suspension.

Closes #42
```
