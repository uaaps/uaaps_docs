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

### Platform Capability Discovery

Static compatibility matrices become outdated. To enable runtime discovery, platforms implementing UAAPS Level 2+ SHOULD expose a **capabilities object** queryable by packages and tools.

#### Capabilities Object Schema

```json
{
  "platform": "claude-code",
  "platform_version": "1.4.2",
  "uaaps_conformance_level": 2,
  "spec_version": "0.6.0",
  "supported_artifacts": ["skills", "commands", "agents", "rules", "hooks", "mcp"],
  "hook_events": [
    "pre-tool-use", "post-tool-use", "pre-prompt", "stop",
    "session-start", "session-end", "sub-agent-end",
    "permission-request", "pre-compact", "notification"
  ],
  "permissions_enforcement": {
    "fs": true,
    "network": true,
    "shell": true
  },
  "eval_headless": true,
  "resolver_version": 1
}
```

#### Capabilities Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `platform` | `string` | **Yes** | Platform identifier (e.g. `claude-code`, `cursor`, `copilot`, `codex`). |
| `platform_version` | `string` | **Yes** | Platform version string. |
| `uaaps_conformance_level` | `number` | **Yes** | Declared conformance level (1, 2, or 3). See §1 Conformance Levels. |
| `spec_version` | `string` | **Yes** | Highest UAAPS spec version supported. |
| `supported_artifacts` | `string[]` | **Yes** | Artifact types the platform can consume. |
| `hook_events` | `string[]` | **Yes** | Hook event names the platform fires. |
| `permissions_enforcement` | `object` | **Yes** | Which permission categories are enforced at runtime. |
| `eval_headless` | `boolean` | **Yes** | Whether the platform supports headless eval execution. |
| `resolver_version` | `number` | No | Dependency resolver version supported. Default `1`. |

#### Querying Capabilities

```bash
# CLI query
aam platform info
aam platform info --json

# Programmatic (within a hook or skill script)
# Read from well-known path: .agent-packages/.platform-capabilities.json
# Written by the platform on session start
```

Platforms MUST write the capabilities object to `.agent-packages/.platform-capabilities.json` at session start. Packages MAY read this file to adapt behavior based on platform support (e.g., falling back to `AGENTS.md` injection when hooks are unsupported).

#### Graceful Degradation

When a package requires an artifact type or hook event not listed in the platform's capabilities:

| Situation | Behavior |
|-----------|----------|
| Unsupported artifact type | `aam install` emits a `WARN`: "Platform does not support `<type>`. These artifacts will be ignored." Installation proceeds. |
| Unsupported hook event | Hook entry is skipped silently. Platform MAY log at debug level. |
| Missing permission enforcement | `aam install` emits a `WARN`: "Platform cannot enforce `<category>` permissions. Package runs with platform defaults." |
| Lower conformance level than package requires | `aam install` emits a `WARN` listing which features will be unavailable. |
