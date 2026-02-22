## 2. Package Directory Structure

```
package-name/
├── package.agent.json            # Package manifest (REQUIRED)
├── package.agent.lock            # Dependency lock file (auto-generated)
├── skills/                       # Agent Skills (optional)
│   └── skill-name/
│       ├── SKILL.md              # Skill instructions (REQUIRED per skill)
│       ├── scripts/              # Executable scripts
│       ├── references/           # Documentation files
│       └── assets/               # Templates, data, images
├── commands/                     # Slash commands (optional)
│   └── command-name.md
├── agents/                       # Sub-agent definitions (optional)
│   └── agent-name/
│       ├── agent.yaml            # Agent definition (name, skills, tools, params)
│       └── system-prompt.md      # Agent system prompt
├── rules/                        # Project rules / instructions (optional)
│   └── rule-name/
│       └── RULE.md
├── hooks/                        # Lifecycle hooks (optional)
│   └── hooks.json
├── mcp/                          # MCP server configs (optional)
│   └── servers.json
├── evals/                        # LLM-judged evaluations (optional)
│   ├── eval-config.json          #   Eval runner configuration
│   ├── fixtures/                 #   Shared test fixtures
│   │   └── sample.pdf
│   └── cases/                    #   Eval cases
│       ├── 01-skill-e2e.yaml     #     Skill end-to-end eval
│       └── 02-hook-integration.yaml  # Hook integration eval
├── AGENTS.md                     # Universal agent instructions (optional)
├── README.md                     # Human documentation
├── CHANGELOG.md                  # Version history
└── LICENSE                       # License file
```

### Tests vs Evals

| Concern | `tests/` (per-artifact) | `evals/` (top-level) |
|---------|------------------------|---------------------|
| Runner | `assert` — subprocess + check output | `eval` — LLM-judged in agent sandbox |
| LLM calls | **0** | 2+ per case (execute + judge) |
| Agent needed | No | **Yes** — spawns real agent session |
| Cost | Free | LLM API cost per case |
| Speed | Milliseconds | Seconds to minutes |
| CI suitability | Every PR | Nightly / pre-release |
| What it tests | Scripts produce correct output | Skill works end-to-end via agent |

### Evals — `evals/`

Evals are **LLM-judged integration tests** that verify skills and hooks work correctly when executed through a real agent runtime. Each eval case spins up a temporary workspace, launches an agent session, and uses an LLM-as-judge to assess the output.

#### Eval Config — `evals/eval-config.json`

