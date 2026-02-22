## 13. Dependency Resolution

Dependency resolution is a core differentiator of UAAPS over raw vendor plugin systems. Like npm, pip, or Cargo, the package manager MUST resolve a directed acyclic graph (DAG) of requirements before installation.

### 13.1 Dependency Types

| Type | Manifest Key | Required to Install? | Semantics |
|------|-------------|---------------------|-----------|
| **Direct** | `dependencies` | **Yes** — install fails | Other agent packages this package requires to function. |
| **Optional** | `optionalDependencies` | No — install succeeds | Enhances functionality if present. Graceful degradation if absent. |
| **Peer** | `peerDependencies` | **Yes** — warn/fail | MUST be provided by the host project or another top-level install. Not auto-installed. |
| **System** | `systemDependencies` | **Yes** — pre-flight check | OS binaries, language runtimes, pip/npm packages, MCP servers. |

### 13.2 Version Constraint Syntax

UAAPS uses **SemVer 2.0** with npm-style range operators:

| Syntax | Meaning | Example |
|--------|---------|---------|
| `1.2.3` | Exact version | Only `1.2.3` |
| `^1.2.3` | Compatible with (`>=1.2.3 <2.0.0`) | Minor + patch updates OK |
| `~1.2.3` | Approximately (`>=1.2.3 <1.3.0`) | Patch updates only |
| `>=1.2.0` | Minimum version | `1.2.0` or higher |
| `>=1.0.0 <3.0.0` | Range | Between `1.0.0` and `2.x.x` |
| `*` | Any version | No constraint |

### 13.3 Lock File — `package.agent.lock`

After resolution, the solver writes a deterministic **lock file** pinning exact versions:

```jsonc
{
  "lockVersion": 1,
  "resolved": {
    "code-review": {
      "version": "1.3.2",
      "source": "marketplace:anthropics/skills",
      "integrity": "sha256-abc123...",
      "dependencies": {
        "git-utils": "1.0.1"
      }
    },
    "testing-utils": {
      "version": "2.4.0",
      "source": "npm:@aamregistry/testing-utils",
      "integrity": "sha256-def456..."
    },
    "git-utils": {
      "version": "1.0.1",
      "source": "marketplace:anthropics/skills",
      "integrity": "sha256-ghi789..."
    }
  },
  "systemChecks": {
    "python": "3.12.1",
    "binaries": {
      "git": "/usr/bin/git",
      "docker": "/usr/local/bin/docker"
    },
    "pip": {
      "pypdf": "4.1.0",
      "pdfplumber": "0.11.4"
    }
  }
}
```

**Behavior**:
- If `package.agent.lock` exists, install uses **locked versions** (reproducible builds).
- `aam install` reads the lock file; `aam update` regenerates it.
- Lock files SHOULD be committed to version control for deterministic environments.

### 13.4 Resolution Algorithm

```
1. Parse root package.agent.json
2. Build initial requirement graph from dependencies + peerDependencies
3. For each unresolved requirement:
   a. Fetch available versions from source (marketplace, npm, local)
   b. Filter by version constraint
   c. Select highest compatible version (prefer stable over pre-release)
   d. Recursively resolve transitive dependencies
4. Detect conflicts:
   a. If two packages require incompatible versions of the same dependency → ERROR
   b. If peerDependency is unmet by root or any ancestor → WARN (or ERROR if strict)
5. Flatten dependency tree (hoist where possible, like npm)
6. Run system dependency pre-flight checks
7. Write package.agent.lock
8. Install resolved packages to install directory
```

### 13.5 Resolution Strategies

| Strategy | Flag | Behavior |
|----------|------|----------|
| **Default** | (none) | Highest compatible version per constraint. |
| **Locked** | `--frozen` | Only use versions from lock file. Fail if lock is stale. |
| **Minimal** | `--minimal` | Lowest version satisfying each constraint (CI reproducibility). |
| **Latest** | `--latest` | Ignore constraints, install latest of everything (dangerous). |
| **Dry-run** | `--dry-run` | Resolve and display tree without installing. |

