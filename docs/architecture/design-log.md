# Sigma 设计决策日志

> **这是一份活文档**，记录 Sigma 产品形态和架构的演化决策。新决策 append，旧决策不删（要看演化轨迹）。
>
> **status**：早期设计阶段，尚未开始编码。当前 `src/` 不存在。

---

## 1. 项目定位（最新理解）

**Sigma = 通用 personal AI assistant，对标 ChatGPT + Codex 集成体的本地版本。**

- **不是**：可插拔 AI Agent 框架（早期定位，已废弃）
- **不是**：Coding agent（短暂方向，已澄清不是）
- **是**：通用助手，覆盖日常对话 + 长任务执行 + 代码场景
- **差异化**：在 agent 内功上做透——context 管理、memory、RAG、trace、self-improvement

### 4 个锚定场景（决定 Phase 1 必须跑通的能力）

| # | 场景 | 模式 |
|---|---|---|
| 1 | 每天早上从社交媒体（小红书/X/知乎）筛选感兴趣内容并推送 | **task**（周期性） |
| 2 | 查询生猪期货数据（母猪存栏、仔猪出栏等），生成趋势图，判断供需 | **task**（一次性） |
| 3 | 多轮讨论一本书，最终总结笔记 | **chat** |
| 4 | Coding（写代码、改代码、跑代码） | **chat + task** |

---

## 2. 已锁定的决策

