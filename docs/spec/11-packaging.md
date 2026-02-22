## 12. Packaging & Distribution

### UAAPS Registry

UAAPS supports two registry types that share the same package format (`.aam` archives) but differ in transport:

| Type | When to use |
|------|-------------|
| **Local filesystem** | Offline use, air-gapped environments, CI mirrors, monorepos |
| **HTTP remote** | Public registry, private org registry, hosted SaaS |

The `aam` CLI auto-detects the registry type from the URL scheme:

```bash
# HTTP remote registry
aam install @myorg/code-review --registry https://aamregistry.io

# Local filesystem registry
aam install @myorg/code-review --registry file:///opt/aam-registry

# Default registry (HTTP, configured in ~/.aam/config.yaml)
aam install @myorg/code-review
```

Registry configuration is managed in `~/.aam/config.yaml` or `.aam/config.yaml`:

```yaml
registries:
  default: https://aamregistry.io
  sources:
    - name: myorg-remote
      url: https://pkg.myorg.internal/aam
      auth: token                          # see §12.6 for auth types
    - name: myorg-local
      url: file:///opt/aam-registry
```

> Other distribution channels (Git marketplace, npm bridge, direct install, project-bundled) are supported by the `aam` CLI — see `aam_cli_new.md`.

### 12.5 Local Filesystem Registry

A local filesystem registry is a **directory tree** on disk following a defined layout. No server process is required. It is suitable for offline installs, CI artifact mirrors, and air-gapped enterprise environments.

#### Directory Layout

```
<registry-root>/
├── index.json                          # Full package index (all packages + versions)
├── packages/
│   ├── code-review/                    # Unscoped package
│   │   ├── meta.json                   # Package metadata (all versions)
│   │   └── versions/
│   │       ├── 1.0.0.aam
│   │       ├── 1.0.0.aam.sha256        # Detached SHA-256 checksum
│   │       ├── 1.1.0.aam
│   │       └── 1.1.0.aam.sha256
│   └── myorg--code-review/             # Scoped package (@myorg/code-review)
│       ├── meta.json
│       └── versions/
│           ├── 2.0.0.aam
│           └── 2.0.0.aam.sha256
└── dist-tags.json                      # Global dist-tag → version map
```

#### `index.json`

A flat list of all packages in the registry. Implementations MUST regenerate this file after every publish or unpublish operation.

```json
{
  "formatVersion": 1,
  "updatedAt": "2026-02-22T14:00:00Z",
  "packages": [
    { "name": "code-review",         "latest": "1.1.0", "versions": ["1.0.0", "1.1.0"] },
    { "name": "@myorg/code-review",  "latest": "2.0.0", "versions": ["2.0.0"] }
  ]
}
```

#### `packages/<name>/meta.json`

Per-package metadata covering all published versions.

```json
{
  "name": "@myorg/code-review",
  "versions": {
    "2.0.0": {
      "version": "2.0.0",
      "description": "Code review skills for myorg",
      "author": "myorg",
      "publishedAt": "2026-02-20T10:00:00Z",
      "integrity": "sha256-abc123...",
      "tarball": "versions/2.0.0.aam"
    }
  },
  "dist-tags": {
    "latest": "2.0.0",
    "stable": "2.0.0"
  }
}
```

#### `dist-tags.json`

Registry-wide dist-tag snapshot for fast tag resolution without reading every `meta.json`.

```json
{
  "code-review":        { "latest": "1.1.0", "stable": "1.0.0" },
  "@myorg/code-review": { "latest": "2.0.0" }
}
```

#### Filesystem Registry Rules

| Rule | Requirement |
|------|-------------|
| Package directory name MUST use the `scope--name` mapping (§12.2) | MUST |
| Every `.aam` file MUST have a sibling `.aam.sha256` file | MUST |
| `index.json` MUST be regenerated atomically after each mutation | MUST |
| Symlinks outside the registry root are forbidden | MUST NOT |
| The registry root MAY be read-only (install-only mirror) | MAY |

#### CLI Commands for Local Registries

```bash
# Initialise a new local registry
aam registry init file:///opt/aam-registry

# Publish a package to a local registry
aam pkg publish --registry file:///opt/aam-registry

# Rebuild index.json after manual archive placement
aam registry reindex file:///opt/aam-registry

# List all packages in a local registry
aam registry ls file:///opt/aam-registry
```

---

### 12.6 HTTP Registry Protocol