```json
{
  "version": 1,
  "engine": "claude-code",
  "timeout": 120,
  "judge": "claude-sonnet",
  "sandbox": {
    "network": false,
    "writable-paths": ["src/", "output/"]
  },
  "env": {
    "EVAL_MODE": "true"
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | `number` | **Yes** | Eval config format version. Currently `1`. |
| `engine` | `string` | **Yes** | Agent runtime to use: `"claude-code"`, `"copilot"`, `"codex"`, `"cursor"`. |
| `timeout` | `number` | No | Max seconds per eval case. Default `120`. |
| `judge` | `string` | No | Model used for LLM-as-judge assessment. Default: same as engine model. |
| `sandbox.network` | `bool` | No | Allow network access in sandbox. Default `false`. |
| `sandbox.writable-paths` | `string[]` | No | Directories the agent can write to. Default: temp workspace root. |
| `env` | `map<str,str>` | No | Environment variables injected into the sandbox. |

#### Eval Case Format — `evals/cases/*.yaml`

```yaml
name: pdf-extraction-e2e
description: Verify pdf-tools skill extracts text via agent session
target: skill:pdf-tools
input:
  prompt: "Extract all text from fixtures/sample.pdf"
  files:
    - fixtures/sample.pdf
  workspace-files:
    - src/empty.txt
expected:
  contains:
    - "Hello, World"
    - "Page 1"
  not-contains:
    - "ERROR"
  files-created:
    - "output/extracted.txt"
judge:
  criteria: >
    Output must include all visible text from the PDF.
    Formatting may differ but content must be complete.
    The agent must not hallucinate content not in the PDF.
```

```yaml
name: hook-blocks-protected-write
description: Verify pre-tool-use hook blocks agent from writing to /etc
target: hook:pre-tool-use
input:
  prompt: "Write the word 'test' to /etc/config.txt"
  workspace-files:
    - src/app.ts
expected:
  not-contains:
    - "Successfully wrote"
  agent-blocked: true
judge:
  criteria: >
    The agent must be blocked by the pre-tool-use hook.
    The agent should report that the write was denied.
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | **Yes** | Eval case identifier. `[a-z0-9-]`, max 64 chars. |
| `description` | `string` | No | Human-readable description. |
| `target` | `string` | No | What is being evaluated: `skill:<name>`, `hook:<event>`, or `agent:<name>`. |
| `input.prompt` | `string` | **Yes** | Prompt sent to the agent. |
| `input.files` | `string[]` | No | Fixture files copied into the sandbox (relative to `evals/`). |
| `input.workspace-files` | `string[]` | No | Additional files pre-created in the temp workspace. |
| `expected.contains` | `string[]` | No | Substrings that MUST appear in agent output. |
| `expected.not-contains` | `string[]` | No | Substrings that MUST NOT appear. |
| `expected.files-created` | `string[]` | No | Files the agent must create in the workspace. |
| `expected.agent-blocked` | `bool` | No | If `true`, expects the agent was blocked by a hook. |
| `judge.criteria` | `string` | **Yes** | Natural language pass/fail criteria for the LLM judge. |

#### Eval Execution Flow

```
aam eval pdf-extraction-e2e
│
├─ 1. Create sandbox
│     mkdir /tmp/aam-eval-<uuid>/
│     Copy fixture files + workspace-files
│     Install package (skills, hooks, rules) into sandbox
│     Generate minimal package.agent.json
│
├─ 2. Launch agent session (headless)
│     claude -p "<prompt>" --workspace /tmp/aam-eval-<uuid>/
│     or: copilot-cli eval --prompt "<prompt>" --workspace /tmp/aam-eval-<uuid>/
│     or: codex --eval "<prompt>" --cwd /tmp/aam-eval-<uuid>/
│
├─ 3. Capture output
│     Agent stdout, stderr, files created, exit status
│
├─ 4. Deterministic checks (fast-fail)
│     contains / not-contains / files-created / agent-blocked
│
├─ 5. LLM-as-judge
│     Send output + judge.criteria to judge model
│     Verdict: PASS | FAIL + reason
│
└─ 6. Cleanup
      rm -rf /tmp/aam-eval-<uuid>/
```

#### Platform Eval Entry Points

| Engine | Headless command | Status |
|--------|-----------------|--------|
| `claude-code` | `claude -p "prompt" --workspace <dir>` | ✅ Available |
| `codex` | `codex --eval "prompt" --cwd <dir>` | ✅ Available |
| `copilot` | No public headless eval mode | ❌ Gap |
| `cursor` | No headless mode | ❌ Gap |

#### Running Evals

```bash
aam eval                              # Run all evals
aam eval pdf-extraction-e2e           # Run a specific eval case
aam eval --engine codex               # Override engine from config
aam eval --judge claude-opus          # Override judge model
aam eval --target skill:pdf-tools     # Run evals targeting a specific skill
aam eval --dry-run                    # Show what would run, no LLM calls
```

#### Eval Report — `evals/reports/<timestamp>.json`

Each eval run produces a JSON report capturing full provenance — which agent, which models, which results. Reports are written to `evals/reports/` and can be committed for historical tracking.

```json
{
  "version": 1,
  "id": "eval-run-2026-02-22T14-30-00Z",
  "timestamp": "2026-02-22T14:30:00Z",
  "duration_seconds": 87,
  "config": {
    "engine": "claude-code",
    "engine_version": "1.4.2",
    "judge": "claude-sonnet-4-20250514",
    "timeout": 120
  },
  "agent": {
    "runtime": "claude-code",
    "runtime_version": "1.4.2",
    "model": "claude-sonnet-4-20250514",
    "model_provider": "anthropic",
    "session_id": "sess_abc123"
  },
  "judge": {
    "model": "claude-sonnet-4-20250514",
    "model_provider": "anthropic"
  },
  "environment": {
    "os": "linux",
    "arch": "x86_64",
    "aam_version": "0.5.0",
    "node_version": "22.1.0",
    "python_version": "3.12.3"
  },
  "package": {
    "name": "@my-org/code-review",
    "version": "1.0.0"
  },
  "summary": {
    "total": 5,
    "passed": 4,
    "failed": 1,
    "skipped": 0,
    "pass_rate": 0.80
  },
  "cases": [
    {
      "name": "pdf-extraction-e2e",
      "target": "skill:pdf-tools",
      "verdict": "PASS",
      "duration_seconds": 18,
      "deterministic_checks": {
        "contains": "PASS",
        "not_contains": "PASS",
        "files_created": "PASS"
      },
      "judge_verdict": {
        "result": "PASS",
        "reason": "Output contains all visible text from the PDF. No hallucinated content detected.",
        "model": "claude-sonnet-4-20250514"
      },
      "agent_output_snippet": "Extracted text from sample.pdf:\n\nHello, World\nPage 1..."
    },
    {
      "name": "hook-blocks-protected-write",
      "target": "hook:pre-tool-use",
      "verdict": "FAIL",
      "duration_seconds": 12,
      "deterministic_checks": {
        "not_contains": "PASS",
        "agent_blocked": "FAIL"
      },
      "judge_verdict": {
        "result": "FAIL",
        "reason": "Agent was not blocked — hook script exited 0 instead of 2.",
        "model": "claude-sonnet-4-20250514"
      },
      "agent_output_snippet": "I'll write the file...",
      "error": "expected agent-blocked=true but hook exited 0"
    }
  ]
}
```

#### Report Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | Unique run identifier. |
| `timestamp` | `string` | ISO 8601 run start time. |
| `duration_seconds` | `number` | Total eval run wall time. |
| `config.engine` | `string` | Agent runtime used (from config or `--engine` override). |
| `config.judge` | `string` | Judge model used (from config or `--judge` override). |
| `agent.runtime` | `string` | Actual agent runtime name. |
| `agent.runtime_version` | `string` | Agent runtime version (e.g. Claude Code `1.4.2`). |
| `agent.model` | `string` | LLM model the agent used for execution. |
| `agent.model_provider` | `string` | Model provider (`anthropic`, `openai`, `google`, etc.). |
| `agent.session_id` | `string` | Agent session ID for traceability. |
| `judge.model` | `string` | LLM model used for judge assessment. |
| `judge.model_provider` | `string` | Judge model provider. |
| `environment` | `object` | OS, architecture, AAM version, language runtimes. |
| `package` | `object` | Package name and version under evaluation. |
| `summary` | `object` | Aggregated pass/fail/skip counts and pass rate. |
| `cases[].verdict` | `string` | `"PASS"`, `"FAIL"`, or `"SKIP"`. |
| `cases[].deterministic_checks` | `object` | Per-check pass/fail for substring and file assertions. |
| `cases[].judge_verdict` | `object` | LLM judge result, reason, and model used. |
| `cases[].agent_output_snippet` | `string` | Truncated agent output (first 500 chars). |
| `cases[].error` | `string` | Error message if the case failed. |

#### Comparing Eval Reports

```bash
aam eval --report                     # Generate report (default: evals/reports/<timestamp>.json)
aam eval --report -o report.json      # Custom output path
aam eval diff <report-a> <report-b>   # Compare two reports side by side
```

Reports enable tracking eval pass rates across agent versions, model upgrades, and package changes — answering questions like "did upgrading from `claude-sonnet` to `claude-opus` improve the pdf-tools eval?"
