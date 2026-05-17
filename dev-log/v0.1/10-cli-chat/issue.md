# Issue 10 — CLI client `sigma chat`

**Milestone**：v0.1
**Type**：`feat(app)`
**依赖**：issue 09
**预计 PR 大小**：S-M

---

## 背景

CLI 是 Phase 1 的唯一客户端（[架构总览 § 2.2](../../../docs/architecture/overview.md#22-交互形态分阶段)）。本 issue 实现 `sigma chat` 子命令，通过 HTTP/SSE 跟 server 说话。

依赖红线：**`app/` 不直接调内核**（[依赖规则 § 3](../../../docs/architecture/dependency-rules.md#3-红线违反必拒)）。所以 `app/cli.py` 只能 import `core` 的数据结构 + `httpx`，**不准 import `chat/` `agent/` 等任何内核模块**。

## 范围

### 1. `src/sigma/app/cli.py`

用 `typer` 构造命令树：

```
sigma --help
sigma serve [--host ...] [--port ...]            # 起 server
sigma chat                                        # 进入交互式 REPL（新 session）
sigma chat "Q"                                    # 单次问答（新 session）
sigma chat --resume <session_id>                  # REPL（续 session）
sigma chat --new                                  # 等价于不带参数
sigma chat --list                                 # 列最近 20 条 session
```

Entry point：`sigma = sigma.app.cli:main`（issue 01 已声明）。

### 2. Server 自启动策略

CLI 启动时按以下顺序：

1. 探测 `http://127.0.0.1:7777/healthz`
2. 200 → 复用
3. 失败 → fork 后台 `sigma serve` 子进程（写 PID 到 `~/.sigma/server.pid`、stdout/stderr 重定向到 `~/.sigma/server.log`）；轮询 healthz 直到通（最多 5 秒）
4. 仍失败 → 友好报错退出

`sigma serve` 是显式前台运行（不 fork），方便 debug。后台启动只在 CLI 自动 spawn 时用。

### 3. SSE 客户端

用 `httpx.AsyncClient.stream()` 读 SSE：

- 解析 `event: chunk` / `data: {...}`
- 把每个 `AgentChunk` 渲染到终端：
  - `text_delta` → 直接 print（不换行，flush）
  - `tool_call` → 一行灰色提示「→ 调用 tool: read_file(path=README.md)」
  - `tool_result` → 一行灰色提示「✓ tool 完成」/失败时红色
  - `done` → 换行收尾
  - `error` → 红色错误信息 + 退出码 1
- 用 `rich` 做颜色（已在 typer 依赖里？没的话就 `colorama` 或裸 ANSI）

### 4. REPL 模式

- 简单 readline-style loop：`> ` 提示符 + 输入一行 + 流式打印响应
- `Ctrl+C`：取消当前 in-flight 请求（HTTP 断开），不退出 REPL
- `Ctrl+D` / `:quit` / `exit`：退出 REPL
- `:help` / `:list` / `:resume <id>`：内部命令（未来扩展，0.1 至少做 `:quit`）

### 5. `--list`

```
$ sigma chat --list
ID                     Title                 Updated
abc123...              帮我总结这篇...        2026-05-17 10:23
def456...              在 README 里查找...    2026-05-17 09:14
```

### 6. 配置加载

CLI 读 `~/.sigma/config.yaml`（不存在则用 issue 01 的 `config.example.yaml` 默认值）。本 issue 顺手加 `sigma config init` 命令：复制 example 到 `~/.sigma/config.yaml`。

## 设计要点

- **绝不 import 内核模块**（pre-commit 会查）
- **`core/` 数据结构允许 import**——CLI 反序列化 SSE chunk 时复用 `AgentChunk` dataclass，避免裸 dict
- **server 启动失败的报错要给具体 hint**（端口占用 / 配置错 / api_key 没设）
- **不要在 CLI 加业务逻辑**——所有"什么时候建议升级 task / 什么时候提示用户"都是 server 侧（agent 输出 chunk）的事

## 验收标准

- [ ] `tests/app/test_cli.py`（用 `typer.testing.CliRunner` + mock httpx）：
  - `sigma chat --list` 正确请求 `/chat/sessions` 并打印
  - `sigma chat "Q"` 创建 session + 发消息 + 流式打印
  - server 不可达时给出友好提示
- [ ] 集成 e2e（在 issue 11 做）：真启 server + CLI，跑通退出标准的多轮对话场景
- [ ] `grep -rE "^from sigma\.(agent|chat|server|llm|tools|trace|checkpoint|context|rag|memory|skill|task)" src/sigma/app/` 必须为空（依赖规则红线）
- [ ] `sigma --help` / `sigma chat --help` 输出整洁
- [ ] 未配置 api_key 时启动 server 给出明确报错（不是堆栈）

## 不做

- 不做 task 子命令（0.2）
- 不做 trace 子命令（0.6）
- 不做 memory / rag 子命令（0.3 / 0.4）
- 不做 TUI 全屏渲染（v2+）
- 不做配置向导（v2）
- 不做 server 自动停止 / 健康监控（v2）

## 相关文档

- [架构总览 § 2.2](../../../docs/architecture/overview.md#22-交互形态分阶段)
- [Chat 模块 § 6](../../../docs/modules/chat/README.md#6-cli-映射phase-1)
- [依赖规则 § 3](../../../docs/architecture/dependency-rules.md#3-红线违反必拒)
