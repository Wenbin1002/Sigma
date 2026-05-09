# 代码规范

## 命名约定

| 类型 | 规范 | 示例 |
|------|------|------|
| Port 接口 | `XxxPort` | `STTPort`, `LLMPort`, `AgentRuntimePort` |
| 实现类 | 描述性名称（provider + 能力） | `WhisperSTT`, `OpenAILLM`, `SimpleLoopAgent` |
| Registry dict | `REGISTRY` | `REGISTRY: dict[str, type] = {...}` |
| 工厂函数 | `create` | `def create(config) -> XxxPort` |
| 配置键 | snake_case | `agent_runtime`, `vector_store` |
| 文件名 | snake_case | `simple_loop.py`, `elevenlabs_tts.py` |
| 测试文件 | `test_<被测模块>.py` | `test_whisper.py` |

## 文件组织

```
src/<domain>/
  __init__.py          # 可选，保持简洁
  registry.py          # REGISTRY dict + create() 工厂
  <adapter_a>.py       # 一个文件一个 adapter
  <adapter_b>.py
```

- **一个 adapter 一个文件**
- **一个 Port 一个文件**（在 `src/ports/` 中）
- **Registry 集中管理**（每个域一个 `registry.py`）

## 类型标注

- 所有公开接口（Port 方法、工厂函数）必须有类型标注
- 内部实现推荐标注，但不强制
- 使用 `from __future__ import annotations` 支持延迟求值

```python
from __future__ import annotations
from typing import AsyncIterator, Protocol

class STTPort(Protocol):
    async def transcribe(self, audio: bytes) -> str: ...
```

## 异步约定

- Port 方法：`async def` 或返回 `AsyncIterator`
- 流式输出：统一用 `AsyncIterator`，不用 callback
- 禁止在 async 上下文中做阻塞 I/O（用 `asyncio.to_thread()` 包装）

```python
# ✅ 正确
def stream(self, context: Context, input: str) -> AsyncIterator[AgentChunk]: ...

# ❌ 错误：用 callback
def stream(self, context: Context, input: str, on_chunk: Callable) -> None: ...
```

## Docstring 风格

使用 Google 风格：

```python
class WhisperSTT:
    """Whisper API 语音转文字实现。

    使用 OpenAI Whisper API 将音频转为文本。
    支持多语言自动检测。

    Args:
        config: 包含 api_key 和 model 的配置对象

    Example:
        stt = WhisperSTT(config)
        text = await stt.transcribe(audio_bytes)
    """
```

## Import 规范

```python
# 标准库
import asyncio
from dataclasses import dataclass
from typing import AsyncIterator

# 第三方库（仅在域目录中）
import openai

# 项目内部（遵循依赖规则）
from ports.stt import STTPort
from core.schemas import Context
```

顺序：标准库 → 第三方 → 项目内部，各组之间空一行。

## 错误处理

- Adapter 内部捕获外部 SDK 异常，转为统一的项目异常
- Port 层面不定义具体异常类型（保持简洁）
- 错误信息应包含足够上下文用于调试

## 格式化工具

- Formatter: `ruff format`
- Linter: `ruff check`
- 行宽: 88（ruff 默认）

```bash
# 格式化
ruff format src/ tests/

# 检查
ruff check src/ tests/
```

## 相关文档

- [依赖规则](../architecture/dependency-rules.md) — Import 约束详解
- [Ports & Adapters](../architecture/ports-and-adapters.md) — 接口设计模式
- [添加 Adapter](adding-an-adapter.md) — 实践中的规范应用
