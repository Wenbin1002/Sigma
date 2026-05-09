# LLM

> 大语言模型调用层。被 Agent Runtime 内部消费，提供文本生成、工具调用解析等能力。

## 在架构中的位置

- **所属层**: Adapters
- **Port 接口**: `src/ports/llm.py` → `LLMPort`
- **实现目录**: `src/llm/`（未来可能在 `src/agent/` 内部直接使用）
- **被谁调用**: Agent Runtime（内部）
- **依赖**: 外部 LLM API

## Port 接口定义

```python
class LLMPort(Protocol):
    def stream(self, messages: list[dict], tools: list[dict] | None = None) -> AsyncIterator[str]: ...
    async def generate(self, messages: list[dict], tools: list[dict] | None = None) -> str: ...
```

## 可用实现

| 实现 | 类名 | 文件 | 配置值 | 状态 |
|------|------|------|--------|------|
| OpenAI | `OpenAILLM` | `src/llm/openai.py` | `openai` | V0 |
| DeepSeek | `DeepSeekLLM` | `src/llm/deepseek.py` | `deepseek` | V0 |

## 工作原理

LLM 层封装模型 API 调用细节：认证、重试、流式 token 输出、function calling 格式解析、token 用量统计。Agent Runtime 通过 LLMPort 调用模型，不关心底层用的是哪个 provider。

流式输出是核心——Agent 拿到每个 token 后立即向上层透传，实现打字机效果和语音流式合成。

## 配置示例

```yaml
llm:
  provider: openai
  model: gpt-4o
```

## 扩展方式

### 添加新 LLM Provider

1. 创建文件 `src/llm/<provider>.py`
2. 实现 `LLMPort` Protocol
3. 注册到 `src/llm/registry.py`
4. 添加配置项

### 约束

- 流式输出必须用 AsyncIterator
- Tool calling 结果格式统一为 `list[dict]`（与 OpenAI function calling 格式兼容）

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 单 provider，单 model | OpenAI / DeepSeek |
| V1 | 多 provider fallback，retry | OpenAI + DeepSeek + Qwen |
| V2 | 路由（按任务类型选 model）、缓存 | 大模型规划 + 小模型执行 |
| V3 | 成本优化、并发限流、自适应选模型 | — |

## 可扩展能力

- 路由策略：简单任务用便宜模型，复杂任务用强模型
- Prompt 缓存 / Semantic 缓存
- 本地模型接入（Ollama / vLLM）
- 多模态输入（图片、音频直传）
- 多 provider 负载均衡和故障转移

## 相关文档

- [Agent 模块](../agent/) — LLM 的主要消费方
- [Ports & Adapters](../../architecture/ports-and-adapters.md)
- [依赖规则](../../architecture/dependency-rules.md)
