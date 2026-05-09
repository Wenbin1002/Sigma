# Tools

> 让 Agent 能调用外部工具：搜索、读写文件、执行代码、操作浏览器。

## 在架构中的位置

- **所属层**: Adapters
- **Port 接口**: `src/ports/tools.py` → `ToolPort`
- **实现目录**: `src/tools/`
- **被谁调用**: Agent Runtime（内部 Tool Router）
- **依赖**: MCP Server、外部工具服务

## Port 接口定义

```python
class ToolPort(Protocol):
    async def execute(self, name: str, params: dict) -> str: ...
    async def list_tools(self) -> list[ToolDef]: ...
```

## 可用实现

| 实现 | 类名 | 文件 | 配置值 | 状态 |
|------|------|------|--------|------|
| Native | `NativeTools` | `src/tools/native.py` | `native` | V0 |
| MCP | `MCPTools` | `src/tools/mcp.py` | `mcp` | V1 planned |

## 工作原理

Agent 在决策循环中判断需要调用工具时，通过 ToolPort 执行。工具系统负责：

1. **工具发现**：列出可用工具及其参数定义
2. **风险分级**：标记每个工具的风险等级（read / write / destructive）
3. **执行控制**：高风险操作暂停并请求用户确认
4. **结果返回**：工具执行结果以字符串形式返回给 Agent

MCP（Model Context Protocol）模式下，工具通过标准协议动态发现和调用，支持远程 MCP Server。

## 配置示例

```yaml
tools:
  provider: native
  # 或
  provider: mcp
  mcp_servers:
    - url: "http://localhost:3000"
```

## 风险分级

| 级别 | 说明 | 用户确认 |
|------|------|---------|
| `read` | 只读操作（搜索、查看） | 不需要 |
| `write` | 写入操作（创建文件、发消息） | 可配置 |
| `destructive` | 破坏性操作（删除、修改系统） | 必须确认 |

## 扩展方式

### 添加新工具

1. 创建文件 `src/tools/<tool_set>.py`
2. 实现 `ToolPort` Protocol
3. 注册到 `src/tools/registry.py`
4. 添加配置项

或直接接入现有 MCP Server。

### 约束

- 工具执行结果必须是可序列化的字符串
- 高风险操作必须有确认机制
- 工具执行应有超时控制

## 演进路径

| 阶段 | 能力 | 候选方案 |
|------|------|---------|
| V0 | 几个硬编码 Python 函数 | Native |
| V1 | MCP 协议接入，动态工具发现 | MCP Client |
| V2 | 沙箱执行、权限模型、工具组合 | Docker sandbox |
| V3 | 工具市场、用户自定义工具、运行时注册 | MCP Server 生态 |

## 可扩展能力

- 风险分级自动化（静态分析工具签名）
- 工具结果缓存（相同输入不重复执行）
- 工具编排（一个高级工具 = 多个底层工具的 workflow）
- 浏览器操作、代码执行、文件系统操作作为标准工具集
- 用户自定义工具运行时热加载

## 相关文档

- [Agent 模块](../agent/) — 工具的调用方
- [Ports & Adapters](../../architecture/ports-and-adapters.md)
- [运行模式](../../architecture/runtime-modes.md) — Realtime 模式下的 tool calling
