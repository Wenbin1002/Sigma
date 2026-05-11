---
name: git-workflow
description: |
  Unified git workflow skill covering branch creation, commits, and pull requests. Triggers on: "建分支", "checkout", "new branch", "commit", "提交", "save", "done", "提 PR", "submit PR", "create PR". Dispatches internally based on user intent. Enforces branch naming, commit message format, architecture checks, and PR coherence review.
---

# Git Workflow Skill

统一的 git 工作流技能，覆盖从建分支到提 PR 的完整流程。

## Design Principles

This skill exists to keep the git history clean, reviewable, and easy to navigate months later. A few ground rules flow from that goal:

- **All git text in English** — commit messages, PR titles, and PR bodies are all in English. The git log and GitHub history are shared artifacts; English keeps them consistent and searchable.
- **No Co-Authored-By or Signed-off-by** — this project doesn't use DCO or co-author tracking; adding them creates noise in the log.
- **No AI attribution in PR body** — don't add "Generated with Claude Code" or similar footers; the PR should look like any human-written PR.
- **Scopes come from the reference table only** — inventing ad-hoc scopes (like `architecture` or `docs` as a scope) fragments the log and makes filtering unreliable.
- **Always fetch before branching** — the local `main` can be days behind `origin/main`; branching from a stale base guarantees merge conflicts later.

---

## Shared Conventions

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

Scope tells you *where* the change lives. Determine it by priority:

1. **Changed files under `src/xxx/`** → scope is that directory name (`voice`, `agent`, `core`, `runtime`, etc.)
2. **Not in `src/` but has a clear category** → `deps` (dependency files) or `config` (runtime config like config.yaml)
3. **Nothing above matches** → `project` (catch-all for project infrastructure: docs, skills, CI, scripts, etc.)
4. **Multiple directories** → comma-join the main scopes, or pick the one with the biggest impact

#### Reference Table

| Scope | Typical paths |
|-------|--------------|
| `voice` | `src/voice/` |
| `agent` | `src/agent/` |
| `rag` | `src/rag/` |
| `memory` | `src/memory/` |
| `tools` | `src/tools/` |
| `trace` | `src/trace/` |
| `core` | `src/core/` |
| `ports` | `src/ports/` |
| `runtime` | `src/runtime/` |
| `context` | `src/context/` |
| `app` | `src/app/` |
| `deps` | `pyproject.toml`, `requirements.txt` |
| `config` | `config.yaml` |
| `project` | Everything else (README, CLAUDE.md, skills/, CI, scripts, etc.) |

**Note**: `docs` is a type, not a scope. When writing documentation, the scope points to the module it describes (e.g., `docs(voice)`). For global docs, use `docs(project)`.

### Commit Message Format

```
<type>(<scope>): <subject>

[body]

[footer]
```

- `type` and `scope` are required
- subject: imperative mood, lowercase start, no period, under 50 chars (hard limit 72)
- body (optional): explain *why*, wrap at 72 chars
- footer (optional): `Closes #123`, `BREAKING CHANGE:`

**Example 1:**
Input: Added a new Whisper-based STT implementation under src/voice/
Output: `feat(voice): add whisper STT implementation`

**Example 2:**
Input: Fixed a bug where the runtime dispatched duplicate events
Output: `fix(runtime): prevent duplicate event dispatch in pipeline`

**Example 3:**
Input: Updated pyproject.toml to bump openai SDK version
Output: `chore(deps): upgrade openai sdk to 1.12.0`

### Architecture Checks

Before each commit, verify the change respects the project's layered architecture. This catches coupling issues early — see `references/architecture-checks.md` for the full checklist and grep commands.

If a violation is found, warn the user and explain the concern, but don't block the commit — let them decide.

---

## Branch — Creating a Branch

Triggers: "建分支", "new branch", "checkout -b", or when starting a new task

### Why fetch first?

Your local `main` might be commits behind `origin/main`. Branching from a stale base means you'll inherit outdated code and face unnecessary conflicts at PR time. Always start from the freshest `origin/main`.

### Flow

1. **`git fetch origin`** — sync remote state
2. Determine `type` and `short-desc` from the task at hand
3. **`git checkout -b <type>/<short-desc> origin/main`** — branch from remote main

### Naming

Format: `<type>/<short-desc>`

- type matches commit types above
- short-desc uses English, kebab-case, concisely describes the intent

