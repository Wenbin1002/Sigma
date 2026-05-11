# Agent Protocol & Context 接口

> 所有 agent（内置 / 用户编写）对外遵循的统一契约。

---

## Agent Protocol

```python
class Agent(Protocol):
    name: str
    description: str
    triggers: list[str]                  # @-mention / 路由关键词
    tools: list[str]                     # tool 白名单
    
    async def run(
        self,
        task: str,                       # 派下来的 task description
        ctx: Context,                    # Sigma 提供的能力窗口
    ) -> AsyncIterator[AgentChunk]:
        ...
```

**输入 / 输出**：
- 输入：task description + Context
- 输出：流式 `AgentChunk`（含 text / tool_call / artifact / proxy_decision / needs_input ...）
- 内部：完全黑盒——Sigma 不关心 agent 内部怎么实现

**设计决策**：

- Protocol 不暴露 LangGraph——agent 内部用什么 runtime 是自己的事
- 所有 agent（内置的用 LangGraph、用户写的用纯 Python）对 supervisor 来说是同质的
- 这让 Sigma 可以换底层实现而不影响用户 agent

---

## Context 接口

Context 是 Sigma 给 agent 的"能力窗口"，agent 通过它与 Sigma 生态交互：

```python
class Context:
    # LLM 调用
    llm: LLMHandle                    # ctx.llm.chat(...) / ctx.llm.stream(...)
    
    # Tool 调用
    async def use_tool(self, name: str, **kwargs) -> Any
    
    # Skill 加载
    async def load_skill(self, name: str) -> str
    
    # 状态持久化（checkpoint 会存）
    state: dict
    
    # 跟用户/上级 agent 交互
    async def ask_user(self, question: str) -> str       # 阻塞等回答
    async def report_progress(self, msg: str)            # 进度汇报（不阻塞）
    
    # 上下文
    history: list[Message]             # 对话历史（如果有）
    parent_task: str                   # 被派的任务描述
    
    # 子 agent 调用
    async def spawn_agent(self, name: str, task: str) -> AsyncIterator[AgentChunk]
    
    # 成本/限制
    budget: Budget                     # 还剩多少 token/钱
```

---

## 设计原则

1. **Context 是稳定接口**——内部实现可以随时换，用户 agent 不感知
2. **不需要 import LangGraph**——通过 Context 就能用 Sigma 的全部能力
3. **能力都走 Context 统一口径**——LLM / Tool / Skill / Checkpoint / 交互 / 子 agent

---

## BlockedException 协议

Agent 卡住时的结构化通信：

```python
class BlockedException(Exception):
    reason: Literal[
        "missing_resource",     # 缺 API key / 文件 / 登录
        "ambiguous_decision",   # 决策歧义
        "missing_info",         # 缺信息
    ]
    detail: str
    needed_input: NeededInput
```

**两类卡住**：
- 资源性（缺 API key / 文件 / 登录）→ 直接 bubble up 用户
- 认知性（决策歧义 / 缺信息）→ 先尝试自决 → 再 bubble up 主 agent → 最后才问用户

详见 [Multi-agent § 三级回退](../../features/multi-agent.md)。
