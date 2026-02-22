## 11. Migration Guides

### From Claude Code Plugin → Universal Package

```
.claude-plugin/plugin.json    →  package.agent.json
commands/*.md                 →  commands/*.md (unchanged)
agents/*.md                   →  agents/<name>/system-prompt.md + agent.yaml (add structured def)
skills/*/SKILL.md             →  skills/*/SKILL.md (unchanged)
.claude/prompts/*.md          →  commands/*.md (add name field to frontmatter)
hooks/hooks.json              →  hooks/hooks.json (rename events)
.mcp.json                     →  mcp/servers.json (restructure)
CLAUDE.md                     →  AGENTS.md (rename)
```

### From Cursor Plugin → Universal Package

```
.cursor-plugin/plugin.json    →  package.agent.json
skills/*/SKILL.md             →  skills/*/SKILL.md (unchanged)
.cursor/prompts/*.md          →  commands/*.md (add name field to frontmatter)
rules/*.mdc                   →  rules/*/RULE.md (restructure frontmatter)
mcp.json                      →  mcp/servers.json (restructure)
.cursor/hooks.json            →  hooks/hooks.json (rename events)
AGENTS.md                     →  AGENTS.md (unchanged)
```

### From GitHub Copilot → Universal Package

```
.github/copilot-instructions.md    →  AGENTS.md (rename + restructure)
.github/instructions/*.instructions.md  →  rules/*/RULE.md (applyTo → globs)
.github/agents/*.agent.md          →  agents/*.md (remove .agent suffix)
AGENTS.md                          →  AGENTS.md (unchanged)
```

#### `.instructions.md` to `RULE.md` Frontmatter Mapping

```yaml
# Copilot (.instructions.md)        # Universal (RULE.md)
---                                  ---
applyTo: "src/**/*.ts"              globs: "src/**/*.ts"
---                                  alwaysApply: false
                                     description: "TypeScript standards"
                                     ---
```

### From `.mdc` Rule → `RULE.md`

```yaml
# .mdc (Cursor native)                    # RULE.md (Universal)
---                                        ---
description: React standards               description: React standards
globs: src/**/*.tsx                        globs: src/**/*.tsx
alwaysApply: false                         alwaysApply: false
---                                        ---
# Content...                               # Content...
```

The formats are functionally identical. Only the file extension changes.

---

## 11.1 Install Mode — Smooth Adoption Before Full Vendor Implementation

The `installMode` field in `package.agent.json` allows a package author to declare how the package SHOULD be deployed on platforms that have not yet natively implemented the UAAPS standard. This enables smooth adoption: one package can target both UAAPS-native environments and platforms still using their own plugin system.

### Mode Values

| Value | Install target | When to use |
|-------|---------------|-------------|
| `uaaps` (default) | `.agent-packages/` — full UAAPS layout | Platform supports UAAPS natively |
| `plugin` | Vendor-native location (see table below) | Platform uses its own plugin system |

### `installMode` in `package.agent.json`

```jsonc
{
  "name": "@myorg/code-review",
  "version": "1.0.0",

  // Omit entirely to use "uaaps" for all platforms (recommended default)
  "installMode": {
    "default": "uaaps",          // Fallback for any platform not listed
    "claude-code": "plugin",     // Deploy as native Claude Code plugin
    "cursor": "uaaps"            // Deploy via UAAPS on Cursor
  }
}
```

A scalar string shorthand MAY be used when the same mode applies to all platforms:

```jsonc
"installMode": "plugin"   // plugin mode on every platform
```

### Plugin Mode — Vendor-Native Deploy Targets

When `installMode` resolves to `"plugin"` for a given platform, the `aam` CLI translates and deploys artifacts to the platform's native location using the same mapping logic as `aam pkg build --target <platform>` (see §12.4):

| Platform | `plugin` mode install target |
|----------|-----------------------------|
| `claude-code` | `.claude/` (skills, commands, agents, hooks) |
| `cursor` | `.cursor/rules/` (rules as `.mdc`), `.cursor/skills/` |
| `copilot` | `.github/instructions/`, `.github/agents/`, `.github/skills/` |
| `codex` | `AGENTS.md` composite inject |

### CLI Override

Users MAY override the manifest declaration at install time:

```bash
# Force plugin mode regardless of manifest
aam install @myorg/code-review --install-mode plugin

# Force uaaps mode regardless of manifest
aam install @myorg/code-review --install-mode uaaps

# Target a specific platform's plugin layout
aam install @myorg/code-review --install-mode plugin --platform claude-code
```

### Deprecation Path

Once a platform vendor implements UAAPS natively, `"plugin"` mode for that platform becomes a no-op — the `aam` CLI SHOULD silently promote it to `"uaaps"` and emit an informational message:

```
⚠ installMode "plugin" for claude-code is ignored — platform supports UAAPS natively.
  Update package.agent.json to remove the per-platform override.
```

Package authors SHOULD remove per-platform `plugin` overrides once the target platform is listed as UAAPS-native in the compatibility matrix (§10).
