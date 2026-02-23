## 17. Security Considerations

1. **Only install packages from trusted sources** — skills can execute code and access the filesystem.
2. **Audit all scripts** before enabling — look for network calls, file access, and unexpected operations.
3. **Use `allowed-tools`** to restrict skill capabilities where possible.
4. **Hook commands** run as the user — apply least-privilege principles.
5. **MCP servers** SHOULD NOT expose secrets; use environment variable injection.
6. **Environment Variables**: Packages MUST explicitly declare required environment variables in the `env` manifest field. Host platforms SHOULD prompt users securely for these values rather than storing them in plaintext.
7. **Permissions Model**: The `permissions` field in `package.agent.json` is OPTIONAL. When it is present, platforms MUST enforce it as defined in §17.1 Permissions Model below. A platform that ignores a declared `permissions` field is non-compliant. When the field is absent, platforms MAY apply their own default restrictions.
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

---

### 17.2 Threat Model

This section identifies the primary threat actors, attack surfaces, and mitigations relevant to the UAAPS ecosystem. Implementations SHOULD use this model to guide security decisions.

#### Threat Actors

| Actor | Capability | Motivation |
|-------|-----------|------------|
| **Malicious package author** | Publishes a package with harmful skills, hooks, or scripts | Data exfiltration, credential theft, resource hijacking |
| **Compromised registry** | Serves tampered archives or metadata | Supply-chain poisoning at scale |
| **Dependency confusion attacker** | Publishes a public package with the same name as a private one | Hijack installs that resolve against the public registry |
| **Man-in-the-middle** | Intercepts registry traffic | Serve tampered archives, steal auth tokens |
| **Compromised maintainer** | Publishes a malicious update to an existing trusted package | Supply-chain attack via trusted channel |
| **Typosquatter** | Publishes packages with names similar to popular ones | Trick users into installing malicious packages |

#### Attack Surfaces

| Surface | Vector | Relevant Artifacts |
|---------|--------|-------------------|
| **Skill scripts** | Execute arbitrary code during skill activation (`scripts/`) | Skills |
| **Hook commands** | Execute shell commands at lifecycle events with user privileges | Hooks |
| **MCP servers** | Launch external processes with network and filesystem access | MCP configs |
| **Shell binaries** | Invoke system executables with attacker-controlled arguments | Skills, hooks |
| **Network access** | Exfiltrate data or download additional payloads | Skills, MCP servers |
| **Filesystem write** | Overwrite project files, inject malicious code, plant backdoors | Skills, hooks |
| **Environment variables** | Leak secrets declared in `env` to untrusted code | All artifacts |
| **Transitive dependencies** | Inject malicious code via a deeply-nested dependency | Dependencies |
| **Lock file manipulation** | Modify `package.agent.lock` to point to tampered archives | Lock file |

#### Mitigations Mapped to Spec Mechanisms

| Threat | Mitigation | Spec Mechanism |
|--------|-----------|----------------|
| Malicious package author | Permissions restrict what a package can do | §17.1 Permissions Model |
| Malicious package author | Code review before install | `aam validate`, `aam audit` |
| Compromised registry | Checksum verification on every install | §14.4 Lock File Integrity |
| Compromised registry | Signature verification (Sigstore/GPG) | §14.1–§14.3 Signing |
| Compromised registry | Provenance attestation (SLSA L2+) | §14.5 Provenance |
| Dependency confusion | Scoped package names (`@org/name`) | §12.2 Scoped Names |
| Dependency confusion | Policy gates restrict allowed scopes | §15.1 Policy Gates |
| MITM | HTTPS-only registry transport | §12.6 HTTP Registry Protocol |
| MITM | Lock file integrity hashes | §14.4 Lock File Integrity |
| Compromised maintainer | Yank + deprecation (but no unpublish) | §12.7 Package Lifecycle |
| Compromised maintainer | Approval workflows before publish | §15.2 Approval Workflows |
| Typosquatting | Registry-side name similarity checks | Registry implementation (out of scope) |
| Transitive dependencies | `aam tree` exposes full graph; `aam audit` checks vulnerabilities | §13.10 Dependency Commands |
| Lock file manipulation | `--frozen` verifies hashes against locked values | §14.4 Lock File Integrity |
| Env variable leakage | Explicit `env` declarations with required/optional flags | §3 Manifest `env` field |

