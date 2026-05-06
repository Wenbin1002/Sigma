# Tools

让 Agent 能调用外部工具：搜索、读写文件、执行代码、操作浏览器。

## 特性

- 工具注册和发现
- 风险分级（read / write / destructive）
- 人工确认（高风险操作前暂停）
- 支持 MCP 协议

## 可用实现

| Adapter | 说明 | 配置值 |
|---------|------|--------|
| Native | 硬编码的工具函数 | `native` |
| MCP | 通过 MCP 协议动态发现工具 | `mcp` |

## 扩展

实现 `ToolPort` 接口 + 注册即可。或直接接入现有 MCP Server。
