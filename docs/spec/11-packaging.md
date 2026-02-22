## 12. Packaging & Distribution

### UAAPS Registry

Packages are published to and installed from a **UAAPS registry** — an HTTP server implementing the UAAPS registry protocol. The official public registry is `registry.agentpkg.io`; organizations can self-host a private registry.

```bash
# Install from the default registry
aam install @myorg/code-review

# Install from a private registry
aam install @myorg/code-review --registry https://pkg.myorg.internal/aam

# Publish to a registry
aam pkg publish --registry https://pkg.myorg.internal/aam
```

Registry configuration is managed in `~/.aam/config.yaml` or `.aam/config.yaml`:

```yaml
registries:
  default: https://registry.agentpkg.io
  sources:
    - name: myorg
      url: https://pkg.myorg.internal/aam
```

> Other distribution channels (Git marketplace, npm bridge, direct install, project-bundled) are supported by the `aam` CLI — see `aam_cli_new.md`.

### 12.1 Archive Distribution Format

Packages are distributed as **`.aam` archives** (gzipped tar), providing a binary-safe, single-file distribution unit.

```
my-package-1.0.0.aam   # produced by: aam pkg pack
```

**Archive constraints:**
- Maximum size: **50 MB**
- Must contain `package.agent.json` at root
- No symlinks outside the package directory
- No absolute paths
- SHA-256 checksum stored in `package.agent.lock` post-install

### 12.2 Scoped Package Names

Following npm convention, packages support a `@scope/name` format for namespacing under an organization or author:

| Format | Example | Use Case |
|--------|---------|----------|
| Unscoped | `code-review` | Community / public packages |
| Scoped | `@myorg/code-review` | Organization packages |

**Name rules:**
- Unscoped: `[a-z0-9-]`, max 64 chars
- Scope: `@[a-z0-9][a-z0-9_-]{0,63}`
- Full scoped name: max 130 chars

**Filesystem mapping** — the `@scope/name` format is converted using double-hyphen:

| Package Name | Filesystem Name |
|-------------|-----------------|
| `code-review` | `code-review` |
| `@myorg/code-review` | `myorg--code-review` |

The `--` separator is reversible and unambiguous because valid name segments cannot start with hyphens.

### 12.3 Dist-Tags

Dist-tags are **named aliases for package versions**, enabling installs like `aam install @org/agent@stable` or enterprise gates like `bank-approved`.

#### Default Tags

| Tag | Behavior |
|-----|----------|
| `latest` | Automatically set to the newest published version |
| `stable` | Opt-in; set manually via `aam dist-tag add` |

#### Custom Tags

Organizations can define arbitrary tags (e.g., `staging`, `bank-approved`, `qa-passed`).

**Tag rules:**
- Lowercase alphanumeric + hyphens only
- Max 32 characters
- Cannot be a valid SemVer string (prevents ambiguity)

#### Manifest Declaration (optional)

```jsonc
// package.agent.json
{
  "name": "@myorg/agent",
  "version": "1.2.0",
  "dist-tags": {
    "stable": "1.1.0",     // Maintained by publisher
    "latest": "1.2.0"
  }
}
```

### 12.4 Portable Bundles

A portable bundle is a **self-contained, pre-compiled archive** for a specific target platform. It contains artifacts already transformed to the platform's native format — no install-time resolution needed.

**Use case:** Distributing packages via Slack, email, or air-gapped environments without a registry.

```bash
aam pkg build --target cursor
# → dist/my-package-1.0.0-cursor.bundle.aam

aam pkg build --target copilot
# → dist/my-package-1.0.0-copilot.bundle.aam

aam pkg build --target all
# → one bundle per configured platform
```

**Bundle structure (tar.gz internally):**

```
my-package-1.0.0-cursor.bundle.aam
├── bundle.json               # Bundle manifest
├── .cursor/
│   ├── skills/...            # Pre-compiled platform artifacts
│   ├── rules/...
│   └── commands/...
└── package.agent.json        # Original manifest for reference
```

**`bundle.json` schema:**

```json
{
  "format": "uaaps-bundle",
  "version": "1.0",
  "package": "@author/my-agent",
  "package_version": "1.2.0",
  "target": "cursor",
  "built_at": "2026-02-19T14:30:00Z",
  "checksum": "sha256:...",
  "artifacts": [
    { "type": "skill", "name": "my-skill", "path": ".cursor/skills/author--my-skill/" },
    { "type": "agent", "name": "my-agent", "path": ".cursor/rules/agent-author--my-agent.mdc" }
  ]
}
```

**Installing from a bundle:**

```bash
aam install ./dist/my-package-1.0.0-cursor.bundle.aam
# Deploys immediately — no dependency resolution needed
```
