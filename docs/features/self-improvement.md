# Self-Improvement

> Sigma 怎么从用户互动中学习——**让 agent 越用越懂用户**。
>
> 这是 Sigma 真正的差异化能力之一，但需要建立在 memory / trace / realtime 之上。

---

## 1. 是什么 / 不是什么

### 是

- **从用户反馈和互动中学习**——更新 memory、调整偏好、改进 prompt 选择
- **闭环驱动**——signal 提取 → 写入 memory → 影响下次行为 → 看新的 signal
- **本地学习**——所有 signal 处理在本地，不上传

### 不是

- ❌ **不是模型微调**（不在 Sigma 范围）
- ❌ **不是 RAG 自动更新**（那是 RAG 模块的事）
- ❌ **不是无监督学习**（要可解释、可审计、可被用户 override）

---

## 2. 关键洞察：Realtime 是 self-improvement 的天然落地

文本 chat 的反馈密度低——每分钟 1-2 次有效信号。Realtime 不一样：

```
1 分钟 Realtime 对话可能产生：
  - 5-10 次打断 / 被打断
  - 2-3 次"重说一遍"
  - 多次沉默时长变化
  - 笑声 / 叹息 / 语速变化
  - 显式偏好（"以后 xxx"）
```

**信号密度高 + 有"语气" 维度** → Self-improvement 在 Realtime 模式下能看到效果。在文本 chat 里也有，但更稀疏。