> **Status: Skeleton** — endpoint signatures and auth model are defined below. Full request/response schemas, pagination rules, rate-limit headers, and error codes will be detailed in a dedicated Registry Protocol document in a future spec revision.

An HTTP registry is an HTTPS server implementing the UAAPS Registry Protocol. The official public registry is `https://aamregistry.io`. Organizations MAY self-host a private registry.

#### Base URL

All endpoints are relative to the registry base URL. Implementations MUST serve the API over HTTPS. Plain HTTP MUST NOT be used for registries handling private packages or authentication tokens.

#### Endpoint Index

> **TODO**: This is a preliminary endpoint list and is not final. Additional endpoints (e.g. search, package transfer, org management, audit log) will be added in the Registry Protocol document.

| Method | Path | Purpose | Auth required |
|--------|------|---------|--------------|
| `GET` | `/` | Registry metadata & capabilities | No |
| `GET` | `/packages` | List all packages (paginated) | No (public) / Yes (private) |
| `GET` | `/packages/:name` | Package metadata (all versions) | No (public) / Yes (private) |
| `GET` | `/packages/:name/:version` | Single version metadata | No (public) / Yes (private) |
| `GET` | `/packages/:name/:version/tarball` | Download `.aam` archive | No (public) / Yes (private) |
| `GET` | `/packages/:name/:version/signature` | Fetch signature bundle | No |
| `GET` | `/packages/:name/dist-tags` | List dist-tags for package | No |
| `PUT` | `/packages/:name/dist-tags/:tag` | Set a dist-tag | Yes |
| `DELETE` | `/packages/:name/dist-tags/:tag` | Remove a dist-tag | Yes |
| `POST` | `/packages` | Publish a new package version | Yes |
| `DELETE` | `/packages/:name/:version` | Unpublish a version | Yes |
| `GET` | `/approvals` | List pending approval requests | Yes |
| `POST` | `/approvals/:id/approve` | Approve a publish request | Yes |
| `POST` | `/approvals/:id/reject` | Reject a publish request | Yes |

#### Authentication

HTTP registries MAY require authentication. The `aam` CLI supports the following auth types, configured per registry in `~/.aam/config.yaml`:

| Auth type | Config value | Transport |
|-----------|-------------|-----------|
| No auth (public) | `auth: none` | — |
| Static token | `auth: token` | `Authorization: Bearer <token>` header |
| OIDC / Sigstore keyless | `auth: oidc` | Short-lived token via OIDC provider |
| Basic (legacy, not recommended) | `auth: basic` | `Authorization: Basic <base64>` header |

```yaml
# ~/.aam/config.yaml
registries:
  sources:
    - name: myorg
      url: https://pkg.myorg.internal/aam
      auth: token
      token: "${MYORG_AAM_TOKEN}"       # resolved from environment variable
```

Tokens MUST be stored in environment variables or a secrets manager. Tokens MUST NOT be committed to version control. The `aam login` command handles interactive token acquisition and secure local storage.

```bash
aam login --registry https://pkg.myorg.internal/aam   # interactive login
aam logout --registry https://pkg.myorg.internal/aam  # remove stored credential
aam whoami --registry https://pkg.myorg.internal/aam  # show current identity
```

#### Error Response Format

All error responses MUST use `application/json` with this structure:

```json
{
  "error": {
    "code": "PACKAGE_NOT_FOUND",
    "message": "Package @myorg/code-review@3.0.0 does not exist.",
    "docs": "https://aamregistry.io/errors/PACKAGE_NOT_FOUND"
  }
}
```

#### Standard Status Codes

| Code | Meaning |
|------|---------|
| `200` | Success |
| `201` | Published successfully |
| `400` | Malformed request |
| `401` | Authentication required |
| `403` | Insufficient permissions |
| `404` | Package or version not found |
| `409` | Version already exists (publish conflict) |
| `422` | Validation failed (manifest schema error) |
| `429` | Rate limit exceeded |
| `503` | Registry temporarily unavailable |

> Full pagination headers, rate-limit headers, conditional request support (`ETag`, `If-None-Match`), and approval workflow request/response bodies will be defined in the Registry Protocol document.

### 12.1 Archive Distribution Format

Packages are distributed as **`.aam` archives** (gzipped tar), providing a binary-safe, single-file distribution unit.

```
my-package-1.0.0.aam   # produced by: aam pkg pack
```

**Archive constraints:**
- Maximum size: **50 MB**
- MUST contain `package.agent.json` at root
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
