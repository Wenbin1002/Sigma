# Trace

> 追踪 Agent 的每一步执行，用于调试、回放和质量评估。

## 在架构中的位置

- **所属层**: 横切关注点（贯穿所有层）
- **性质**: 内部基础设施模块（**不走 Port 模式**）
- **目录**: `src/trace/`
- **被谁调用**: 所有模块直接 import `trace.api`
- **依赖**: 存储后端

## 为什么不走 Port

Trace 和 Agent/Memory/RAG 本质不同：

- 不存在"今天 JSONL 明天 Langfuse 后天 OTel 来回切"的场景
- 定了就不怎么动，不需要 Registry + config 动态选择
- 调用遍布全局，过度抽象反而碍事

做法：暴露固定 API，内部实现随时可换，调用方不感知。

## API

```python
# src/trace/api.py — 所有模块直接 import 这个
from trace.api import log, span

async def log(event: TraceEvent) -> None: ...
def span(name: str) -> ContextManager: ...
```

## 工作原理

Trace 模块记录系统运行的每一步：

- **每轮对话**：输入、输出、延迟、token 消耗
- **工具调用**：调用了什么、参数、结果、耗时
- **成本统计**：累计 token 用量和费用
- **错误追踪**：失败原因、重试次数

每个 trace event 包含时间戳、类型、数据和 span 层级关系。通过 span 嵌套，可以还原完整的执行路径树。

## 配置示例

```yaml
trace:
  output_dir: ./traces/
```

## 约束

- Trace 记录不能阻塞主流程（异步写入）
- 敏感信息需要脱敏处理
- Span 嵌套层级要清晰，便于调试

## 演进路径

| 阶段 | 能力 | 做法 |
|------|------|------|
| V0 | 本地 JSONL 日志 | `src/trace/backend.py` 写 JSONL |
| V1 | 结构化 trace（每步耗时、token、工具调用链） | 丰富 event 类型 |
| V2 | 在线可视化、成本看板 | 内部换为 Langfuse SDK，API 不变 |
| V3 | Eval 框架、A/B 测试 | 扩展 API |

## 后续能力

- 成本统计和预算告警
- 对话质量自动评分
- Replay：回放历史对话用于调试
- 性能 profiling（每步延迟分布、瓶颈定位）

## 相关文档

- [架构总览](../../architecture/overview.md)
