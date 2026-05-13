# Roadmap

> **V1 是唯一的完整交付目标。** 0.1 → 0.6 是到达 V1 的六个里程碑，每个可独立 demo / 独立测试。
>
> CLI only — Web UI 和 Realtime 语音放 V2+。

---

## 总览

```
0.1 ──→ 0.2 ──→ 0.2.5 ─→ 0.3 ──→ 0.4 ──→ 0.5 ──────→ 0.6 ＝ V1
骨架      Task    Coding    Memory   RAG    Multi-Agent  周期
单Agent   引擎    单 agent  Context  讨论书 Skill         社媒
对话      基建    场景4✓    偏好MVP  场景3✓ 场景2✓        场景1✓
                  (dogfood)                 场景4增强
```

| Milestone | 主题 | 核心模块 | 解锁场景 |
|-----------|------|---------|----------|
| **0.1** | 骨架 + 单 Agent 对话 | Agent · LLM · Tool · Trace · Chat · Server · CLI | — |
| **0.2** | Task 引擎 | Task · Chat↔Task | — |
| **0.2.5** | Coding（单 agent）+ dogfood | Coding tools · Plan/Edit/Run loop · 错误自修复 | 场景 4 ✓（单 agent） |
| **0.3** | Memory + Context + Self-Improvement MVP | Memory · Context · Improvement | — |
| **0.4** | RAG + 讨论书 | RAG · Context 增强 | 场景 3 ✓ |
| **0.5** | Multi-Agent + Skill + 数据分析 | Agent(multi) · Skill · LLM(multi-provider) | 场景 2 ✓ · 场景 4 增强 |
| **0.6** | 周期 Task + 推送 + 社媒 | Task(cron) · 推送 · Trace viewer | 场景 1 ✓ |
| **V1** | = 0.6 完成，4 个核心场景全部跑通 | 全部 | **全部** ✓ |

### 4 个核心场景

| # | 场景 | 模式 | 解锁于 |
|---|------|------|--------|
| 1 | 每天早上从社交媒体（小红书/X/知乎）筛选感兴趣内容并推送 | task（周期性） | 0.6 |
| 2 | 查询生猪期货数据，生成趋势图，判断供需走向 | task（一次性） | 0.5 |
| 3 | 多轮讨论一本书，最终总结笔记 | chat | 0.4 |
| 4 | Coding（写代码、改代码、跑代码） | chat + task | **0.2.5**（单 agent）→ 0.5（multi-agent 协作增强） |

---

## 0.1：骨架 + 单 Agent 对话

> 🚧 **当前阶段**

**目标**：跑通最基础的 chat，验证内核选型。

**做什么**：
- `src/` 项目骨架（按 [架构总览](architecture/overview.md) § 9 项目结构搭建）
- LangGraph 内核接入（Master Agent 单节点 graph）
- LLMPort + 1 个 adapter（OpenAI 或 DeepSeek）
- ToolPort + 内置 tool（read_file / write_file / shell / web_search / grep / glob / edit_file / git）
- TracerPort + JSONLTracer
- CheckpointerPort + LangGraph SqliteSaver
- Server（HTTP/SSE）暴露最简 chat API
- CLI client：`sigma chat`

**不做**：Task / Skill / Sub-agent / RAG / Memory / 多 provider 路由

**退出标准**：
- [ ] `sigma chat` 能跟 LLM 多轮对话
- [ ] 工具调用流跑通（agent 调 read_file 读文件回答）
- [ ] Trace JSONL 文件能看到每次 LLM 调用 / tool call
- [ ] 中断恢复：kill 进程后 `sigma chat --resume <session>` 能续上

---

## 0.2：Task 引擎

**目标**：Chat / Task 双视图跑通，Task 基建完整可用。

**做什么**：
- Task Engine：状态机（queued → running → completed / paused / failed / cancelled）
- Task queue + SQLite 持久化
- Pause/resume 机制（基于 LangGraph interrupt + `Command(resume=...)`）
- Chat ↔ Task 打通：升级建议（chat 里 agent 建议转 task）+ 完成回流（task 结果注入 chat）
- CLI：`sigma task new / list / view / pause / resume / cancel`

