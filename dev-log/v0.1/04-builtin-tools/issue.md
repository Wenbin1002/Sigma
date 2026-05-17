# Issue 04 — 内置 tools（file / shell / web / git）

**Milestone**：v0.1
**Type**：`feat(tools)`
**依赖**：issue 02
**预计 PR 大小**：M

---

## 背景

roadmap § 0.1 列出的内置 tool：`read_file / write_file / shell / web_search / grep / glob / edit_file / git`。0.1 的退出标准之一就是"工具调用流跑通（agent 调 read_file 读文件回答）"。

参考：[Tools 模块](../../../docs/modules/tools/README.md)、[Ports & Adapters § 3.2](../../../docs/architecture/ports-and-adapters.md#32-toolport)。

## 范围

### 1. `src/sigma/tools/base.py`

- `ToolPort` Protocol 实现基类 `BaseTool`
- `@tool` 装饰器：把普通 async 函数包成 `BaseTool` 实例（自动从 type hints + docstring 生成 `parameters_schema` 和 `description`）

```python
@tool
async def read_file(path: str) -> str:
    """读取文件内容。

    Args:
        path: 文件绝对路径或相对当前工作目录的路径
    """
    ...
```

### 2. `src/sigma/tools/registry.py`

```python
REGISTRY: dict[str, BaseTool] = {}

def register(tool: BaseTool) -> None: ...
def get(name: str) -> BaseTool: ...
def enabled(config) -> list[BaseTool]: ...   # 根据 config.tools.builtin 过滤
```

启动时由 `sigma.tools.builtin.__init__` 自动 `register()` 所有内置 tool。

### 3. `src/sigma/tools/builtin/`

| tool | 文件 | 功能 |
|---|---|---|
| `read_file` | `file.py` | 读文件，参数：path、可选 offset/limit |
| `write_file` | `file.py` | 写文件（覆盖），参数：path、content |
| `edit_file` | `file.py` | 字符串替换式编辑（old_str → new_str；要求 old_str 唯一） |
| `list_dir` | `file.py` | 列目录 |
| `grep` | `search.py` | 调 ripgrep（缺失时 fallback 到 Python 实现），参数：pattern、path、可选 glob |
| `glob` | `search.py` | 文件名 glob 匹配 |
| `shell` | `shell.py` | 跑 shell 命令，受 sandbox 约束（见下） |
| `web_search` | `web.py` | 搜索网络（用 DuckDuckGo HTML 端点或 Brave API），返回 [title/url/snippet] 列表 |
| `git` | `git.py` | 跑 git 子命令（被 shell sandbox allowlist 兜底） |

### 4. Shell sandbox（最简版）

0.1 不上 docker / firejail，只做"配置驱动的护栏"：

- **CWD 锁**：执行命令的工作目录必须在配置的 `working_root` 之内，参数 path 越界拒绝
- **deny list**：匹配 `rm -rf /` / `sudo` / `curl | sh` 等的命令直接拒绝（regex 匹配，宽松一点：怀疑就拒绝）
- **timeout**：默认 60 秒（可配）
- **输出截断**：stdout/stderr 各最多 64 KB，超过截断并在 `metadata.truncated=true`
- 危险命令 / 越界 / 超时全部用 `ToolResult(success=False, error="...")` 返回，**不要 raise**

更复杂的沙箱（docker / firejail）是 V5+ ，本 issue 不做。

### 5. `web_search` 选择

为了 0.1 顺利跑通，**默认走免登录的 DuckDuckGo HTML 接口**（`https://duckduckgo.com/html/?q=...`）。失败时返回 `ToolResult(success=False, error=...)`，不阻塞 chat。

如果将来要换 Brave / Google / Bing，再加 provider 字段；0.1 先不做这层抽象。

### 6. `git` tool

直接 subprocess 调 `git`：参数是 `args: list[str]`。例如 `git(args=["status"])`。这样 LLM 完全控制语义，不用我们枚举所有子命令。

## 设计要点

- **一个 file/web/search/shell/git 一个文件**（不是一个 tool 一个文件——粒度过细）
- **每个 tool 都返回 `ToolResult`**：失败用 `success=False`，不让异常逃出（异常会让 LangGraph node 直接 fail，agent 没机会自我纠正）
- **`@tool` 装饰器自动从类型注解推导 JSON Schema**：`str` → `{"type":"string"}`、`int` → `{"type":"integer"}`、`list[str]` → `{"type":"array","items":{"type":"string"}}`；不支持复杂类型时报错（让人在 docstring 里手写 `parameters_schema=`）
- **`recall_memory` / `search_knowledge` / `load_skill` / `create_task` / `cancel_task` 都不做**——对应模块都没实现（[Tools § 8](../../../docs/modules/tools/README.md#8-内置-tool-清单v0v1) 是 V0/V1 的远期清单，0.1 只做基础 8 个）
- **MCP / HTTP tool 不做**（0.5 才接 MCP）

## 验收标准

- [ ] 单元测试覆盖每个 tool 的 happy path + 关键失败路径（路径越界 / 命令拒绝 / git 不存在等）
- [ ] `read_file` / `write_file` / `edit_file` 的语义跟 Claude Code 内置 tool 对齐：
  - `edit_file` 的 `old_string` 必须唯一，否则失败
  - `write_file` 覆盖现有内容
- [ ] `shell` 测试：构造越界 cwd / deny 命中 / 超时三种情况
- [ ] `grep` 在没装 ripgrep 的环境也能跑（fallback）
- [ ] `tools.registry.enabled(config)` 根据 `config.tools.builtin` 列表过滤——未启用的 tool 不进 registry
- [ ] `grep -rE "^from sigma\.(llm|trace|checkpoint|agent)" src/sigma/tools/` 必须为空

## 不做

- 不实现 `recall_memory` / `search_knowledge` / `load_skill` / `create_task` / `cancel_task`（对应模块在 0.3-0.5 才有）
- 不接 MCP server（0.5）
- 不做 docker / firejail 沙箱（V5+）
- 不做 `requires_approval` 流程（0.2 task 视图加上之后再做）

## 相关文档

- [Tools 模块](../../../docs/modules/tools/README.md)
- [Ports & Adapters § 3.2](../../../docs/architecture/ports-and-adapters.md#32-toolport)
