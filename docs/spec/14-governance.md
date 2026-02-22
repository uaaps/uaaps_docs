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
| `version.yank` | Version marked as yanked |
| `tag.set` | Dist-tag added or updated |
| `tag.remove` | Dist-tag removed |
| `version.approve` | Version approved |
| `version.reject` | Version rejected |
| `ownership.transfer` | Package ownership transferred |
