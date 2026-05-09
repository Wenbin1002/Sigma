# 依赖规则

本文档是 Sigma 项目 import 约束的权威参考。**违反任何一条 = 架构退化，必须修复。**

## 规则总表

| 模块 | 允许 import | 禁止 import |
|------|------------|------------|
| `core/` | 标准库 | 任何外部包、任何其他 src/ 模块 |
| `ports/` | `core/` | 域目录、runtime/、外部 SDK |
| `runtime/` | `ports/`, `core/` | 域目录（voice/, agent/, rag/ 等） |
| `voice/` | `ports/`, `core/`, 外部 SDK | 其他域目录、runtime/ |
| `agent/` | `ports/`, `core/`, 外部 SDK | 其他域目录、runtime/ |
| `rag/` | `ports/`, `core/`, 外部 SDK | 其他域目录、runtime/ |
| `memory/` | `ports/`, `core/`, 外部 SDK | 其他域目录、runtime/ |
| `tools/` | `ports/`, `core/`, 外部 SDK | 其他域目录、runtime/ |
| `trace/` | `ports/`, `core/`, 外部 SDK | 其他域目录、runtime/ |
| `context/` | `ports/`, `core/`, 外部 SDK | 其他域目录、runtime/ |
| `app/` | `runtime/`, `ports/`, `core/` | 域目录 |

## 依赖图

```
                    ┌─────┐
                    │ app │
                    └──┬──┘
                       │
                       ▼
                  ┌─────────┐
                  │ runtime │
                  └────┬────┘
                       │
                       ▼
                   ┌───────┐
          ┌────────│ ports │────────┐
          │        └───┬───┘        │
          │            │            │
          ▼            ▼            ▼
     ┌─────────┐  ┌────────┐  ┌────────┐
     │ voice/  │  │ agent/ │  │  rag/  │  ...（所有域目录）
     └─────────┘  └────────┘  └────────┘
          │            │            │
          ▼            ▼            ▼
     外部 SDK      外部 SDK     外部 SDK

                   ┌──────┐
                   │ core │  ← 所有模块都可 import，core 自身无依赖
                   └──────┘
```

**箭头方向 = 允许 import 方向。禁止反向。域之间无箭头 = 禁止互相 import。**

## 为什么这样设计

| 规则 | 保证了什么 |
|------|-----------|
| `core/` 无外部依赖 | 数据结构稳定，不受框架更新影响 |
| `runtime/` 只 import `ports/` | 替换任何 adapter 不需要改 runtime 代码 |
| 域之间不互引 | 各域可独立开发、测试、替换 |
| 域只 import `ports/` + 外部 SDK | 实现可独立部署为微服务 |
| 禁止反向依赖 | 避免循环依赖，保持分层清晰 |

## 常见违规及修复

| 违规类型 | 错误示例 | 正确做法 |
|---------|---------|---------|
| Runtime import 域实现 | `from voice.whisper import WhisperSTT` | 使用 registry：`voice.registry.create(config)` |
| 域 import 域 | `from memory.local import LocalMemory`（在 rag/ 中） | 通过 Port 接口或 graph state 通信 |
| Core import 外部包 | `import openai`（在 core/ 中） | 外部依赖放到对应域目录 |
| 域 import runtime | `from runtime.task_manager import TaskManager` | 通过 Port 接口暴露需要的能力 |

## 自检方法

在提交代码前，检查新文件的 import 语句：

```bash
# 检查 runtime/ 是否违规 import 域目录
grep -r "from voice\|from agent\|from rag\|from memory\|from tools\|from trace\|from context" src/runtime/

# 检查域之间是否互相 import（以 voice/ 为例）
grep -r "from agent\|from rag\|from memory\|from tools\|from trace\|from context" src/voice/

# 检查 core/ 是否有外部依赖
grep -r "^import \|^from " src/core/ | grep -v "from core\|import dataclass\|import typing\|from typing\|from datetime\|from enum"
```

## 相关文档

- [架构总览](overview.md) — 系统分层设计
- [Ports & Adapters](ports-and-adapters.md) — 接口模式详解
- [添加 Adapter](../guides/adding-an-adapter.md) — 如何正确添加实现
