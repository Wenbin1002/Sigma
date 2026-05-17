# Issue 11 — E2E 验收（4 条退出标准）

**Milestone**：v0.1
**Type**：`test(project)`
**依赖**：issue 10
**预计 PR 大小**：S

---

## 背景

roadmap § 0.1 列出 4 条退出标准。前 10 个 issue 各自有单元测试，但**端到端是否真的工作**需要一个统一的验收脚本。本 issue 把 4 条标准写成可重复执行的 e2e 测试。

## 范围

### 1. `tests/e2e/conftest.py`

- pytest fixture：临时 `~/.sigma/`（用 `tmp_path` + monkeypatch `HOME`）
- fixture：自动启 server + 关 server
- fixture：从环境变量读 `SIGMA_LLM_API_KEY` / `SIGMA_LLM_BASE_URL`，缺失时跳过 e2e（标记 `pytest.mark.skipif`）

### 2. `tests/e2e/test_exit_criteria.py`

每条退出标准一个测试：

#### 2.1 多轮对话

```python
def test_chat_multi_turn():
    sid = client.post("/chat/sessions").json()["id"]
    answer1 = collect_text(client.stream_chat(sid, "我叫小明"))
    answer2 = collect_text(client.stream_chat(sid, "我叫什么名字?"))
    assert "小明" in answer2
```

#### 2.2 工具调用流（read_file）

```python
def test_tool_call_read_file(tmp_path):
    f = tmp_path / "hello.txt"
    f.write_text("magic-token-9527")
    answer = run_chat(f"读 {f} 文件，告诉我里面写了什么")
    assert "magic-token-9527" in answer
    # 同时验证 trace 里出现了 read_file
    trace = load_latest_trace()
    assert any(e["type"] == "tool_call" and e["payload"]["tool"] == "read_file"
               for e in trace)
```

#### 2.3 Trace JSONL 含 llm_call / tool_call

```python
def test_trace_jsonl_contains_llm_and_tool():
    sid = ...
    run_chat_with_session(sid, "读 README 第一行")
    trace = load_latest_trace()
    types = {e["type"] for e in trace}
    assert "llm_call" in types
    assert "tool_call" in types
    assert "agent_start" in types and "agent_end" in types
```

#### 2.4 中断恢复

```python
def test_kill_and_resume():
    sid = client.post("/chat/sessions").json()["id"]
    # 启一个长 prompt（多轮 tool call）的 send，中途 kill server
    task = asyncio.create_task(client.stream_chat(sid, "..."))
    await asyncio.sleep(2)
    kill_server()
    # 重启 server
    start_server()
    # resume 同 session 再发一句
    answer = await collect_text(client.stream_chat(sid, "继续"))
    # 至少 session 还活着、能发新消息（不验证"恰好接上"，那是 LangGraph 的事）
    assert answer  # 非空响应
```

### 3. README 跑法

在本 issue PR 里更新 `tests/e2e/README.md`：

```bash
export SIGMA_LLM_API_KEY=...
export SIGMA_LLM_BASE_URL=https://your-relay/v1
pytest tests/e2e/ -v
```

### 4. `docs/roadmap.md` 退出标准勾选

PR 合并后，把 roadmap.md § 0.1 的四个 `[ ]` 改成 `[x]`，并在 0.1 节顶部把 🚧 改成 ✅。

## 设计要点

- **e2e 跑真 LLM**——必然花钱+不稳定。所以：
  - 单独一个 marker `@pytest.mark.e2e`，CI 默认不跑
  - `pytest -m e2e` 才会触发
  - 失败时给清晰报错（api_key 没设 / server 起不来 / LLM 超时）
- **不要在 e2e 里 mock LLM**——那是单元测试的职责，e2e 就是要踩真路径
- **prompt 写得保守**：用 read_file 而不是 web_search（web 不稳定），用「我叫小明」而不是依赖模型常识的 prompt——尽量减少 flake

## 验收标准

- [ ] 4 个测试全部通过（`pytest -m e2e tests/e2e/`）
- [ ] 缺 api_key 时整组测试跳过，不报错
- [ ] 测试结束后清理临时 `~/.sigma/`，不污染本机
- [ ] roadmap.md § 0.1 标志位更新

## 不做

- 不做 CI 集成（CI 跑钱+不稳，0.1 阶段先本地手跑）
- 不做性能基准（v2+）
- 不做覆盖率门槛（先有再说）

## 相关文档

- [roadmap.md § 0.1](../../../docs/roadmap.md)
- [v0.1 README](../README.md)