#### Residual Risks

The following risks are **not fully mitigated** by this specification. Implementations and users SHOULD apply additional controls:

| Risk | Why It Persists | Recommended Mitigation |
|------|----------------|----------------------|
| **Zero-day in trusted package** | No mechanism prevents a trusted author from introducing a subtle vulnerability | Automated security scanning, eval regression testing |
| **Permission escalation via shell** | `shell.binaries: ["git"]` permits `git` subcommands that may invoke arbitrary executables (e.g. `--upload-pack`) | Platforms SHOULD sanitize arguments to allowed binaries |
| **MCP server compromise** | MCP servers run as external processes with broad access | Platforms SHOULD sandbox MCP servers with minimal permissions |
| **Social engineering** | Users may approve dangerous permission prompts without reading | UX design: require explicit confirmation for `shell.allow: true` and broad `fs.write` |
| **Registry account takeover** | Not addressed by this spec | Registry implementations SHOULD enforce MFA and session timeouts |

---

### 17.3 Transitive Permission Composition

When a package depends on other packages, each with their own `permissions` declarations, the effective permission set must be determined. UAAPS uses a **root-governs** model.

#### Composition Rules

| Rule | Description |
|------|-------------|
| **Root authority** | The root package's `permissions` field (the package installed directly by the user) is the maximum permission boundary. No dependency can exceed it. |
| **Dependency permissions** | Each dependency's `permissions` are checked against the root's permissions. If a dependency requests a capability not granted by the root, the platform MUST deny it at runtime. |
| **Absent permissions** | If the root package omits `permissions`, platform defaults apply to the entire dependency tree. If a dependency declares `permissions` but the root does not, the dependency's permissions are advisory only. |
| **Union within root boundary** | The effective permission set is the union of all dependencies' declared permissions, intersected with the root's permission boundary. |

#### Permission Escalation via `permissionGrants`

If a root package needs to explicitly grant broader permissions to a specific dependency (e.g., a dependency needs `shell` access but the root normally restricts it), the root MUST declare this in a `permissionGrants` field:

```jsonc
// Root package's package.agent.json
{
  "permissions": {
    "fs": { "read": ["src/**"], "write": ["output/**"] },
    "shell": { "allow": false }
  },
  "permissionGrants": {
    "code-formatter": {
      "shell": { "allow": true, "binaries": ["prettier"] }
    }
  }
}
```

| Rule | Requirement |
|------|-------------|
| `permissionGrants` keys MUST be package names from `dependencies` | MUST |
| Granted permissions MUST NOT exceed what the root could declare for itself | MUST |
| `aam validate` MUST warn when `permissionGrants` grants `shell.allow: true` without a `binaries` list | MUST |
| `aam install` MUST display permission grants to the user for review | MUST |
| If a dependency requires permissions not granted by root or `permissionGrants`, installation SHOULD warn | SHOULD |

#### Effective Permission Calculation

```
effective(dep) = declared(dep) ∩ (declared(root) ∪ permissionGrants(dep))
```

Where `∩` is the intersection (least-privilege) and `∪` is the union of explicit grants. If `declared(dep)` is absent, the dependency inherits the root's permissions boundary.

#### Example

```
Root declares:     fs.read=["src/**"], fs.write=["output/**"], shell.allow=false
Dependency A:      fs.read=["**"], shell.allow=true, binaries=["prettier"]
permissionGrants:  { "A": { shell: { allow: true, binaries: ["prettier"] } } }

Effective for A:   fs.read=["src/**"], fs.write=["output/**"],
                   shell.allow=true, binaries=["prettier"]
                   (fs.read narrowed to root's boundary; shell granted explicitly)
```
