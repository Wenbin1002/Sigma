# Roadmap

> 各模块可平行探索。不必等 V0 完成才碰 RAG，不必等 V1 完成才试 multi-agent。版本只是"主线集成"的节奏。

---

## V0：Hello Agent（最小可用骨架）

> 🚧 **当前阶段**

**目标**：跑通最基础的 chat，验证内核选型。

**做什么**：
- `src/` 项目骨架（按 [架构总览](architecture/overview.md) § 7 项目结构搭起来）
- LangGraph 内核接入（Master Agent 单节点 graph）
- LLMPort + 1 个 adapter（OpenAI 或 DeepSeek）
- ToolPort + 几个内置 tool（read_file / write_file / shell）
- TracerPort + JSONLTracer
- CheckpointerPort + LangGraph SqliteSaver
- Server (HTTP/SSE) 暴露最简 chat API
- CLI client：`sigma chat`

**不做**：Task / Skill / Sub-agent / RAG / Memory / 多 provider 路由

**退出标准**：
- [ ] `sigma chat` 能跟 LLM 多轮对话
- [ ] 工具调用流跑通（agent 调 read_file 读文件回答）
- [ ] Trace JSONL 文件能看到每次 LLM 调用 / tool call
- [ ] 中断恢复：kill 进程后 `sigma chat --resume <session>` 能续上

---

## V1：Task 系统 + 4 个核心场景中的 1-2 个

**目标**：Chat / Task 双视图跑通，验证 4 个核心场景中 1-2 个最容易的。

**做什么**：
- Task Engine：状态机（queued / running / paused / completed / failed / cancelled）
- Task queue + 持久化（基于 SQLite）
- Task CLI：`sigma task new / list / view / pause / resume / cancel`
- Pause/resume 机制（基于 LangGraph interrupt + Command(resume=...)）
- Chat ↔ Task context 打通（chat 升级时带历史）
- 选 1-2 个核心场景跑通（推荐：场景 4 coding，因为 tool 已有）

**不做**：周期性 task（cron）、multi-agent、RAG/Memory（先用极简版）

**退出标准**：
- [ ] `sigma task new "..."` 能起一次性 task 后台跑
- [ ] Task 卡 L3 能进 paused，`sigma task resume` 能续
- [ ] Trace 里能完整看到 task 执行（含 pause / resume）
- [ ] 选定的 1-2 个场景能完整跑通（端到端 demo）

---

## V2：Multi-agent + Skill 系统 + Self-improvement MVP

**目标**：扩展模型完整落地，支持复杂场景；self-improvement 闭环跑起来。

**做什么**：
- Master Agent + Supervisor 路由
- @-mention 显式召唤
- Sub-agent 三级回退（BlockedException 协议 + L2 主 agent 代答 + L3 升级用户）
- Sub-agent 用户扩展点（`~/.sigma/agents/`）
- Skill 系统（扫描 + metadata 注入 + 渐进式加载）
- Skill 用户扩展点（`~/.sigma/skills/`）
- 至少 2-3 个内置 sub-agent（researcher / coder / analyst）
- 至少 5+ 内置 skill
- **Self-improvement MVP**：显式偏好提取 + Memory 写入闭环 + 用户可见 / 可编辑

**退出标准**：
- [ ] 用户能写 sub-agent 并被 Supervisor 路由
- [ ] 用户能写 skill 并被自动加载
- [ ] 三级回退完整工作（含 trace 审计）
- [ ] 4 个核心场景全部跑通
- [ ] 用户明说"以后用 X 风格"后，下次对话能感知到 memory 已生效

---

## V3：RAG + Memory（Sigma 真正的差异化能力）

**目标**：把 context engineering 做到位；Self-improvement 加入隐式 signal。

**做什么**：
- RAG：多 index 管理（按 task / session 配置）
- 至少接一个 vector store（Qdrant / Chroma / LanceDB）
- 文档索引 pipeline（chunking / embedding / metadata）
- 检索时带引用
- Memory 三层：global（用户偏好）/ session（对话上下文）/ task（临时 state）
- Memory extraction（每次会话结束自动总结进 global memory）
- Context Engine 把 RAG / Memory / 历史压缩拼装到 LLM 上下文
- **Self-improvement V2**：隐式 signal 提取（重复问 / 转话题 / task 反馈）

