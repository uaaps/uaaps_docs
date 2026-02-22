## 5. Commands (Slash Commands & Reusable Templates)

Commands are **markdown prompt files** that serve two roles:

1. **User-invoked** — triggered by typing `/command-name`.
2. **Reusable templates** — referenced by agents and skills via `{{variable}}` interpolation.

A command with no `variables` is a simple slash command. A command with `variables` is a parameterized template that can also be referenced by agents and skills.

### Simple Command

```markdown
---
name: review
description: Run a comprehensive code review on the current file
---

Review the current file for:
1. Code organization and structure
2. Error handling patterns
3. Performance implications
4. Security vulnerabilities
5. Test coverage gaps

Provide specific, actionable feedback with line references.
```

### Parameterized Command (Template)

```markdown
---
name: audit-finding
description: "Structured command for documenting a single audit finding"
variables:
  - name: control_id
    description: "Control identifier"
    required: true
  - name: severity
    description: "Finding severity level"
    required: true
    enum: [critical, high, medium, low]
    default: medium
  - name: evidence
    description: "Supporting evidence"
    required: false
    default: "No evidence provided"
---

# Audit Finding: {{control_id}}

## Severity: {{severity}}

Analyze the following evidence and produce a structured finding:

{{evidence}}
```

### Frontmatter Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | **Yes** | Identifier, `[a-z0-9-]`, max 64 chars. |
| `description` | `string` | Recommended | Shown in command listings and help. |
| `variables` | `Variable[]` | No | Template variables with optional defaults and enums. |

Each variable in the `variables` array has:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | **Yes** | Variable identifier used in `{{name}}` interpolation. |
| `description` | `string` | No | Human-readable description. |
| `required` | `boolean` | No | Whether the variable must be provided. Default: `false`. |
| `enum` | `string[]` | No | Allowed values. |
| `default` | `string` | No | Default value when not provided. |

### Platform Mapping

| Platform | Native Support | Location | Invocation |
|----------|---------------|----------|------------|
| Claude Code | ✅ Yes | `commands/` (.md files), `.claude/prompts/` | `/plugin:command` |
| Cursor | ✅ Prompts / ⚠️ Commands via rules | `.cursor/prompts/` | No native `/command` from plugins |
| GitHub Copilot | ✅ Yes | `.github/prompts/<name>.prompt.md` | Via prompt picker |
| Codex | ⚠️ Via skills | Skills with `disable-model-invocation` | `$skill-name` |

### Migration Strategy
For platforms without native command support, commands can be **converted to skills** with `disable-model-invocation: true` to achieve similar user-initiated behavior.