**不做**：周期性 task（cron）、multi-agent、RAG / Memory、coding 专用循环（下一个 milestone）

**退出标准**：
- [ ] `sigma task new "帮我总结这篇文章"` 能后台跑 task
- [ ] Task 卡住能进 paused，`sigma task resume` 能续
- [ ] Chat 中建议升级 task，task 完成后回流通知
- [ ] Trace 里能完整看到 task 执行（含 pause / resume）

---

## 0.2.5：Coding（单 agent 形态）+ dogfood

**目标**：让 Sigma 在**单 agent 形态**下能稳定写 / 改 / 跑代码，从此 Sigma 能反哺自身开发——0.3 之后所有 milestone 都用 Sigma 自己加速。

**为什么单独成一个 milestone**（而不是放在 0.5 multi-agent 里）：
- Dogfood 红利早收 = 后续 4 个 milestone 全部受益
- 单 agent coding 跟 multi-agent 协作是**两个独立的设计空间**，先各自跑通再组合，符合 "小步走"
- 0.5 接 supervisor 时只需把 0.2.5 的 coder 包成 sub-agent，不会重写

**做什么**：
- Coding tool 完整化：精准 patch（diff-style edit，避免整文件覆盖）/ apply_patch / lint / test runner / code search（基于 0.1 的 grep/glob 增强）
- Plan / Edit / Run / Verify 循环：单 agent 内置 prompt 编排（不引入 sub-agent 概念）
- 错误自修复循环：跑测试 → 失败 → 读错误 → 改代码 → 重跑（最多 N 轮，超过则三级回退）
- Shell 沙箱基建：CWD 锁定 / 黑名单（rm -rf / / sudo / curl | sh）/ 超时 / 输出截断
- Coding-aware context：自动收集相关文件（imports / 定义 / 测试），打包进 LLM context
- CLI：`sigma code "<task>"` 直接进入 coding task；`sigma chat` 内 LLM 自主使用 coding tool
- **Dogfood 验收**：用 Sigma 自己开发 0.3 的 Memory 模块的某个子任务（端到端可见进展）

**不做**：
- coder sub-agent（留到 0.5，届时把本 milestone 的 coding 逻辑包进 sub-agent）
- 多文件大型重构（context 还没有 RAG，先做小到中等改动）
- 代码库索引（等 0.4 RAG）

**退出标准**：
- [ ] `sigma code "在 src/utils.py 加 fibonacci 函数并写测试"` 能端到端跑通（写代码 / 跑测试 / 失败自修复 / 通过）
- [ ] `sigma code` 能在 Sigma 自己的代码库里跑（用 Sigma 改 Sigma 的小功能）
- [ ] 错误自修复循环上限可配置，超限触发 task pause（三级回退 L3）
- [ ] Shell 沙箱：危险命令拒绝执行，CWD 越界拒绝
- [ ] 至少 1 次"用 Sigma 写 Sigma"产出的 PR 合入 main

---

## 0.3：Memory + Context + Self-Improvement MVP

**目标**：让 Sigma 有记忆、会记偏好——"越用越懂你"的起点。

**做什么**：
- Memory 三层存储：global（用户偏好）/ session（对话上下文）/ task（临时 state）
- Memory CLI：`sigma memory list / edit / delete`
- Context Engine：多源拼装（system prompt + memory + 历史压缩 + tool/skill metadata）+ token budget 管理
- Self-improvement MVP：**显式偏好**提取（用户明说"以后用更简洁的风格"）→ memory 写入 → 下次对话自动生效
- 用户可查看 / 编辑 / 删除 memory 条目（可审计）

**不做**：RAG（下个 milestone）、隐式信号提取（V2+）、audio metadata

