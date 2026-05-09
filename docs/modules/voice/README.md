# Voice Pipeline

> 实时语音交互：用户说话 → AI 听懂 → AI 回复 → 播放给用户。支持打断。

## 在架构中的位置

- **所属层**: Adapters
- **Port 接口**: `src/ports/stt.py` → `STTPort`，`src/ports/tts.py` → `TTSPort`
- **实现目录**: `src/voice/`
- **被谁调用**: Runtime（Voice Pipeline）
- **依赖**: 外部 STT/TTS/VAD 服务

## Port 接口定义

```python
class STTPort(Protocol):
    async def transcribe(self, audio: bytes) -> str: ...

class TTSPort(Protocol):
    def synthesize(self, text: str) -> AsyncIterator[bytes]: ...
```

## Pipeline 流程

```
Mic → VAD → STT → Agent Runtime → TTS → Speaker
                                    ↑
                              Interrupt（打断检测）
```

## 可用实现

| 实现 | 类名 | 文件 | 配置值 | 状态 |
|------|------|------|--------|------|
| Whisper API | `WhisperSTT` | `src/voice/whisper.py` | `whisper` | V0 |
| FunASR（中文优化） | `FunASRSTT` | `src/voice/funasr.py` | `funasr` | V1 planned |
| Edge TTS | `EdgeTTS` | `src/voice/edge_tts.py` | `edge` | V0 |
| CosyVoice | `CosyVoiceTTS` | `src/voice/cosyvoice.py` | `cosyvoice` | V1 planned |
| Silero VAD | `SileroVAD` | `src/voice/silero_vad.py` | `silero` | V0 |

## 工作原理

Voice Pipeline 管理完整的语音 I/O 循环。VAD（Voice Activity Detection）检测用户何时开始/停止说话，触发 STT 转写。转写文本送入 Agent Runtime，Agent 的输出流式送到 TTS 合成并播放。

打断机制：当用户在 AI 播放时说话，VAD 检测到活动后立即停止当前 TTS 播放，开始新一轮 STT。

流控由 `src/runtime/voice_pipeline.py` 中的 async 事件循环处理，不走 LangGraph——实时音频对延迟的要求不适合图模型。

## 配置示例

```yaml
voice:
  stt: { provider: whisper }
  tts: { provider: edge }
  vad: { provider: silero }
```

## 双模式

语音交互支持两种模式：

| 模式 | 路径 | 适用场景 |
|------|------|---------|
| 级联（Cascade） | Audio → STT → LLM → TTS → Audio | 通用交互 |
| 端到端（Realtime） | Audio → Multimodal LLM → Audio | 需感知语气的场景 |

详见 [运行模式](../../architecture/runtime-modes.md)。

## 扩展方式

### 添加新 STT/TTS Adapter

1. 创建文件 `src/voice/<name>.py`
2. 实现 `STTPort` 或 `TTSPort` Protocol
3. 注册到 `src/voice/registry.py`
4. 添加配置项

### 约束

- STT/TTS 可独立为 gRPC streaming 服务（GPU 机器单独部署）
- VAD 策略可插拔（能量检测 / 模型检测 / 混合）
- 音频预处理 pipeline（降噪、回声消除）作为 middleware 插入

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 基础 STT + TTS，API 调用 | Whisper API, Edge TTS |
| V1 | 流式 STT、打断、VAD 优化 | FunASR, Deepgram, Silero |
| V2 | 低延迟本地推理、语音克隆 | Whisper.cpp, CosyVoice, Fish Speech |
| V3 | 多语言实时切换、说话人识别 | Qwen2-Audio |
| V4 | Realtime 模式（端到端） | OpenAI Realtime API, Gemini Live API |

## 相关文档

- [架构总览](../../architecture/overview.md)
- [运行模式](../../architecture/runtime-modes.md)
- [依赖规则](../../architecture/dependency-rules.md)
