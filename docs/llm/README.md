# LLM

大语言模型调用层。被 Agent Runtime 内部消费，提供文本生成、工具调用解析等能力。

## 特性

- 流式输出（streaming tokens）
- 多 provider 支持
- 工具调用格式解析（function calling）
- Token 用量统计

## 可用实现

| Adapter | 说明 | 配置值 |
|---------|------|--------|
| OpenAI | GPT-4o 系列 | `openai` |
| DeepSeek | DeepSeek 系列 | `deepseek` |

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

## 扩展方式

实现 `LLMPort` 接口 + 注册即可。