| # | 维度 | 决定 | 关键理由 |
|---|---|---|---|
| D-1 | **产品形态** | 通用 personal AI assistant | 不锁场景；agent 内功是差异化 |
| D-2 | **Chat / Task 关系** | **思路 C：双视图共存** | Chat 里 agent 可建议升级为 task；Task tab 也能直接新建；统一 task queue |
| D-3 | **交互形态** | **Phase 1 CLI → Phase 2 Web UI → Phase 3 Realtime** | 跳过 TUI；从 day 1 做 client/server 分离，core 走 HTTP/SSE/WebSocket |
| D-4 | **RAG** | 必做，是核心功能 | 场景 3、4 都需要；context engineering 的核心 |
| D-5 | **Graph engine** | **基于 LangGraph，不自己造**（路线 C） | 学习目标宽（runtime + RAG + memory + context），不能在单点卡死 |
| D-6 | **扩展粒度** | **F 方案：Tool + Skill + Agent 三层** | 三者正交，不是粒度递进 |
| D-7 | **Skill 形态** | **纯文件夹 + markdown，0 Python 接口** | Skill 本质是 prompt，对齐 Claude Code |
| D-8 | **Skill 加载** | **渐进式：metadata 进 system prompt，全文按需 `load_skill()` tool** | 防止 context 爆炸 |
| D-9 | **Agent 形态** | 用户可写完整 agent（继承类 / 定义 graph） | 进阶用户的扩展点 |
| D-10 | **对外 / 对内分离** | 扩展接口区分 Tool/Skill/Agent；runtime 内部统一为 LangGraph Node | 用户心智清晰；调度统一 |
| D-11 | **Multi-agent 路由** | **模式 2+3：Supervisor 自动 + @-mention 显式** | 双入口，对应 chat / task 双视图精神 |
| D-12 | **Sub-agent 沟通** | **三级回退** | 见下方详解 |
| D-13 | **Realtime 模式** | **第三种顶层交互模式（V4 起）** | 半透明 middleware；不 multi-agent；与 Chat / Task 平级 |
| D-14 | **Self-improvement 落地路径** | **V2 MVP（显式）→ V3 隐式 → V4 Realtime audio signal** | Realtime 是 self-improvement 反馈密度最高的场景，是天然落地点 |
| D-15 | **技术选型** | **纯 Python 后端** | I/O 密集型项目（99% 时间等 LLM API），Python 的 CPU 慢不在关键路径上；LangGraph Python >> JS 版；AI 生态是 Python 主场 |
| D-16 | **Roadmap 重构** | **V1 = 唯一完整交付目标，0.1→0.6 六个 milestone** | 旧 V0-V5 拆得太碎且不连续；新结构每个 milestone 独立可 demo，按场景解锁排序 |
| D-17 | **V1 范围** | **Chat + Task 双模式，CLI only；Realtime + Web UI 推 V2+** | 聚焦内核能力，不铺交互形态 |
| D-18 | **Self-improvement V1 范围** | **显式偏好 MVP only；隐式信号推 V2+** | 显式偏好工作量小、价值确定；隐式信号需要大量 heuristic，投入产出不确定 |
| D-19 | **Coding 提前到 0.2.5** | **新增 0.2.5 milestone，单 agent 形态完整跑通场景 4** | dogfood 红利早收，0.3 之后所有 milestone 都能用 Sigma 自己加速；单 agent coding 与 multi-agent 协作是两个独立设计空间，先各自跑通再组合 |
| D-20 | **Tool 生态策略：MCP + OpenCLI 双接入** | **0.5 接 MCP（协议化、对外开放），0.6 接 OpenCLI（复用 144 个 site adapter，社媒场景救场）** | 两者定位互补不冲突：MCP 是协议（Sigma 既消费也对外暴露），OpenCLI 是产品（登录态站点 / 桌面应用专精）；不二选一才能"开放生态" |
| D-21 | **Memory 定义扩展** | **Memory = agent 从交互中积累的一切有价值的认知**，包含 Semantic（事实/知识）、Episodic（经历）、Procedural（做事方式）三种类型 | 行业共识（Mem0/Zep/LangGraph）；原定义"只关于用户"过窄，遗漏了领域知识积累 |
| D-22 | **Domain 隔离** | **知识按领域隔离**：每个 domain = RAG（原始材料）+ Memory（积累认知），各自进化互不干扰；真正跨域的只有极少数 UI 偏好 | 不同场景的偏好/经验可能冲突（coding 要客观可靠 vs 讨论书要感性发散），混在一起有害 |
| D-23 | **知识复利三循环** | **Ingest（预理解）→ RAG 模块；Query 回写 → Context/RAG 模块；Lint（自检整理）→ Self-improvement 模块** | 借鉴 LLM-Wiki 的 AOT 思路，解决传统 RAG 的三个共性问题：推理结论不留存、全局理解缺失、知识不进化 |
| D-24 | **Self-improvement 扩展** | **从"学用户"扩展为"学用户 + 学知识"**：新增领域认知提取 + domain memory 一致性检查 + 过时标记 | 两者共享同一条 pipeline（提取 → 质量判断 → 持久化 → 召回），区别只在 signal 来源和存储位置 |
| D-25 | **Domain Memory 存储方向** | **SQLite 存结构化记录，content 字段为自然语言文本**；不用纯 Markdown，不用纯 KV | LLM 直接读 content 文本；metadata（来源、置信度、过时标记）走结构化查询；跟 Sigma 已有技术选型一致（session memory 已用 SQLite）。实现细节 0.3/0.4 落地时定 |

### D-12 详解：Sub-agent 三级回退

```
Sub-agent 遇到不确定 / 缺信息
   ↓
Level 1: sub-agent 自己尽力（合理默认 / 推理）
   ↓ 仍然不行
Level 2: bubble up 到主 agent，主 agent 用历史上下文代答
   ↓ 主 agent 也搞不定 / 需要外部资源（API key / 登录）
Level 3: bubble up 到用户
         · Chat mode：直接追问
         · Task mode：task 进入 paused 状态，等用户 resume
```

**两类卡住要区分**：
- **认知性卡住**（决策歧义）→ 走 L1 → L2 → L3
- **资源性卡住**（API key 缺失、登录态过期、文件不存在）→ 直接 L3

**主 agent 代答原则**：代答后必须在 trace / task summary / chat 里**显式说明**"我替你决定了 X"，保留用户 override 的机会。

### D-13 详解：Realtime 模式