详见 [Realtime 模式 § 7](realtime-mode.md#7-self-improvement-的天然落地)。

---

## 3. Signal 来源

### 3.1 显式 signal（用户明说）

| Signal | 例子 | 处理 |
|---|---|---|
| 偏好声明 | "以后用更专业的术语" | 写入 global memory |
| 反馈 | "你回答太长了" | 调短回复偏好 |
| 纠正 | "我是金融行业不是 IT" | 修正 global memory |
| 风格 | "用正式语气" | 风格偏好 |
| 主题兴趣 | "对 X 不感兴趣" | 兴趣过滤 |

### 3.2 隐式 signal（行为推理）

#### 来自文本对话

| Signal | 推理 |
|---|---|
| 用户重复问 | 上次回答没解决问题 |
| 用户说"还有吗"/"继续" | 当前回答信息量不够 |
| 用户突然换话题 | 当前话题不感兴趣 / 已解决 |
| 长时间没回 | 用户在思考 / 已离开 |
| 直接给答案 vs 反问 | 用户决策模式 |

#### 来自 Realtime（独有）

| Signal | 来源 | 推理 |
|---|---|---|
| 打断 | Audio metadata | "我话太多了" |
| "重说" / "再来一次" | Transcript | 上次回答不够好 |
| 沉默时长 | Audio metadata | 思考 vs 走神 |
| 语速变化 | Audio metadata | 投入 / 疲倦 |
| 笑声 / 叹息 | Audio metadata | 互动质量 |

#### 来自 Task

| Signal | 推理 |
|---|---|
| 用户取消了 task | 任务方向错了 |
| 用户多次修改 task description | 第一次描述不准 |
| 用户对 artifact 反馈 | 输出质量信号 |
| 周期 task 用户停用 | 价值不够 |

---

## 4. 反馈到哪里

```
Signal 提取
   ↓
分类
   ├── 偏好类      → Global Memory（"用户喜欢简洁"）
   ├── 知识类      → RAG / Memory（"用户提到自己在金融业"）
   ├── 兴趣类      → Memory + 推送过滤（场景 1 用户兴趣模型）
   ├── 风格类      → Global Memory + Agent prompt 调优
   └── 元能力类    → Agent metadata（"这个 agent 经常被用户中断 → 调短风格"）
```

详见 [Memory 模块](../modules/memory/)。

---

## 5. 闭环示例（场景 1：社媒推送）

```
Day 1:
  Sigma 推送 10 条
  → 用户点开 3 条（小红书 X 篇 / X 帖 X 条 / 知乎 X 篇）
  → Signal: "用户偏好 X 平台 + Y 主题"
  → 写入 task memory（场景 1 task 跨实例 memory）

Day 2:
  推送时优先 X 平台 + Y 主题
  → 用户点开 5 条（更准了）
  → 强化 signal

Day 7:
  Sigma 学到稳定模型 → 推送质量稳定提升
  → 用户开始反馈："这个分类的别推了"（显式负反馈）
  → 进一步细化模型
```

**关键**：每个 signal 都进 trace，可审计、可回退。

---

## 6. 设计原则

### 6.1 保守 > 激进

错误的 self-improvement 比没有 self-improvement 更糟——它会污染所有未来对话。

- **宁可漏，不写错**：低置信度的 signal 不写 memory，写也标 `confidence=low`
- **可解释**：每个 memory 记录来源（哪次对话 / 哪个 signal）
- **可审计**：CLI / Web UI 提供查看和编辑入口
- **可回退**：检测到 memory 错误时能回滚

### 6.2 用户掌控

- 默认开启，但用户可关闭某类 signal
- 用户可见 self-improvement 在做什么（"我刚刚学到你不喜欢 X"）
- 提供 `sigma memory edit` / `sigma improvement disable <category>`

### 6.3 不做模型层学习

Sigma 不微调模型、不训练 embedding。所有 self-improvement 通过：
- 写 memory（影响 context engineering）
- 调 prompt（影响 agent 行为）
- 调推送过滤（影响什么内容呈现给用户）

模型本身保持原样——可替换、可升级。

---

## 7. 实现位置

```
src/improvement/
  signals.py          # Signal 定义和提取协议
  extractors/         # 各种 signal extractor
    explicit.py       #   显式声明提取
    pattern.py        #   行为模式提取
    audio.py          #   Realtime audio signal（V4）
  pipeline.py         # Signal → 反馈路由
  audit.py            # 审计 / 可解释性 / 用户可见
```

---

## 8. 实现路径（V2-V5）

### V2：MVP

- 显式偏好提取（用户明说的）
- 简单的 memory 写入闭环
- 用户可见 / 可编辑

### V3：隐式 signal

- 文本对话的隐式 signal（重复问 / 换话题等）
- 周期 task 的反馈循环（场景 1 关键）

### V4：Realtime signal

- 跟 Realtime 模式同步上线
- Audio metadata signal 提取（打断 / 沉默）
- Self-improvement 闭环可观察

### V5：跨 session 学习

- 学习用户进步轨迹
- 跨 session 模式识别（"用户每周一情绪低落"）
- 主动调整 agent 风格 / 推送内容

---

## 9. 与其他模块关系

| 模块 | Self-improvement 怎么作用 |
|---|---|
| [Memory](../modules/memory/) | 主要写入目标 |
| [Realtime](realtime-mode.md) | 主要 signal 来源（V4 起） |
| [Trace](../modules/trace/) | 所有 signal 进 trace；可视化"agent 学到了什么" |
| [Agent](../modules/agent/) | Agent metadata 可被 self-improvement 修改（如"经常被打断" → 调短风格） |
| [Context](../modules/context/) | Memory 影响 context 拼装 |
| [Task](../modules/task/) | 周期 task 是 self-improvement 最大受益者（场景 1） |

---

## 10. 未决问题

| 问题 | 状态 |
|---|---|
| Signal 提取的精确 schema | V2 落地时定 |
| 自动 vs 用户确认（提取的 memory 默认开还是用户确认才生效） | V2 |
| Self-improvement 的"撤销"机制 | V3 |
| 跨 device 的 sync（多设备共享学习成果） | V5+ |
| 模型层学习（fine-tune）是否纳入 | 暂不（保持模型可替换） |

---

## 11. 相关文档

- [Memory 模块](../modules/memory/) — Self-improvement 的主要写入目标
- [Realtime 模式](realtime-mode.md) — Self-improvement 最佳落地
- [Trace 模块](../modules/trace/) — 可观察性和审计
- [架构总览](overview.md)
- [设计决策日志](design-log.md)
