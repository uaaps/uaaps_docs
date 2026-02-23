# Implementers Guide

## Overview

This guide is for platform developers and tool authors who want to implement UAAPS support in their agent runtime, IDE, or toolchain. It defines what you need to build, how to test it, and how to declare your conformance level.

## Key References

- [Introduction & Principles](../spec/00-introduction.md) — Design goals, conformance levels, and formal syntax
- [Package Manifest](../spec/02-manifest.md) — `package.agent.json` schema
- [Skills](../spec/03-skills.md) — The primary portable artifact
- [Commands](../spec/04-commands.md) — Slash commands and parameterized templates
- [Compatibility Matrix](../spec/09-compatibility.md) — Per-platform artifact support and capability negotiation
- [Migration Guides](../spec/10-migration.md) — Migrating existing plugin formats
- [Validation Rules](../spec/15-validation.md) — Conformance requirements
- [Security](../spec/16-security.md) — Permissions model, threat model, and permission composition

## Conformance Levels

UAAPS defines three conformance levels. Each level subsumes all requirements of lower levels. See the [Introduction](../spec/00-introduction.md#conformance-levels) for full definitions.

### Level 1 — Reader (Minimum Viable)

Implement this first. It enables your platform to consume skills from any UAAPS package.

| Step | What to Build |
|------|--------------|
| 1 | Parse `package.agent.json` (JSON and optionally YAML). Extract `name`, `version`, `description`. |
| 2 | Scan `skills/` directory for subdirectories containing `SKILL.md`. |
| 3 | Parse YAML frontmatter from each `SKILL.md` to extract `name` and `description` (lazy metadata loading). |
| 4 | On skill activation, load the full markdown body and make `scripts/`, `references/`, `assets/` available. |
| 5 | Ignore unknown manifest fields, frontmatter keys, and `x-*` vendor extensions without error. |

**Test**: Install a sample UAAPS package with 2-3 skills. Verify skills are discoverable by name and activatable.

### Level 2 — Installer (Full Consumption)

| Step | What to Build |
|------|--------------|
| 6 | Discover and invoke commands from `commands/` (markdown files with frontmatter). |
| 7 | Discover and activate agents from `agents/` (`agent.yaml` + `system-prompt.md`). |
| 8 | Discover and apply rules from `rules/` (glob-scoped `RULE.md` files). |
| 9 | Load `hooks/hooks.json` and execute hook commands at lifecycle events your platform supports. |
| 10 | Load `mcp/servers.json` and launch MCP servers. |
| 11 | Resolve dependencies from `package.agent.lock` (at minimum, support `--frozen` locked installs). |
| 12 | Enforce `permissions` (fs, network, shell) at runtime when the field is present. |
| 13 | Run `systemDependencies` pre-flight checks before installation. |
| 14 | Write `.agent-packages/.platform-capabilities.json` at session start. |

**Test**: Install a package with skills, commands, agents, rules, hooks, and MCP servers. Verify all artifact types function. Install a package with `permissions` and verify enforcement.

### Level 3 — Publisher (Full Lifecycle)

| Step | What to Build |
|------|--------------|
| 15 | Support `aam init` to scaffold a new package, `aam validate` to check conformance. |
| 16 | Support `aam pkg pack` to create `.aam` archives. |
| 17 | Support at least checksum + one signing method (Sigstore recommended). |
| 18 | Support `aam publish` to HTTP and filesystem registries. |
| 19 | Support `aam test` (deterministic assert tests) and `aam eval` (LLM-judged evaluations). |
| 20 | Support client-side policy gates from `config.yaml`. |
| 21 | Support generating and verifying SLSA-aligned provenance attestations. |

**Test**: Author a package, run tests and evals, publish to a local filesystem registry, install from the registry in a clean environment.

## Declaring Conformance

Your platform MUST declare its conformance level in two places:

1. **`aam platform info` output** (or equivalent):
```
UAAPS Conformance: Level 2 (Installer)
Spec Version: 0.6.0
Platform: my-agent-runtime v3.1.0
```

2. **Platform capabilities file** (`.agent-packages/.platform-capabilities.json`):
```json
{
  "platform": "my-agent-runtime",
  "platform_version": "3.1.0",
  "uaaps_conformance_level": 2,
  "spec_version": "0.6.0",
  "supported_artifacts": ["skills", "commands", "agents", "rules", "hooks", "mcp"],
  "hook_events": ["pre-tool-use", "post-tool-use", "stop"],
  "permissions_enforcement": { "fs": true, "network": false, "shell": true },
  "eval_headless": false,
  "resolver_version": 1
}
```

## Common Implementation Pitfalls

| Pitfall | Guidance |
|---------|---------|
| Rejecting unknown manifest fields | MUST ignore them — this breaks forward compatibility |
| Hardcoding platform-specific paths | Use `${PACKAGE_ROOT}` variable for package-relative paths |
| Loading all skills eagerly | Use progressive disclosure: metadata first, body on activation |
| Ignoring `permissions` field | If present, enforcement is mandatory for Level 2+ |
| Mixing project and global lock files | A global install MUST NOT read `./package.agent.lock` |
| Treating yank as unpublish | Yanked versions MUST remain downloadable for existing lock files |

## Schema Validation

Use the JSON Schema files in [`docs/schemas/`](../schemas/README.md) for programmatic validation of all structured files. The schemas target JSON Schema 2020-12 and can be used with `ajv`, `jsonschema`, or any compliant validator.

> **Important**: The prose specification is authoritative. The JSON Schema files are informational aids. In case of conflict, the prose governs.
