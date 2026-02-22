## 17. Security Considerations

1. **Only install packages from trusted sources** — skills can execute code and access the filesystem.
2. **Audit all scripts** before enabling — look for network calls, file access, and unexpected operations.
3. **Use `allowed-tools`** to restrict skill capabilities where possible.
4. **Hook commands** run as the user — apply least-privilege principles.
5. **MCP servers** SHOULD NOT expose secrets; use environment variable injection.
6. **Environment Variables**: Packages MUST explicitly declare required environment variables in the `env` manifest field. Host platforms SHOULD prompt users securely for these values rather than storing them in plaintext.
7. **Permissions Model**: The `permissions` field in `package.agent.json` is OPTIONAL. When it is present, platforms MUST enforce it as defined in §17.1. A platform that ignores a declared `permissions` field is non-compliant. When the field is absent, platforms MAY apply their own default restrictions.
8. **Sandbox** untrusted packages before production deployment.
9. **Verify lock file integrity** — `package.agent.lock` hashes prevent supply-chain tampering.
10. **Run `aam audit`** regularly to check dependencies for known vulnerabilities.
11. **System dependency auto-install** never uses `sudo` without explicit `--allow-sudo` flag.
12. **Transitive dependencies** SHOULD be audited — `aam tree` reveals the full graph.

---

### 17.1 Permissions Model

The `permissions` field in `package.agent.json` is OPTIONAL. When present, it declares the minimum capabilities a package requires. Packages SHOULD request the narrowest permissions needed to function.

When `permissions` is present, platforms MUST deny any capability not explicitly declared within it. When `permissions` is absent, platform-default restrictions apply and no specific enforcement is mandated by this spec.

#### Filesystem Permissions

```jsonc
"fs": {
  "read":  ["src/**", "docs/**"],   // Glob patterns the package MAY read
  "write": ["output/**"]            // Glob patterns the package MAY write or delete
}
```

| Rule | Requirement |
|------|-------------|
| Patterns use forward-slash glob syntax (`**`, `*`, `?`) | MUST |
| Paths are relative to the project root at install time | MUST |
| Absolute paths are forbidden in `fs` permissions | MUST NOT |
| A `write` grant does not imply `read` — declare both if needed | MUST |
| Attempts to access paths outside declared globs MUST be denied | MUST |
| Platform MAY present a user prompt to approve `write` grants at install time | MAY |

#### Network Permissions

```jsonc
"network": {
  "hosts":   ["api.github.com", "*.npmjs.org"],
  "schemes": ["https"]
}
```

| Rule | Requirement |
|------|-------------|
| `hosts` entries are exact hostnames or single-level wildcard (`*.example.com`) | MUST |
| Multi-level wildcards (`**.example.com`) are forbidden | MUST NOT |
| `schemes` restricts URI scheme; permitted values: `https`, `http`, `wss`, `ws` | MUST |
| If `schemes` is omitted, only `https` is permitted | MUST |
| IP address literals and CIDR ranges are not supported as `hosts` values | MUST NOT |
| Outbound connections to hosts or schemes not in the declared list MUST be blocked | MUST |
| Omitting `network` entirely means no network access is permitted | MUST |

#### Shell Permissions

```jsonc
"shell": {
  "allow":    true,
  "binaries": ["git", "npm", "python3"]
}
```

| Rule | Requirement |
|------|-------------|
| `allow: false` (or omitting `shell`) prohibits all shell execution | MUST |
| `allow: true` without `binaries` permits any binary — use only when unavoidable | SHOULD NOT |
| When `binaries` is present, only the listed executable names MAY be invoked | MUST |
| Binary names are matched against the basename of the resolved executable | MUST |
| Shell metacharacters and interpreter flags (`-c`, `eval`, etc.) that bypass the allow-list MUST be rejected | MUST |

#### Enforcement Requirements

Platforms MUST enforce permissions at runtime. The following violations MUST result in a denied operation and a structured error surfaced to the user:

| Violation | Required response |
|-----------|-------------------|
| `fs` read outside declared globs | Deny + log |
| `fs` write/delete outside declared globs | Deny + log |
| Network connection to undeclared host | Deny + log |
| Network connection on undeclared scheme | Deny + log |
| Shell invocation when `allow: false` | Deny + log |
| Shell invocation of binary not in `binaries` | Deny + log |

Platforms that cannot enforce a specific permission category (e.g., no network interception) MUST warn the user at install time that enforcement is partial and indicate which categories are unenforced.

#### Permission Audit at Install and Validate Time

Both `aam install` and `aam validate` MUST run a permission security audit and emit `WARN`-level output for over-privileged or missing declarations (see full condition list in §16 Validation Rules → Permissions Security Audit). These warnings MUST NOT block the operation — they are advisory only.

Example output during install:

```
$ aam install @myorg/data-exporter

  ✓ Resolved 3 packages
  ✓ Integrity verified
  ⚠ Permission warnings for @myorg/data-exporter:
      • No permissions declared — platform-default restrictions apply.
      • (Add a "permissions" field for explicit least-privilege.)

  Installed successfully. Run `aam validate --permissions` for details.
```

The `aam validate --permissions` flag runs the audit in isolation without a full package validation pass.
