# Issue 01 — 项目骨架 + 工具链

**Milestone**：v0.1
**Type**：`chore(project)`
**依赖**：无（这是起点）
**预计 PR 大小**：S

---

## 背景

`docs/roadmap.md` § 0.1 第一项：搭好 [`架构总览 § 9`](../../../docs/architecture/overview.md#9-项目结构) 描述的项目结构。当前 repo 没有 `src/`，所有后续 issue 都依赖这一步定下骨架和工具链。

## 范围

### 1. `src/` 目录骨架（只建空目录 + `__init__.py`）

按 [架构总览 § 9](../../../docs/architecture/overview.md#9-项目结构) 创建：

```
src/
  core/                  # 数据结构 / 事件（issue 02 填充）
  ports/                 # Protocol 定义（issue 02 填充）
  llm/                   # issue 03
  tools/                 # issue 04
  trace/                 # issue 05
  checkpoint/            # issue 06
  agent/                 # issue 07
  chat/                  # issue 08
  server/                # issue 09
  app/                   # issue 10
tests/
config.example.yaml      # 示例配置（含 OpenAI 兼容 base_url 字段）
```

不在 0.1 范围的目录（`task/`, `skill/`, `context/`, `rag/`, `memory/`, `realtime/`, `improvement/`）**本 issue 不创建**。等到对应 milestone 再加，避免空目录污染。

### 2. `pyproject.toml`

- Python ≥ 3.11
- 项目名：`sigma`
- console_scripts：`sigma = sigma.app.cli:main`（实际 entry 在 issue 10 实现）
- 必需依赖（仅声明 0.1 用到的，不预先装多余的）：
  - `langgraph`
  - `httpx`（OpenAI 兼容中转 / SSE client）
  - `typer`（CLI）
  - `fastapi` + `uvicorn`（server）
  - `pydantic` 仅用于配置加载（schemas 用 dataclass，详见 issue 02）
  - `pyyaml`
- dev 依赖：`ruff`、`pytest`、`pytest-asyncio`、`anyio`

### 3. 工具链

- **Ruff**：format + lint，行宽 88（[code-style.md](../../../docs/guides/code-style.md)）
- **pre-commit hook**：保留并完善已有的 `.githooks/pre-commit`，新增对 `src/` 的依赖规则自检（grep 命令直接抄 [dependency-rules.md § 5](../../../docs/architecture/dependency-rules.md#5-自检方法)）
- **`.gitignore`**：补 `~/.sigma/` 不会出现在 repo 内，但要 ignore `dist/` `*.egg-info/` `.pytest_cache/` `.ruff_cache/` `__pycache__/`

### 4. `config.example.yaml`

最小可跑配置，覆盖 0.1 用到的所有键：

```yaml
llm:
  default:
    provider: openai_compat        # OpenAI 兼容中转
    model: gpt-4o-mini             # 占位
    base_url: https://YOUR_RELAY/v1
    api_key_env: SIGMA_LLM_API_KEY

tools:
  builtin:
    - read_file
    - write_file
    - edit_file
    - list_dir
    - shell
    - grep
    - glob
    - web_search
    - git
  shell:
    cwd_lock: true
    timeout_seconds: 60
    deny: ["rm -rf /", "sudo", "curl | sh"]

trace:
  provider: jsonl
  output_dir: ~/.sigma/traces

checkpoint:
  provider: sqlite
  db_path: ~/.sigma/checkpoint.db

server:
  host: 127.0.0.1
  port: 7777
```

## 设计要点

- **不引入未用到的依赖**——后续 milestone 再加（vector store / cron / MCP client...）
- **不创建 0.1 不会用到的空目录**——避免"刚搭骨架就有死代码"
- **`pyproject.toml` 用 `setuptools` + `src/` layout**——`packages = ["sigma"]`，所有 import 路径以 `sigma.*` 开头（避免 `from llm.registry` 这种顶层撞库名风险）。需要同步更新 [code-style.md § Import](../../../docs/guides/code-style.md#import-规范) 的示例。
- **CLAUDE.md 项目结构表格**保持不变——0.1 不引入新模块。

## 验收标准

- [ ] `pip install -e ".[dev]"` 一次成功
- [ ] `ruff format src/ tests/` `ruff check src/ tests/` 干净通过
- [ ] `pytest` 跑通（即使 0 条用例，也不能报错）
- [ ] `python -c "import sigma"` 不报错
- [ ] `.githooks/pre-commit` 在违反依赖规则时报错（手动构造 `from llm.openai import x` 写在 `ports/llm.py` 里验证）
- [ ] `sigma --help` 能跑（实际 entry 是 stub，issue 10 替换）

## 不做

- 不写任何业务逻辑
- 不实现任何 Port / Adapter（issue 02+）
- 不接 CI（先本地保证，CI 等到有内容跑了再加）

## 相关文档

- [架构总览 § 9](../../../docs/architecture/overview.md#9-项目结构)
- [依赖规则](../../../docs/architecture/dependency-rules.md)
- [代码规范](../../../docs/guides/code-style.md)
