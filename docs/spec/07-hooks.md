## 8. Hooks (Lifecycle Event Handlers)

Hooks execute shell commands at specific points in the agent lifecycle. They run **outside** the context window with zero token overhead.

### Universal `hooks.json` Format

```json
{
  "version": 1,
  "hooks": {
    "pre-tool-use": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "bash ${PACKAGE_ROOT}/hooks/scripts/validate.sh",
          "timeout": 30
        }]
      }
    ],
    "post-tool-use": [
      {
        "matcher": "Write|Edit",
        "hooks": [{
          "type": "command",
          "command": "npx prettier --write ${file}"
        }]
      }
    ],
    "stop": [
      {
        "hooks": [{
          "type": "prompt",
          "prompt": "Check if all tasks are complete. Context: $ARGUMENTS",
          "timeout": 30
        }]
      }
    ],
    "session-start": [
      {
        "hooks": [{
          "type": "command",
          "command": "bash ${PACKAGE_ROOT}/hooks/scripts/setup.sh"
        }]
      }
    ]
  }
}
```

### Unified Hook Events

| Universal Event | Claude Code | Cursor | Copilot | Can Block? | Description |
|----------------|-------------|--------|---------|------------|-------------|
| `pre-tool-use` | `PreToolUse` | `beforeShellCommand` / `beforeMcpCall` | `preToolUse` | **Yes** | Before tool execution |
| `permission-request` | `PermissionRequest` | N/A | N/A | **Yes** | Permission dialog shown |
| `post-tool-use` | `PostToolUse` | `afterFileEdit` | N/A | Feedback | After tool execution |
| `pre-prompt` | `UserPromptSubmit` | `beforeSubmitPrompt` | `userPromptSubmitted` | **Yes** / Copilot: logging | Before prompt processed |
| `session-start` | `SessionStart` | N/A | `sessionStart` | Context inject | New session begins |
| `session-end` | `SessionEnd` | N/A | `sessionEnd` | Cleanup | Session terminates |
| `stop` | `Stop` | `stop` | N/A | **Yes** | Agent finishes responding |
| `sub-agent-end` | `SubagentStop` | N/A | N/A | **Yes** | Sub-agent finishes |
| `pre-compact` | `PreCompact` | N/A | N/A | No | Before context compaction |
| `notification` | `Notification` | N/A | N/A | No | System notification |

> **Copilot hook gaps**: Copilot currently supports 4 hook events vs Claude Code's 10. The `stop`, `post-tool-use`, `permission-request`, `sub-agent-end`, `pre-compact`, and `notification` events have no Copilot equivalent. Community feature request [#1157](https://github.com/github/copilot-cli/issues/1157) tracks parity efforts.

### Hook Types

| Type | Description | Use Case |
|------|-------------|----------|
| `command` | Execute a shell command | Deterministic rules (lint, format, validate) |
| `prompt` | Query an LLM for context-aware decision | Stop/continue decisions, complex validation |

### Blocking Response Schema

For hooks that support blocking (`pre-tool-use`, `permission-request`, `pre-prompt`, `stop`):

```json
{
  "hookSpecificOutput": {
    "hookEventName": "pre-tool-use",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Reason shown to user or agent",
    "updatedInput": {}
  }
}
```

For `stop` / `sub-agent-end`:

```json
{
  "decision": "block",
  "reason": "Tests not yet executed"
}
```

#### Exit Code Convention

| Code | Behavior |
|------|----------|
| `0` | Success. JSON on stdout parsed for structured control. |
| `2` | Blocking error. `stderr` fed back to agent. |
| Other | Non-blocking error. Logged only. |

### Hook Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `matcher` | `string` | No | Regex pattern for filtering by tool name. |
| `hooks[].type` | `string` | **Yes** | `"command"` or `"prompt"`. |
| `hooks[].command` | `string` | For `command` | Shell command. Receives JSON on stdin. |
| `hooks[].prompt` | `string` | For `prompt` | LLM prompt. `$ARGUMENTS` for input. |
| `hooks[].timeout` | `number` | No | Timeout in seconds. |

