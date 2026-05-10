# Realtime 模式

> Sigma 的第三种顶层交互模式——**实时语音对话 + 长期记忆 + 持续学习**。
>
> 在 ChatGPT Realtime / Gemini Live 等原生 realtime API 之上，长出 Sigma 自己的 memory / RAG / self-improvement 层。

---

## 1. 是什么 / 不是什么

### 是

- **实时语音双向对话**——模型直接消费/生成音频 token，能感知语气、口音、停顿、笑声
- **延迟 < 500ms**——接近真人对话节奏
- **可打断（barge-in）**——用户中途说话立即停止模型输出
- **Sigma 半透明 middleware**——会话前注入 memory，会话中拦截 tool call，会话后沉淀知识

### 不是

- ❌ 不是 STT → LLM → TTS 的级联（那种丢失所有语气信息）
- ❌ 不是 Chat 模式加个语音输入（节奏、能力、UI 完全不同）
- ❌ 不是给所有场景的——多 agent 协作、复杂 reasoning 不适合实时

---

## 2. 三种模式对比

| | Chat | Task | **Realtime** |
|---|---|---|---|
| 节奏 | 实时往返（文本） | 后台异步 | 实时往返（语音） |
| 用户在线 | 是 | 否 | **必须** |
| 延迟要求 | 秒级 | 无 | **< 500ms** |
| Multi-agent | 支持 | 支持 | **不直接支持** |
| 三级回退 L3 | 弹追问 | task pause | **直接打断对话** |
| Memory / RAG | 内核 context 拼装 | 内核 context 拼装 | **会话前注入 + 会话中 tool call** |
| 中途可打断 | 否 | 否 | **是（barge-in）** |
| 典型场景 | 问答、讨论 | 长任务、周期任务 | 陪练、陪伴、辅导 |

**Realtime 是真正不同的交互范式**，不是 Chat 的扩展。所以是**顶层模式**，跟 Chat / Task 平级。

---

## 3. Sigma 在 Realtime 中的角色：半透明 Middleware

```
┌─── 会话前 ──────────────────────────────────────────┐
│  Sigma 准备：                                          │
│  - Recall global memory（用户偏好 / 背景）              │
│  - Recall session memory（之前聊过什么）                │
│  - 可选 RAG（教材 / 资料 / 题库）                       │
│  - 拼装成 Realtime session 的 system prompt             │
│  - 配置 session 参数（语气 / 速度 / 可调用 tool）         │
└─────────────────────────────────────────────────────┘
                         ↓
┌─── 会话中 ──────────────────────────────────────────┐
│                                                       │
│  User ←──── WebSocket（双向音频流）────→ Realtime Model │
│                       ↓                                 │
│              ┌────────────────────┐                    │
│              │ Sigma Middleware   │                    │
│              │                    │                    │
│              │ • Transcript 实时记录                    │
│              │ • Tool call 拦截执行：                    │
│              │   - recall_memory()                     │
│              │   - search_knowledge() (RAG)            │
│              │   - create_task()                       │
│              │   - save_note()                         │
│              │ • 实时 signal 提取（打断 / 沉默 / 重做）   │
│              │ • 结果注入回 session                      │
│              └────────────────────┘                    │
└─────────────────────────────────────────────────────┘
                         ↓
┌─── 会话后 ──────────────────────────────────────────┐
│  Sigma 沉淀：                                          │
│  - Transcript → Memory extraction                      │
│  - 关键决策 / 偏好 → Global memory                     │
│  - 互动 signal → Self-improvement signal               │
│  - 可选生成会话报告（学习进度 / 互动质量）                │
└─────────────────────────────────────────────────────┘
```

**"半透明"** = 用户跟模型直接对话，但 Sigma 的存在用户能感知到（看 transcript / 看记忆提取结果 / 控制开关）——不是完全隐身。

---

## 4. Killer 场景

Realtime + Sigma 组合的杀手场景（这些**单纯 ChatGPT Realtime 都做不好**）：

### 4.1 语言学习陪练

```
ChatGPT Realtime alone：每次都从零开始；不记得你的薄弱点；不知道你昨天练了什么
Sigma 加上：
  · 记得：发音弱点（"r/l 经常混"）、词汇错误模式、上次练习内容
  · RAG：教材、词汇表、语法规则
  · Self-improvement：根据"用户哪些话经常需要重说"调整难度
```