Examples:
- `feat/voice-streaming`
- `fix/pipeline-duplicate-events`
- `refactor/extract-port-base`
- `chore/upgrade-openai-sdk`

### Guidelines

- One branch, one purpose — keeps PRs focused and reviewable
- Confirm the branch name with the user before creating it

---

## Commit — Committing Code

Triggers: "commit", "提交", "save", "done", "finished"

### Why check the branch first?

Committing directly to `main` bypasses the PR review process and can't be easily undone once pushed. Catching this early saves headaches.

### Flow

1. **Check current branch** — `git branch --show-current`
   - On `main`/`master`: stop and suggest creating a feature branch first (explain why: direct commits to main bypass review)
   - On a feature branch: proceed
2. **`git status`** — see what changed
3. **`git diff --staged`** (+ `git diff`) — understand the changes
4. **`git log --oneline -5`** — confirm style consistency
5. **Analyze the changes**:
   - Which directory? → determines scope
   - One logical change or several? If multiple unrelated changes, suggest splitting (a commit should tell one story)
   - Check architecture constraints (see `references/architecture-checks.md`)
6. **Determine scope** — walk through the priority rules:
   - List all changed file paths
   - Match against: `src/xxx/` → directory name → `deps`/`config` → `project`
   - Only use scopes from the reference table
7. **Generate commit message** — follow the shared format
8. **Ask user to confirm** the message
9. **Execute the commit**
10. **Don't push** — pushing is a separate, conscious decision (happens at PR time)

### Splitting Commits

If the diff contains unrelated changes, suggest splitting — a clean history makes `git bisect`, `git log`, and code review dramatically easier:

```
The current diff contains multiple unrelated changes:
1. New voice implementation (feat)
2. Pipeline bug fix (fix)

Splitting these into separate commits keeps the history navigable.
Want me to help stage them separately?
```

---

## PR — Creating a Pull Request

Triggers: "提 PR", "submit PR", "create PR", "push and PR"

### Flow

#### 1. Pre-check

```bash
git fetch origin
git log origin/main..HEAD --oneline
```

Confirm the branch has commits ready to submit.

#### 2. Coherence Review

Review all commits to check whether the branch serves **one purpose**:

- **Coherent**: all commits orbit the same goal → proceed to step 3
- **Incoherent**: commits address unrelated concerns → suggest splitting

How to judge:
- Same or strongly related scopes (e.g., `ports` + `runtime` for one feature) → coherent
- Different types serving one goal (e.g., `feat` + `test` + `docs` for one feature) → coherent
- Unrelated scope/purpose combos → incoherent

**When incoherent, suggest splitting** (but it's a suggestion, not a mandate — the user decides):

```
This branch contains multiple unrelated changes:

1. feat(voice): add streaming support — commits: a1b2c3, d4e5f6
2. fix(runtime): fix duplicate events — commits: g7h8i9

Splitting into separate PRs makes each one easier to review and revert if needed:
- feat/voice-streaming
- fix/pipeline-duplicate-events

Want me to help split them via cherry-pick?
```

#### 3. Rebase on latest main

```bash
git rebase origin/main
```

Keep the branch up-to-date with main. Don't squash here — GitHub's Squash Merge handles that at merge time.

If rebase produces conflicts, help the user resolve them.

#### 4. Push

```bash
git push -u origin <branch-name>
```

Standard push. If a prior rebase requires force-pushing, explain the risk and use `--force-with-lease` (safer than `--force` because it won't overwrite someone else's push).

#### 5. Generate PR title + body

- **PR title** = the commit message that will survive after squash merge (follows the shared `<type>(<scope>): <subject>` format)
- Analyze the full diff across all branch commits to generate an accurate title
- **Ask user to confirm the PR title**

PR body format (this becomes the squash merge commit body, so keep it concise and meaningful):

```markdown
## Summary
<Distilled from all commits, keep it concise>
```

#### 6. Create the PR

```bash
gh pr create --title "<PR title>" --body "<body>"
```

#### 7. Done

Output the PR URL. Don't auto-merge — that's the user's call.

### PR Guidelines

- Always run the coherence review, even for single-commit branches — it's a quick sanity check
- The PR title becomes the final commit message after squash merge, so it matters
- Splitting is a recommendation, not a command — if the user wants to proceed as-is, respect that but note the trade-off (harder to review, harder to revert partially)
