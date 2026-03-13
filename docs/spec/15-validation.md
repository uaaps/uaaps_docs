## 16. Validation Rules

| Rule | Constraint |
|------|-----------|
| Package `name` | `[a-z0-9-]` max 64 chars, or scoped `@scope/name` max 130 chars |
| Package `version` | Valid SemVer 2.0 |
| Skill `description` | Max 1024 chars, MUST describe WHAT + WHEN |
| SKILL.md body | RECOMMENDED < 5,000 tokens / 500 lines |
| Frontmatter | Valid YAML, no tabs |
| Scripts | MUST be self-contained or document dependencies |
| Hook commands | MUST handle JSON on stdin |
| File references | Forward slashes only, relative paths |
| Dependency versions | Valid SemVer range syntax (`^`, `~`, `>=`, exact) |
| Dependency graph | MUST be acyclic — circular dependencies are an error |
| Peer dependencies | MUST be satisfied by root or ancestor package |
| System dependencies | Pre-flight check MUST pass (or explicit `--skip-checks`) |
| Lock file integrity | Hash MUST match on `--frozen` install |
| Lock file source metadata | `package.agent.lock` version `2` entries MUST store `source` as an object with required fields for the selected source `type` |
| Manifest `files` | Entries MUST be relative, use forward slashes, and MUST NOT use negated patterns |
| Archive completeness | `aam pkg pack` MUST fail if the computed packlist omits a referenced artifact, hook, or MCP file |
| Dependency groups | `dependencyGroups` keys MUST match `[a-z0-9][a-z0-9-]*` and each group MUST declare at least one of `dependencies` or `systemDependencies` |
| Extras | `extras` keys MUST match `[a-z0-9][a-z0-9-]*` and each extra MUST declare at least one of `dependencies`, `systemDependencies`, `artifacts`, or `permissions` |
| Extra artifact refs | `extras.*.artifacts` entries MUST resolve to names declared in the top-level `artifacts` registry |
| Namespace uniqueness | No two skills with same `name` within a single package |
| Resolution overrides | `resolutions` entries MUST reference packages in the tree |
| Resolver version | `resolverVersion` MUST be a positive integer. Unknown versions are a fatal error. |
| YAML round-trip | `package.agent.yaml` MUST use YAML 1.2 with JSON-compatible types only (no anchors, aliases, tags, or merge keys) |
| Workspace config | `workspace.agent.json` `packages` globs MUST resolve to directories containing `package.agent.json` |
| Forward-compatibility | Tools MUST NOT reject manifests containing unrecognized fields (see §1 Forward-Compatibility Parsing Rules) |
| Permission grants | `permissionGrants` keys MUST reference packages listed in `dependencies` |
| Provenance level | `require_provenance` in config MUST be one of: `L0`, `L1`, `L2`, `L3` |
| Deprecated field | When `deprecated` is present, `deprecated.message` is REQUIRED |
| Conformance claim | Platforms claiming a conformance level MUST implement ALL requirements of that level |

### Permissions Security Audit

The following conditions do not fail validation but MUST produce a `WARN`-level diagnostic during both `aam validate` and `aam install`. They indicate a package that may be unintentionally over-privileged.

| Condition | Warning message |
|-----------|----------------|
| `permissions` field absent | `⚠ No permissions declared — platform-default restrictions apply. Consider adding a permissions field for explicit least-privilege.` |
| `shell.allow: true` and `binaries` is empty or absent | `⚠ Shell access is unrestricted. Add a binaries allow-list to limit which executables may be invoked.` |
| `fs.write` contains `**` or `.` | `⚠ Filesystem write scope is very broad. Narrow the write globs to only the output paths the package requires.` |
| `network.hosts` contains `*` (bare wildcard) | `⚠ Network host wildcard allows any host. Specify explicit hosts or scoped wildcards (e.g. *.example.com).` |
| `network.schemes` contains `http` | `⚠ Plain HTTP is declared as an allowed network scheme. Prefer https unless the target endpoint requires it.` |

These warnings MUST be displayed to the user and MUST be included in the output of `aam validate --json` under a `warnings` array. They MUST NOT block installation or publication. When extras are selected during install, the same audit rules MUST be applied to the effective permission set after selected extra permissions are merged.

### Vendor Extension Validation

The following rules apply to all `x-*` keys in `package.agent.json`:

| Rule | Constraint |
|------|-----------|
| Key naming | MUST match `x-[a-z0-9-]+`, vendor-id max 32 chars |
| Value type | MUST be a JSON object — scalars and arrays are errors |
| Unknown keys | Tools MUST silently ignore unrecognised `x-*` keys |
| Count | `aam validate` MUST warn if more than 5 `x-*` keys are declared |
| Unregistered vendor-id | Registry SHOULD produce a `WARN`; tools MUST NOT error |

The following conditions do not block validation but MUST produce a `WARN`-level diagnostic:

| Condition | Warning message |
|-----------|----------------|
| More than 5 `x-*` keys | `⚠ More than 5 vendor extension keys declared. Consider consolidating metadata — extension sprawl can reduce manifest readability.` |
| `x-*` key vendor-id not in registered platform list | `⚠ Vendor-id "<vendor-id>" is not a registered platform. This key will not be indexed by the registry.` |
| `x-*` value is not an object | `✕ Vendor extension "x-<vendor-id>" value MUST be a JSON object. Scalar and array values are invalid.` (this condition is an **error**, not a warning) |