### Hooks Directory

```
hooks/
├── hooks.json           # REQUIRED — hook definitions
├── scripts/             # Shell scripts referenced by hooks
│   ├── validate.sh
│   ├── setup.sh
│   └── format.sh
└── tests/               # Hook tests (optional)
    ├── test-config.json
    ├── fixtures/        # Simulated event payloads
    │   ├── pre-tool-use-write.json
    │   └── stop-incomplete.json
    └── cases/
        ├── 01-pre-tool-block.yaml
        └── 02-post-tool-format.yaml
```

### Hook Testing

Hooks MAY include a `tests/` directory with **deterministic** test cases that verify hook scripts behave correctly for simulated lifecycle events. These tests use the `assert` runner — no LLM calls, no agent sandbox, CI-safe.

> **Eval-based testing** (LLM-judged, agent sandbox) lives in the top-level `evals/` directory. See [§ Evals](01-package-format.md#evals-evals) in the package format spec.

#### Test Config — `tests/test-config.json`

```json
{
  "version": 1,
  "timeout": 30,
  "env": {
    "HOOK_TEST": "true",
    "PACKAGE_ROOT": "."
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | `number` | **Yes** | Test config format version. Currently `1`. |
| `timeout` | `number` | No | Max seconds per test case. Default `30`. |
| `env` | `map<str,str>` | No | Environment variables injected during test runs. |

#### Fixture Payloads — `tests/fixtures/*.json`

Fixtures simulate the JSON event that a hook receives on stdin during real execution.

```json
{
  "hookEventName": "pre-tool-use",
  "toolName": "Write",
  "toolInput": {
    "file_path": "/src/app.ts",
    "content": "console.log('hello');"
  }
}
```

#### Test Case Format — `tests/cases/*.yaml`

```yaml
name: pre-tool-block-forbidden-path
description: Verify pre-tool-use hook blocks writes to protected directories
event: pre-tool-use
hook-index: 0
input:
  fixture: fixtures/pre-tool-use-write.json
  overrides:
    toolInput.file_path: "/etc/passwd"
expected:
  exit-code: 2
  stderr-contains:
    - "blocked"
    - "protected path"
  stdout-json:
    hookSpecificOutput:
      permissionDecision: deny
```

```yaml
name: post-tool-format-success
description: Verify post-tool-use hook runs formatter without error
event: post-tool-use
hook-index: 0
input:
  fixture: fixtures/pre-tool-use-write.json
expected:
  exit-code: 0
  not-contains:
    - "ERROR"
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | `string` | **Yes** | Test case identifier. `[a-z0-9-]`, max 64 chars. |
| `description` | `string` | No | Human-readable description of what is tested. |
| `event` | `string` | **Yes** | Hook event to test (e.g. `pre-tool-use`, `stop`). |
| `hook-index` | `number` | No | Index of the hook group in the event array. Default `0`. |
| `input.fixture` | `string` | No | Path to a fixture JSON file (relative to `tests/`). |
| `input.overrides` | `map` | No | Dot-path overrides applied on top of the fixture. |
| `expected.exit-code` | `number` | No | Expected exit code from the hook script. |
| `expected.stderr-contains` | `string[]` | No | Substrings that MUST appear in stderr. |
| `expected.stdout-json` | `object` | No | JSON structure that stdout MUST match (deep partial match). |
| `expected.not-contains` | `string[]` | No | Substrings that MUST NOT appear in combined output. |

#### Running Hook Tests

```bash
aam test --hooks                          # Run all hook tests in current package
aam test --hooks --case 01-pre-tool-block # Run a single hook test case
aam test --hooks --event pre-tool-use     # Run tests for a specific event only
```