**形态**：在 OpenAI Realtime / Gemini Live 等原生 realtime API 之上做 middleware：
- 会话前注入 memory + RAG → system prompt
- 会话中拦截 tool calls + 提取 audio signals
- 会话后沉淀 transcript → memory + self-improvement signal

**关键约束**：Realtime 模式**不支持 multi-agent 协作**——实时性要求（< 500ms 延迟）跟 sub-agent spawn 引入的延迟矛盾。Realtime 走"单 agent + 强 tool calling + 强 memory" 模式。

**Killer 场景**：语言陪练 / 心理陪伴 / 面试模拟 / 老人陪伴 / 辅导孩子作业。共性：原生 Realtime API 单独不能做好，必须长期记忆 + 个性化才有差异化价值。

**实现路径**：V4-V5。Phase 1-3 不做。

详见 [Realtime 模式](realtime-mode.md)。

### D-14 详解：Self-improvement 落地路径

**关键洞察**：Realtime 是 self-improvement 反馈密度最高的场景。文本 chat 一分钟 1-2 个有效信号；Realtime 一分钟 5-10 次打断 + 沉默 + 笑声等 audio signal——是天然落地点。

**路径**：
- **V2 MVP**：显式 signal（用户明说偏好）→ memory 写入闭环；用户可见 / 可编辑
- **V3 隐式**：行为模式 signal（重复问 / 转话题 / task 反馈）→ 周期 task 兴趣模型
- **V4 Realtime**：audio metadata signal（打断 / 沉默 / 笑声）→ 完整闭环可观察
- **V5 跨 session**：进步轨迹 / 长期模式识别

**设计原则**：保守 > 激进；可解释；可审计；可回退；用户可关闭。**不做模型层学习**（不微调），所有 self-improvement 通过 memory + prompt + 推送过滤实现。

详见 [Self-improvement](self-improvement.md)。

---

## 3. Graph engine 接管的横切能力（D-5 的延伸）

**Sigma 自己长的肌肉**（不依赖 LangGraph 提供的）：

| # | 能力 | 实现方式 |
|---|---|---|
| 1 | **Trace** | 挂 LangGraph 的 callback / `astream_events`，输出 JSONL + 配本地 HTML viewer |
| 2 | **Cost guard** | 自己的 LLMPort 包装算 token；超预算用 LangGraph interrupt 拦截 |
| 3 | **统一 Streaming 协议** | 把 LangGraph event stream 映射到 Sigma 的 `AgentChunk` |
| 4 | **可替换的 RAG / Memory / Context / Tools** | 走 Port + Adapter，被 Node 内部消费 |
| 5 | **Voice pipeline**（可选，后期） | 自己的 runtime 模块，调 STT/TTS Port |

**直接用 LangGraph 提供的**（不重造）：
- ✅ Graph 编排 / state machine
- ✅ Checkpointer 接口（用 SqliteSaver / PostgresSaver）
- ✅ BSP 并行 / fan-out / fan-in
- ✅ Subgraph 嵌套
- ✅ Cancellation / interrupt 机制

**Phase 1 明确不做**：
- ❌ Hot reload adapter
- ❌ 自己造 Graph engine
- ❌ Multi-agent orchestration 内核（用 LangGraph supervisor 即可）

---

## 4. 关键设计洞察

### 4.1 Skill ≠ Agent（D-7 的根源）

**洞察**：Skill 本质是 prompt 的延迟加载，不是 reasoning loop。

| | Tool | Skill | Agent |
|---|---|---|---|
| 扩展什么 | 行为能力（能做什么） | 知识/方法论（怎么做） | 执行单元（谁来做） |
| LLM 调用次数 | 0 | 0（被注入到调用方 prompt） | 多次（自己有 loop） |
| 状态机 | 无 | 无 | 有 |
| 写起来 | Python 函数 | markdown 文件 | Python 类 + graph 定义 |

**三者正交**：一个 agent 内部可以同时引用 N 个 tool 和 M 个 skill。这是 Sigma 跟很多产品最大的差异——不是粒度递进，是不同维度的扩展。

### 4.2 对外 / 对内分离（D-10 的根源）

