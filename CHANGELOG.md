# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
