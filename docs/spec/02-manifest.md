## 3. Package Manifest — `package.agent.json`

The manifest is the **single required file**. It identifies the package and declares its contents.

### Schema

```jsonc
{
  // === REQUIRED ===
  "name": "my-package",                        // [a-z0-9-] or @scope/name, max 64/130 chars
  "version": "1.0.0",                          // SemVer

  // === RECOMMENDED ===
  "description": "Brief description",          // Max 1024 chars
  "author": "org-or-user",
  "license": "Apache-2.0",
  "repository": "https://github.com/org/repo",
  "homepage": "https://docs.example.com",      // Project homepage or docs URL

  // === OPTIONAL ===
  "category": "Developer Tools",               // Marketplace category
  "keywords": ["testing", "python"],            // Discovery tags
  "engines": {                                  // Platform compatibility
    "claude-code": ">=1.0",
    "cursor": ">=2.2",
    "copilot": "*",
    "codex": "*"
  },

  // === ARTIFACT DECLARATIONS (optional, explicit registry; auto-discovered if omitted) ===
  // Declaring artifacts explicitly enables richer registry metadata and description per artifact.
  "artifacts": {
    "skills": [
      { "name": "pdf-tools", "path": "skills/pdf-tools/", "description": "Extract text from PDFs" }
    ],
    "agents": [
      { "name": "code-auditor", "path": "agents/code-auditor/", "description": "Security audit agent" }
    ],
    "commands": [
      { "name": "review", "path": "commands/review.md", "description": "Code review command" },
      { "name": "audit-finding", "path": "commands/audit-finding.md", "description": "Finding template" }
    ]
  },

  // === DEPENDENCIES (optional) ===
  "dependencies": {                             // Other agent packages required
    "code-review": "^1.0.0",                   //   SemVer range
    "testing-utils": ">=2.1.0 <3.0.0"
  },
  "optionalDependencies": {                     // Nice-to-have, install doesn't fail
    "ai-image-gen": "^1.0.0"
  },
  "peerDependencies": {                         // Must be provided by host environment
    "eslint-rules": ">=3.0.0"
  },
  "systemDependencies": {                       // OS-level / runtime requirements
    "python": ">=3.10",
    "node": ">=18",
    "packages": {                               //   Package manager installs
      "pip": ["pypdf", "pdfplumber>=0.10"],
      "npm": ["prettier", "eslint"],
      "brew": ["graphviz"],
      "apt": ["poppler-utils"]
    },
    "binaries": ["git", "docker"],              //   Must exist on $PATH
    "mcp-servers": ["filesystem"]              //   Required MCP servers
  },

  // === ENVIRONMENT & SECRETS (optional) ===
  "env": {                                      // Required environment variables
    "GITHUB_TOKEN": { "description": "Required for PR creation", "required": true },
    "DEBUG_MODE": { "description": "Enable verbose logging", "required": false, "default": "false" }
  },

  // === PERMISSIONS (optional) ===
  "permissions": {                              // Requested agent capabilities
    "fs": ["read", "write"],                    // Filesystem access scope
    "network": ["api.github.com"],              // Allowed network hosts
    "shell": false                              // Allow arbitrary shell execution
  },

  // === QUALITY (optional) ===
  "quality": {
    "tests": [
      { "name": "unit-tests", "command": "pytest tests/", "description": "Unit tests" }
    ],
    "evals": [
      {
        "name": "accuracy-eval",
        "path": "evals/accuracy.yaml",
        "description": "Measures accuracy against benchmark",
        "metrics": [
          { "name": "accuracy", "type": "percentage" }
        ]
      }
    ]
  },

  // === COMPONENT OVERRIDES (optional, defaults use convention) ===
  "skills": "./skills",                         // Default: ./skills
  "commands": "./commands",                     // Default: ./commands
  "agents": "./agents",                         // Default: ./agents
  "rules": "./rules",                           // Default: ./rules
  "hooks": "./hooks/hooks.json",                // Default: ./hooks/hooks.json
  "mcp": "./mcp/servers.json",                  // Default: ./mcp/servers.json

  // === VENDOR EXTENSIONS (optional) ===
  "x-claude": {
    "marketplace": "anthropics/skills"
  },
  "x-cursor": {
    "category": "Developer Tools"
  }
}
```

### Format Support

The canonical format is **JSON** (`package.agent.json`), but tools SHOULD also accept **YAML** (`package.agent.yaml`) for author convenience:

```yaml
# package.agent.yaml (equivalent to JSON, author-friendly)
name: my-package           # or @scope/my-package for scoped
version: 1.0.0
description: Brief description
author: org-or-user
license: Apache-2.0
homepage: https://docs.example.com

skills: ./skills
commands: ./commands
agents: ./agents

dependencies:
  code-review: "^1.0.0"

x-claude:
  marketplace: anthropics/skills

x-cursor:
  category: Developer Tools
```

**Rules**:
- If both `.json` and `.yaml` exist, **JSON takes precedence**.
- CLI tools (`aam install`, etc.) generate `package.agent.json` as the canonical output.
- Lock files (`package.agent.lock`) are always JSON.

### Vendor Mapping

| Manifest Field | Claude Code (`plugin.json`) | Cursor (`.cursor-plugin/plugin.json`) |
|----------------|----------------------------|---------------------------------------|
| `name` | `name` | `name` |
| `version` | `version` | `version` |
| `description` | `description` | `description` |
| `homepage` | N/A | N/A |
| `skills` | (auto-discovered in `skills/`) | (auto-discovered in `skills/`) |
| `commands` | `commands` | N/A (uses rules) |
| `agents` | `agents` | N/A (uses rules/agents) |
| `hooks` | `hooks` path | `.cursor/hooks.json` |
| `mcp` | `mcpServers` | `mcp.json` |
| `dependencies` | N/A (no native equivalent) | N/A (no native equivalent) |
| `systemDependencies` | Skill `compatibility` field | N/A |
| `quality` | N/A | N/A |