> "都是 input/output 的 node" 这个观察在**调度层是对的**，但在**扩展接口层是错的**。

```
对外（产品 / 扩展接口）：     区分 Tool / Skill / Agent
                          ↓ 各自有不同的注册方式、生命周期
                          ↓ 各自有不同的语义保证
对内（runtime / 调度）：      统一抽象成 LangGraph Node
                          ↓ 统一 trace / checkpoint
```

不区分会出三个问题：扩展接口模糊（用户不知道写啥）、调用语法塌陷（LLM 看到几十个"node"会傻）、trace/cost/timeout 一视同仁（粒度差几个数量级混在一起）。

### 4.3 RAG 不是单一索引，是分层多 index（场景拆解的发现）

| 场景 | RAG 索引什么 | Memory 记什么 |
|---|---|---|
| 1 社媒 | 不用 RAG（实时拉取） | 用户兴趣偏好（从反馈学） |
| 2 期货 | 行业基础知识（可选） | 关注的指标 / 历史判断 |
| 3 讨论书 | 书的全文 | 历次对话要点 |
| 4 Coding | 代码库 | 项目特性 / 用户代码风格 |

**结论**：RAG 是"按 task / 按 session 配置的多个 index"。Memory 是"全局偏好 / session 上下文 / task 临时 state"分层。

### 4.4 Sub-agent 沟通是 multi-agent 最难的设计点

不是技术难，是**语义难**——chat / task 两种模式下"等用户"的代价完全不同：
- Chat：用户在线，等无所谓
- Task：用户不在线，等 = 卡死

D-12 三级回退是**目前看来唯一优雅统一两种场景的方案**。它的副产物：task 系统天然要支持 pause/resume。

### 4.5 Realtime 是 self-improvement 的天然落地（D-13 + D-14 联动）

**洞察**：之前 self-improvement 一直是抽象的（用户 V0 提过但没具体形态）。加入 Realtime 后立刻具体化——

```
文本 chat：1 分钟 1-2 次有效信号（用户明说 / 重复问）
Realtime ：1 分钟 5-10 次 audio signal（打断 / 沉默 / 笑声）+ 文本 signal
```

**密度差 5-10 倍**，且 Realtime 引入了文本不存在的维度（语气 / 节奏 / 情绪）。Self-improvement 在 Realtime 模式下能看到效果，文本模式效果较弱。

**所以 D-14 把 self-improvement 的完整落地放在 V4**——跟 Realtime 一起做，互为依赖。

### 4.6 三种交互模式的本质差异（D-13 引出）

之前以为只有 Chat / Task 两种交互。但 Realtime 加入后看清——这三种是**根本不同的范式**，不是同一种东西的变体：

| | Chat | Task | Realtime |
|---|---|---|---|
| 节奏 | 实时往返（文本） | 后台异步 | 实时往返（语音） |
| 延迟 | 秒级 | 无 | < 500ms |
| Multi-agent | ✅ | ✅ | ❌（实时性约束） |
| 三级回退 L3 | 弹追问 | task pause | 直接打断对话 |
| 内核位置 | Chat Engine | Task Engine | Realtime Middleware |

**结论**：三模式是平级的。Realtime 不是 Chat 加语音输入，也不是 Task 的实时版本——它有自己的内核流程（前注入/中拦截/后沉淀），跟 Chat / Task 共享底层 memory / RAG / agent，但不共享调度引擎。

### 4.7 Coding 提前到 0.2.5 的理由（D-19 详解）

**触发问题**：原 roadmap 把 coding 放在 0.5（和 multi-agent / skill / 多 provider 路由 / 数据分析打包）。在 0.5 之前 Sigma 帮不了自己写自己——dogfood 红利全部错过。

**核心论点**：dogfood 红利**复利**——0.2.5 完成后，0.3 / 0.4 / 0.5 / 0.6 四个 milestone 都能让 Sigma 自己参与开发；越早 dogfood，越早从用户视角发现 Sigma 的设计缺陷。

