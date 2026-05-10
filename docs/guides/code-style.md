# 代码规范

## 命名约定

| 类型 | 规范 | 示例 |
|------|------|------|
| Port 接口 | `XxxPort` | `LLMPort`, `ToolPort`, `TracerPort` |
| Adapter 实现 | 描述性名称（provider + 能力） | `OpenAILLM`, `MCPTool`, `JSONLTracer` |
| Sub-agent 类 | `XxxAgent` | `ResearcherAgent`, `CoderAgent` |
| Registry dict | `REGISTRY` | `REGISTRY: dict[str, type] = {...}` |
| 工厂函数 | `create` | `def create(config) -> XxxPort` |
| 配置键 | snake_case | `cost_guard`, `vector_store` |
| 文件名 | snake_case | `openai.py`, `multi_agent.py` |
| 测试文件 | `test_<被测模块>.py` | `test_openai.py` |

## 文件组织

```
src/<domain>/
  __init__.py          # 可选，保持简洁
  registry.py          # REGISTRY dict + create() 工厂（如果是有 Port 的域）
  <adapter_a>.py       # 一个文件一个 adapter
  <adapter_b>.py
```

- **一个 adapter 一个文件**
- **一个 Port 一个文件**（在 `src/ports/` 中）
- **Registry 集中**（每个有 Port 的域一个 `registry.py`）

## 类型标注

- 所有公开接口（Port 方法、工厂函数、Agent metadata）必须有类型标注
- 内部实现推荐标注，但不强制
- 使用 `from __future__ import annotations` 支持延迟求值

```python
from __future__ import annotations
from typing import AsyncIterator, Protocol

class LLMPort(Protocol):
    async def stream(self, messages: list[Message]) -> AsyncIterator[LLMChunk]: ...
```

## 异步约定

- Port 方法：`async def` 或返回 `AsyncIterator`
- 流式输出：统一用 `AsyncIterator`，**禁用 callback**
- 禁止在 async 上下文中做阻塞 I/O（用 `asyncio.to_thread()` 包装）

```python
# ✅ 正确
async def stream(self, ...) -> AsyncIterator[LLMChunk]: ...

# ❌ 错误：用 callback
def stream(self, ..., on_chunk: Callable) -> None: ...
```

## Docstring 风格

使用 Google 风格：

```python
class OpenAILLM:
    """OpenAI Chat Completions 实现。

    支持 streaming、tool calling、token usage 上报。

    Args:
        config: 含 api_key、model、base_url 的配置对象

    Example:
        llm = OpenAILLM(config)
        async for chunk in llm.stream(messages):
            print(chunk.content)
    """
```

## Import 规范

```python
# 1. 标准库
import asyncio
from dataclasses import dataclass
from typing import AsyncIterator

# 2. 第三方库（仅在域目录 / 内核模块中）
import openai
from langgraph.graph import StateGraph

# 3. 项目内部（遵循依赖规则）
from ports.llm import LLMPort
from core.schemas import Message, LLMChunk
```

顺序：标准库 → 第三方 → 项目内部，各组之间空一行。

## 错误处理

- Adapter 内部捕获外部 SDK 异常，转为统一的项目异常
- Sub-agent 卡住时 raise `BlockedException`，**不要 raise 通用异常**
- 错误信息应包含足够上下文用于调试

```python
# Sub-agent 卡住的标准方式
from sigma import BlockedException

if not file.exists():
    raise BlockedException(
        reason="missing_file",
        detail=f"需要的文件不存在：{file}",
        needed_input=NeededInput(type="file", description="提供文件路径"),
    )
```

## 数据结构

- 跨模块通信用 `dataclass` 或 `TypedDict`，**不用裸 dict**
- 数据结构定义集中在 `src/core/schemas.py`
- 入参出参可序列化（`str / bytes / int / float / bool / list / dict / dataclass`）

## 格式化工具

- Formatter: `ruff format`
- Linter: `ruff check`
- 行宽: 88（ruff 默认）

```bash
ruff format src/ tests/
ruff check src/ tests/
```

## Agent / Skill / Tool 的特殊规范

### Agent

```python
class XxxAgent(Agent):
    name = "xxx"                    # 唯一，全小写下划线
    description = "..."             # 一段话，写清擅长 / 不擅长
    triggers = [...]                # 可选关键词
    tools = [...]                   # tool 子集
    allowed_skills = [...] | None   # 可选
```

### Skill (SKILL.md front-matter)

```yaml
---
name: skill_name              # 必需，对应文件夹名
description: |
  一段话描述，给 supervisor / agent 看
triggers: [...]               # 可选
allowed_tools: [...]          # 可选
---
```

### Tool

```python
@tool
def name(...) -> ReturnType:
    """简洁描述（会成为 LLM 看到的 tool description）。
    
    Args:
        ...
    
    Returns:
        ...
    """
```

## 相关文档

- [依赖规则](../architecture/dependency-rules.md) — Import 约束详解
- [Ports & Adapters](../architecture/ports-and-adapters.md) — 接口设计模式
- [添加扩展](adding-an-adapter.md) — 实践中的规范应用