**退出标准**：
- [ ] 用户说"以后回答用中文"，下次新 session 自动用中文
- [ ] `sigma memory list` 能看到提取的偏好，可编辑
- [ ] 隔天开新 session 还记得偏好
- [ ] 长对话（50+ 轮）context 压缩正常，不崩不截断

---

## 0.4：RAG + 讨论书场景

**目标**：Sigma 能基于外部文档回答问题，场景 3（讨论书）端到端跑通。

**做什么**：
- RAG 多 index 管理（每个 index 独立：书、代码库、行业知识等）
- Indexing pipeline：parse → chunk → embed → vector store
- 至少接 1 个 vector store（Qdrant / Chroma / LanceDB）
- 检索带引用（保留源位置 metadata）
- Context Engine 增强：RAG 检索结果作为 context 源之一，参与 token budget 和优先级排序
- CLI：`sigma rag create <name> / index <path> / search <query> / list / delete`
- 场景 3（多轮讨论一本书 + 总结笔记）端到端跑通

**不做**：Multi-agent（单 agent + RAG tool 即可）、代码库索引优化（后续迭代）

**退出标准**：
- [ ] 上传一本书 → 基于内容多轮问答，回答带引用（章节 / 页码）
- [ ] 讨论 50 轮后让 Sigma 总结笔记，质量可用
- [ ] 多个 index 共存互不干扰
- [ ] RAG 检索结果在 context 中正确参与 token budget 分配

---

## 0.5：Multi-Agent + Skill + 数据分析场景

**目标**：扩展模型完整落地（Tool / Skill / Agent 三层正交），场景 2（期货数据分析）端到端跑通；场景 4（Coding）从单 agent 升级为 multi-agent 协作形态。

