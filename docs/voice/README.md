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

## 扩展

实现对应 Port 接口（`STTPort` / `TTSPort`）+ 注册即可。
