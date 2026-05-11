# 用户 Agent 安装与加载

> 用户自己编写或从社区下载的 agent 如何接入 Sigma。

---

## 文件结构

```
~/.sigma/agents/
  stock-analyst/
    agent.py                  # 必需：实现 Agent Protocol
    README.md                 # 可选：说明文档
    requirements.txt          # 可选：额外依赖（用户自行管理）
    prompts/                  # 可选：prompt 模板
    
  xhs-monitor/
    agent.py
    ...
```

唯一硬性要求：**目录下有 `agent.py`，里面有一个实现了 Agent Protocol 的类。**

---

## 加载机制

Sigma 启动时：

1. 扫描 `~/.sigma/agents/` 下所有子目录
2. import `agent.py`，找到 `Agent` 子类
3. 检查是否满足 Protocol（name / description / triggers / run）
4. 注册到 supervisor 的路由表——跟内置 agent 平等对待

**加载失败不阻断启动**——报 warning，跳过这个 agent，继续加载其他的。

---

## 分发模型

### V1：本地文件夹

手动复制 / git clone：

```bash
git clone https://github.com/someone/sigma-stock-analyst ~/.sigma/agents/stock-analyst
```

这是最简单的方式，跟 Claude Code 的 MCP server / oh-my-zsh 插件一样——clone 即用。

### V5+（未来）：Registry

```bash
sigma install agent stock-analyst          # 从 registry 下载
sigma install agent gh:user/my-agent       # 从 GitHub 下载
```

V1 只做本地扫描。文件夹结构已经为未来 registry 留好了口子。

---

## 依赖管理

**Sigma 不管用户 agent 的依赖**——跟用户自己跑 Python 脚本一样：

- 用户 agent `import pandas`？用户自己 `pip install pandas`
- 可选放一个 `requirements.txt` 供参考
- 未来可考虑自动检测并提示（但不自动安装）

这跟产品定位一致：用户有编程能力、在自己机器跑、类似 OpenClaw / Claude Code。

---

## 安全模型

不做沙箱——**用户 agent 跟 Sigma 同权限运行**。

理由：
- 用户部署在自己机器
- 用户有编程能力，能审查代码
- 跟"你自己写个 Python 脚本"没区别
- 类比：VSCode extension 也是同权限跑的

唯一的约束是 **tool 白名单**——agent 声明的 `tools` 列表限制了它能调哪些 tool。但这更多是"防手滑"而非"防恶意"。