**做什么**：
- Master Agent + Supervisor 路由
- @-mention 显式召唤（`@researcher 帮我查数据`）
- Sub-agent 三级回退（L1 自己尽力 → L2 主 agent 代答 → L3 升级用户）+ BlockedException 协议
- 至少 3 个内置 sub-agent：researcher（资料收集）/ coder（代码执行）/ analyst（数据分析）
- **Coder sub-agent = 把 0.2.5 的 coding 单 agent 形态包成 sub-agent**——复用 plan/edit/run/verify 循环、tool 集合、shell 沙箱；新增的只是 supervisor 路由 + BlockedException 协议
- Sub-agent 用户扩展点（`~/.sigma/agents/`）
- Skill 系统：扫描 `~/.sigma/skills/` + metadata 注入 system prompt + 渐进式 `load_skill()` 加载
- 至少 5+ 内置 skill
- Skill 用户扩展点（`~/.sigma/skills/`）
- LLM 多 provider：接入 ≥ 2 个 provider + cost-aware 路由（简单任务走便宜模型）
- **Tool 生态接入（第一步）：MCP adapter**——实现 MCP client（stdio + HTTP/SSE），让 Sigma 能消费任意 MCP server（filesystem / postgres / 各家 SaaS API）；对外开放生态的奠基（详见 [D-20](architecture/design-log.md#2-已锁定的决策)）
- 场景 2（查询期货数据 + 生成趋势图 + 判断供需）端到端跑通
- 场景 4 增强：multi-agent 协作（researcher 查依赖文档 + coder 写代码 + analyst 解释结果）

**不做**：周期性 task（cron）、推送通道、Realtime

**退出标准**：
- [ ] 用户说"帮我分析生猪期货"，Supervisor 自动路由到 analyst agent
- [ ] `@researcher 查一下母猪存栏数据` 直接派给 researcher
- [ ] `@coder 帮我重构这个函数` 或 Supervisor 自动路由到 coder agent
- [ ] Sub-agent 缺信息时三级回退完整工作（trace 可审计）
- [ ] 用户写的 skill 能被自动发现和加载
- [ ] 用户写的 sub-agent 能被 Supervisor 路由
- [ ] 场景 2：期货数据分析端到端跑通（含趋势图生成）
- [ ] 场景 4 增强：coder + researcher 协作完成跨模块改动（如改 API 时同时更新文档）
- [ ] Cost-aware 路由：同一 session 内简单问题走便宜模型，复杂分析走强模型
- [ ] MCP adapter 跑通：至少接入 1 个外部 MCP server（filesystem 或 postgres），researcher / analyst 能消费

---

## 0.6：周期 Task + 推送 + 社媒场景

**目标**：最后一个核心场景落地，V1 完整交付。

**做什么**：
- 周期性 task 调度（cron 表达式 / 自然语言周期）
- 推送通道（至少 1 种：邮件 / 桌面通知 / 自定义 webhook）
- **Tool 生态接入（第二步）：OpenCLI adapter**——subprocess + JSON 调 `opencli`，复用其 144 个 site adapter（小红书 / X / 知乎 / Bilibili / Reddit / HackerNews ...），不自己写社媒抓取（详见 [D-20](architecture/design-log.md#2-已锁定的决策)）
- CLI：`sigma task schedule` / 推送配置
- 场景 1（每天早上从社媒筛选内容并推送）端到端跑通
- Trace HTML viewer 完整版（timeline / 嵌套树 / cost 面板 / 过滤）
- 全量集成测试 + 4 场景回归

**不做**：Web UI、Realtime、隐式 self-improvement、为 OpenCLI 写新 site adapter（不在本项目范围）

**退出标准**：
- [ ] `sigma task schedule "每天早上8点推送社媒摘要"` 自动按时执行
- [ ] 推送到邮件 / webhook 能收到
- [ ] OpenCLI adapter 跑通：通过 `opencli xiaohongshu / x / zhihu` 等命令拿到结构化数据并入 Sigma context
- [ ] 用户反馈（点开 / 忽略）能影响下次推送内容（通过 memory + 显式偏好）
- [ ] Trace viewer 能打开 HTML 查看完整执行链路
- [ ] **V1 退出标准**：4 个核心场景全部端到端跑通，可作为完整项目 demo

---

## V1 = 0.6 完成

所有 milestone 退出标准达成时，V1 即达成。V1 是一个**完整可用的 personal AI assistant**：

- ✅ Chat + Task 双模式
- ✅ Tool / Skill / Agent 三层扩展
- ✅ Tool 生态：内置 + MCP + OpenCLI 三源（开放生态）
- ✅ Multi-agent 协作 + 三级回退
- ✅ Memory 三层 + 显式偏好学习
- ✅ RAG 多 index + 引用
- ✅ Context engineering + token budget
- ✅ Trace 全链路可观测
- ✅ Cost guard + 多 provider 路由
- ✅ 周期 task + 推送
- ✅ CLI 全功能
- ✅ 4 个核心场景全部跑通

---

## V2+（展望）

V1 之后按真实需求出现的顺序做，不预先承诺优先级：

- **Realtime 模式**：第三种交互模式（实时语音），OpenAI Realtime / Gemini Live middleware
- **Web UI**：React / Svelte 前端，走同一套 HTTP/SSE/WebSocket API
- **隐式 Self-improvement**：行为模式信号（重复问 / 转话题 / task 反馈）+ audio metadata（V4 Realtime）
- **Trace viewer 增强**：time-travel / replay / diff / 跨 session 对比
- **Agent / Skill 分发**：从 registry 下载别人的 agent / skill（含安全沙箱）
- **更多 LLM provider**：Qwen / Ollama / 本地模型
- **Eval 框架**：agent 行为质量评估

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

- **跟着 4 个核心场景走**——任何 milestone 如果不能让某个核心场景前进，就不做
- **学习价值优先**——选实现路径时优先选能学到东西的，不优先选最简的
- **不预先做 Port**——只在已有 ≥ 2 个真实需求时才做抽象
- **小步走**——每个 milestone 可独立 demo / 独立写博客复盘
- **先跑通再打磨**——每个场景先端到端跑通，再优化细节

## 相关文档

- [架构总览](architecture/overview.md)
- [设计决策日志](architecture/design-log.md) — 路线图变更的演化记录
- [贡献指南](../CONTRIBUTING.md)
