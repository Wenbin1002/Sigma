# Observability

追踪 Agent 的每一步执行，用于调试、回放和质量评估。

## 特性

- 每轮对话完整 trace（输入、输出、延迟、token 消耗）
- 工具调用记录
- 成本统计
- 可升级到 Langfuse 可视化

## 可用实现

| Adapter | 说明 | 配置值 |
|---------|------|--------|
| JSONL | 本地 JSONL 文件 | `jsonl` |

## 扩展

实现 `TracePort` 接口 + 注册即可。
