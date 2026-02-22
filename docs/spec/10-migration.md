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