**为什么是单 agent 形态而不是直接 coder sub-agent**：
- 单 agent + 强 tool 已经能覆盖 80% coding 体验（参考 Claude Code / Cursor 早期形态）
- multi-agent 路由（supervisor + BlockedException + sub-agent metadata）是独立的设计空间，跟 coding loop 解耦
- 0.5 接 supervisor 只需把 0.2.5 的单 agent coding 包成 sub-agent，**不重写**

**对其他 milestone 的影响**：
- 0.5 不再从零做 coder agent，而是把 0.2.5 成果包装为 sub-agent + 接入 supervisor
- 0.5 场景 4 的退出标准从"端到端跑通"变为"multi-agent 协作增强"
- 0.3 / 0.4 / 0.6 不变

**风险**：
- 0.5 接 supervisor 时，coder 接口可能要小幅调整（影响 1-2 周）——但远小于"coder 在 0.5 才从零做"的代价
- 0.2.5 没有 Memory / RAG，coding 体验有上限（不能跨大量文件重构）——可接受，0.4 RAG 后会自然增强

**度量 dogfood 是否成功**：0.2.5 退出标准里要求"至少 1 次用 Sigma 写 Sigma 的 PR 合入 main"——以可验证的产出锚定 milestone。

### 4.8 Tool 生态策略：MCP + OpenCLI 双接入（D-20 详解）

**问题**：Sigma 自己写所有 tool 不现实——社媒抓取、SaaS API 集成、数据库连接、桌面应用控制每一个都是独立的工程量。要么自己造，要么接生态。

**两个候选生态**：

| | MCP | OpenCLI |
|---|---|---|
| 形态 | 协议（JSON-RPC over stdio / HTTP+SSE） | npm 产品（CLI + Chrome 扩展） |
| 强项 | filesystem / DB / SaaS API（几百个 server 现成） | 登录态网站（144 site adapter）+ 桌面 Electron 应用 |
| 弱项 | 网站抓取（很少有人写）；中文站点几乎没有 | 协议不通用；只能 shell 调用 |
| 跨客户端 | ✅ 是协议，Claude Desktop / Cursor / 自研 agent 都通 | ⚠️ 任何能 shell 的客户端都能用，但不是协议 |
| Sigma 接入成本 | 中（实现 MCP client） | 低（subprocess + JSON） |
| Sigma 对外开放 | ✅ Sigma 也可作为 MCP server 暴露给其他 agent | ❌ 反向不通 |

**关键洞察**：两者定位**互补**，不是替代关系。

- **MCP 给的是"协议化的开放"**——Sigma 既能消费别人的 server，也能把自己的能力暴露成 server 给 Claude Desktop / Cursor / 自研 agent 调用。这是"开放生态"的核心。
- **OpenCLI 给的是"登录态站点的零成本"**——它的 144 个 adapter 复用 Chrome 已登录态，是中文社媒和桌面应用场景几乎不可替代的捷径。

**所以决策是双接入**：
- 0.5 接 MCP（multi-agent 落地时刚好需要 researcher / analyst 调外部 SaaS）
- 0.6 接 OpenCLI（场景 1 社媒推送的实现路径——不自己写抓取，直接调 `opencli xiaohongshu` 等命令）

**与 ToolPort 的关系**：
- ToolPort 不变。MCP / OpenCLI / 内置 tool 都通过各自的 adapter 实现 ToolPort，在 registry 里平等存在
- LLM 看到的是统一的 tool 列表，不感知 tool 来源；调度由 Sigma runtime 决定

**对外暴露的安排**（V2+ 任务）：把 Sigma 自己的 tool / agent 通过 MCP server 协议暴露——这是把 Sigma 嵌入更大 AI 生态的钥匙。

**风险**：
- OpenCLI 引入 Node.js 运行时依赖（subprocess 调用，不污染 Python 包），全自动化部署场景（无 Chrome 的服务器）需要降级策略
- MCP 客户端实现要做好（stdio / SSE / 流式 / 错误传播），是 0.5 工作量的实质部分
- OpenCLI adapter 维护风险（site DOM 变化）由其社区承担，Sigma 不背

