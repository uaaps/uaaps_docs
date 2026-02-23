# Changelog

All notable changes to the UAAPS specification are documented here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## [0.6.0-draft] — 2026-02-23

### Added
- §1 Introduction: Specification versioning strategy (major/minor/patch rules for the spec itself)
- §1 Introduction: Forward-compatibility parsing rules (consumers MUST ignore unknown fields)
- §1 Introduction: Conformance levels — Level 1 (Reader), Level 2 (Installer), Level 3 (Publisher)
- §1 Introduction: ABNF syntax definitions for all identifiers (package names, version ranges, skill refs, etc.)
- §3 Manifest: YAML round-trip guarantee (YAML 1.2, JSON-compatible types only)
- §3 Manifest: `dist-tags`, `resolutions`, and `deprecated` fields added to manifest schema
- §5 Commands: Namespace collision resolution rules
- §6 Agents: `system-prompt.md` frontmatter field reference table
- §9 MCP: `servers.json` field reference table
- §10 Compatibility: Platform capability negotiation protocol and `.platform-capabilities.json`
- §12.6 Registry: Structured error code constants, content negotiation, pagination, search/deprecation/yank endpoints
- §12.7 Packaging: Package lifecycle states (Published, Deprecated, Yanked) with immutability guarantee
- §13.4 Dependencies: Formalized resolution algorithm with named phases, priority ordering, pre-release handling
- §13.4 Dependencies: Resolver versioning (`resolverVersion` manifest field)
- §13.14 Dependencies: Workspace / monorepo support (`workspace.agent.json`)
- §14.5 Signing: SLSA-aligned provenance attestation (L0–L3) with in-toto format
- §15.3 Governance: Structured audit log format with JSON schema
- §16 Validation: New validation rules for resolver version, workspaces, forward-compat, permission grants, provenance
- §17.2 Security: Formal threat model (actors, surfaces, mitigations, residual risks)
- §17.3 Security: Transitive permission composition model with `permissionGrants`
- §18 Glossary: 12 new terms (Yank, Deprecation, Provenance, Conformance Level, Workspace, etc.)
- Implementers guide expanded from stub to full conformance guide
- All JSON schemas upgraded from Draft-07 to JSON Schema 2020-12
- Schema authority statement: prose spec is normative, schemas are informational

### Changed
- §12 Packaging: Subsections reordered to numerical sequence (§12.1–§12.6)
- §12.6 Registry: Status upgraded from "Skeleton" to "Draft" with substantially expanded content
- Introduction section headings unnumbered (resolves duplicate §2 in combined full.md)
- Permissions schema in `package-manifest.schema.json` fixed to match prose spec structure

### Fixed
- Broken `#evals-evals` anchor links verified correct (not broken — Python-Markdown collapses em-dash correctly)
- Stale changelog reference to `§5.5 Prompts` → `§5 Commands`
- Copilot issue `#1157` citation replaced with generic reference (unverifiable URL)
- Self-referential `§17.1` citation clarified in security section
- Appendix headings made consistent (Appendix A, Appendix B)
- Eval `engine` field: `copilot` and `cursor` values marked as reserved (unsupported until headless modes available)
- Model IDs in eval examples noted as illustrative

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
- §5 Commands: reusable parameterized prompt templates with variable interpolation
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
