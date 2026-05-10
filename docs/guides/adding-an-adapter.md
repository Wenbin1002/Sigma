# 添加扩展

> Sigma 有四种扩展方式：**Tool / Skill / Agent / LLM provider**。每种门槛、形态、注册方式都不同。

## 选哪种？

```
扩展什么？
   ├── 一个动作（"发邮件"、"抓网页"）              → Tool
   ├── 一段方法论（"研究怎么做"、"代码风格"）        → Skill
   ├── 一个独立 reasoning loop（"研究员"、"分析师"）→ Agent
   └── 接入新 LLM provider                       → LLM Adapter
```

详见 [架构总览 § 4](../architecture/overview.md#4-扩展模型tool--skill--agent-三层正交)。

---

## A. 添加 Tool

**门槛**：⭐⭐ 写 Python 函数 + 注册。

### 步骤

#### 1. 创建文件

`src/tools/builtin/<tool_name>.py` 或 `~/.sigma/tools/<tool_name>.py`：

```python
from sigma import tool

@tool
def send_email(to: str, subject: str, body: str) -> bool:
    """发送邮件。
    
    Args:
        to: 收件人
        subject: 主题
        body: 正文
    
    Returns:
        bool: 发送是否成功
    """
    # 实现细节...
    return True
```

#### 2. 注册到 registry

如果是内置 tool，自动通过 decorator 注册。如果手动管理：

```python
# src/tools/registry.py
from tools.builtin.email import send_email

REGISTRY: dict[str, ToolPort] = {
    ...
    "send_email": send_email,
}
```

#### 3. 启用配置

```yaml
# config.yaml
tools:
  builtin:
    - send_email
```

#### 4. 写测试

`tests/tools/test_send_email.py`：

```python
import pytest
from tools.builtin.email import send_email

def test_send_email_success():
    # 用 mock，不依赖真实 SMTP
    ...
```

### 高风险 Tool

如果 tool 涉及破坏性操作，标记需要确认：

```python
@tool(requires_approval=True)
def delete_file(path: str) -> bool:
    """删除文件（需要用户确认）。"""
    ...
```

详见 [Tools 模块 § 6](../modules/tools/#6-tool-的安全考虑)。

### 接入 MCP server（不写 tool 代码）

如果想复用 MCP 生态：

```yaml
tools:
  mcp_servers:
    - name: github
      command: npx
      args: ["@anthropic/mcp-github"]
```

Sigma 自动把 MCP server 暴露的 tool 注入。

---

## B. 添加 Skill

**门槛**：⭐ 写一段 markdown，**不写 Python**。

### 步骤

#### 1. 创建目录

```
~/.sigma/skills/<skill_name>/
  SKILL.md              # 必需
  references/           # 可选
  templates/            # 可选
```

#### 2. 写 SKILL.md

```markdown
---
name: research
description: |
  擅长资料收集、网页抓取、信息总结、引用规范。
triggers: [研究, 调研, research]
allowed_tools: [search_web, fetch_url, read_file]
---

# Research Skill

## 方法论

1. 先明确研究目标
2. 数据源选择...
...

## 引用规范

详见 references/citation-styles.md
```

#### 3. 完成

Sigma 启动时自动扫描，**无需注册**。

详见 [Skill 模块](../modules/skill/)。

---

## C. 添加 Sub-agent

**门槛**：⭐⭐⭐ 继承 `sigma.Agent`，定义 graph。

### 步骤

#### 1. 创建目录

```
~/.sigma/agents/<agent_name>/
  agent.py              # 必需
  prompts/              # 可选
  tools/                # 可选私有 tool
```

#### 2. 实现 agent

```python
# agent.py
from sigma import Agent, StateGraph, START, END
from typing import TypedDict
from tools import search_web, summarize

class ResearchState(TypedDict):
    task: str
    sources: list[str]
    findings: list[dict]
    report: str

class ResearcherAgent(Agent):
    name = "researcher"
    description = "擅长资料收集、网页抓取、信息总结。不擅长写代码或数据分析。"
    triggers = ["调研", "查一下", "research"]
    tools = [search_web, summarize]
    
    def build_graph(self) -> StateGraph:
        g = StateGraph(ResearchState)
        g.add_node("plan", self.plan)
        g.add_node("collect", self.collect)
        g.add_node("synthesize", self.synthesize)
        g.add_edge(START, "plan")
        g.add_edge("plan", "collect")
        g.add_edge("collect", "synthesize")
        g.add_edge("synthesize", END)
        return g
    
    async def plan(self, state: ResearchState) -> dict:
        ...
    
    async def collect(self, state: ResearchState) -> dict:
        ...
    
    async def synthesize(self, state: ResearchState) -> dict:
        ...
```

#### 3. 关键约束

- **必须声明 metadata**：`name` / `description` / `triggers`
  - Supervisor 看 `description` 路由
  - 用户用 `@name` 召唤
- **声明 `tools`**：限定 agent 能调用的 tool 子集
- **卡住时 raise BlockedException**（不要 raise 通用异常）：

```python
from sigma import BlockedException

if missing_data:
    raise BlockedException(
        reason="missing_information",
        detail="需要用户补充什么...",
        needed_input=NeededInput(...),
    )
```

详见 [Multi-agent § 4](../architecture/multi-agent.md#4-三级回退sub-agent-沟通问题)。

#### 4. 完成

Sigma 启动时自动扫描，**无需手动注册**。

---

## D. 添加 LLM Provider

**门槛**：⭐⭐ 实现 LLMPort + 注册。

### 步骤

#### 1. 创建 adapter

`src/llm/<provider>.py`：

```python
"""Custom LLM Adapter"""

from typing import AsyncIterator
from ports.llm import LLMPort, LLMChunk, Message, ToolSchema

class CustomLLM:
    """自定义 LLM provider 实现。"""
    
    def __init__(self, config):
        self.api_key = config.api_key
        self.model = config.model
        # ...
    
    async def stream(
        self,
        messages: list[Message],
        tools: list[ToolSchema] | None = None,
        **kwargs,
    ) -> AsyncIterator[LLMChunk]:
        # 调用 provider API，转换 chunk 格式
        ...
    
    async def generate(
        self,
        messages: list[Message],
        tools: list[ToolSchema] | None = None,
        **kwargs,
    ) -> LLMResponse:
        ...
```

#### 2. 关键约束

- **`LLMChunk` 必须含 token usage**——cost guard 依赖
- **流式用 `AsyncIterator[LLMChunk]`**，不传 callback
- **遵循 OpenAI 消息格式**（事实标准）

详见 [Ports & Adapters § 3.1](../architecture/ports-and-adapters.md#31-llmport)。

#### 3. 注册 registry

```python
# src/llm/registry.py
from llm.custom import CustomLLM

REGISTRY: dict[str, type[LLMPort]] = {
    "openai": OpenAILLM,
    "deepseek": DeepSeekLLM,
    "custom": CustomLLM,    # ← 新增
}
```

#### 4. 配置

```yaml
llm:
  default:
    provider: custom
    model: my-model
    api_key: "..."
```

#### 5. 写测试 + 更新模块文档

`tests/llm/test_custom.py` + 更新 `docs/modules/llm/README.md` 内置 adapter 表。

---

## PR 前检查清单

通用：
- [ ] 没有违反 [依赖规则](../architecture/dependency-rules.md)（特别：域目录不互引、`app/` 不调内核）
- [ ] 测试通过
- [ ] 文档已更新

Tool 特有：
- [ ] 高风险操作标记 `requires_approval=True`
- [ ] 参数有 type hint（用于生成 LLM tool schema）

Skill 特有：
- [ ] front-matter 含 `name` / `description`
- [ ] `triggers` 关键词代表性强

Agent 特有：
- [ ] metadata（`name` / `description` / `triggers`）齐全
- [ ] `tools` 子集声明
- [ ] 内部 raise `BlockedException` 而非通用异常
- [ ] Graph 定义清晰、节点单一职责

LLM 特有：
- [ ] `LLMChunk` 含 token usage
- [ ] 流式不传 callback
- [ ] OpenAI 消息格式兼容

---

## 常见问题

### Q: 我的 Tool 需要调用其他模块的功能怎么办？

通过其他 tool 间接调用（推荐）。或在 Tool 内部调用 [Memory](../modules/memory/) / [RAG](../modules/rag/) 等内核模块的接口（允许，因为 tool 在内核层而非域层）。

### Q: 我的 Skill 想引用其他 Skill 的内容？

目前不直接支持（U-决策）。临时方案：在 SKILL.md 里 prompt agent "如果需要 X 知识，可以 load_skill('other-skill')"。

### Q: 我的 Agent 想调用其他 Agent？

允许。在 graph 里 spawn sub-agent 即可。但请慎重——嵌套层级深时调试和成本控制都难。详见 [Agent 模块 § 7](../modules/agent/#7-agent-之间协作)。

### Q: 一个文件能放多个 adapter 吗？

不推荐。**一个 adapter 一个文件**。如果非常相似（同一 SDK 不同模型），可放一起。

---

## 相关文档

- [架构总览](../architecture/overview.md)
- [Multi-agent](../architecture/multi-agent.md)
- [Skill 模块](../modules/skill/)
- [Tools 模块](../modules/tools/)
- [LLM 模块](../modules/llm/)
- [依赖规则](../architecture/dependency-rules.md)
- [代码规范](code-style.md)