### 13.6 Package Dependencies (Agent-to-Agent)

Agent packages can depend on other agent packages. This enables composition:

```jsonc
// package.agent.json for "full-stack-review"
{
  "name": "full-stack-review",
  "version": "1.0.0",
  "dependencies": {
    "code-review": "^1.0.0",           // Reuse code review skills
    "security-scanning": "^2.0.0",     // Reuse security agents
    "testing-utils": "^1.5.0"          // Reuse test generation
  }
}
```

**Transitive resolution**: If `code-review@1.3.2` itself depends on `git-utils@^1.0.0`, the solver installs `git-utils` automatically.

**Namespace isolation**: Each installed dependency's skills, commands, and agents are namespaced under `dependency-name:artifact-name` to prevent collisions.

### 13.7 System Dependencies

System dependencies are **not installed** by the package manager — they are **verified** via pre-flight checks.

```jsonc
"systemDependencies": {
  // === Runtime requirements ===
  "python": ">=3.10",                  // Check: python3 --version
  "node": ">=18",                      // Check: node --version

  // === Package manager installs ===
  "packages": {
    "pip": ["pypdf>=4.0", "pdfplumber"],
    "npm": ["prettier@^3.0", "eslint"],
    "brew": ["graphviz", "poppler"],
    "apt": ["poppler-utils", "tesseract-ocr"],
    "cargo": ["ripgrep"]
  },

  // === Binary availability ===
  "binaries": ["git", "docker", "gh"],  // Must exist on $PATH

  // === MCP server requirements ===
  "mcp-servers": ["filesystem", "github"]
}
```

#### Pre-flight Check Flow

```
aam install my-package
  → Checking system dependencies...
  ✓ python 3.12.1 (>=3.10)
  ✓ node 20.11.0 (>=18)
  ✓ git found at /usr/bin/git
  ✗ docker not found on $PATH
  ✓ pip: pypdf 4.1.0 installed
  ✗ pip: pdfplumber not installed
  
  ⚠ Missing system dependencies:
    • docker: Install via https://docs.docker.com/get-docker/
    • pdfplumber: Run `pip install pdfplumber`
  
  Install package anyway? [y/N/auto-install]
```

#### Auto-Install Behavior

| Package Manager | Auto-install | Flag |
|----------------|-------------|------|
| `pip` | ✅ Supported | `--install-system-deps` |
| `npm` | ✅ Supported | `--install-system-deps` |
| `brew` / `apt` | ⚠️ Prompt only | Requires `--install-system-deps --allow-sudo` |
| `binaries` | ❌ Never | Manual install instructions provided |
| `mcp-servers` | ✅ Via MCP registry | `--install-system-deps` |

### 13.8 Skill-Level Dependencies

Individual skills can declare their own dependencies in SKILL.md frontmatter, complementing the package-level manifest:

```yaml
---
name: pdf-processing
description: Extract text and fill forms in PDFs.
compatibility:
  requires:
    - python>=3.10
    - pypdf>=4.0
    - pdfplumber
  packages:                          # UAAPS extension
    - code-review/git-utils          # Depends on another skill
  optional:
    - tesseract-ocr                  # For OCR, graceful degradation
---
```

**Resolution order**:
1. Package-level `systemDependencies` checked first (covers most skills).
2. Skill-level `compatibility.requires` checked on activation (lazy check).
3. Skill-level `compatibility.packages` resolved transitively if referencing other skills.

### 13.9 Conflict Resolution

#### Version Conflicts

When two packages require incompatible versions of the same dependency:

```
my-app
├── code-review@1.3.2
│   └── git-utils@^1.0.0  →  resolves to 1.0.5
└── deployment@2.0.0
    └── git-utils@^2.0.0  →  resolves to 2.1.0   ← CONFLICT
```

**Resolution strategies**:

| Strategy | Behavior | When to Use |
|----------|----------|-------------|
| **Fail** (default) | Error, refuse to install | Safety-first environments |
| **Duplicate** | Install both versions, isolate namespaces | When skills don't share state |
| **Override** | Use `resolutions` field to force a version | When you know compatibility holds |
| **Peer promote** | Lift to `peerDependencies`, let host decide | Library packages |

