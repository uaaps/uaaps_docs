## 6. Agents (Sub-agent Definitions)

Agents are **specialized personas** with domain expertise that can be selected automatically or manually. *(Note: This is distinct from `AGENTS.md`, which provides project-wide instructions).*

Agents are defined as a **directory** containing two files, which provides richer structured configuration than a single markdown file.

### Directory Structure

```
agents/
└── code-auditor/
    ├── agent.yaml        # REQUIRED — structured agent definition
    └── system-prompt.md  # REQUIRED — the agent's full system prompt
```

### `agent.yaml` Format

```yaml
name: code-auditor
description: "Security-focused code review agent"
version: 1.0.0

system_prompt: system-prompt.md    # Path to the system prompt file

# Skills this agent uses (resolved from package or dependencies)
skills:
  - pdf-tools                      # from this package
  - code-review                    # from dependency

# Commands this agent references
commands:
  - audit-finding                  # from this package

# Tool access (platform-dependent)
tools:
  - file_read
  - file_write
  - shell

# Behavioral parameters
parameters:
  temperature: 0.3
  style: professional
  output_format: markdown
```

### `system-prompt.md` Format

```markdown
---
description: Security review specialist for vulnerability analysis
capabilities:
  - OWASP Top 10 detection
  - Dependency vulnerability scanning
  - Secret detection
  - Cryptographic implementation review
---

You are a security review specialist. When analyzing code:

1. Check for injection vulnerabilities
2. Verify authentication patterns
3. Review cryptographic implementations
4. Scan for hardcoded secrets
5. Assess dependency security

Provide severity ratings (Critical/High/Medium/Low) with remediation.
```

### `agent.yaml` Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | **Yes** | Agent identifier. |
| `description` | `string` | **Yes** | Role summary for automatic selection. |
| `system_prompt` | `string` | **Yes** | Relative path to `system-prompt.md`. |
| `skills` | `string[]` | No | Skills this agent activates. |
| `commands` | `string[]` | No | Commands (including parameterized templates) this agent uses. |
| `tools` | `string[]` | No | Tool whitelist (platform-dependent). |
| `parameters` | `map` | No | Behavioral parameters (temperature, style, etc.). |

> **Legacy format**: A single `agent-name.md` file (without a separate `agent.yaml`) is also accepted for backward compatibility. In this case the frontmatter fields from the `system-prompt.md` table below apply directly to the single file.

### Platform Mapping

| Platform | Native Support | Migration |
|----------|---------------|-----------|
| Claude Code | ✅ Yes (`agents/` dir) | Direct |
| Cursor | ⚠️ No native `agents/` | Convert `system-prompt.md` → RULE.md with `alwaysApply: true` |
| GitHub Copilot | ✅ `.github/agents/*.agent.md` | Convert to `.agent.md` |
| Codex | ⚠️ Via skills | Convert to skill |