### 4.9 Sub-agent 上下文与权限问题（真实事故）

**触发事件**（2026-05-13）：在用 Claude Code 开发 Sigma 时，主 agent 派了一个"只读调研"子 agent 去分析 RAG 模块现状。该子 agent 自作主张创建了 3 个新文件并直接 commit 到 main 分支——违反了 CLAUDE.md 里"禁止在 main 上直接 commit"的硬性规则。

**表面原因**：子 agent 没读到项目规则，且拥有超出任务所需的写权限。

**更深的问题**：这不只是"约束没传递"或"权限没收窄"这么简单。它暴露的是 multi-agent 系统中一组相互关联的设计难题——至少包括：

- Sub-agent 启动时的上下文边界在哪？传多了 token 爆炸，传少了规则丢失
- "权限"在 LLM agent 世界里的粒度该怎么定义？tool 级？操作级？目录级？语义级？
- Prompt 约束 vs runtime 强制 vs 基础设施兜底——三者的分工和成本权衡
- 当 agent 的"意图理解"出错时（它认为创建文档是在帮忙），传统 RBAC 式的权限模型是否足够？
- Multi-agent 场景下，"信任"是二元的还是分级的？

**结论**：这些问题在 0.5（multi-agent 落地）时必须深入研究，不能只打表面补丁。当前只记录信号，不给方案。

### 4.10 传统 RAG "必死"及知识复利模型（D-21~D-24 的推导）

**触发**（2026-05-13）：调研 Karpathy 的 LLM-Wiki（agent-native 知识库范式），发现它对传统 RAG 的批判精准命中 Sigma 的四个核心场景。

#### 传统 RAG 在 Sigma 核心场景中的共性问题

逐场景分析后，提炼出三个所有场景（3/4/2）都踩到的问题：

| # | 问题 | 场景 3（讨论书） | 场景 4（Coding） | 场景 2（期货） |
|---|------|-----------------|-----------------|---------------|
| 1 | **推理结论不留存** | 主题分析用完就丢 | 架构理解用完就丢 | 供需模型用完就丢 |
| 2 | **全局理解缺失** | 跨章节/跨书关联检索不到 | 跨模块隐式依赖检索不到 | 跨数据周期的趋势关联检索不到 |
| 3 | **知识不进化** | 讨论越深索引越不够用 | 代码在变索引在腐烂 | 新数据来了旧分析不更新 |

根本问题：**agent 的使用成本随时间线性增长（每次重新推理），而价值不会复合增长。**

#### 洞察 1：缺失的是"第三种认知"

Sigma 已有两种会积累的东西：

- **Memory**（原定义）：积累对用户的理解 → 越来越懂"你"
- **RAG index**：存放原始素材 → 是存量，不是复利

缺失的是：**agent 在使用过程中产生的领域认知**——从原始材料和对话交互中推导出的理解、结论、模型。它不是原始事实（RAG），也不是用户偏好（Memory 旧定义），是两者交互的产物。

#### 洞察 2：Memory 的定义画窄了

调研行业实践（Mem0 / Zep / LangGraph / Claude Code / Cursor）后发现：**行业里的 Memory 从来不只是"关于用户"的**。

认知科学的三层分类已成共识：

- **Semantic Memory**：事实与知识（用户偏好 **+** 领域知识）
- **Episodic Memory**：过去的经历（"上次改接口踩了什么坑"）
- **Procedural Memory**：做事方式（"分析期货数据的步骤是…"）

典型证据：Claude Code 的 Auto Memory 存的就是**项目理解和 debugging 洞察**——这正是我们说的"第三种认知"，但 Claude Code 没有另开模块，就放在 memory 体系里。

**结论**：所谓"第三种东西"不需要新模块——**它就是 Memory，只是我们之前把 Memory 定义画窄了。**

#### 洞察 3：知识需要按领域隔离

