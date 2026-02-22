## Appendix: Confidence Legend

| Level | Meaning | Example |
|-------|---------|---------|
| **Official** | Verified against vendor documentation | Hook event names, manifest schema |
| **Community** | Confirmed by multiple community sources | Pseudo-XML rule structure, Cursor RULE.md folders |
| **Inferred** | Derived from behavior or early previews | Parallel agent counts, some adoption claims |

Claims marked as Inferred should be validated before building production tooling.

---

## Appendix A: Comparison of Key Differences

| Aspect | Claude Code | Cursor |
|--------|------------|--------|
| Manifest location | `.claude-plugin/plugin.json` | `.cursor-plugin/plugin.json` |
| Rule format | `CLAUDE.md` (freeform) | `.mdc` / `RULE.md` (YAML frontmatter) |
| Rule scoping | File-based hierarchy | `globs`, `alwaysApply`, agent-decided |
| Hook config | `hooks/hooks.json` | `.cursor/hooks.json` |
| Hook events | `PreToolUse`, `PostToolUse`, `Stop`... | `beforeShellCommand`, `afterFileEdit`, `stop`... |
| Skill standard | agentskills.io (creator) | agentskills.io (adopter, v2.2+) |
| Commands | Native `commands/` dir | Via rules or skills |
| Sub-agents | Native `agents/` dir | Via rules or imported |
| Team rules | N/A (org-wide via API) | Dashboard-managed (Team/Enterprise) |
| Distribution | `/plugin marketplace`, npm, git | Plugin marketplace, git |
| Root variable | `${CLAUDE_PLUGIN_ROOT}` | Working directory relative |
| MCP config (plugin) | `.mcp.json` (leading dot) | `mcp.json` (no dot) |
| MCP config (project) | `.mcp.json` | `.cursor/mcp.json` |
| MCP config (global) | `~/.claude/.mcp.json` | `~/.cursor/mcp.json` |
| Instruction files | `CLAUDE.md` | `AGENTS.md`, `CLAUDE.md`, `GEMINI.md` |
| Dependency resolution | N/A (no native resolver) | N/A (no native resolver) |
| System dep checks | Via skill `compatibility` field | N/A |
| Lock file | N/A | N/A |

---

*This specification is a living document. Contributions welcome at [github.com/spazyCZ/agent-package-manager](https://github.com/spazyCZ/agent-package-manager).*
