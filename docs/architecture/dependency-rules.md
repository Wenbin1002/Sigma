# 依赖规则

> Sigma 的 import 约束。**违反任何一条 = 架构退化，必须修复。**
>
> 规则不为"未来可拆 gRPC"服务，只为单人维护期保持模块边界清晰、便于重构。

---

## 1. 模块层级

```
                    ┌─────┐
                    │ app │  CLI / Web 客户端
                    └──┬──┘
                       │
                       ▼
                  ┌─────────┐
                  │ server  │  HTTP/SSE Core API
                  └────┬────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   ┌────────┐    ┌──────────┐   ┌──────────┐
   │ agent/  │    │  task/    │   │ context/  │  内核模块
   │ skill/  │    │  chat/    │   │  rag/     │  (互相之间通过 core schema 通信)
   │         │    │           │   │ memory/   │
   └────┬────┘    └─────┬─────┘   └────┬─────┘
        │               │              │
        └───────┬───────┴──────────────┘
                ▼
            ┌───────┐
            │ ports │   只放真有多实现的能力契约
            └───┬───┘
                │
        ┌───────┼────────┬────────┐
        ▼       ▼        ▼        ▼
    ┌──────┐ ┌─────┐ ┌──────┐ ┌──────────┐
    │ llm/  │ │tools│ │trace │ │checkpoint│  域目录（adapter 实现）
    └──────┘ └─────┘ └──────┘ └──────────┘
        │       │       │         │
        ▼       ▼       ▼         ▼
       外部 SDK / MCP / SQLite / ...

                  ┌──────┐
                  │ core │  ← 所有模块都可 import；core 自身只用标准库
                  └──────┘
```

**箭头方向 = 允许 import 方向**。禁止反向。

---

## 2. 规则总表

| 模块 | 允许 import | 禁止 import |
|------|------------|------------|
| `core/` | 标准库 | 任何外部包、任何其他 `src/` 模块 |
| `ports/` | `core/` | 域目录、内核模块、`server/`、`app/` |
| `llm/` `tools/` `trace/` `checkpoint/`（域目录） | `ports/`、`core/`、外部 SDK | 其他域目录、内核模块、`server/`、`app/` |
| `agent/` `skill/` `task/` `chat/` `context/` `rag/` `memory/`（内核模块） | `ports/`、`core/`、外部 SDK（langgraph 等）、**核心模块互相允许 import**（见下） | `server/`、`app/`、域目录的具体实现（要走 registry） |
| `server/` | 内核模块、`ports/`、`core/` | `app/`、域目录的具体实现（走 registry） |
| `app/` | `server/`、`core/`（client 数据结构） | 内核模块、域目录、`ports/` |

### 2.1 关键变更点（vs 旧规则）

旧规则严格禁止"域之间互相 import"。**新规则放宽**：

- ✅ 内核模块（`agent/`, `task/`, `context/`, `rag/`, `memory/` 等）**可以互相 import**
- ❌ 域目录（`llm/`, `tools/`, `trace/`, `checkpoint/`）**仍然禁止互相 import**

**为什么放宽内核模块**：
- 内核模块本身就是 Sigma 的核心算法，互相协作是正常的（agent 调 context，context 调 rag/memory，task 调 agent...）
- 严格禁止互引会逼出"为合规而设计的中间层"，对单人项目是负资产
- 真正要保护的边界是 `ports/` 抽象（保证可替换的能力解耦）+ `app/`/`server/` 边界（保证 client/server 分离）

**为什么仍然禁止域目录互引**：
- 域目录的本质是"Port 的具体实现"
- 域目录互引 = 实现层耦合 = Port 抽象失效
- 比如 `llm/openai.py` 不应该 import `tools/python_tool.py`，因为这破坏了"换 LLM provider 不影响 tool"的保证

---

## 3. 红线（违反必拒）

这三条是不可妥协的：

1. **`ports/` 不依赖任何具体实现**——`ports/` 里 import 任何域目录就是退化
2. **域目录之间不互引**——`llm/`、`tools/`、`trace/`、`checkpoint/` 互相禁止 import
3. **`app/` 不直接调内核**——必须走 `server/` 暴露的 HTTP/SSE API（client/server 分离）

---

## 4. 常见违规及修复

| 违规 | 错误示例 | 正确做法 |
|------|---------|---------|
| ports 依赖具体实现 | `from llm.openai import OpenAILLM`（在 `ports/llm.py` 里） | ports 只定义 Protocol，不依赖任何 adapter |
| 域之间互引 | `from tools.python_tool import ...`（在 `llm/openai.py` 里） | 通过 core schema 通信；不该互调时考虑是否设计有问题 |
| 内核模块直接 import 域实现 | `from llm.openai import OpenAILLM`（在 `agent/master.py` 里） | 走 registry：`from llm.registry import create as create_llm; llm = create_llm(config)` |
| App 直接调内核 | `from agent.master import MasterAgent`（在 `app/cli.py` 里） | 通过 `server/` 暴露的 HTTP API：`http_client.post("/chat", ...)` |
| Core 引外部包 | `import openai`（在 `core/` 里） | core 只放数据结构 / 事件，外部依赖放对应域 |

---

## 5. 自检方法

**Pre-commit hook 已经覆盖部分检查**（见 `.githooks/pre-commit`）。手动检查：

```bash
# ports/ 不应 import 任何域目录
grep -rE "^from (llm|tools|trace|checkpoint)" src/ports/

# 域目录之间不互引
for d in llm tools trace checkpoint; do
  grep -rE "^from (llm|tools|trace|checkpoint)" src/$d/ | grep -v "from $d"
done

# app/ 不应直接 import 内核模块
grep -rE "^from (agent|skill|task|chat|context|rag|memory|server)" src/app/ \
  | grep -v "from server"

# core/ 不应有外部依赖
grep -rE "^import |^from " src/core/ | grep -vE "^src/core/.*from (typing|datetime|enum|dataclasses|abc)"
```

---

## 6. 为什么这样设计（精简版）

| 规则 | 真实收益 |
|------|---------|
| `core/` 只用标准库 | 数据结构稳定，不被框架更新拖死 |
| `ports/` 不依赖实现 | Protocol 抽象的根本前提 |
| 域目录互不依赖 | 替换 LLM provider / Tracer 时不会牵动其他 adapter |
| 内核模块允许互引 | 单人项目避免过度设计；内核协作是正常需求 |
| `app/` 走 `server/` | Phase 2 加 Web UI 时前端只是另一个 client |

---

## 7. 已废弃的规则

- ❌ "runtime/ 只 import ports/"（旧规则）——内核模块已合并到具体职责模块（agent/task/context...），无统一的 runtime/
- ❌ "可序列化、为 gRPC 拆分预留"——单人项目这是过度设计，废弃
- ❌ "域之间禁止 import"覆盖所有内核模块——已放宽，只对 LLM/Tools/Trace/Checkpoint 这种域 adapter 保留

---

## 8. 相关文档

- [架构总览](overview.md) — 项目整体分层
- [Ports & Adapters](ports-and-adapters.md) — Port 设计原则和清单
- [设计决策日志](design-log.md) — 依赖规则演化历史
