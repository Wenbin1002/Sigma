# 贡献指南

感谢你对 Sigma 项目的兴趣！本文档帮助你从零开始参与贡献。

## 前置要求

- Python 3.11+
- Git
- GitHub 账号

## 环境搭建

```bash
# Fork 并 clone
git clone <your-fork-url>
cd Sigma

# 安装依赖
pip install -e ".[dev]"

# 验证
python -m pytest  # 当前可能没有测试，正常
```

## 开发流程

所有代码变更必须通过分支 + PR 合入，**禁止直接在 main 上提交**。

### 1. 建分支

```bash
git fetch origin
git checkout -b <type>/<short-desc> origin/main
```

分支命名格式：`<type>/<short-desc>`

| type | 用途 |
|------|------|
| `feat` | 新功能 |
| `fix` | 修复 bug |
| `refactor` | 重构（不改行为） |
| `chore` | 依赖、配置、工具链 |
| `perf` | 性能优化 |
| `test` | 测试 |
| `docs` | 文档 |

示例：`feat/voice-streaming`、`fix/pipeline-timeout`、`docs/add-rag-guide`

### 2. 开发

编码时注意：
- 遵守 [代码规范](docs/guides/code-style.md)
- 遵守 [依赖规则](docs/architecture/dependency-rules.md)

### 3. Commit

Commit message 格式：`<type>(<scope>): <subject>`

```bash
git add src/voice/elevenlabs_tts.py src/voice/registry.py
git commit -m "feat(voice): add ElevenLabs TTS adapter"
```

scope 从以下选取：`voice | agent | rag | memory | tools | trace | core | ports | runtime | context | app | deps | config | project`

### 4. 提 PR

```bash
git push -u origin <branch-name>
gh pr create --title "<type>(<scope>): <subject>" --body "..."
```

PR 会经过架构约束检查，确保没有违反依赖规则。

## 架构红线

这三条规则**不可违反**，违反会导致 PR 被拒：

1. **`runtime/` 只 import `ports/`** — 永远不直接 import 域目录的实现
2. **域目录之间不互相 import** — `voice/` 不能 import `agent/`，反之亦然
3. **`core/` 无外部依赖** — 只用 Python 标准库

详见 [依赖规则](docs/architecture/dependency-rules.md)。

## 推荐贡献方向

### 新手友好

- 为现有模块添加新的 Adapter（参见 [添加 Adapter 指南](docs/guides/adding-an-adapter.md)）
- 完善文档和注释
- 添加测试用例

### 进阶

- 实现新的 RAG 策略
- 优化语音延迟
- 接入新的 LLM provider
- 在 `experiments/` 中做对比实验

## 获取帮助

- 在 Issue 中提问
- 查阅 [完整文档](docs/index.md)
- 阅读 [架构总览](docs/architecture/overview.md) 理解系统设计
