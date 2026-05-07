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

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 本地 JSONL 日志 | JSONL |
| V1 | 结构化 trace（每步耗时、token、工具调用链） | JSONL + 查询工具 |
| V2 | 在线可视化、成本看板 | Langfuse, LangSmith |
| V3 | Eval 框架、A/B 测试、自动回归检测 | 自研 eval pipeline |

## 可扩展能力

- Trace 后端可替换（本地文件 / Langfuse / OpenTelemetry）
- 成本统计和预算告警
- 对话质量自动评分
- Replay：回放历史对话用于调试
- 性能 profiling（每步延迟分布、瓶颈定位）

## 扩展方式

实现 `TracePort` 接口 + 注册即可。
