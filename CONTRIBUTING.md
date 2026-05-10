# 贡献指南

感谢你对 Sigma 的兴趣！本文档帮助你从零开始参与贡献。

## 前置要求

- Python 3.11+
- Git
- GitHub 账号

## 环境搭建

```bash
git clone <your-fork-url>
cd Sigma

pip install -e ".[dev]"
python -m pytest   # 早期可能没有测试，正常
```

## 开发流程

所有变更必须通过分支 + PR 合入，**禁止直接在 main 上提交**。

### 1. 建分支

```bash
git fetch origin
git checkout -b <type>/<short-desc> origin/main
```

| type | 用途 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 bug |
| `refactor` | 重构（不改行为） |
| `chore` | 依赖、配置、工具链 |
| `perf` | 性能优化 |
| `test` | 测试 |
| `docs` | 文档 |

示例：`feat/openai-llm-adapter`、`fix/task-pause-race`、`docs/chat-task-modes`

### 2. 开发

- 遵守 [代码规范](docs/guides/code-style.md)
- 遵守 [依赖规则](docs/architecture/dependency-rules.md)
- 改动如涉及架构决策，请先在 issue / discussion 里同步思路，避免大改返工

### 3. Commit

格式：`<type>(<scope>): <subject>`

```bash
git add src/llm/openai.py src/llm/registry.py
git commit -m "feat(llm): add OpenAI LLM adapter"
```

scope 取自：`agent | skill | task | chat | context | rag | memory | tools | llm | trace | checkpoint | core | ports | server | app | docs | deps | config | project`

### 4. 提 PR

```bash
git push -u origin <branch-name>
gh pr create --title "<type>(<scope>): <subject>" --body "..."
```

PR 会经过架构约束检查（`.githooks/pre-commit` 和 CI）。

## 架构红线（不可违反）

详见 [依赖规则](docs/architecture/dependency-rules.md)。三条不可妥协：

1. **`ports/` 不依赖任何具体实现**——`ports/` 里 import 任何域目录就是退化
2. **域目录之间不互引**——`llm/`、`tools/`、`trace/`、`checkpoint/` 互相禁止 import
3. **`app/` 不直接调内核**——必须走 `server/` 暴露的 HTTP/SSE API（client/server 分离）

## 推荐贡献方向

按门槛从低到高：

### 极低门槛：写 Skill（不需要写一行 Python）

Skill 是纯文件夹 + markdown，描述一个领域的方法论：

```
~/.sigma/skills/your-skill/
  SKILL.md              # 必需，含 metadata
  references/           # 可选，详细参考资料
  templates/            # 可选，模板文件
```

详见 [Skill 模块](docs/modules/skill/) 和 [添加 Adapter](docs/guides/adding-an-adapter.md)。

### 低门槛：接入新 LLM provider / 新 Tool

实现对应 Port + 注册到 registry + 写测试。详见 [添加 Adapter](docs/guides/adding-an-adapter.md)。

### 中门槛：写 Sub-agent

继承 `sigma.Agent`，定义自己的 graph：

```python
class YourAgent(sigma.Agent):
    name = "your_agent"
    description = "..."
    triggers = [...]
    tools = [...]
    
    def build_graph(self):
        ...
```

详见 [Multi-agent](docs/architecture/multi-agent.md) 和 [Agent 模块](docs/modules/agent/)。

### 高门槛：内核改进

- Trace viewer 功能扩展
- Cost guard 策略
- Context engine 优化
- RAG 多 index 管理
- Memory 分层策略

这类改动建议先开 discussion / issue 讨论方案。

## 获取帮助

- 在 Issue 中提问
- 查阅 [完整文档](docs/index.md)
- 阅读 [架构总览](docs/architecture/overview.md) 理解系统
- 阅读 [设计决策日志](docs/architecture/design-log.md) 了解每个决定的来龙去脉
