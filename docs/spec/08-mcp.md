## 9. MCP Server Configuration

Model Context Protocol servers extend agent capabilities with external tool access.

### Universal `servers.json` Format

```json
{
  "servers": {
    "my-service": {
      "command": "node",
      "args": ["${PACKAGE_ROOT}/mcp-server/index.js"],
      "env": {
        "API_KEY": "${API_KEY}"
      }
    }
  }
}
```

### Platform Mapping

| Platform | Plugin Location | Project Location | Global Location | Root Variable |
|----------|----------------|-----------------|----------------|---------------|
| Claude Code | `.mcp.json` at plugin root | `.mcp.json` at project root | `~/.claude/.mcp.json` | `${CLAUDE_PLUGIN_ROOT}` |
| Cursor | `mcp.json` at plugin root | `.cursor/mcp.json` | `~/.cursor/mcp.json` | Working dir relative |
| Universal | `mcp/servers.json` | `mcp/servers.json` | N/A | `${PACKAGE_ROOT}` |

> **Critical**: Cursor plugin MCP uses `mcp.json` (no dot prefix, no `.cursor/` wrapper). Project/global MCP uses `.cursor/mcp.json`. Claude Code always uses `.mcp.json` (leading dot).