**退出标准**：
- [ ] 上传文档 → 基于内容问答（带引用）
- [ ] 隔天 chat 还记得偏好
- [ ] Sub-agent 能用 RAG 工具查行业知识
- [ ] 长对话（50+ 轮）不崩
- [ ] 周期 task（场景 1 雏形）能从用户反馈中学（点开/忽略 → 兴趣模型）

---

## V4：Realtime + Web UI + 周期性 Task

**目标**：第三种交互模式上线；从 CLI-only 升级到完整产品形态；Realtime + self-improvement 完整闭环。

**做什么**：
- **Realtime 模式**：
  - 接 OpenAI Realtime API
  - WebSocket session 管理
  - Middleware 三层（前注入 memory / 中拦截 tool / 后沉淀 transcript）
  - 1-2 个杀手场景 demo（推荐：语言陪练 / 心理陪伴）
  - CLI: `sigma realtime start`
- **Web UI**（Phase 2）：React/Svelte + 走同一套 HTTP/SSE/WebSocket API
  - Chat 视图 / Task 视图 / Skill 列表 / Agent 列表 / Realtime 视图（transcript 实时展示）
- **周期性 task**（cron 调度）
- 推送通道至少 1 种（邮件 / 桌面通知 / 自定义 webhook）
- 场景 1（每天早上社媒推送）端到端跑通
- **Self-improvement V3**：Audio metadata signal（打断 / 沉默）+ 完整闭环可观察

**退出标准**：
- [ ] `sigma realtime start` 能跟模型实时语音对话；memory 注入生效
- [ ] Realtime 会话后 transcript / 偏好自动入 memory
- [ ] 浏览器打开 Sigma Web UI 跟 CLI 完全等价
- [ ] 周期性 task 按 cron 自动跑
- [ ] 场景 1 能稳定每天产出推送，质量随用户反馈提升
- [ ] 至少 1 个 realtime 杀手场景可演示

---

## V5+：高级特性

按真实需求出现的顺序做，不预先承诺：

- **第二个 Realtime provider**（Gemini Live）；视情况抽 RealtimePort
- **Self-improvement V4**：跨 session 学习；进步轨迹分析
- **Trace viewer**：本地 HTML viewer 完整版（time-travel / replay / diff）
- **Cost-aware routing**：简单任务走便宜 model，复杂任务走贵 model
- **Agent / Skill 分发**：从 registry 下载别人的 agent / skill（含安全沙箱）
- **Voice 模块**（Cascade STT/TTS，如果重新启用）
- **更多 LLM provider**
- **Eval 框架**（agent 行为质量评估）

---

## 平行实验线

这些可以随时独立做，不阻塞主线：

| 实验 | 目标 | 目录 |
|------|------|------|
| `experiments/rag/` | 测试不同 chunking / embedding 策略 | — |
| `experiments/memory/` | 测试什么 prompt 能提取有用 memory | — |
| `experiments/agent/` | 试验不同 agent 模式（ReAct / Plan-Execute / ...） | — |
| `experiments/trace-viewer/` | 本地 trace viewer 原型 | — |
| `experiments/cost-routing/` | cost-aware 路由策略对比 | — |

---

## 路线图原则

- **跟着 4 个核心场景走**——任何 V 版本如果不能让某个核心场景前进，就不做
- **学习价值优先**——选实现路径时优先选能学到东西的，不优先选最简的
- **不预先做 Port**——只在已有 ≥2 个真实需求时才做抽象
- **小步走**——每个 V 版本可独立 demo / 独立写博客复盘

## 相关文档

- [架构总览](architecture/overview.md)
- [设计决策日志](architecture/design-log.md) — 路线图变更的演化记录
- [贡献指南](../CONTRIBUTING.md)
