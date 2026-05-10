# Chat / Task 模式

> Sigma 的两种异步交互模式：**Chat（对话流）** 和 **Task（后台任务）**。两者共存、互通，统一 task queue。
>
> Sigma 还有第三种模式 **Realtime**（实时语音），是独立的同步模式，详见 [Realtime 模式](realtime-mode.md)。本文档只讲 Chat / Task。

---

## 1. 双视图概览

```
┌─────────────────────────┐    ┌──────────────────────────┐
│  Chat 视图              │    │  Task 视图                │
├─────────────────────────┤    ├──────────────────────────┤
│  · 实时对话              │    │  · 任务列表               │
│  · 问答 / 工具调用       │    │  · 进度 / artifact / log  │
│  · 短交互               │    │  · 后台执行               │
│  · Agent 可建议升级      │    │  · 周期 / 一次性          │
│    为 task               │    │                          │
└────────────┬────────────┘    └────────────┬─────────────┘
             │                              │
             └──────────────┬───────────────┘
                            ▼
                    统一 Task Queue
```

**关键原则**：

1. **Chat 是默认模式**——用户进来直接聊
2. **Task 是显式模式**——长任务、周期性任务、需要后台跑的事
3. **两条入口最终汇入同一套执行机制**——只是路由方式不同

---

## 2. Task 起源的两条路

### 2.1 Chat 升级路径（agent 主动建议）

```
User: 帮我把这个 repo 所有 README 翻译成英文，保持原格式
  ↓
Master Agent 判断：这是长任务（多文件 / 多次 LLM 调用）
  ↓
回复用户：「这看起来要花一阵，要不要起个后台任务？我估计 X 分钟」
  ↓
User: 好
  ↓
Sigma 创建 Task #N，进入 task queue，开始后台执行
  ↓
Chat 里出现一张 task 卡片（可点击跳转 Task 视图）
```

**判断标准**（agent 自己决定，不写死规则）：
- 预计执行时间 / token 数
- 是否涉及大量文件操作
- 是否需要多步骤推理
- 是否需要"长时间无人值守"

### 2.2 Task tab 直接新建（用户显式）

```
User 在 Task 视图：[+ New Task]
  → 输入 task description
  → 选择 schedule（一次性 / 每日 / cron 表达式）
  → 选择目标 agent（默认 Master，可 @-mention 指定）
  → 提交
  ↓
进入 task queue
```

**适合**：用户已经知道自己要起 task（典型：周期性任务，不可能每天都到 chat 里聊一遍）。

### 2.3 两条路汇入同一处

```python
# 不论从哪条路进来
task = Task(
    description: str,
    schedule: OneShot | Daily | Cron,
    target_agent: str = "master",   # 默认 master，可指定
    context_seed: ChatContext | None = None,  # chat 升级时把对话历史带进来
)
TaskQueue.enqueue(task)
```

---

## 3. Task 状态机

```
              ┌────────────┐
              │   queued    │  入队，等待执行
              └──────┬──────┘
                     │ 调度器拉起
                     ▼
              ┌────────────┐
              │   running   │  执行中
              └──┬───┬───┬──┘
                 │   │   │
       完成      │   │   │   遇到 L3 回退（缺资源 / 用户决策）
       ◄────────┘   │   └──────┐
                    │          │
              ┌─────▼─────┐    │
              │ completed  │    │
              └────────────┘    ▼
                          ┌──────────┐
                          │  paused   │  等用户输入
                          └────┬─────┘
                               │ 用户提供输入 → resume
                               ▼
                          (回到 running)

                          错误 / 不可恢复
                  ┌─────────────────────────┐
                  │                         │
                  ▼                         ▼
            ┌──────────┐              ┌──────────┐
            │  failed   │              │ cancelled │
            └──────────┘              └──────────┘
```

### 3.1 状态详解

| 状态 | 含义 | 可转移到 |
|---|---|---|
| `queued` | 入队，等调度 | `running`, `cancelled` |
| `running` | 执行中 | `completed`, `paused`, `failed`, `cancelled` |
| `paused` | 卡在 L3 回退，等用户输入 | `running`（resume）, `cancelled` |
| `completed` | 正常完成，artifact 已产出 | （终态） |
| `failed` | 不可恢复错误 | （终态，可手动 retry → 起新 task） |
| `cancelled` | 用户主动取消 | （终态） |

### 3.2 状态持久化

- 基于 LangGraph 的 checkpointer（SqliteSaver 或更强后端）
- 每个 Node 完成后自动 checkpoint
- `paused` / 进程崩溃 / 主动 kill 后，重启能 resume
- Trace 记录所有状态转移（who / when / why）

---

## 4. Pause / Resume：与 multi-agent 三级回退打通

Task 进入 `paused` 状态的唯一来源是 **multi-agent 三级回退的 L3**——sub-agent 自己解决不了 + 主 agent 也代答不了 + 必须等用户输入。

