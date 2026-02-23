## 15. Governance & Policy Gates

Governance controls who can install what, and provides a tamper-evident record of actions. Policy gates are **client-side** (no server required); approval workflows require an HTTP registry.

### 15.1 Client-Side Policy Gates

Configured in `config.yaml`, enforced by the CLI **before** any install or publish:

```yaml
governance:
  install_policy:
    allowed_scopes: ["@myorg", "@trusted-vendor"] # only allow these scopes
    require_signature: true                        # block unsigned packages
    require_tag: "stable"                          # only install tagged versions
    blocked_packages: ["@sketchy/*"]               # glob-matched block list
  publish_policy:
    require_signature: true                        # must sign before publishing
```

| Gate | Trigger | Effect |
|------|---------|--------|
| `allowed_scopes` | `aam install` | Block packages outside listed scopes |
| `require_signature` | `aam install` | Block unsigned packages |
| `require_tag` | `aam install` | Only allow versions with a specific tag |
| `blocked_packages` | `aam install` | Block specific packages (glob patterns) |
| `require_signature` | `aam pkg publish` | Require signing before publish |

### 15.2 Approval Workflows (HTTP Registry)

When `require_approval: true` is set and publishing to an HTTP registry:

1. `aam pkg publish` → uploads with `approval_status: pending`
2. Approvers receive notification
3. Approver runs `aam approve @org/agent@1.2.0`
4. Only approved packages appear in search/install

### 15.3 Audit Events

Every registry mutation is logged immutably:

| Event | Description |
|-------|-------------|
| `package.publish` | New version published |
| `version.deprecate` | Version marked as deprecated |
| `version.yank` | Version marked as yanked |
| `version.yank.undo` | Yank reversed |
| `tag.set` | Dist-tag added or updated |
| `tag.remove` | Dist-tag removed |
| `version.approve` | Version approved |
| `version.reject` | Version rejected |
| `ownership.transfer` | Package ownership transferred |

#### Structured Audit Log Format

Registries MUST store audit events in a structured, machine-parseable format. Each log entry MUST conform to the following schema:

```json
{
  "version": 1,
  "event": "package.publish",
  "timestamp": "2026-02-22T14:30:00Z",
  "actor": {
    "identity": "user@myorg.com",
    "auth_method": "oidc"
  },
  "target": {
    "package": "@myorg/code-review",
    "version": "1.3.0"
  },
  "metadata": {
    "signature_method": "sigstore",
    "provenance_level": "L2",
    "source_commit": "abc123def456",
    "archive_integrity": "sha256-abc123..."
  },
  "registry": "https://pkg.myorg.internal/aam"
}
```

#### Audit Log Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | `number` | **Yes** | Audit log format version. Currently `1`. |
| `event` | `string` | **Yes** | Event type from the audit events table above. |
| `timestamp` | `string` | **Yes** | ISO 8601 timestamp of the event. |
| `actor.identity` | `string` | **Yes** | Authenticated identity that performed the action. |
| `actor.auth_method` | `string` | **Yes** | Authentication method used: `oidc`, `token`, `basic`, `anonymous`. |
| `target.package` | `string` | **Yes** | Full package name (scoped or unscoped). |
| `target.version` | `string` | No | Package version affected (absent for package-level events like `ownership.transfer`). |
| `metadata` | `object` | No | Event-specific metadata. Content varies by event type. |
| `registry` | `string` | **Yes** | Registry URL where the event occurred. |

#### Querying Audit Logs

```bash
aam audit-log @myorg/code-review                     # All events for a package
aam audit-log @myorg/code-review --event publish      # Filter by event type
aam audit-log --actor user@myorg.com --since 2026-01-01  # Filter by actor and date
```

HTTP registries SHOULD expose audit logs via `GET /packages/:name/audit-log` with pagination support (see §12.6 Pagination).