#### Resolution Overrides

```jsonc
// package.agent.json
{
  "resolutions": {
    "git-utils": "2.1.0"              // Force all transitive refs to this version
  }
}
```

#### Skill Name Conflicts

When two installed packages export skills with the same `name`:

- Skills are **namespaced** by package: `code-review:lint-check` vs `testing:lint-check`.
- If a skill is referenced without namespace, the **closest scope wins** (project > personal > plugin).
- Explicit disambiguation: `aam resolve skill lint-check` lists all matches.

### 13.10 Dependency Tree Commands

| Command | Description |
|---------|-------------|
| `aam install` | Install all dependencies from manifest (or lock file). |
| `aam install <pkg>` | Add a package and resolve. |
| `aam update` | Re-resolve all dependencies, update lock file. |
| `aam update <pkg>` | Update a single package within constraints. |
| `aam tree` | Display the full resolved dependency tree. |
| `aam tree --depth=1` | Show only direct dependencies. |
| `aam why <pkg>` | Explain why a package is installed (which parent requires it). |
| `aam check` | Verify all dependencies are installed and compatible. |
| `aam check --system` | Run system dependency pre-flight checks only. |
| `aam outdated` | List packages with newer versions available. |
| `aam audit` | Check dependencies for known security issues. |
| `aam dedupe` | Flatten tree, remove unnecessary duplicates. |
| `aam prune` | Remove installed packages not in the dependency tree. |

### 13.11 Install Directory Layout

```
.agent-packages/                       # Install root (analogous to node_modules/)
├── .lock                              # package.agent.lock (symlink or copy)
├── code-review/                       # Installed package
│   ├── package.agent.json
│   ├── skills/
│   │   └── lint-review/
│   │       └── SKILL.md
│   └── commands/
│       └── review.md
├── testing-utils/                     # Transitive dependency
│   ├── package.agent.json
│   └── skills/
│       └── test-gen/
│           └── SKILL.md
└── git-utils/                         # Shared dependency (hoisted)
    ├── package.agent.json
    └── skills/
        └── git-diff/
            └── SKILL.md
```

**Hoisting**: Like npm, shared dependencies are hoisted to the top level of `.agent-packages/` when version-compatible. This reduces duplication and simplifies skill discovery.

### 13.12 Dependency Resolution in CI/CD

For reproducible builds in CI environments:

```yaml
# .github/workflows/agent-check.yml
- name: Install agent packages (frozen)
  run: aam install --frozen

- name: Verify system dependencies
  run: aam check --system

- name: Audit for security issues
  run: aam audit --severity=high
```

The `--frozen` flag ensures CI uses exactly the versions from `package.agent.lock`, failing if the lock file is out of date.

### 13.13 Install Modes & Package Cache

The `aam` CLI supports three install modes. The mode determines where resolved packages are written and whether a shared cache is consulted.

#### Install Mode Overview

| Mode | Install Location | Lock File Used | Typical Use |
|------|-----------------|---------------|-------------|
| **Project-local** (default) | `./.agent-packages/` | `./package.agent.lock` | Per-project installs, checked into VCS |
| **Global** | `~/.aam/packages/` | `~/.aam/global.lock` | User-wide utilities, not project-specific |
| **CI ephemeral** | `$AAM_INSTALL_DIR` or system temp | `./package.agent.lock` (read-only) | Fresh environment per pipeline run |

```bash
aam install                          # Project-local (default)
aam install --global                 # Global install
aam install --frozen                 # Project-local, lock file is authoritative
AAM_INSTALL_DIR=/tmp/aam aam install # CI ephemeral override
```

#### Installing from a Specific Vendor

When multiple vendors publish a package with the same base name, always qualify with the `@scope` prefix to target the correct author:

```bash
# Unscoped — installs the community package named "code-review"
aam install code-review

# Scoped — installs @myorg's code-review, not @otherorg's
aam install @myorg/code-review
aam install @otherorg/code-review

# Specific version from a specific vendor
aam install @myorg/code-review@1.3.2
aam install @myorg/code-review@^1.0.0

# Specific dist-tag from a vendor (e.g. enterprise-approved tag)
aam install @myorg/code-review@stable
aam install @myorg/code-review@bank-approved

# From a private registry owned by a specific vendor
aam install @myorg/code-review --registry https://pkg.myorg.internal/aam
```

Both `@myorg/code-review` and `@otherorg/code-review` can be installed simultaneously — they are stored under `myorg--code-review/` and `otherorg--code-review/` respectively (see §12.2 and §13.13 author collision rule). Skills from each are namespaced accordingly: `myorg--code-review:lint-check` vs `otherorg--code-review:lint-check`.

#### Shared Download Cache

All install modes share a **download cache** at `~/.aam/cache/`. The cache stores downloaded `.aam` archives keyed by `package@version` + integrity hash. Packages are never re-downloaded if a valid cached copy exists.

```
~/.aam/
├── cache/                                        # Shared download cache (all modes)
│   ├── myorg--code-review-1.3.2-sha256-abc123.aam
│   ├── otherorg--code-review-2.0.0-sha256-def456.aam
│   └── testing-utils-2.4.0-sha256-ghi789.aam
├── packages/                                     # Global install root
│   ├── myorg--code-review/                       # @myorg/code-review
│   ├── otherorg--code-review/                    # @otherorg/code-review
│   └── testing-utils/                            # unscoped package
├── global.lock                                   # Global install lock file
└── config.yaml                                   # CLI configuration
```

**Author collision rule**: The `@scope/name` → `scope--name` filesystem mapping (defined in §12.2) is the sole mechanism preventing collisions between same-named packages from different authors. Two packages that share a base name but differ in scope (`@myorg/code-review` and `@otherorg/code-review`) MUST be stored under distinct directory names (`myorg--code-review/` and `otherorg--code-review/` respectively) in both the cache and global packages directory. Implementations MUST NOT flatten scoped names to their bare base name in any install root.

#### Cache Invalidation Rules

| Trigger | Behaviour |
|---------|-----------|
| Integrity hash mismatch | Cache entry is rejected; package re-downloaded |
| `aam cache clean` | Entire cache cleared |
| `aam cache clean <pkg>@<version>` | Single entry evicted |
| `aam install --no-cache` | Cache bypassed for this run; result not cached |
| Registry republish of same version | Cache is **not** invalidated automatically — use `--no-cache` or bump version |

Cached archives MUST be verified against their `sha256` integrity hash before extraction. A corrupted or tampered archive MUST be rejected and the cache entry evicted.

#### Lock File Precedence Rules

When both a user-level (`~/.aam/global.lock`) and project-level (`./package.agent.lock`) lock file exist, resolution follows this precedence order (highest wins):

1. **`./package.agent.lock`** — project lock file always takes precedence for project-local installs.
2. **`~/.aam/global.lock`** — governs `--global` installs only; never consulted for project-local resolution.
3. **`resolutions` field** in `package.agent.json` — overrides both lock files for named packages (see §13.9).

Implementations MUST NOT mix project-local and global lock files in the same resolution pass. A global install (`--global`) MUST NOT read or modify `./package.agent.lock`.

#### Global vs Local Resolution Rules

| Rule | Project-local | Global |
|------|--------------|--------|
| Artifact lookup scope | `./.agent-packages/` only | `~/.aam/packages/` only |
| Lock file written | `./package.agent.lock` | `~/.aam/global.lock` |
| Hoisting target | `./.agent-packages/` root | `~/.aam/packages/` root |
| Peer dependency resolution | Ancestor chain within project tree | Global install tree |
| Available to agent platforms | Only when agent runs in project directory | Always — platform SHOULD scan `~/.aam/packages/` |

Agent platforms SHOULD load globally installed packages as a lower-priority fallback after project-local packages. Project-local packages MUST shadow global packages of the same name.
