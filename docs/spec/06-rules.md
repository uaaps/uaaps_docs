## 7. Rules (Project Rules / Instructions)

**Version:** 1.3.0  
**Purpose:** A platform-agnostic rule definition format that can be compiled into agent-specific configuration files for GitHub Copilot, Cursor, Claude Code, and AGENTS.md (OpenAI Codex / open standard).

---

### Overview

A Universal Rule is a single source of truth for coding conventions, project standards, and agent behaviors. A compiler/converter reads UARS files and emits the correct format for each target platform.

```
.aam/rules/
  ├── python-standards.rule.md
  ├── api-conventions.rule.md
  ├── service-patterns.rule.md
  └── ...
```

---

### Rule File Format

Each rule is a Markdown file with a YAML frontmatter block followed by freeform Markdown content.

#### File naming

```
<rule-name>.rule.md
```

#### Full schema

```yaml
---
# ─── Identity ───────────────────────────────────────────────────────────────
name: string          # [required] Unique rule identifier (slug-style, e.g. "python-standards")
version: string       # [optional] Semver string. Defaults to "1.0.0"
description: string   # [recommended] One-line summary shown in UIs and hover tooltips

# ─── Applicability ──────────────────────────────────────────────────────────
apply:
  mode: always | intelligent | files | manual
  #   always      → injected into every session / all files
  #   intelligent → agent decides based on description + context
  #   files       → applied when any matched path is in context
  #   manual      → only when explicitly referenced by the user
  globs:          # [required when mode=files] list of glob patterns
    - "src/**/*.py"
    - "tests/**/*.py"

# ─── Agent overrides ────────────────────────────────────────────────────────
# Platform-specific features not expressible in the general apply: config.
# Use these only for capabilities unique to a given platform.
agents:
  cursor:
    priority: number   # [optional] Cursor-specific priority override (higher = more likely to be selected) — Inferred; not in Cursor's public .mdc schema
    alwaysApply: false      # pin rule as always-on regardless of apply.mode

# ─── References ─────────────────────────────────────────────────────────────
refs:                 # [optional] other rule names or template files to attach
  - service-template  # maps to Cursor's @file references
---
```

#### Rule content (Markdown body)

The body is plain Markdown. It becomes the instruction content in all target platforms.

```markdown
# Python Coding Standards

- Follow the PEP 8 style guide.
- Use type hints for all function signatures.
- Write docstrings for all public functions and classes.
- Use 4 spaces for indentation.
- Prefer `pathlib.Path` over `os.path` for file system operations.
```

---

### Mode Mapping Table

| UARS `apply.mode` | Copilot output | Cursor output | Claude Code output | AGENTS.md output |
|---|---|---|---|---|
| `always` | `applyTo: "**"` in frontmatter | `alwaysApply: true` | Listed in `CLAUDE.md` body (no paths filter) | Appended to root `AGENTS.md` |
| `intelligent` | No `applyTo` (manual attach) | `alwaysApply: false`, no `globs` | Listed in `CLAUDE.md` with note for agent | Appended to root `AGENTS.md` under a labelled section |
| `files` | `applyTo: <glob>` | `alwaysApply: false`, `globs: <glob>` | `paths:` list in `.claude/rules/<n>.md` | Appended to inferred or explicit subdirectory `AGENTS.md` |
| `manual` | No `applyTo` (manual attach) | `alwaysApply: false`, no `globs` | Not auto-included; referenced explicitly | Not emitted (AGENTS.md has no manual-only concept) |

---

### Compiled Output Examples

Given the rule below:

```yaml
---
name: python-standards
description: Coding conventions for Python files
apply:
  mode: files
  globs:
    - "**/*.py"
---
# Python Coding Standards
- Follow PEP 8.
- Use type hints for all function signatures.
- Write docstrings for public functions.
- Use 4 spaces for indentation.
```

#### → GitHub Copilot

**Path:** `.github/instructions/python-standards.instructions.md`

