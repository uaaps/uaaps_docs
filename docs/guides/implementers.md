# Implementers Guide

!!! note "Stub"
    This guide is a placeholder. Content will be added in a future revision.

## Overview

This guide is for platform developers and tool authors who want to implement UAAPS support in their agent runtime, IDE, or toolchain.

## Key References

- [Introduction & Principles](../spec/00-introduction.md) — Understand the design goals
- [Package Manifest](../spec/02-manifest.md) — `package.agent.json` schema
- [Commands](../spec/04-commands.md) — Slash commands and parameterized templates
- [Compatibility Matrix](../spec/09-compatibility.md) — Per-platform artifact support
- [Migration Guides](../spec/10-migration.md) — Migrating existing plugin formats
- [Validation Rules](../spec/15-validation.md) — Conformance requirements

## Minimum Viable Implementation

To claim UAAPS compliance, a platform must minimally support:

1. Reading `package.agent.json` at the package root
2. Discovering skills from the `skills/` directory (or declared `artifacts.skills`)
3. Reading `SKILL.md` frontmatter for lazy-loading (`name` + `description`)
4. Activating the full `SKILL.md` body on demand

Support for commands, agents, hooks, and MCP servers is recommended but optional.
