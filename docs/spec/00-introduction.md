# Universal Agentic Artifact Package Specification (UAAPS)

> **Version**: 0.5.0-draft  
> **Date**: 2026-02-22 (revised)  
> **Purpose**: Define a universal, portable package standard for AI agent artifacts — analogous to Docker for containers or npm for JavaScript packages.  
> **Goal**: Write once, deploy to any agent platform. Simplify migration. Eliminate vendor lock-in.

---

## 1. Design Principles

| Principle | Description |
|-----------|-------------|
| **Portability** | A single package works across Claude Code, Cursor, Copilot, Codex, and any compliant platform. |
| **Progressive Disclosure** | Metadata loads first; full instructions load on activation; resources load on-demand. |
| **Convention over Configuration** | Standard directory names eliminate per-platform mapping; overrides are optional. |
| **Filesystem-First** | All artifacts are files on disk — no databases, no APIs required, no binary formats. |
| **Composability** | Packages can be installed together; namespacing prevents conflicts. |
| **Deterministic Resolution** | Lock files ensure reproducible installs across machines and CI. |
| **Backward Compatibility** | Existing Claude Code plugins and Cursor plugins work with minimal changes. |