```markdown
---
description: Coding conventions for Python files
applyTo: '**/*.py'
---
# Python Coding Standards
- Follow PEP 8.
- Use type hints for all function signatures.
- Write docstrings for public functions.
- Use 4 spaces for indentation.
```

#### → Cursor

**Path:** `.cursor/rules/python-standards.mdc`

```markdown
---
description: Coding conventions for Python files
globs: **/*.py
alwaysApply: false
---
# Python Coding Standards
- Follow PEP 8.
- Use type hints for all function signatures.
- Write docstrings for public functions.
- Use 4 spaces for indentation.
```

#### → Claude Code

**Path:** `.claude/rules/python-standards.md`

```markdown
---
paths:
  - "**/*.py"
---
# Python Coding Standards
- Follow PEP 8.
- Use type hints for all function signatures.
- Write docstrings for public functions.
- Use 4 spaces for indentation.
```

#### → AGENTS.md (OpenAI Codex / open standard)

**Path:** `AGENTS.md` (root — for `always` / `intelligent` modes)  
**Path:** `src/AGENTS.md` (subdirectory — compiler infers from `apply.globs` for `files` mode)

> AGENTS.md has no frontmatter or glob fields. Scope is determined entirely by the **directory** the file is placed in. The compiler groups all rules targeting the same directory into a single `AGENTS.md` file.

```markdown
<!-- AUTO-GENERATED BY AAM — python-standards -->
# Python Coding Standards
- Follow PEP 8.
- Use type hints for all function signatures.
- Write docstrings for public functions.
- Use 4 spaces for indentation.
```

---

### Field Reference

#### Top-level fields

| Field | Required | Type | Description |
|---|---|---|---|
| `name` | ✅ | `string` | Unique slug identifier for the rule |
| `version` | ❌ | `string` | Semver version string |
| `description` | ✅ | `string` | One-line summary; used in Copilot hover and Cursor agent decisions |
| `apply.mode` | ✅ | `enum` | `always` \| `intelligent` \| `files` \| `manual` |
| `apply.globs` | ⚠️ | `string[]` | Required when `mode=files`. Glob patterns relative to workspace root |
| `agents` | ❌ | `object` | Per-platform overrides |
| `refs` | ❌ | `string[]` | Rule names or template file paths to attach (Cursor `@file` style) |

#### `agents.*` override fields

| Field | Platform | Description |
|---|---|---|
| `enabled` | all | Whether to emit output for this platform |
| `alwaysApply` | `cursor` | Pin rule as always-on regardless of `apply.mode` (Cursor-specific) |
| `override` | `codex` | Emit `AGENTS.override.md` instead of `AGENTS.md` for full subdirectory override (Codex-specific) |

---

### Compiler Behavior

A UARS compiler (`aam compile` or equivalent) reads `.aam/rules/*.rule.md` files and writes platform-specific outputs.

#### Output path conventions

| Platform | Output path pattern |
|---|---|
| Copilot | `.github/instructions/<name>.instructions.md` |
| Cursor | `.cursor/rules/<name>.mdc` |
| Claude Code | `.claude/rules/<name>.md` (+ entry in `CLAUDE.md` for `always` mode) |

#### Merge strategy for `always` mode rules (Claude Code)

Rules with `mode: always` are injected directly into `CLAUDE.md` under a generated section header. Rules with `mode: files` or others are written as separate files under `.claude/rules/` and referenced from `CLAUDE.md`.

```markdown
<!-- AUTO-GENERATED BY AAM — DO NOT EDIT BELOW THIS LINE -->

## Always-on Rules

- [python-standards](.claude/rules/python-standards.md)
- [api-conventions](.claude/rules/api-conventions.md)
```

#### Merge strategy for AGENTS.md

AGENTS.md has no native concept of glob-scoped rules — it applies to all files within the directory it's placed in. The compiler handles this with two behaviors:

**Directory inference from globs:** When `apply.mode` is `files`, the compiler extracts the longest common directory prefix from `apply.globs`. For example, `src/api/**/*.ts` → `src/api/`. The rule content is appended to `<prefix>/AGENTS.md`. If multiple rules share the same inferred directory, they are all merged into a single file in insertion order.

