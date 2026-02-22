## 7. Rules (Project Rules / Instructions)

Rules provide **persistent context** — coding standards, architecture conventions, and project-specific guidance.

### Universal Format: `RULE.md`

```markdown
---
description: "TypeScript coding standards for this project"
globs: "src/**/*.{ts,tsx}"
alwaysApply: false
---

# TypeScript Standards

- Use strict mode
- Prefer `const` over `let`
- Use early returns
- Named exports only
- Co-locate tests as `*.test.ts`
```

### Frontmatter Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `description` | `string` | **Yes** | Purpose of rule. Used for agent auto-selection. |
| `globs` | `string` | No | File patterns for auto-attachment. |
| `alwaysApply` | `boolean` | No | Always include in context. Default: `false`. |

### Platform Mapping

| Platform | Native Format | Migration |
|----------|--------------|-----------|
| Claude Code | `CLAUDE.md` (freeform) | Copy content into `CLAUDE.md` |
| Cursor | `.mdc` / `RULE.md` in `.cursor/rules/` | Copy to `.cursor/rules/` |
| Codex | `AGENTS.md` | Copy into `AGENTS.md` sections |
| Universal | `AGENTS.md` (project root) | Direct (supported everywhere) |

### `AGENTS.md` as Universal Fallback
`AGENTS.md` at the project root is the **most portable instruction format** — supported by Claude Code, Cursor, Codex, Copilot, Gemini CLI, Jules, Amp, and others. Every universal package SHOULD include an `AGENTS.md` for maximum compatibility.
