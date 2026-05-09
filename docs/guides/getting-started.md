# 新手指南

## 前置要求

- Python 3.11+
- Git
- 一个代码编辑器（推荐 VS Code / Cursor）

## 环境搭建

```bash
# Clone 仓库
git clone <repo-url>
cd Sigma

# 安装依赖（开发模式）
pip install -e ".[dev]"

# 复制配置模板
cp config.example.yaml config.yaml
# 编辑 config.yaml 填入必要的 API key
```

## 项目当前状态

Sigma 处于**早期开发阶段**：

- ✅ 架构设计完成
- ✅ 接口定义（Port Protocol）设计完成
- 🚧 V0 实现进行中
- 📋 代码目录 `src/` 正在建设

这意味着当前最需要的贡献是：实现各模块的 Adapter、完善文档、在 `experiments/` 中做技术验证。

## 快速理解架构

**三句话版本：**

1. 所有模块通过 Port（Protocol 接口）通信，`runtime/` 只看到接口不看到实现
2. 切换任何技术方案 = 改 `config.yaml` 里的 provider 字段
3. 新增技术 = 写 adapter 文件 + 注册到 registry + 加配置

**想深入了解**：阅读 [架构总览](../architecture/overview.md)。

## 第一次贡献建议

### 最简单的开始

1. 阅读一个模块文档（如 [Voice](../modules/voice/)）
2. 在 `experiments/` 中写一个 proof-of-concept
3. 如果验证可行，转为正式 Adapter 贡献

### 常见贡献类型

| 类型 | 难度 | 说明 |
|------|------|------|
| 添加 Adapter | ⭐⭐ | 为现有模块接入新 provider |
| 文档改进 | ⭐ | 修正、补充、翻译 |
| 实验验证 | ⭐⭐ | 在 experiments/ 中对比方案 |
| Bug 修复 | ⭐⭐ | 修复已知问题 |
| 新功能 | ⭐⭐⭐ | 需先 Issue 讨论 |

## 应该先读的文件

按优先级排列：

1. **[CLAUDE.md](../../CLAUDE.md)** — 项目规则总纲（3 分钟）
2. **[架构总览](../architecture/overview.md)** — 系统设计（10 分钟）
3. **[依赖规则](../architecture/dependency-rules.md)** — Import 约束（5 分钟）
4. **你感兴趣的模块文档** — 如 [Voice](../modules/voice/)、[Agent](../modules/agent/)（5 分钟）
5. **[添加 Adapter](adding-an-adapter.md)** — 如果准备写代码（5 分钟）

## 开发工作流

```
1. git fetch origin
2. git checkout -b feat/my-feature origin/main
3. 开发 + 测试
4. git add + git commit（遵循 commit 规范）
5. git push + 提 PR
```

详见 [贡献指南](../../CONTRIBUTING.md)。

## 相关文档

- [贡献指南](../../CONTRIBUTING.md) — 完整开发流程
- [代码规范](code-style.md) — 命名和风格
- [添加 Adapter](adding-an-adapter.md) — 最常见贡献方式的详细步骤
