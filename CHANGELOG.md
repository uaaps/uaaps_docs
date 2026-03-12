# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.7.0-draft] — 2026-02-23

### Added
- §7 Rules: Full Universal Agent Rule Specification (UARS v1.3) — replaces the previous Cursor-centric stub
- Two CLI namespaces for rules: `aam pkg rules` (package authoring) and `aam convert rules` (standalone conversion)
- `aam pkg rules compile/import/diff/clean/list` sub-commands
- `aam convert rules -s <platform> -t <platform>` with standardised flags (`-s`, `-t`, `--type`, `--dry-run`, `--force`, `--verbose`)
- UARS `apply.mode` enum: `always` | `intelligent` | `files` | `manual`
- Mode mapping table and per-platform transformation tables for all four platforms
- Conflict detection with generation guards and `.aam/conflicts/` pending files
- Idempotency guarantee and watch mode for `aam pkg rules compile`
- AGENTS.md merge strategy (directory inference, root aggregation, guard-block management)
- Codex `AGENTS.override.md` support via `agents.codex.override: true`
- Validation rules, error exit codes (0–4), and worked migration examples
- `docs/schemas/rule-frontmatter.schema.json` updated to UARS v1.3 (`name`, `apply.*`, `agents.*`, `refs`)

### Changed
- `agents.*` override block now limited to platform-specific capabilities only (`enabled`, `alwaysApply`/`priority` for Cursor, `override` for Codex); scope/path fields removed
- Standalone conversion command renamed from `aam converter rules` → `aam convert rules`
- Flags renamed: `--from`/`--to` → `--source-platform`/`-s` and `--target-platform`/`-t`

## [0.5.0-draft] — 2026-02-22

### Added
- Skill testing: `tests/` directory with deterministic assert runner (§4 Skills)
- Hook testing: `tests/` directory with fixtures and assert runner (§8 Hooks)
- Evals system: top-level `evals/` with agent sandbox execution model (§2 Package Format)
- Two-tier testing model: `tests/` (assert, CI-safe, $0) vs `evals/` (LLM-judged, agent sandbox)
- `aam test` and `aam eval` CLI commands

## [0.4.0-draft] — 2026-02-19

### Added
- Initial specification covering package format, manifest, skills, commands, prompts, agents, rules, hooks, MCP, compatibility matrix, migration guides, packaging & distribution, dependency resolution, signing & verification, governance, validation rules, security considerations, glossary, and appendix.
- MkDocs-based documentation site with Material theme.
- GitHub Actions workflow for automated GitHub Pages deployment.
