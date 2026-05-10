# Skill 模块

> Skill 是 **Sigma 的"知识/方法论扩展点"**——纯 markdown 文件，**0 Python 接口**。

> 详见 [架构总览 § 4.3](../../architecture/overview.md#43-skill)。

---

## 1. 定位

**Skill ≠ Agent**：

| | Skill | Agent |
|---|---|---|
| 本质 | 一段 prompt + 配套资源 | 完整的 reasoning loop |
| LLM 调用 | 0（被注入到调用方 prompt） | 多次（自己有 loop） |
| 状态机 | 无 | 有 |
| 写起来 | markdown 文件 | Python 类 + graph |

**Skill 是 prompt 的延迟加载**——通过把方法论抽到 markdown，让任意 agent（master 或 sub）都能在需要时按需引用。

---

## 2. 文件夹约定

```
~/.sigma/skills/
  research/
    SKILL.md              # 必需，含 metadata + 主要内容
    references/           # 可选，详细参考资料（按需读取）
      citation-styles.md
      data-sources.md
    templates/            # 可选，模板/资源文件
      report-outline.md
  
  writing/
    SKILL.md
    templates/
      blog-post.md
      readme.md
```

**约定**：
- `SKILL.md` 是入口，必须存在
- 子目录（`references/`、`templates/` 等）按 skill 自己定义，agent 通过 tool 读取
- 路径必须 self-contained（skill 内容引用自己目录下的文件用相对路径）

---

## 3. SKILL.md 格式

```markdown
---
name: research
description: |
  擅长资料收集、网页抓取、信息总结、引用规范。
  适合需要拉取多源数据并产出报告的任务。
triggers: [研究, 调研, 查一下, research]
allowed_tools: [search_web, fetch_url, read_file]
---

# Research Skill

## 方法论

1. 先明确研究目标（用户想要什么 / 决策依据是什么）
2. 数据源选择...
3. ...

## 引用规范

详见 references/citation-styles.md

## 模板

输出报告参考 templates/report-outline.md
```

**Front-matter（YAML）字段**：

| 字段 | 必需 | 用途 |
|---|---|---|
| `name` | ✅ | 唯一标识，对应文件夹名 |
| `description` | ✅ | Supervisor / agent 看这个判断要不要 load_skill |
| `triggers` | 可选 | 触发关键词，提高匹配准确度 |
| `allowed_tools` | 可选 | 这个 skill 允许 agent 调用哪些 tool 的子集 |

---

## 4. 渐进式加载机制

**问题**：如果一次性把所有 skill 的 SKILL.md 全文塞进 system prompt，context 会爆。

**解决**：两阶段加载。

### 4.1 阶段 1：metadata 注入

Sigma 启动时扫描所有 skill，把 **name + description + triggers** 注入 system prompt：

```
You have access to the following skills. Use load_skill(name) to load full instructions:

- research: 擅长资料收集、网页抓取、信息总结、引用规范...
  triggers: [研究, 调研, 查一下, research]
- writing: 擅长长文写作、blog post、技术文档...
  triggers: [写, 起草, write]
- ...
```

### 4.2 阶段 2：按需加载

Agent 判断要用某个 skill 时调内置 tool：

```python
@sigma.builtin_tool
def load_skill(name: str) -> str:
    """Load the full content of a skill, including its SKILL.md."""
    skill_path = SKILLS_DIR / name / "SKILL.md"
    return skill_path.read_text()
```

加载后内容进入 agent 的 conversation context，agent 后续推理就带着这段 prompt。

### 4.3 子资源访问

Skill 内引用的 references/ / templates/ 文件，通过普通的 `read_file` tool 访问（agent 看到 SKILL.md 里的路径，自己决定读不读）。

---

## 5. Skill 在 multi-agent 中的角色

**Skill 是"全局可用的方法论"**，不绑定特定 agent：

```
Master Agent ──────── load_skill("research") ──→ Research SKILL.md
Researcher Agent ──── load_skill("citation") ──→ Citation SKILL.md
Coder Agent ──────── load_skill("git-workflow") ──→ Git SKILL.md
```

任何 agent 都能 `load_skill(name)`，互不干扰。

但 sub-agent 可以**限制自己只看哪些 skill**（在 metadata 里声明）：

```python
class ResearcherAgent(sigma.Agent):
    name = "researcher"
    allowed_skills = ["research", "citation", "data-sources"]   # 子集白名单
```

---

## 6. 与 Tool 的关系

Skill 通过 `allowed_tools` 字段声明能调用的 tool 子集：

```yaml
---
name: research
allowed_tools: [search_web, fetch_url, read_file]
---
```

**Tool 是 skill 的"行为词汇表"**——skill 用 markdown 描述方法论，描述会自然引用 tool 名字（"用 `search_web` 找最新数据，再用 `fetch_url` 拉详情"）。

如果 skill 文档提到一个 tool，但 `allowed_tools` 里没有，运行时报错或忽略——这是个 guard。

---

## 7. 内置 skill 清单（V2 阶段）

预计提供：

| Skill | 用途 |
|---|---|
| `research` | 资料收集和总结的方法论 |
| `writing` | 长文写作风格和结构 |
| `coding` | 代码风格、git 流程、commit message |
| `data-analysis` | 数据探索、可视化的方法论 |
| `summarize` | 长文总结策略 |

详见各 skill 自己的 SKILL.md。

---

## 8. 实现位置

```
src/skill/
  loader.py           # 启动扫描 + metadata 注入
  registry.py         # skill 元数据缓存
  builtin_tool.py     # load_skill 工具的实现
```

```
~/.sigma/skills/      # 用户自定义 skill 安装目录
sigma/skills/         # Sigma 自带的 skill（可被覆盖）
```

---

## 9. 未决问题

| 问题 | 状态 |
|---|---|
| Skill 之间是否可以引用（A skill 引用 B skill 的内容） | 未决 |
| 用户是否能从 registry 下载第三方 skill（安全模型） | 未决（V5+） |
| Skill 版本管理 | 未决（V5+） |

---

## 10. 相关文档

- [架构总览 § 4.3](../../architecture/overview.md#43-skill)
- [Multi-agent](../../architecture/multi-agent.md) — sub-agent 怎么用 skill
- [Tools 模块](../tools/) — Skill 引用的 tool 来源
- [设计决策日志 § 4.1](../../architecture/design-log.md) — Skill ≠ Agent 的洞察来源
