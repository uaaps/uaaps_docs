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
    "fs": {                                     // Filesystem access — path-scoped
      "read":  ["src/**", "docs/**"],           //   Glob patterns the package may read
      "write": ["output/**", ".cache/**"]       //   Glob patterns the package may write/delete
    },
    "network": {                                // Network access
      "hosts": ["api.github.com", "*.npmjs.org"], // Exact host or wildcard subdomain
      "schemes": ["https"]                      // Allowed URI schemes (https | http | wss)
    },
    "shell": {                                  // Shell execution
      "allow": true,                            //   false = no shell; true = allow listed binaries
      "binaries": ["git", "npm", "python3"]     //   Explicit allow-list (ignored when allow=false)
    }
  },

  // === QUALITY (optional) ===
  "quality": {
    "tests": [
      { "name": "unit-tests", "command": "pytest tests/", "description": "Unit tests" }
    ],
    "evals": [
      {
        "name": "accuracy-eval",
        "path": "evals/cases/accuracy-eval.yaml",
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

  // === DIST-TAGS (optional) ===
  // Publisher-managed version aliases. See §12.3 for full semantics.
  "dist-tags": {
    "stable": "1.0.0",                          // Manually maintained alias
    "latest": "1.0.0"                           // Auto-set on publish
  },

  // === DEPENDENCY OVERRIDES (optional) ===
  // Force a specific resolved version for a transitive dependency. See §13.9.
  "resolutions": {
    "git-utils": "2.1.0"                        // Force all transitive refs to this version
  },

  // === INSTALL MODE (optional, for adoption transitioning) ===
  // Controls deployment target when installing on platforms with partial UAAPS support.
  // See §11.1 for full semantics and deprecation path.
  "installMode": {                              // Default: "uaaps" for all platforms
    "default": "uaaps",                         // "uaaps" | "plugin"
    "claude-code": "plugin",                    // Per-platform override
    "cursor": "uaaps"
  },

  // === VENDOR EXTENSIONS (optional) ===
  // Keys MUST be "x-<vendor-id>" where vendor-id matches [a-z0-9-], max 32 chars.
  // Values MUST be JSON objects (not scalars or arrays).
  // Tools MUST silently ignore unrecognised x-* keys.
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
  marketplace: anthropics/skills   # x-<vendor-id>, value MUST be an object

x-cursor:
  category: Developer Tools
```

**Rules**:
- If both `.json` and `.yaml` exist, **JSON takes precedence**.
- CLI tools (`aam install`, etc.) generate `package.agent.json` as the canonical output.
- Lock files (`package.agent.lock`) are always JSON.

**YAML round-trip guarantee**: To ensure lossless JSON-to-YAML conversion, `package.agent.yaml` files MUST conform to the following constraints:

| Constraint | Requirement |
|-----------|-------------|
| YAML version | MUST use YAML 1.2 |
| Scalar types | MUST use only JSON-compatible types: strings, numbers, booleans, null |
| Anchors & aliases | MUST NOT use YAML anchors (`&`), aliases (`*`), or merge keys (`<<`) |
| Custom tags | MUST NOT use YAML tags (e.g. `!!python/object`) |
| Key types | All mapping keys MUST be strings |
| Duplicate keys | MUST NOT contain duplicate keys in the same mapping |

Tools that convert between JSON and YAML MUST produce output that round-trips without data loss. A manifest that cannot be losslessly converted between formats is non-conformant.

### Vendor Extensions

Vendor extensions allow platforms and tools to attach platform-specific metadata to a package manifest without conflicting with the core schema.

#### Naming Rules

- Keys MUST use the prefix `x-<vendor-id>`, where `vendor-id` is a lowercase alphanumeric slug matching `[a-z0-9-]+`, maximum 32 characters (e.g. `x-claude`, `x-cursor`, `x-copilot`).
- The `vendor-id` SHOULD match either a registered platform identifier from the [compatibility matrix](09-compatibility.md) or the package's own scope identifier (e.g. `x-myorg` for `@myorg/` scoped packages).
- Extension values MUST be JSON objects. Scalar values (strings, booleans, numbers) and arrays MUST NOT be used as the top-level value of an `x-*` key.
- Packages SHOULD NOT declare more than **5** `x-*` keys. `aam validate` MUST warn when more than 5 are present (see §16 Validation).

#### Interoperability Rules

- Tools and runtimes MUST silently ignore unrecognised `x-*` keys. Tools MUST NOT error or refuse to install a package solely because of an unknown `x-*` key.
- Registry indexing: only `x-<known-platform>` keys (those matching a registered platform) are indexed and searchable. Unknown `x-*` keys are stored but not indexed.
- Registry validation SHOULD produce a `WARN`-level diagnostic for `x-*` keys that do not match any registered platform identifier, to alert the author that the key will not be indexed.

#### Conflict Avoidance

- Two packages MUST NOT declare the same `x-*` key with structurally incompatible schemas. The registry MAY enforce a canonical JSON Schema per registered `x-<vendor>` key.
- Authors MUST NOT use another vendor's registered `x-*` key to override or shadow that vendor's metadata.
- Private extensions for in-house tooling SHOULD use the package owner's scope as the vendor-id (e.g. `x-myorg`) to avoid collisions with future registered platforms.

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
