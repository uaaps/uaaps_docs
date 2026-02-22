## 4. Skills — `SKILL.md`

Skills are the **primary portable artifact** — already standardized via the Agent Skills Open Standard (agentskills.io) and adopted by Claude Code, Cursor, Copilot, Codex, and others.

### SKILL.md Format

```markdown
---
# === REQUIRED ===
name: skill-name                    # [a-z0-9-], max 64 chars
description: >                      # Max 1024 chars. WHAT it does + WHEN to use it.
  Extract text and tables from PDF files, fill forms, merge documents.
  Use when working with PDF files or when the user mentions PDFs.

# === OPTIONAL (Open Standard) ===
license: Apache-2.0
compatibility:                      # Only if skill has special requirements
  requires:
    - python>=3.10
    - pypdf
metadata:                           # Arbitrary key-value extensions
  author: my-org
  version: "1.0"

# === OPTIONAL (Platform Extensions) ===
allowed-tools: Read Grep Glob Bash  # Pre-approved tools (experimental)
context: fork                       # Claude Code: run in isolated sub-agent
agent: Explore                      # Claude Code: sub-agent config
disable-model-invocation: true      # Claude Code: user-only invocation
---

# Skill Title

## Instructions
Step-by-step guidance...

## Examples
Concrete usage examples...
```

### Frontmatter Field Reference

| Field | Type | Required | Standard | Description |
|-------|------|----------|----------|-------------|
| `name` | `string` | **Yes** | agentskills.io | `[a-z0-9-]`, max 64 chars |
| `description` | `string` | **Yes** | agentskills.io | What + When. Max 1024 chars. |
| `license` | `string` | No | agentskills.io | License identifier or filename |
| `compatibility` | `map` | No | agentskills.io | Environment requirements |
| `metadata` | `map<str,str>` | No | agentskills.io | Extension key-value pairs |
| `allowed-tools` | `string` | No | Extension | Space-delimited tool whitelist |
| `context` | `string` | No | Claude ext. | `"fork"` for isolated execution |
| `agent` | `string` | No | Claude ext. | Sub-agent config name |
| `disable-model-invocation` | `bool` | No | Claude ext. | Restrict to user-only invocation |

### Skill Directory

```
skill-name/
├── SKILL.md         # REQUIRED
├── scripts/         # Executable code (Python, Bash, JS)
├── references/      # Additional documentation
├── assets/          # Templates, images, data
├── examples/        # Example inputs/outputs
└── tests/           # Skill tests (optional)
    ├── test-config.json
    └── cases/
        ├── 01-basic.yaml
        └── 02-edge-cases.yaml
```

### Progressive Disclosure

| Phase | What Loads | Token Cost |
|-------|-----------|------------|
| 1. Metadata | `name` + `description` from all skills | ~50 tokens/skill |
| 2. Activation | Full `SKILL.md` body | ~2,000–5,000 tokens |
| 3. Execution | `scripts/`, `references/`, `assets/` on-demand | Variable |

### Skills Discovery & Precedence

Skills can exist at multiple scopes. The universal resolver follows this precedence:

| Scope | Location | Precedence | Namespaced? |
|-------|----------|-----------|-------------|
| **Managed / Enterprise** | Admin-deployed | Highest | No |
| **Project** | `./skills/` (in package or `.claude/skills/` or `.github/skills/`) | Overrides personal | No |
| **Personal** | `~/.claude/skills/` or `~/.github/skills/` or user profile | Lowest non-plugin | No |
| **Plugin / Package** | `<package>/skills/` | Always namespaced | Yes (`pkg:skill`) |

**Resolution rules**:
- When project and personal skills share the same `name`, **project wins**.
- Plugin/package skills are **always namespaced** (`package-name:skill-name`) so they never conflict.
- Enterprise/managed rules override all other scopes.
- Monorepo support: skills in nested subdirectory `.claude/skills/` or `.github/skills/` are auto-discovered.

> **Disputed precedence**: ChatGPT investigation claimed enterprise > personal > project. Our investigation (verified against community guides) found enterprise > project > personal. The enterprise tier being highest is agreed; the project > personal ordering is confirmed by Claude Code behavior where `.claude/skills/` in the project overrides `~/.claude/skills/` for same-named skills.

### Skill Testing

Skills MAY include a `tests/` directory with **deterministic** test cases that verify scripts behave correctly. These tests use the `assert` runner — no LLM calls, no agent sandbox, CI-safe.

> **Eval-based testing** (LLM-judged, agent sandbox) lives in the top-level `evals/` directory. See [§ Evals](01-package-format.md#evals-evals) in the package format spec.

#### Test Config — `tests/test-config.json`

```json
{
  "version": 1,
  "timeout": 30,
  "env": {
    "SKILL_TEST": "true"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | `number` | **Yes** | Test config format version. Currently `1`. |
| `timeout` | `number` | No | Max seconds per test case. Default `30`. |
| `env` | `map<str,str>` | No | Environment variables injected during test runs. |

#### Test Case Format — `tests/cases/*.yaml`

```yaml
name: basic-pdf-extraction
description: Verify script extracts text from a single-page PDF
input:
  command: "python scripts/extract.py tests/fixtures/sample.pdf"
  stdin: null
  files:
    - tests/fixtures/sample.pdf
expected:
  exit-code: 0
  stdout-contains:
    - "Hello, World"
    - "Page 1"
  not-contains:
    - "ERROR"
    - "Traceback"
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | **Yes** | Test case identifier. `[a-z0-9-]`, max 64 chars. |
| `description` | `string` | No | Human-readable description of what is tested. |
| `input.command` | `string` | **Yes** | Shell command to execute (relative to skill root). |
| `input.stdin` | `string` | No | Data piped to stdin. |
| `input.files` | `string[]` | No | Fixture files required (relative to skill root). |
| `expected.exit-code` | `number` | No | Expected exit code. Default `0`. |
| `expected.stdout-contains` | `string[]` | No | Substrings that MUST appear in stdout. |
| `expected.stderr-contains` | `string[]` | No | Substrings that MUST appear in stderr. |
| `expected.not-contains` | `string[]` | No | Substrings that MUST NOT appear in combined output. |
| `expected.stdout-json` | `object` | No | JSON structure stdout MUST match (deep partial match). |

#### Running Skill Tests

```bash
aam test                             # Run all assert tests in current package
aam test skill-name                  # Run tests for a specific skill
aam test skill-name --case 01-basic  # Run a single test case
```

### Platform Compatibility

| Platform | Skill Location | Status |
|----------|---------------|--------|
| Claude Code | `~/.claude/skills/`, `.claude/skills/`, plugin `skills/` | ✅ Full support |
| Cursor | Plugin `skills/`, imported as agent-decided rules | ✅ Supported (v2.2+) |
| GitHub Copilot | `.github/skills/`, `.claude/skills/` (also supported), user profile | ✅ Supported |
| OpenAI Codex | `~/.codex/skills/`, `.agents/skills/` | ✅ Supported |
| Amp, Goose, OpenCode, Letta | Various paths | ✅ Supported |