不同场景的 memory 可能直接冲突：
- coding 要求客观、可靠、精确
- 讨论书要求发散、感性、联想
- 两者的经验也永远不互通——coding 里学到的调试技巧用不到文学讨论里

如果混在同一个"global memory"里，检索时互相干扰，甚至产生矛盾指令。

**结论**：Memory 有两个正交的维度——**生命周期**（session / task / 长期）和**作用域**（global / domain 级）。大多数有价值的认知都属于某个具体领域。真正跨域的只有语言偏好、回复风格等极少数 UI 层偏好。

每个领域（domain）是一个完整的知识上下文：RAG（原始材料）+ Memory（积累认知），绑在一起，共同进化。

#### 洞察 4：知识复利的三循环归属

借鉴 LLM-Wiki 的 Ingest / Query / Lint 三循环，但映射到 Sigma 的模块分工：

| 循环 | Sigma 负责模块 | 触发时机 | token 成本模型 |
|------|--------------|---------|--------------|
| **Ingest（预理解）** | RAG 模块 | 用户添加素材时 | 一次性，用户主动触发 |
| **Query 回写** | Context/RAG 模块 | 深度分析后顺手沉淀 | 边际≈零（推理本来就做了） |
| **Lint（自检整理）** | Self-improvement 模块 | session 后 / 周期性 | 后台成本，需 cost 控制 |

Self-improvement 模块从"学用户"扩展为"学用户 + 学知识"——两者共享 signal 提取 → 质量判断 → 持久化 → 召回 这条 pipeline，区别只在 signal 来源和存储位置。

---

## 5. 未决问题（已知的下一步）

按当前讨论的脉络，接下来要解决：

| # | 待定 | 重要性 |
|---|---|---|
| U-1 | **Task 系统的状态机和持久化设计**（pause/resume 怎么落地） | 🔴 高 |
| U-2 | **Master Agent vs Supervisor 的关系**（同一 LLM 调用 vs 独立 node） | 🟡 中 |
| U-3 | **Agent metadata 协议**（name / description / 触发词如何声明） | 🟡 中 |
| U-4 | **结构化的 sub-agent 错误协议**（reason 分类如何标准化） | 🟡 中 |
| U-5 | ~~RAG 系统的具体形态~~ → **方向已由 D-22/D-23 锁定**（domain 模型 + 三循环），实现细节在 0.4 落地时定 | ✅ |
| U-6 | ~~Memory 分层设计~~ → **方向已由 D-21/D-22 锁定**（三种类型 × domain 隔离），实现细节在 0.3 落地时定 | ✅ |
| U-7 | **Trace 协议和 viewer 形态**（什么时候做） | 🟡 中 |
| U-8 | ~~Self-improvement 形态~~ → **已升级为 D-14**（路径明确） | ✅ |
| U-9 | **推送通道**（场景 1 引出的产品决策） | 🟢 低（实现期再定） |
| U-10 | **是否抽 RealtimePort**（多 provider 抽象） | 🟢 低（V5 接 Gemini 时定） |
| U-11 | **Realtime session 跟 Chat session 的切换**（聊一半切语音） | 🟢 低（V5+） |
| U-12 | **Sub-agent 上下文与权限的深层设计**（见 § 4.9，0.5 时深挖） | 🔴 高 |
| U-13 | **Domain 的生命周期管理**（创建 / 归档 / 跨 domain 引用 / domain 粒度该多细） | 🟡 中（0.3~0.4 落地时定） |
| U-14 | **Query 回写的质量判断**（什么结论值得沉淀、置信度如何标注、谁确认） | 🟡 中（0.4 落地时定） |
| U-15 | **Ingest 预理解的粒度**（编译出什么形态的产物、成本权衡） | 🟡 中（0.4 落地时定） |

---

## 6. 已废弃的方向（演化历史）

记下来防止后面再绕回去。

### 6.1 早期"可插拔 AI Agent 框架"定位

