# Tools 模块

> Tool 是 Sigma 的**行为扩展点**——能做什么动作。

> 详见 [架构总览 § 4.2](../../architecture/overview.md#42-tool)。

---

## 1. 定位

Tool 是 Sigma 三层扩展模型中"门槛最低"的一层：

| | Tool | Skill | Agent |
|---|---|---|---|
| 扩展什么 | **行为** | 知识 / 方法论 | 执行单元 |
| 写起来 | Python 函数（5 分钟） | Markdown（30 分钟） | Python 类 + graph（几小时-几天） |
| LLM 调用 | 0 | 0（注入 prompt） | 多次 |

---

## 2. Tool 的三种来源

### 2.1 内置 Python 函数（最简）

```python
from sigma import tool

@tool
def read_file(path: str) -> str:
    """读取文件内容。"""
    with open(path) as f:
        return f.read()
```

文件放在 `src/tools/builtin/` 或用户的 `~/.sigma/tools/`。

### 2.2 MCP Server（生态接入）

挂载 MCP server，复用 MCP 工具生态：

```yaml
tools:
  mcp_servers:
    - name: github
      command: npx
      args: ["@anthropic/mcp-github"]
    - name: filesystem
      command: npx
      args: ["@modelcontextprotocol/server-filesystem", "/path"]
```

Sigma 自动把 MCP server 暴露的 tool 接入到 ToolPort 体系。

### 2.3 HTTP Tool（V4+）

远程 HTTP API 包装为 tool（暂不实现）。

---

## 3. ToolPort

```python
class ToolPort(Protocol):
    name: str
    description: str
    parameters_schema: dict           # JSON Schema (OpenAI tool calling 格式)
    
    async def execute(self, **kwargs) -> ToolResult: ...
```

详见 [Ports & Adapters § 3.2](../../architecture/ports-and-adapters.md#32-toolport)。

**ToolResult** 是结构化的：

```python
@dataclass
class ToolResult:
    success: bool
    data: Any
    artifact: Artifact | None = None    # 文件 / 图片 / 报告等
    error: str | None = None
    metadata: dict = field(default_factory=dict)
```

---

## 4. Tool 注册

```python
# src/tools/registry.py
REGISTRY: dict[str, type[ToolPort]] = {
    "read_file": ReadFileTool,
    "write_file": WriteFileTool,
    "shell": ShellTool,
    "search_web": SearchWebTool,
    ...
}
```

启用通过 config：

```yaml
tools:
  builtin:
    - read_file
    - write_file
    - shell
    - search_web
  mcp_servers: [...]
```

未启用的 tool 不进入 agent 视野。

---

## 5. Agent 看到 Tool 的方式

每个 sub-agent 声明自己能用哪些 tool：

```python
class CoderAgent(sigma.Agent):
    tools = [read_file, write_file, shell, run_tests]
```

Master agent 看所有启用的 tool（或从 supervisor 那里得到精简列表）。

LLM 调用时，工具以 OpenAI tool calling schema 注入：

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "read_file",
        "description": "...",
        "parameters": { ... }
      }
    },
    ...
  ]
}
```

---

## 6. Tool 的安全考虑

某些 tool 危险（`shell`、`write_file`、`delete_file`）。控制方式：

### 6.1 Allowlist（启动时）

```yaml
tools:
  builtin:
    - read_file        # 安全
    - shell            # 危险但启用
  shell:
    allowed_commands: [git, ls, cat, ...]   # 子级 allowlist
```

### 6.2 用户确认（运行时）

某些 tool 标记为 `requires_approval: true`，调用时 agent 输出 `needs_approval` chunk，等用户在 chat / Task 视图确认。

```python
@tool(requires_approval=True)
def delete_file(path: str) -> bool:
    ...
```

### 6.3 沙箱（V5+）

危险 tool 在 docker / firejail 沙箱里跑（不在 Phase 1 范围）。

---

## 7. MCP 接入细节

启动时：
1. 读 config.yaml 的 `mcp_servers`
2. 启动每个 MCP server（subprocess）
3. 通过 MCP 协议握手 / 列出 server 暴露的 tool
4. 包装成 ToolPort 注册到 registry

运行时：
- Agent 调 MCP tool 跟调内置 tool 完全一样
- 失败时（MCP server 挂了）通过 BlockedException 报告
- Trace 里标注 tool 来源（builtin / mcp:github / mcp:filesystem）

---

## 8. 内置 Tool 清单（V0/V1）

| Tool | 用途 |
|---|---|
| `read_file` | 读文件 |
| `write_file` | 写文件 |
| `list_dir` | 列目录 |
| `shell` | 执行 shell 命令（带 allowlist） |
| `search_web` | 搜索网络 |
| `fetch_url` | 抓取网页 |
| `recall_memory` | 调用 [Memory](../memory/) 召回 |
| `search_knowledge` | 调用 [RAG](../rag/) 检索 |
| `load_skill` | 加载 [Skill](../skill/) 全文 |
| `create_task` | 创建后台 task（chat → task 升级用） |
| `cancel_task` | 取消 task |

---

## 9. 实现位置

```
src/tools/
  base.py             # ToolPort 基类、@tool 装饰器
  registry.py         # 注册表 + 工厂
  
  builtin/
    file.py           # read_file / write_file / list_dir
    shell.py          # shell tool + allowlist
    web.py            # search_web / fetch_url
    memory.py         # recall_memory
    knowledge.py      # search_knowledge
    skill.py          # load_skill
    task.py           # create_task / cancel_task
  
  mcp/
    client.py         # MCP server 客户端
    adapter.py        # MCP tool → ToolPort 包装
```

---

## 10. 未决问题

| 问题 | 状态 |
|---|---|
| Shell tool 的 allowlist 默认值 | V0 落地时定 |
| `requires_approval` 的 UX（chat 弹卡片 vs Web UI 弹窗） | V1 |
| 沙箱方案 | V5+ |

---

## 11. 相关文档

- [Skill 模块](../skill/) — Skill 引用 tool
- [Agent 模块](../agent/) — Agent 调用 tool
- [Ports & Adapters § 3.2](../../architecture/ports-and-adapters.md#32-toolport)
- [架构总览](../../architecture/overview.md)