**Root aggregation for `always` / `intelligent`:** Rules with these modes are appended to the root `AGENTS.md`. Each rule contributes a clearly labelled section with an auto-generated comment header so diffs remain readable:

```markdown
<!-- AUTO-GENERATED BY AAM — DO NOT EDIT BELOW THIS LINE -->

<!-- rule: python-standards -->
# Python Coding Standards
- Follow PEP 8.
- Use type hints for all function signatures.

<!-- rule: general-conventions -->
# General Conventions
- Prefer explicit over implicit.
- Keep functions under 40 lines.
```

**Override files:** When `agents.codex.override: true`, the compiler emits `AGENTS.override.md` in the target directory instead of `AGENTS.md`. This is a Codex-specific extension that completely replaces parent-scope instructions for that directory rather than appending to them.

**Precedence note:** AGENTS.md files are hierarchical — Codex concatenates files from the root down, with files closer to the current directory winning because they appear later in the combined prompt. The compiler respects this by placing scoped rules in the most specific matching directory.

---

### Converter Specification

This section is the normative specification for the UARS converter — the component that transforms `.aam/rules/*.rule.md` source files into platform-specific instruction artifacts (and back).

The converter exposes **two distinct command namespaces** for two different use cases:

| Use case | Command namespace | When to use |
|---|---|---|
| **Package authoring** | `aam pkg rules *` | You are building or maintaining a UAAPS package and want rules to be a first-class part of it, stored in `.aam/rules/` and compiled into the package's distributed artifacts |
| **Standalone conversion** | `aam convert rules *` | You want to translate existing platform-native rules directly from one format to another without creating or managing a UAAPS package |

---

### Use Case 1 — Package Rules (`aam pkg rules`)

Use these commands when rules live inside a UAAPS package (`.aam/rules/`). This is the full authoring workflow: import → edit → compile → distribute as part of the package.

#### CLI Command Group

```
aam pkg rules compile [--target <platform>] [--dry-run] [--watch] [--force]
aam pkg rules import  [--from <platform>]   [--source <path>] [--overwrite] [--dry-run]
aam pkg rules diff    [--target <platform>]
aam pkg rules clean   [--target <platform>]
aam pkg rules list
```

---

#### `aam pkg rules compile`

Reads all `.aam/rules/*.rule.md` files and writes platform-specific outputs to their canonical paths.

##### Flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--target` | `string` | `all` | Platform(s) to emit: `copilot`, `cursor`, `claude`, `codex`, or `all` |
| `--dry-run` | `bool` | `false` | Print what would be written without performing any file I/O |
| `--watch` | `bool` | `false` | Re-compile on any change under `.aam/rules/` |
| `--force` | `bool` | `false` | Overwrite manually-edited output files (see §Conflict Detection) |
| `--out` | `string` | workspace root | Root directory for all relative output paths |

Source rules are always read from `.aam/rules/` relative to the package root.

##### Processing pipeline

The compiler executes the following stages in order for each invocation:

```
1. Discovery    — glob .aam/rules/**/*.rule.md
2. Parse        — YAML frontmatter + Markdown body per file
3. Validate     — apply all validation rules (§Validation Rules); abort on error
4. Filter       — skip rules where agents.<target>.enabled = false
5. Transform    — apply per-platform field mapping (see §Per-platform transformations)
6. Group        — group transformed outputs by their resolved output file path
7. Merge        — concatenate rules sharing the same output path in alphabetical name order
8. Conflict     — compare each output path against its on-disk state (see §Conflict Detection)
9. Write        — atomic write (write to .tmp, then rename) for each output file
```

##### Per-platform transformations

**Copilot** (`--target copilot`)

Output path: `.github/instructions/<name>.instructions.md`