### 4.2 心理陪伴

```
ChatGPT Realtime alone：不知道你是谁；不懂你的背景
Sigma 加上：
  · 记得：家庭情况、最近压力源、之前提过的人和事
  · 风格：从历史对话学到"你需要的是倾听还是建议"
  · 隐私：本地存储，不上云
```

### 4.3 面试模拟

```
ChatGPT Realtime alone：每次面试体验雷同
Sigma 加上：
  · 历史表现：上次的弱点 / 进步轨迹
  · RAG：目标公司题库、岗位 JD、行业知识
  · Self-improvement：从用户回答中学"哪些回应方式让用户表现更好"
```

### 4.4 老人陪伴

```
ChatGPT Realtime alone：模型不知道老人的健康/家庭/方言
Sigma 加上：
  · 长期 memory：健康状况、家庭成员、爱好、方言偏好
  · 定时 task：主动问候、用药提醒（task mode 触发 realtime 会话）
  · RAG：健康知识、用药指南
```

### 4.5 辅导孩子作业

```
ChatGPT Realtime alone：不知道孩子知识水平
Sigma 加上：
  · Memory：孩子的薄弱知识点积累、学习风格
  · RAG：教材、题目库
  · Task：定时复习、错题归纳
```

**共性模式**：

```
Realtime API（裸大脑）
  +
Sigma 三层包裹：
  1. 会话前：Memory + RAG → System Prompt 注入
  2. 会话中：Tool 执行（RAG / Memory / Task）+ Signal 提取
  3. 会话后：Transcript → 知识沉淀 + Self-improvement
```

---

## 5. 三级回退在 Realtime 中的特殊处理