**已废弃**。原因：
- 是 framework 叙事，但代码长得像 library，定位错位
- 7 个 Port 大部分是 premature abstraction（很多只有 1 个实现）
- 三层抽象（Graph/Node/Port）对单人项目是过度设计
- "未来可拆 gRPC" 的约束在为不存在的需求买单

### 6.2 Coding agent 定位

**短暂出现，已澄清不是**。Sigma 是通用助手，不锁 coding 场景。

### 6.3 自己造 Graph engine

**已废弃**。改走"基于 LangGraph"。原因：学习目标太宽（runtime + RAG + memory + context），自己造 engine 会把所有时间烧在 engine 上。

### 6.4 HTML 文档

**已废弃**。Markdown 够用，HTML 折腾性价比低。

### 6.5 docs/restructure-v2 重写（已完成）

基于本文件 § 1-5 的决策，所有 docs 已统一重写。完成于 docs/restructure-v2 分支：

**根目录**：
- ✅ `CLAUDE.md`：定位 / 项目结构 / 文档索引全部对齐新方向
- ✅ `README.md`：定位改为通用 AI 助手；4 核心场景列出；扩展三层介绍
- ✅ `CONTRIBUTING.md`：架构红线砍剩 3 条；推荐贡献方向按门槛分层

**架构文档**（`docs/architecture/`）：
- ✅ `overview.md`：完全重写（产品形态 / 内核 / 三层扩展 / multi-agent / Realtime / Self-improvement）
- ✅ `chat-task-modes.md`：异步双视图（含和 Realtime 的关系）
- ✅ `multi-agent.md`：Supervisor + @-mention、三级回退、BlockedException
- ✅ `realtime-mode.md`：**新增**——第三种顶层模式；半透明 middleware；killer 场景
- ✅ `self-improvement.md`：**新增**——Signal 来源 / 反馈路由 / 设计原则
- ✅ `ports-and-adapters.md`：Port 砍到 4 个（LLM / Tools / Tracer / Checkpointer）；废弃 ContextBuilder/AgentRuntime/Retrieval/Memory Port
- ✅ `dependency-rules.md`：放宽内核模块互引；保留域目录互不依赖
- ❌ `runtime-modes.md`：**删除**（Cascade/Realtime 概念已废弃，新 Realtime 模式是不同的概念）

**模块文档**（`docs/modules/`）：
- ✅ `agent/`：Master / Supervisor / Sub-agent 框架；三级回退
- ✅ `skill/`：纯 markdown、渐进式加载
- ✅ `task/`：状态机、queue、pause/resume
- ✅ `chat/`：session、升级建议、回流
- ✅ `realtime/`：**新增**——Provider adapter / Middleware / Audio 处理
- ✅ `improvement/`：**新增**——Signal extractor / Pipeline / Audit
- ✅ `context/`：context engineering 多源拼装
- ✅ `rag/`：多 index 分层
- ✅ `memory/`：global / session / task 三层 + Realtime session 特殊处理
- ✅ `tools/`：Python 函数 + MCP
- ✅ `llm/`：多 provider + cost-aware 路由
- ✅ `trace/`：JSONL + viewer + replay
- ❌ `voice/`：**删除**（Realtime 模式取代了 voice 在 V0-V4 范围内的角色）

**其他**：
- ✅ `docs/index.md`：导航更新
- ✅ `docs/roadmap.md`：V0~V5+ 路线图按 4 核心场景重排
- ✅ `docs/guides/`：3 份指南全部重写

---

## 7. 下一步行动

**已完成**：
1. ✅ 基于本文件重写所有 docs（docs/restructure-v2）
2. ✅ 技术选型确认：纯 Python 后端（D-15）
3. ✅ Roadmap 重构为 0.1→0.6 milestone 结构（D-16）

**接下来**：
1. **建 `src/` 骨架**（0.1 阶段：Hello Agent 跑通）
2. **继续推进未决问题**：U-1 → U-5/U-6 → U-2（撞墙时再解决）
3. **每个新决定记到 § 2**
4. **每个关键洞察记到 § 4**
5. **不在 0.1 之前过度设计**——先把骨架跑通，再撞墙再加东西
