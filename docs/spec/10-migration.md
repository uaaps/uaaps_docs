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

### 11.1 Install Mode — Smooth Adoption Before Full Vendor Implementation

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
| `codex` | `.agents/skills/` + `AGENTS.md` |

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

### 11.2 Future Build Target — UAAPS Package → Codex Native Structure

This section defines the intended conversion model for a future toolchain that exports a UAAPS package into a Codex-native repository layout.

The primary entry points are expected to be:

```bash
aam pkg build --target codex
aam install ./my-package --install-mode plugin --platform codex
```

The Codex export is a **lossy platform build target**. The tool MUST preserve the portable meaning of supported artifacts where possible, and MUST emit warnings for UAAPS concepts that Codex does not natively model.

### Output Layout

A Codex-native export SHOULD produce a repository tree shaped like:

```text
<output-root>/
├── .agents/
│   └── skills/
│       ├── <skill-name>/
│       │   ├── SKILL.md
│       │   ├── scripts/
│       │   ├── references/
│       │   ├── assets/
│       │   └── agents/
│       │       └── openai.yaml
│       ├── cmd-<command-name>/
│       │   ├── SKILL.md
│       │   └── agents/
│       │       └── openai.yaml
│       └── agent-<agent-name>/
│           ├── SKILL.md
│           └── agents/
│               └── openai.yaml
└── AGENTS.md
```

`AGENTS.md` is OPTIONAL and is emitted only when the package contains root instructions or rules that can be compiled into Codex instruction files.

### Artifact Mapping

| UAAPS artifact | Codex-native output | Conversion behavior |
|----------------|---------------------|---------------------|
| `skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` | Copy skill body and supported frontmatter. |
| `skills/<name>/scripts/`, `references/`, `assets/` | Same relative path under `.agents/skills/<name>/` | Copy verbatim. |
| UAAPS skill vendor metadata | `.agents/skills/<name>/agents/openai.yaml` | Emit only when Codex-specific metadata is available or derived. |
| `commands/<name>.md` | `.agents/skills/cmd-<name>/SKILL.md` | Convert command to a generated skill for explicit user invocation. |
| `agents/<name>/agent.yaml` + `system-prompt.md` | `.agents/skills/agent-<name>/SKILL.md` | Convert agent definition into a generated skill persona. |
| `rules/` + root `AGENTS.md` | `AGENTS.md` | Compile/merge into Codex instruction file(s). |
| `hooks/` | No native output | Emit warning: unsupported on Codex. |
| `mcp/servers.json` | No direct 1:1 file | Emit Codex-side metadata only when exact mapping exists; otherwise warn. |
| `tests/`, `evals/` | No runtime output | Omit from native runtime layout; MAY remain in bundle metadata. |
| `package.agent.json` | Not consumed by Codex runtime | MAY be retained in bundle/reference metadata, but MUST NOT be required at runtime. |

### Skill Conversion Rules

For a UAAPS skill exported to Codex:

1. The tool MUST preserve `name`, `description`, `license`, `metadata`, and `allowed-tools` when they are representable in Codex.
2. The tool SHOULD preserve the `SKILL.md` body unchanged except for path rewrites needed to keep relative references valid.
3. The tool MUST copy `scripts/`, `references/`, and `assets/` directories when present.
4. The tool SHOULD emit `agents/openai.yaml` only for Codex-specific metadata and policy, not for portable fields already represented in `SKILL.md`.

### Command Conversion Rules

Codex does not define a native slash-command artifact. A UAAPS command export therefore uses a generated skill:

1. Each command MUST be converted into a generated skill directory named `cmd-<command-name>` unless the user requests a different naming strategy.
2. The generated `SKILL.md` MUST carry a description explaining that it is the Codex-native form of the source UAAPS command.
3. The command template body MUST become the skill instruction body.
4. Command variables SHOULD be rendered into the generated instructions as explicit fill-in guidance or parameter prompts.
5. The exporter SHOULD write `agents/openai.yaml` with `policy.allow_implicit_invocation: false` so the generated skill behaves like an explicit user-invoked action rather than an always-eligible implicit skill.

### Agent Conversion Rules

Codex does not define a native `agents/` artifact separate from skills. A UAAPS agent export therefore uses a generated skill:

1. Each agent MUST be converted into a generated skill directory named `agent-<agent-name>` unless the user requests a different naming strategy.
2. The generated `SKILL.md` MUST combine:
   - the agent description from `agent.yaml`
   - the behavioral instructions from `system-prompt.md`
3. Referenced UAAPS skills SHOULD remain separate skills and SHOULD be mentioned in the generated instructions only when needed for discovery or orchestration guidance.
4. Structured fields from `agent.yaml` that have no Codex-native equivalent MUST trigger a warning and MAY be serialized into vendor metadata for informational use only.

### Rules And `AGENTS.md`

Rules and repository-wide instructions SHOULD be compiled into Codex instruction files using the Codex rules conversion pipeline defined in §7.

When both are present:

1. Root package `AGENTS.md` content SHOULD appear first.
2. Converted rule content SHOULD be appended in deterministic order.
3. If a rule targets a subdirectory-specific Codex instruction file, the exporter SHOULD emit the appropriate `AGENTS.md` or `AGENTS.override.md` file in that directory.

### Unsupported Or Partial Features

The Codex export tool MUST emit a `WARN` for each artifact or field that cannot be represented faithfully. At minimum:

| UAAPS feature | Codex export behavior |
|---------------|-----------------------|
| Hooks | Warn and omit. |
| Hook tests | Warn and omit. |
| Package permissions | Warn if the package depends on host-enforced permissions not available in Codex native export. |
| MCP server config without Codex metadata equivalent | Warn and omit direct conversion. |
| Agent tool whitelist fields with no Codex equivalent | Warn and inline guidance only if safe. |
| Any unknown vendor extension affecting runtime behavior | Warn and preserve only as reference metadata if possible. |

Warnings MUST identify the source path and the reason the information could not be preserved exactly.

### Determinism

For the same package input, platform target, and build options, the Codex export tool MUST produce byte-for-byte stable generated file content except for explicitly declared build metadata such as bundle timestamps.