详见 [Multi-agent § 4](multi-agent.md#4-三级回退sub-agent-沟通问题)，但 Realtime 模式有特殊性：

| Level | Chat / Task | Realtime |
|---|---|---|
| L1 自决 | 同 | 同 |
| L2 主 agent 代答 | 同 | **几乎不用**（Realtime 通常单 agent） |
| L3 升级用户 | Chat 弹追问 / Task pause | **直接在对话里问"我能问一下吗?"** |

**Realtime 没有 multi-agent 不是 bug，是设计约束**——multi-agent 协作引入的延迟和复杂度，跟实时对话的延迟要求矛盾。Realtime 模式下 Sigma 用"单 agent + 强 tool calling + 强 memory/RAG" 的形态。

---

## 6. 会话中可调用的内置 tool

Realtime session 启动时声明给 Realtime API 的 tool 清单：

```python
realtime_tools = [
    # Memory 类
    "recall_memory",          # 临时召回某段记忆
    "save_note",              # 用户说了重要的事，立即存
    
    # 知识类
    "search_knowledge",       # 查 RAG
    
    # 任务类
    "create_task",            # 起个后台 task（"帮我把刚才说的整理成笔记"）
    "schedule_reminder",      # 定时提醒
    
    # 控制类
    "switch_topic",           # 显式切换话题
    "pause_session",          # 让用户暂停（处理实事再回来）
]
```

Tool 的执行延迟必须 < 200ms（不能阻塞实时对话）——所以**重的操作走 task mode**，realtime 中只做轻量调用。

---

## 7. Self-improvement 的天然落地

**Realtime 是 Sigma self-improvement 最容易看到效果的场景**——因为反馈密度高（每分钟几十次互动信号）。

详见 [Self-improvement](self-improvement.md)。

主要 signal：

| Signal | 来源 | 反馈到 |
|---|---|---|
| 用户打断 | Audio metadata | "我话太多" → 调短回复偏好 |
| "再来一次" / "重说" | Transcript | 上次回答不够好 → memory |
| 显式偏好（"以后用更专业的术语"） | Transcript | global memory |
| 沉默时长 | Audio metadata | 是否在思考 / 是否走神 |
| 笑声 / 叹息 | Audio metadata | 互动质量 signal |
| 主动延续 / 转话题 | Transcript | 兴趣偏好 |

---

## 8. 实现路径（V4-V5）

**Phase 1（V0-V3）不做** —— 等 memory / RAG / trace 成熟后再上。

### V4：Realtime MVP

- 接 1 个 Realtime provider（OpenAI Realtime 优先）
- WebSocket session 管理
- 基础 middleware：会话前注入 memory + 会话后存 transcript
- CLI: `sigma realtime start`
- Web UI 加 Realtime 视图（transcript 实时展示）
- 1-2 个杀手场景 demo（推荐：语言陪练 / 心理陪伴）

### V5：Realtime 完整版

- 第二个 Provider 接入（Gemini Live / 其他）
- 完整 self-improvement 闭环
- 会话中的 tool calling 完整支持
- Audio metadata signal 提取
- 跨 session 学习（"这周用户进步轨迹"）

---

## 9. 技术风险

| 风险 | 应对 |
|---|---|
| WebSocket 协议跟现有 HTTP/SSE 不同 | Realtime 走独立的 server endpoint，跟现有架构隔离 |
| 音频流 buffer / barge-in 处理复杂 | 用 provider SDK 处理低层；Sigma 只做 middleware |
| 内核同步操作（memory / RAG）要 < 200ms | 缓存预热；后台预测性查询；超时降级（不阻塞对话） |
| Provider API 形态差异大（OpenAI vs Gemini） | 抽 `RealtimePort`，但**只在确认有 ≥2 个 provider 真要支持时才抽**（避免 premature abstraction） |
| 成本是文本的 5-10 倍 | Cost guard 在 Realtime 模式有更严格的 budget 默认值 |
| 用户期望延迟 < 500ms，任何阻塞都很明显 | Trace 必须能定位每一跳的延迟 |

---

## 10. CLI 映射（V4）

```bash
# 起 realtime 会话
sigma realtime start                          # 默认 provider / 默认 system prompt
sigma realtime start --agent language-coach   # 用某个 sub-agent 配置（system prompt + tool 子集）
sigma realtime start --rag textbook-zh        # 加载特定 RAG index 用于 query

# 历史
sigma realtime list                           # 历史 session 列表
sigma realtime view <session_id>              # 看 transcript
sigma realtime replay <session_id>            # 不真听，看 trace 还原
```

---

## 11. 与其他模块的关系

| 模块 | Realtime 的依赖 |
|---|---|
| [Memory](../modules/memory/) | 会话前注入 + 会话后提取（核心依赖） |
| [RAG](../modules/rag/) | 会话中按需 query（可选） |
| [Tools](../modules/tools/) | 会话中可调用的轻量 tool 子集 |
| [Skill](../modules/skill/) | Realtime session 也能 load_skill 加载方法论（如"语言陪练方法论"） |
| [Agent](../modules/agent/) | "Realtime sub-agent" 是一种特殊 agent，配置 realtime session 参数 |
| [Trace](../modules/trace/) | 每次 Realtime session 都进 trace，含 audio metadata signal |
| [Task](../modules/task/) | 会话中可创建后台 task；周期 task 可触发 realtime 会话（老人陪伴定时问候） |

---

## 12. 已废弃的方向

### ❌ 旧文档的 "Cascade vs Realtime" 双模式

之前文档把 Cascade（STT → LLM → TTS）和 Realtime（原生音频）作为 voice 模块的两种选择。**已废弃**：

- Cascade 不在 Sigma 主线（Voice 模块整体后置到 V5+）
- Realtime 现在是 Sigma 的**顶层模式**，不是 voice 的子模式

新的关系：
- Realtime 模式 = Sigma 顶层交互模式（chat / task / realtime）
- Voice 模块 = 仅在 Cascade 场景需要（V5+ 重新评估）

---

## 13. 未决问题

| 问题 | 状态 |
|---|---|
| 是否抽 RealtimePort（多 provider 抽象） | V4 落地时定 |
| Realtime session 跟 Chat session 是否能合并 / 切换（聊一半切语音） | V5+ |
| Audio metadata signal 提取的具体协议 | V4 |
| Realtime 的 cost guard 策略（更严格的默认值） | V4 |
| Realtime + Multi-agent 是否可能（高级 supervisor 异步派 task）| V5+ |

---

## 14. 相关文档

- [架构总览](overview.md)
- [Chat / Task 模式](chat-task-modes.md) — 另两种模式
- [Self-improvement](self-improvement.md) — Realtime 的最佳落地场景
- [Memory 模块](../modules/memory/) — 跨 realtime session 的记忆
- [Realtime 模块](../modules/realtime/) — 实现细节
- [设计决策日志](design-log.md)
