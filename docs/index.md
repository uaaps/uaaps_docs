# UAAPS — Universal Agentic Artifact Package Specification

**Write once. Deploy to any agent platform.**

UAAPS is the open standard for packaging AI agent artifacts — analogous to Docker for containers or npm for JavaScript packages. A single package works across Claude Code, GitHub Copilot, Cursor, OpenAI Codex, and any compliant platform.

---

## Why UAAPS?

| Problem | UAAPS Solution |
|---------|----------------|
| Skills written for Claude Code can't be used in Copilot | **Portable** — one package, all platforms |
| No lock file to reproduce agent behavior in CI | **Reproducible** — `package.agent.lock` pins exact versions |
| Multiple packages cause silent instruction conflicts | **Composable** — namespaced skills prevent collisions |
| No hooks for `pre-tool-use` or `permission-request` | **Lifecycle hooks** — 10 unified events |
| No way to propagate skill updates across projects | **Registry** — versioned distribution with the `aam` CLI |
| No standard way to verify skills and hooks work correctly | **Testable** — deterministic `tests/` + LLM-judged `evals/` |

---

## Quick Navigation

- **[Full Specification](spec/full.md)** — Complete specification in a single document
- **[Introduction & Principles](spec/00-introduction.md)** — Design goals and core principles
- **[Package Manifest](spec/02-manifest.md)** — `package.agent.json` schema reference
- **[Skills](spec/03-skills.md)** — The primary portable artifact
- **[Hooks](spec/07-hooks.md)** — Lifecycle event handlers
- **[Dependency Resolution](spec/12-dependencies.md)** — Lock files and version resolution
- **[Compatibility Matrix](spec/09-compatibility.md)** — Platform support at a glance
- **[Migration Guides](spec/10-migration.md)** — Migrating from Claude Code, Cursor, or Copilot
- **[Implementers Guide](guides/implementers.md)** — Getting started as a platform implementer

---

## Current Version

**0.5.0-draft** — See the [Changelog](changelog.md) for version history.

---

## Contributing

1. Read the [specification](spec/00-introduction.md)
2. Open an issue with feedback, edge cases, or missing scenarios
3. Submit a PR with spec improvements or examples
4. Test the [Skills Concentrator](https://github.com/spazyCZ/agent-package-manager)
