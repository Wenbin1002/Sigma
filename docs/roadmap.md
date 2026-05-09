# Roadmap

各模块可平行探索。不必等 V0 完成才碰语音，不必等 V1 完成才试 RAG。版本只是"主线集成"的节奏。

---

## V0：能对话（2 周）

> 🚧 **当前阶段**

**目标**：最基础的对话能力跑通。

**做什么**：
- CLI 或简单 Web UI
- 文本输入 → LLM → 文本输出（streaming）
- 语音输入 → STT → LLM → TTS → 语音输出
- 基础 trace（每轮存个 JSON）

**不做**：RAG、Memory、工具调用、多轮状态管理

**退出标准**：
- [ ] 能用语音和 AI 对话，延迟可接受（< 3s 首字）
- [ ] 文本模式作为 fallback 可用
- [ ] 能在日志里看到每轮的输入输出

**关键模块**：[Voice](modules/voice/)、[LLM](modules/llm/)、[Agent](modules/agent/)、[Trace](modules/trace/)

---

## V1：能做事（2-3 周）

**目标**：Agent 能调用工具完成任务。

**做什么**：
- Agent 状态机：input → plan → tool_call → respond
- 2-3 个实际工具（web_search, file_read, code_run）
- 工具调用结果进入上下文
- 会话状态可持久化（重启不丢）

**不做**：复杂的多步规划、并行工具调用

**退出标准**：
- [ ] 用户说"帮我搜一下 X"，AI 能调工具并用结果回答
- [ ] 中断对话后重新连接能恢复上下文
- [ ] 工具调用过程在 trace 里可见

**关键模块**：[Agent](modules/agent/)、[Tools](modules/tools/)

---

## V2：能记住 + 能查资料（3-4 周）

**目标**：长期记忆和知识检索。

**做什么**：
- RAG：上传文档 → 检索 → 带引用回答
- Memory：跨会话记住用户偏好和关键信息
- Context compaction：长对话自动压缩

**不做**：GraphRAG、复杂实体关系

**退出标准**：
- [ ] 上传一份文档后能基于内容问答，回答带引用
- [ ] 隔天再聊，AI 还记得你之前说的重要信息
- [ ] 聊 50 轮以上不会因为 context 太长而崩

**关键模块**：[RAG](modules/rag/)、[Memory](modules/memory/)、[Context](modules/context/)

---

## V3+：更强（持续迭代）

- GraphRAG
- 更好的规划能力
- MCP 工具生态
- 多模态（图片理解）
- Eval 框架
- 生产级部署
- Multi-agent 协作
- Realtime 模式（端到端音频）

---

## 平行实验线

这些可以随时独立做，不阻塞主线：

| 实验 | 目标 | 目录 |
|------|------|------|
| voice-latency | 测试不同 STT/TTS 组合的延迟 | `experiments/voice/` |
| rag-eval | 测试不同 chunking/embedding 策略 | `experiments/rag/` |
| memory-extraction | 测试什么 prompt 能提取有用记忆 | `experiments/memory/` |
| agent-patterns | 试验不同 Agent 框架的手感 | `experiments/agent/` |

## 相关文档

- [架构总览](architecture/overview.md) — 系统设计
- [贡献指南](../CONTRIBUTING.md) — 如何参与