| UARS field | Copilot frontmatter field | Notes |
|---|---|---|
| `name` | `name` | Title-cased: `python-standards` → `Python Standards` |
| `description` | `description` | Verbatim |
| `apply.mode: always` | `applyTo: "**"` | Wildcard — attached to all files |
| `apply.mode: files` | `applyTo: "<globs>"` | Multiple globs joined with `,` |
| `apply.mode: intelligent` | _(omitted)_ | User attaches manually in Copilot UI |
| `apply.mode: manual` | _(omitted)_ | Same as `intelligent` |
| `agents.copilot.enabled: false` | _(file not emitted)_ | |

The Markdown body is passed through verbatim. No merge occurs — one rule produces one file.

---

**Cursor** (`--target cursor`)

Output path: `.cursor/rules/<name>.mdc`

| UARS field | Cursor frontmatter field | Notes |
|---|---|---|
| `description` | `description` | Verbatim |
| `apply.mode: always` | `alwaysApply: true` | No `globs` emitted |
| `apply.mode: files` | `alwaysApply: false`, `globs: <globs>` | Multiple globs joined with `,` |
| `apply.mode: intelligent` | `alwaysApply: false` | No `globs`; Cursor uses `description` for agent selection |
| `apply.mode: manual` | `alwaysApply: false` | No `globs` |
| `agents.cursor.alwaysApply` | `alwaysApply` | Overrides the mode-derived value |
| `agents.cursor.priority` | `priority` | Emitted only when explicitly set _(Inferred — not in Cursor's public `.mdc` schema; field may be silently ignored)_ |
| `agents.cursor.enabled: false` | _(file not emitted)_ | |
| `refs` | `# @file <path>` comments | Prepended to the body as Cursor file references _(Inferred — `@file` syntax is verified in Cursor chat/prompt files; its behavior inside static `.mdc` rule files is unconfirmed)_ |

---

**Claude Code** (`--target claude`)

Two routing rules applied in order:

1. Rules with `apply.mode: always` — body is inlined into `CLAUDE.md` inside the compiler-managed guard block; no separate file is written.
2. All other rules — written to `.claude/rules/<name>.md` with a `paths:` frontmatter block; referenced from `CLAUDE.md`.

| UARS field | Claude output | Notes |
|---|---|---|
| `apply.mode: always` | Body inlined in `CLAUDE.md` guard block | No separate file |
| `apply.mode: files` | `paths:` list in `.claude/rules/<name>.md` | Each glob is a separate list entry |
| `apply.mode: intelligent` | `description:` frontmatter in `.claude/rules/<name>.md` | No `paths:` |
| `apply.mode: manual` | _(not emitted)_ | |
| `agents.claude.enabled: false` | _(not emitted)_ | |

`CLAUDE.md` is managed with a **block-replace strategy**. The compiler owns only the content between guard markers and leaves everything outside them untouched:

```markdown
<!-- aam:begin — managed by aam pkg rules compile; do not edit this block manually -->

## Always-on Rules

<!-- aam:rule:python-standards:begin -->
# Python Coding Standards
- Follow PEP 8.
- Use type hints for all function signatures.
<!-- aam:rule:python-standards:end -->

## File-scoped Rules

- [api-conventions](.claude/rules/api-conventions.md)

<!-- aam:end -->
```

- If `CLAUDE.md` does not exist, the compiler creates it containing only the guard block.
- If `CLAUDE.md` exists but has no guard markers, the compiler appends the guard block at the end and emits a warning.
- On each compile run the compiler replaces only the text between `<!-- aam:begin -->` and `<!-- aam:end -->`.

---

**Codex / AGENTS.md** (`--target codex`)

Fully specified in §Compiler Behavior and §Export: UARS → AGENTS.md Directory Tree. Additional normative requirements:

- The compiler MUST create all intermediate directories required by the resolved placement path.
- All rules targeting the same directory MUST be sorted by `name` (ascending lexicographic) before concatenation, ensuring deterministic output.
- The generation guard comment `<!-- AUTO-GENERATED BY AAM` MUST appear as the first line of every generated file.

---

#### Conflict Detection

Before writing a generated file, the compiler checks whether the existing on-disk file was produced by a previous compile run by looking for the **generation guard** at the top of the file:

| Platform | Guard pattern |
|---|---|
| Copilot | No guard — `.instructions.md` files are fully compiler-owned; always overwritten |
| Cursor | No guard — `.mdc` files are fully compiler-owned; always overwritten |
| Claude Code | `<!-- aam:begin` present anywhere in the file |
| Codex | `<!-- AUTO-GENERATED BY AAM` as the first line |

When the guard is **absent**, the file is treated as manually edited. The compiler:

1. Emits a warning: `conflict: <path> was manually edited — skipping (use --force to overwrite)`
2. Writes the would-be output to `.aam/conflicts/<name>.<target>.pending` for inspection
3. Continues processing remaining rules without aborting

With `--force`, the compiler overwrites the file unconditionally and deletes any corresponding `.pending` conflict file.

---

#### Idempotency

Running `aam pkg rules compile` twice with no source changes MUST produce bit-identical output files:

- Rule ordering within a merged file is always alphabetical by `name`.
- The compiler MUST NOT embed build timestamps or volatile metadata in generated file content.
- Blank line separators between merged rules are always exactly one blank line — no more, no less.
- Frontmatter fields in generated files are emitted in a fixed canonical order.

---

#### Watch Mode (`--watch`)

In watch mode the compiler subscribes to filesystem events on `.aam/rules/`:

1. On `add`, `change`, or `unlink` events, the full compile pipeline re-runs for all targets.
2. After each run the compiler prints a summary line per target:
   ```
   [watch] compiled 5 rules → copilot: 5 files, cursor: 5 files, claude: 3 files, codex: 4 files
   ```
3. Conflict warnings are repeated on every run until resolved or `--force` is used.

Incremental per-file re-compilation is not supported — each change triggers a full pass.

---

#### `aam pkg rules import`

Reads platform-native instruction files and writes UARS `.rule.md` files into the package at `.aam/rules/`.

##### Flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--from` | `string` | — | Source platform: `copilot`, `cursor`, `claude`, `codex` |
| `--source` | `string` | workspace root | Root directory for discovery (overrides default platform paths) |
| `--overwrite` | `bool` | `false` | Overwrite existing `.rule.md` files with the same name |
| `--dry-run` | `bool` | `false` | Print what would be written without performing any file I/O |

##### Discovery paths and splitting per platform

| Platform | Scanned paths | Splitting heuristic |
|---|---|---|
| `copilot` | `.github/instructions/*.instructions.md` | One file → one rule |
| `cursor` | `.cursor/rules/*.mdc` | One file → one rule |
| `claude` | `.claude/rules/*.md` + inline `<!-- aam:rule:*:begin/end -->` blocks in `CLAUDE.md` | One file → one rule; each inline block → one rule |
| `codex` | All `AGENTS.md` / `AGENTS.override.md` files in the directory tree | H1/H2 headings within each file (see §Section Splitting) |

##### Inverse field mapping (platform → UARS)

For `copilot`, `cursor`, and `claude`, existing frontmatter fields map back to UARS fields using the inverse of the per-platform transformation tables above. Unrecognized frontmatter fields are preserved verbatim under `agents.<platform>.*` to ensure round-trip fidelity.

If `--overwrite` is not set and a `.rule.md` file with the same `name` already exists in `.aam/rules/`, the importer skips that rule and emits a warning. With `--overwrite`, the existing file is replaced.

---

#### `aam pkg rules diff`

Compares compiled output currently on disk against what `aam pkg rules compile --dry-run` would produce, and prints a unified diff for each output file that would change.

```bash
aam pkg rules diff                     # diff all targets
aam pkg rules diff --target copilot    # diff only .github/instructions/
```

**Exit codes:** `0` = no changes; `1` = changes exist. Suitable for CI pre-commit hooks.

---

#### `aam pkg rules clean`

Deletes all compiler-generated output files. A file is only deleted if it carries the generation guard comment that proves it was compiler-managed; manually-edited files are never deleted.

```bash
aam pkg rules clean                    # remove generated outputs for all targets
aam pkg rules clean --target cursor    # remove only .cursor/rules/*.mdc
```

---

#### `aam pkg rules list`

Prints all rules currently in the package with their `name`, `apply.mode`, and the set of platforms they will be emitted to.

```bash
aam pkg rules list
# name                  mode          targets
# python-standards      files         copilot, cursor, claude, codex
# api-conventions       files         copilot, cursor, codex
# general-conventions   always        copilot, cursor, claude, codex
```

---

#### Error Codes (`aam pkg rules`)

| Exit code | Meaning |
|-----------|---------|
| `0` | Success — all files written (or no changes in dry-run) |
| `1` | Validation error in one or more rule files — no files written |
| `2` | Conflict detected — one or more output files were manually edited; skipped |
| `3` | I/O error — missing directory, permission denied, or disk full |
| `4` | Dry-run with pending changes — signals a non-empty diff without writing files |

---

### Use Case 2 — Standalone Converter (`aam convert rules`)

Use these commands when you want to translate rules directly between platforms **without managing a UAAPS package**. No `.aam/rules/` directory is created or required — the converter reads from the source platform's native paths and writes directly to the target platform's native paths in one operation.

This is the right tool for:
- Migrating an existing project from Cursor to GitHub Copilot (or any other pair)
- One-off cross-platform sync without committing to the full UAAPS package structure
- CI pipelines that need to keep platform files in sync from a single authoritative source

#### CLI Command Group

```
aam convert rules -s <platform> -t <platform> [--type <artifact-type>] [--source <path>] [--out <path>] [--dry-run] [--force] [--verbose]
aam convert rules diff -s <platform> -t <platform> [--type <artifact-type>] [--source <path>]
```

#### `aam convert rules`

Reads all rules from the source platform's native paths, converts them through the UARS intermediate representation, and writes the results to the target platform's native paths.

##### Options

| Option | Short | Required | Description |
|--------|-------|----------|-------------|
| `--source-platform` | `-s` | **Yes** | Source platform: `cursor`, `copilot`, `claude`, `codex` |
| `--target-platform` | `-t` | **Yes** | Target platform: `cursor`, `copilot`, `claude`, `codex` |
| `--type` | | No | Filter by artifact type: `instruction`, `agent`, `prompt`, `skill`, `rule`, `hooks`, `mcp` |
| `--source` | | No | Root directory to discover source files from (default: workspace root) |
| `--out` | | No | Root directory to write target files to (default: workspace root) |
| `--dry-run` | | No | Show what would be converted without writing files |
| `--force` | | No | Overwrite existing target files (creates `.bak` backup) |
| `--verbose` | | No | Show detailed workaround instructions for warnings |

##### Processing pipeline

```
1. Discovery  — find source platform files under --source (or workspace root)
2. Parse      — read each file and extract frontmatter + body
3. Normalise  — convert to UARS intermediate representation (in-memory; no files written)
4. Validate   — apply all UARS validation rules; abort on error
5. Transform  — apply per-platform output transformation for --to target
6. Conflict   — check target paths for manually-edited files
7. Write      — atomic write to target platform paths under --out
```

The UARS intermediate layer is never written to disk. It exists only to decouple the source and target platform formats.

##### Example — Cursor → Codex

```bash
# Dry-run to preview what would be written
aam convert rules -s cursor -t codex --dry-run

# Execute the conversion
aam convert rules -s cursor -t codex
```

Source discovery: `.cursor/rules/*.mdc`  
Output: `AGENTS.md` (root) or `<dir>/AGENTS.md` (per-directory, inferred from glob prefixes)

##### Example — Cursor → Copilot

```bash
aam convert rules -s cursor -t copilot
```

Source discovery: `.cursor/rules/*.mdc`  
Output: `.github/instructions/<name>.instructions.md`

##### Example — filter by artifact type

```bash
# Convert only rule artifacts (skip skills, prompts, etc.)
aam convert rules -s cursor -t copilot --type rule
```

##### Example — convert from a different source directory

```bash
aam convert rules -s cursor -t codex --source /path/to/other-project
```

#### `aam convert rules diff`

Prints a unified diff showing what the converter **would** change in the target platform's files, without writing anything.

```bash
aam convert rules diff -s cursor -t codex
```

**Exit codes:** `0` = target files already up to date; `1` = changes would be made.

#### Error Codes (`aam convert rules`)

| Exit code | Meaning |
|-----------|----------|
| `0` | Success — all target files written (or no changes in dry-run) |
| `1` | No source platform files found at the expected paths |
| `2` | Parse error in one or more source files |
| `3` | Conflict — one or more target files were manually edited; use `--force` to overwrite |
| `4` | I/O error — permission denied, disk full, or invalid output path |

---

### Import from AGENTS.md

`aam import --from codex` (or `aam import --source <path>`) walks the project tree, discovers all `AGENTS.md` and `AGENTS.override.md` files, parses each one into discrete rules, and writes them as `.aam/rules/*.rule.md` files. The source directory is encoded in `apply.globs` so a subsequent `aam compile --target codex` reconstructs the same file tree.

#### Discovery walk

The importer scans from the project root downward, collecting every file named `AGENTS.md` or `AGENTS.override.md`. Files are processed from the root toward leaves so that parent context is available when inferring `apply.mode`.

```
project/
  AGENTS.md                  → scope: ""  (root)
  src/
    AGENTS.md                → scope: "src/"
    api/
      AGENTS.md              → scope: "src/api/"
      AGENTS.override.md     → scope: "src/api/", override: true
  tests/
    AGENTS.md                → scope: "tests/"
```

#### Section splitting

Each `AGENTS.md` file is split into rules at **H1 (`#`) and H2 (`##`) headings**. Content before the first heading is treated as a single rule with an auto-generated name derived from the file's scope. Each resulting rule gets:

- `name` — slugified heading text, with a numeric suffix if the slug would collide (e.g., `api-rules`, `api-rules-2`)
- `description` — first non-blank line of the section body (truncated to 120 chars)
- `apply.mode` — inferred (see table below)
- `agents.codex.override` — `true` if sourced from `AGENTS.override.md`

#### Mode inference during import

The importer cannot read glob intent from freeform Markdown, so it applies conservative defaults:

| Source directory | Inferred `apply.mode` |
|---|---|
| Root (`AGENTS.md`) | `always` |
| Any subdirectory | `files` + `apply.globs` auto-set to `"<scope>**"` |

The user can manually tighten glob patterns after import. The `description` and content are preserved verbatim so that `intelligent` mode can be set manually when appropriate.

#### Import example

Given this project layout:

```
AGENTS.md
---
# General Conventions
- Prefer explicit over implicit.
- Keep functions under 40 lines.

## Working agreements
- Always run `npm test` after modifying JavaScript files.
```

```
src/api/AGENTS.md
---
# API Development Rules
- All endpoints must include input validation.
- Use standard error response format.
```

Running `aam import --from codex` produces:

**`.aam/rules/general-conventions.rule.md`**
```yaml
---
name: general-conventions
description: Prefer explicit over implicit.
apply:
  mode: always
---
# General Conventions
- Prefer explicit over implicit.
- Keep functions under 40 lines.
```

**`.aam/rules/working-agreements.rule.md`**
```yaml
---
name: working-agreements
description: Always run `npm test` after modifying JavaScript files.
apply:
  mode: always
---
## Working agreements
- Always run `npm test` after modifying JavaScript files.
```

**`.aam/rules/api-development-rules.rule.md`**
```yaml
---
name: api-development-rules
description: All endpoints must include input validation.
apply:
  mode: files
  globs:
    - "src/api/**"
---
# API Development Rules
- All endpoints must include input validation.
- Use standard error response format.
```

---

### Export: UARS → AGENTS.md Directory Tree

`aam compile --target codex` reads all `.aam/rules/*.rule.md` files whose `agents.codex.enabled` is `true` (or unset) and reconstructs the `AGENTS.md` file tree. The placement authority chain is:

```
apply.globs   (inferred: longest common directory prefix)
  └─ "."      (fallback: root AGENTS.md)
```

#### Directory grouping

Rules are grouped by their resolved target directory. All rules sharing the same target are concatenated in alphabetical `name` order and written to a single `AGENTS.md`:

```
.aam/rules/
  general-conventions.rule.md   → scope: ""         ┐
  working-agreements.rule.md    → scope: ""         ├─ → AGENTS.md
  git-workflow.rule.md          → scope: ""         ┘

  api-development-rules.rule.md → scope: "src/api/" ┐
  api-auth-rules.rule.md        → scope: "src/api/" ┘─ → src/api/AGENTS.md

  test-conventions.rule.md      → scope: "tests/"   ──→ tests/AGENTS.md
```

#### Generated file structure

Each generated `AGENTS.md` contains a header guard comment so the file is clearly machine-managed, followed by rule sections separated by blank lines. The rule name is recorded in an inline comment to enable targeted re-import and diff tracking:

```markdown
<!-- AUTO-GENERATED BY AAM — DO NOT EDIT MANUALLY -->
<!-- Regenerate with: aam compile --target codex -->
<!-- Rules: general-conventions, working-agreements -->

# General Conventions
- Prefer explicit over implicit.
- Keep functions under 40 lines.

## Working agreements
- Always run `npm test` after modifying JavaScript files.
```

#### Override files

Rules with `agents.codex.override: true` are written to `AGENTS.override.md` in their target directory. If both regular and override rules exist for the same directory, the compiler writes two separate files.

#### Import → compile workflow

```
aam import --from codex          # existing AGENTS.md files → .aam/rules/*.rule.md
aam compile --target codex       # .aam/rules/*.rule.md → AGENTS.md file tree
```

The generated AGENTS.md tree mirrors the original structure because the importer sets `apply.globs` to `<source-dir>/**`, which the compiler uses to derive the same placement. Section order within a generated file follows alphabetical rule name order; manually adjust rule names to control ordering if needed.

---

### AGENTS.md Platform Notes

AGENTS.md is an open standard originally introduced by OpenAI for Codex and adopted as a de facto cross-platform convention by Cursor, Amp, Jules from Google, Factory, and others. It is not formally stewarded by any foundation. Unlike the other platforms in this spec, AGENTS.md:

- Has **no frontmatter schema** — the entire file is freeform Markdown parsed as plain instructions.
- Uses **directory placement** as its only scoping mechanism (no glob patterns, no `applyTo`).
- Supports a **hierarchical chain** from `~/.codex/AGENTS.md` (global) → project root → subdirectories, all concatenated together at runtime.
- Supports an **override variant** (`AGENTS.override.md`) that suppresses parent-level instructions for a given directory tree.
- Has a **size cap**: Codex stops adding files once the combined size reaches the limit defined by `project_doc_max_bytes` (32 KiB by default).

---

### Validation Rules

A valid UARS file must satisfy:

1. `name` is present, non-empty, and matches `^[a-z0-9][a-z0-9-]*$`
2. `description` is present and non-empty
3. `apply.mode` is one of `always | intelligent | files | manual`
4. If `apply.mode` is `files`, at least one entry in `apply.globs` must exist
5. Markdown body must be non-empty
6. `refs` entries must resolve to another `*.rule.md` file or an existing file path in the repository

---

### Complete Example

```yaml
---
name: api-conventions
version: 1.2.0
description: Standards for REST API endpoints in TypeScript and Python

apply:
  mode: files
  globs:
    - "src/api/**/*.ts"
    - "src/api/**/*.py"

agents:
  copilot:
    enabled: true
  cursor:
    enabled: true
    alwaysApply: false
  claude:
    enabled: true
  codex:
    enabled: true

refs:
  - service-template
---

# API Development Rules

- All endpoints must include input validation using the project's `validate()` helper.
- Use the standard error response format: `{ error: string, code: string, details?: object }`.
- Include OpenAPI documentation comments on every route handler.
- Prefer pagination over returning unbounded lists.
- Log all mutations at `INFO` level with the actor's user ID.
```
