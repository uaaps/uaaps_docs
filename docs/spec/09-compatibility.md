## 10. Cross-Platform Compatibility Matrix

### Artifact Support by Platform

| Artifact | Claude Code | Cursor | Copilot | Codex | Standard | Confidence |
|----------|------------|--------|---------|-------|----------|------------|
| **Skills (SKILL.md)** | ✅ Native | ✅ v2.2+ | ✅ `.github/skills/` | ✅ CLI+App | agentskills.io | Official |
| **Commands** | ✅ Native | ⚠️ Via rules | ✅ `.prompt.md` | ⚠️ Via skills | Package-defined | Official |
| **Agents** | ✅ Native (`agents/*.md`) | ⚠️ Via rules | ✅ `.agent.md` | ⚠️ Via skills | Package-defined | Official |
| **Rules / Instructions** | ✅ `CLAUDE.md` | ✅ `.mdc`/`RULE.md` | ✅ `.instructions.md` | ⚠️ `AGENTS.md` | Package-defined | Official |
| **AGENTS.md** | ✅ | ✅ | ✅ | ✅ | agents.md | Official |
| **Hooks** | ✅ 10 events | ✅ 5 events | ✅ 4 events | ❌ | Package-defined | Official |
| **MCP Servers** | ✅ Native | ✅ Native | ✅ CCA + VS Code | ⚠️ Limited | MCP Protocol | Official |

### Instruction File Compatibility

| File | Claude Code | Cursor | Copilot | Codex | Jules | Confidence |
|------|------------|--------|---------|-------|-------|------------|
| `AGENTS.md` | ✅ | ✅ | ✅ | ✅ | ✅ | Official |
| `CLAUDE.md` | ✅ | ✅* | ❌ | ❌ | ❌ | Official |
| `GEMINI.md` | ❌ | ✅* | ❌ | ❌ | ❌ | Community |
| `.cursor/rules/*.mdc` | ❌ | ✅ | ❌ | ❌ | ❌ | Official |
| `.github/copilot-instructions.md` | ❌ | ❌ | ✅ | ❌ | ❌ | Official |
| `.github/instructions/*.instructions.md` | ❌ | ❌ | ✅ | ❌ | ❌ | Official |
| `.github/agents/*.agent.md` | ❌ | ❌ | ✅ | ❌ | ❌ | Official |

> *Cursor reads `CLAUDE.md` and `GEMINI.md` as agent-specific instruction files alongside `AGENTS.md`.

### GitHub Copilot Artifact Types

| Artifact | Location | Format | Description |
|----------|----------|--------|-------------|
| **Repo instructions** | `.github/copilot-instructions.md` | Plain Markdown | Repository-wide guidance |
| **Path instructions** | `.github/instructions/*.instructions.md` | YAML frontmatter (`applyTo` glob) + Markdown | Path-scoped rules |
| **Custom agents** | `.github/agents/*.agent.md` | Markdown | Agent definitions (VS Code) |
| **Agent instructions** | `AGENTS.md` / `CLAUDE.md` / `GEMINI.md` | Plain Markdown | Agent-specific instructions |

#### Path Instructions Example

```markdown
---
applyTo: "src/**/*.ts"
---

# TypeScript Standards
- Use strict mode
- Prefer interfaces over type aliases
```
