# Voice Pipeline

实时语音交互：用户说话 → AI 听懂 → AI 回复 → 播放给用户。支持打断。

## 特性

- 实时语音输入（VAD + STT）
- 流式语音输出（LLM streaming + TTS）
- 用户打断（说话时停止播放）
- 文本 fallback（语音失败时降级为文本）

## Pipeline

```
Mic → VAD → STT → Agent Runtime → TTS → Speaker
                                    ↑
                              Interrupt（打断检测）
```

## 可用实现

| 组件 | Adapter | 配置值 |
|------|---------|--------|
| STT | Whisper API | `whisper` |
| STT | FunASR（中文优化） | `funasr` |
| TTS | Edge TTS | `edge` |
| TTS | CosyVoice | `cosyvoice` |
| VAD | Silero VAD | `silero` |

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 基础 STT + TTS，API 调用 | Whisper API, Edge TTS |
| V1 | 流式 STT（实时出字）、打断、VAD 优化 | FunASR, Deepgram, Silero |
| V2 | 低延迟本地推理、语音克隆、情感语音 | Whisper.cpp, CosyVoice, Fish Speech |
| V3 | 多语言实时切换、说话人识别 | Qwen2-Audio |
| V4 | Realtime 模式（端到端原生音频） | OpenAI Realtime API, Gemini Live API |

## 双模式

语音交互支持两种模式，详见 [runtime-design.md](../runtime-design.md)：

| 模式 | 路径 | 适用场景 |
|------|------|---------|
| 级联（Cascade） | Audio → STT → LLM → TTS → Audio | 通用交互，不需要感知语气 |
| 端到端（Realtime） | Audio → Multimodal LLM → Audio | 语言学习、情绪陪伴等需感知语气的场景 |

## 可扩展能力

- STT/TTS 可独立为 gRPC streaming 服务（GPU 机器单独部署）
- VAD 策略可插拔（能量检测 / 模型检测 / 混合）
- 音频预处理 pipeline（降噪、回声消除）作为 middleware 插入
- 多语言并行识别

## 扩展方式

实现对应 Port 接口（`STTPort` / `TTSPort`）+ 注册即可。
