# 如何添加 Adapter

Adapter 是 Port 接口的具体实现。这是 Sigma 最常见的贡献类型——为某个模块接入新的技术方案。

## 前置阅读

- [Ports & Adapters](../architecture/ports-and-adapters.md) — 理解模式
- [依赖规则](../architecture/dependency-rules.md) — Import 约束
- 目标模块文档（如 [Voice](../modules/voice/)）— 了解 Port 接口

## 完整步骤

以添加 ElevenLabs TTS 为例：

### 1. 创建 Adapter 文件

路径：`src/voice/elevenlabs_tts.py`

```python
"""ElevenLabs TTS Adapter"""

from ports.tts import TTSPort
from typing import AsyncIterator


class ElevenLabsTTS:
    """ElevenLabs Text-to-Speech 实现"""

    def __init__(self, config):
        self.api_key = config.api_key
        self.voice_id = config.get("voice_id", "default")

    def synthesize(self, text: str) -> AsyncIterator[bytes]:
        """流式合成语音"""
        # 实现细节...
        ...
```

### 2. 实现 Port 接口

确保类满足 `TTSPort` Protocol 的所有方法签名。检查：
- 方法名完全一致
- 参数类型完全一致
- 返回类型完全一致（特别注意 `AsyncIterator` vs `async def`）

### 3. 注册到域 Registry

文件：`src/voice/registry.py`

```python
from voice.elevenlabs_tts import ElevenLabsTTS

REGISTRY: dict[str, type] = {
    "whisper": WhisperSTT,
    "edge": EdgeTTS,
    "elevenlabs": ElevenLabsTTS,  # ← 新增
    # ...
}
```

### 4. 添加配置

文件：`config.yaml`

```yaml
voice:
  tts:
    provider: elevenlabs      # ← 新 provider 值
    api_key: "sk-..."
    voice_id: "custom-voice"
```

### 5. 写测试

文件：`tests/voice/test_elevenlabs_tts.py`

```python
import pytest
from voice.elevenlabs_tts import ElevenLabsTTS


class TestElevenLabsTTS:
    async def test_synthesize_returns_audio_bytes(self):
        # 使用 mock，不依赖真实 API
        ...

    async def test_synthesize_streams_chunks(self):
        ...
```

### 6. 更新模块文档

文件：`docs/modules/voice/README.md` 的"可用实现"表格：

```markdown
| ElevenLabs | `ElevenLabsTTS` | `src/voice/elevenlabs_tts.py` | `elevenlabs` | V1 |
```

## PR 前检查清单

- [ ] 实现了 Port Protocol 的所有方法签名
- [ ] 所有入参出参是可序列化类型（无 callback、无文件句柄）
- [ ] 流式输出使用 `AsyncIterator`（不是 callback）
- [ ] 已注册到对应域的 `REGISTRY`
- [ ] 配置示例已添加
- [ ] 测试通过
- [ ] **没有 import 其他域目录**
- [ ] **没有 import `runtime/`**
- [ ] 模块文档已更新

## 常见问题

### Q: 我的 Adapter 需要依赖其他域的功能怎么办？

不要直接 import 其他域。正确做法：
- 通过 Port 接口获取能力（依赖注入）
- 通过 config 传入需要的配置
- 如果确实需要跨域协调，那是 Runtime 层的事

### Q: 外部 SDK 的依赖怎么处理？

在 `pyproject.toml` 中添加为 optional dependency：

```toml
[project.optional-dependencies]
elevenlabs = ["elevenlabs>=1.0"]
```

### Q: 一个文件只能放一个 Adapter 吗？

推荐一个文件一个 Adapter。如果多个实现非常相似（如同一 SDK 的不同模型），可以放在一个文件里。

## 相关文档

- [Ports & Adapters](../architecture/ports-and-adapters.md) — 模式详解
- [依赖规则](../architecture/dependency-rules.md) — Import 约束
- [代码规范](code-style.md) — 命名和风格