详见 [Multi-agent § 三级回退](multi-agent.md)。

### 4.1 Pause 时发生什么

1. Sub-agent 报告 `BlockedException(reason, needed_input)`
2. 主 agent 评估：能否代答？不能 → 升级 L3
3. Task 状态：`running` → `paused`
4. Task 记录 `pending_question`、`needed_input` schema
5. 通知用户（chat 弹卡片 / Web UI 推通知 / 推送到外部通道）

### 4.2 Resume 时发生什么

1. 用户在 chat / Task 视图提供输入
2. 输入 validate（符合 `needed_input` schema 否）
3. Task 状态：`paused` → `running`
4. LangGraph `Command(resume=user_input)` 重新进入 sub-agent
5. Trace 记录"用户提供了 X，task resume"

---

## 5. Task ↔ Chat 互通

### 5.1 Context 打通

Chat 升级到 task 时，**chat 历史可以带进 task** 作为 context seed：

```
Chat 里聊了 5 轮：
  - 用户介绍了背景：「我在做生猪养殖业的研究」
  - 讨论了关心的指标
  - 决定要分析最近的趋势
  ↓ 升级为 task
Task 创建时，把这 5 轮对话作为初始 context 传入
  ↓ Task 内部 agent 知道用户意图，不会再问"你想做什么"
```

### 5.2 Task 完成后回流到 chat

```
Task #N 完成
  ↓
Chat 里收到通知卡片（artifact 列表 / 关键结果摘要）
  ↓
用户可在 chat 里继续追问：「Task #N 生成的图表里第三个我想再细化一下」
  ↓
Master Agent 能引用 task 产出的 artifact 继续对话
```

### 5.3 周期性 task 怎么和 chat 关联

周期性 task（场景 1：每天早上推送）通常**不需要持续的 chat 上下文**——它是 standalone 的。但用户可在 chat 里：
- 调整任务（"以后早上 8 点推送"）
- 反馈结果（"今天那条小红书的不错，多推这种"）
- 暂停 / 删除

这些反馈会进入 task 的 memory（用户偏好），下次执行时影响行为。

---

## 6. CLI 命令映射（Phase 1）

```bash
# Chat
sigma chat                       # 进入对话流（多轮）
sigma chat "一句话问题"           # 单次问答

# Task
sigma task new "..."              # 新建一次性 task
sigma task new "..." --daily 09:00 # 每天 09:00 跑
sigma task new "..." --cron "0 9 * * 1-5"  # cron 风格
sigma task list                   # 任务列表
sigma task view <id>              # 查看进度 / log / artifact
sigma task pause <id>             # 主动暂停
sigma task resume <id> --input "..."  # 恢复（提供 paused 时缺的输入）
sigma task cancel <id>            # 取消

# 系统
sigma serve                       # Phase 2：起 Web UI server
```

---

## 7. Web UI（Phase 2）映射

```
┌─────────────────────────────────────────────────────────┐
│  Sigma                                       [user]      │
├──────────────┬──────────────────────────────────────────┤
│              │                                          │
│  💬 Chat      │   [Chat 视图：对话流]                     │
│              │   - markdown 渲染                         │
│  📋 Tasks     │   - 代码高亮                              │
│   ○ Running   │   - Task 升级建议卡片                      │
│   ○ Paused    │   - Task 完成回流卡片                      │
│   ✓ Done      │                                          │
│              │                                          │
│  🔧 Skills    │                                          │
│  🤖 Agents    │                                          │
│  ⚙️  Settings  │                                          │
│              │                                          │
└──────────────┴──────────────────────────────────────────┘
```

Tasks 视图的 task 详情页：
- 状态条 / 进度
- Artifact 预览（图片 / markdown / 代码 / 文件）
- 实时日志流
- Trace 链接（→ trace viewer）
- 操作按钮（pause / resume / cancel / 追加指令）

---

## 8. 已废弃的方向

### ❌ "Cascade / Realtime" 双模式

之前文档里描述的 "级联模式 / Realtime 模式"是按**音频处理方式**划分的。在新定位下：
- Voice 不再是一等公民
- 双模式按**交互形态**（chat / task）划分，不按音频处理

### ❌ "Agent 内部判断起不起 task" 的单视图模型

之前 overview.md 把 Task Manager 描述为"agent 可调用的工具"。新方案：
- Agent 主动建议升级（用户最终确认）
- 用户也能直接进 Task 视图新建
- 两条路统一 queue

---

## 9. 相关文档

- [架构总览](overview.md) — 整体定位
- [Realtime 模式](realtime-mode.md) — 第三种交互模式（实时语音）
- [Multi-agent](multi-agent.md) — 三级回退细节（Pause 的来源）
- [Task 模块](../modules/task/) — Task 引擎实现细节（状态机 / queue / 持久化）
- [Agent 模块](../modules/agent/) — Master / Sub-agent 框架
- [设计决策日志](design-log.md) — 每个决定的演化和理由
