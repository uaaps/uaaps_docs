# Changelog

All notable changes to the UAAPS specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.5.0-draft] — 2026-02-22

### Added
- §4 Skills: `tests/` directory structure for deterministic (assert) skill script testing
- §8 Hooks: `tests/` directory structure with fixtures and deterministic hook testing
- §2 Package Format: `evals/` directory expanded with full eval config, case format, and execution flow
- Evals system: LLM-judged integration tests via agent sandbox (`engine`, `judge`, `sandbox` config)
- Tests vs Evals comparison table clarifying the two-tier testing model
- `aam test` CLI for deterministic assert tests (skills + hooks)
- `aam eval` CLI for LLM-judged sandbox evaluations

---

## [0.4.0-draft] — 2026-02-19

### Added
- §12 Packaging & Distribution: archive format (`.aam`), scoped package names, dist-tags, portable bundles
- §13 Dependency Resolution: lock file format, resolution algorithm, conflict handling, CI/CD patterns
- §14 Package Signing & Verification: Sigstore, GPG, registry attestation, verification policy
- §15 Governance & Policy Gates: client-side policy gates, approval workflows, audit log events
- §5.5 Prompts: reusable parameterized prompt templates with variable interpolation
- MkDocs-based documentation structure replacing monolithic `SPECIFICATION.md`

### Changed
- Merged `DESIGN.md` content into main specification
- Agent definitions restructured as two-file directories (`agent.yaml` + `system-prompt.md`)

---

## [0.3.0] — 2026-01-15

### Added
- Initial hook event unification across Claude Code, Cursor, and Copilot
- `AGENTS.md` designated as universal fallback instruction format
- Cross-platform compatibility matrix

---

*Older entries pending backfill.*
