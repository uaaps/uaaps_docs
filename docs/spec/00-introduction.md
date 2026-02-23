# Universal Agentic Artifact Package Specification (UAAPS)

> **Version**: 0.6.0-draft
> **Date**: 2026-02-23  
> **Purpose**: Define a universal, portable package standard for AI agent artifacts — analogous to Docker for containers or npm for JavaScript packages.  
> **Goal**: Write once, deploy to any agent platform. Simplify migration. Eliminate vendor lock-in.

---

## Design Principles

| Principle | Description |
|-----------|-------------|
| **Portability** | A single package works across Claude Code, Cursor, Copilot, Codex, and any compliant platform. |
| **Progressive Disclosure** | Metadata loads first; full instructions load on activation; resources load on-demand. |
| **Convention over Configuration** | Standard directory names eliminate per-platform mapping; overrides are optional. |
| **Filesystem-First** | All artifacts are files on disk — no databases, no APIs required, no binary formats. |
| **Composability** | Packages can be installed together; namespacing prevents conflicts. |
| **Deterministic Resolution** | Lock files ensure reproducible installs across machines and CI. |
| **Backward Compatibility** | Existing Claude Code plugins and Cursor plugins work with minimal changes. |

---

## Normative Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

| Keyword | Meaning |
|---------|---------|
| **MUST** / **REQUIRED** / **SHALL** | An absolute requirement of the specification. Implementations that do not satisfy this are non-conformant. |
| **MUST NOT** / **SHALL NOT** | An absolute prohibition. Implementations that violate this are non-conformant. |
| **SHOULD** / **RECOMMENDED** | There may exist valid reasons to ignore this in particular circumstances, but the full implications must be understood and carefully weighed before doing so. |
| **SHOULD NOT** / **NOT RECOMMENDED** | There may exist valid reasons when this behaviour is acceptable, but the implications must be understood and carefully weighed. |
| **MAY** / **OPTIONAL** | Truly optional. Implementations that omit this remain conformant; those that include it MUST interoperate with implementations that do not. |

Where this spec states a plain imperative (e.g., "Packages include a manifest") without one of these keywords, the statement is informative, not normative.

---

## Specification Versioning

This specification follows **Semantic Versioning 2.0** for its own version number. The version appears in the header of the introduction section.

| Change Type | Spec Version Impact | Example |
|------------|-------------------|---------|
| New optional manifest field | Minor bump | Adding `workspace` support |
| New optional artifact type | Minor bump | Adding a new directory convention |
| Clarification, typo, or errata | Patch bump | Fixing an anchor link, rewording a sentence |
| New required manifest field | **Major bump** | Requiring `permissions` in all manifests |
| Changing resolver behavior | **Major bump** | Altering how pre-release versions are matched |
| Removing a previously-normative feature | **Major bump** | Dropping support for an artifact type |

Implementations SHOULD declare which spec version they target via the `engines` field or `aam platform info` output. Implementations targeting spec version `X.Y` MUST support all features introduced up to and including `X.Y`.

Pre-1.0 versions (`0.x.y`) MAY include breaking changes in minor releases to allow rapid iteration. Post-1.0, the rules above are strictly enforced.

---

## Forward-Compatibility Parsing Rules

To enable graceful evolution of the specification across tool versions:

1. **Manifest fields**: Consumers MUST ignore unrecognized top-level fields in `package.agent.json`. A tool implementing spec v0.5.0 MUST NOT reject a manifest solely because it contains fields introduced in v0.6.0.
2. **Frontmatter fields**: Consumers MUST ignore unrecognized frontmatter keys in `SKILL.md`, `RULE.md`, `agent.yaml`, `system-prompt.md`, and command `.md` files.
3. **Hook events**: Consumers MUST ignore hook event names they do not support. Unknown events in `hooks.json` MUST NOT cause a parse error.
4. **Vendor extensions**: Consumers MUST silently ignore unrecognized `x-*` keys (restated from §3 for emphasis).
5. **Enum values**: When a field specifies an enumeration (e.g., `installMode` values), consumers MUST treat unrecognized enum values as a warning, not an error, unless the field is marked as **closed** (no future values will be added).

These rules ensure that packages authored against a newer spec version remain installable by older tools, with graceful degradation rather than hard failure.

---

## Conformance Levels

Platforms and tools claiming UAAPS support MUST declare a **conformance level**. Each level subsumes all requirements of lower levels.

### Level 1 — Reader

The minimum viable implementation. A Level 1 platform can discover and activate skills from a UAAPS package.

| Requirement | Description |
|-------------|-------------|
| Read `package.agent.json` | Parse the manifest and extract `name`, `version`, `description`. |
| Discover skills | Scan the `skills/` directory (or `artifacts.skills` entries) for `SKILL.md` files. |
| Lazy-load skill metadata | Read `name` + `description` from SKILL.md frontmatter without loading the full body. |
| Activate skill body | On demand, load the full SKILL.md body and associated `scripts/`, `references/`, `assets/`. |
| Ignore unknown fields | Comply with forward-compatibility parsing rules above. |

### Level 2 — Installer

Full package consumption. A Level 2 platform can install, resolve dependencies, and use all artifact types.

| Requirement | Description |
|-------------|-------------|
| All Level 1 requirements | — |
| Commands | Discover and invoke commands from `commands/` directory. |
| Agents | Discover and activate agents from `agents/` directory (`agent.yaml` + `system-prompt.md`). |
| Rules | Discover and apply rules from `rules/` directory (glob-scoped and `alwaysApply`). |
| Hooks | Load `hooks/hooks.json` and execute hook commands at appropriate lifecycle events. |
| MCP servers | Load `mcp/servers.json` and launch configured MCP servers. |
| Dependency resolution | Resolve `dependencies`, `peerDependencies`, and `optionalDependencies` using the lock file. Support `aam install --frozen`. |
| Permissions enforcement | When `permissions` is present in the manifest, enforce `fs`, `network`, and `shell` constraints at runtime. |
| System dependency checks | Run pre-flight checks for `systemDependencies` before installation. |

### Level 3 — Publisher

Full lifecycle support. A Level 3 platform or tool can author, test, sign, publish, and govern packages.

| Requirement | Description |
|-------------|-------------|
| All Level 2 requirements | — |
| Package authoring | Support `aam init`, `aam validate`, `aam pkg pack`. |
| Signing & verification | Support at least checksum + one signing method (Sigstore or GPG). Verify signatures on install. |
| Registry interaction | Support `aam publish`, `aam install` from HTTP and filesystem registries. |
| Testing & evals | Support `aam test` (deterministic assert tests) and `aam eval` (LLM-judged evaluations). |
| Governance | Support client-side policy gates (`allowed_scopes`, `require_signature`, `blocked_packages`). |
| Provenance | Support generating and verifying SLSA-aligned provenance attestations (§14.5). |

### Declaring Conformance

Platforms declare their conformance level via `aam platform info` output and in their documentation:

```
UAAPS Conformance: Level 2 (Installer)
Spec Version: 0.6.0
```

A platform MUST NOT claim a conformance level unless it implements ALL requirements of that level. Partial implementations MUST declare the highest level they fully satisfy.

---

## Syntax Definitions (ABNF)

The following ABNF grammar (per [RFC 5234](https://www.rfc-editor.org/rfc/rfc5234)) defines the syntax of key identifiers used throughout this specification. Where the prose spec and this grammar conflict, the grammar is authoritative.

```abnf
; === Package naming ===
package-name     = unscoped-name / scoped-name
unscoped-name    = LOWER 0*63( LOWER / DIGIT / "-" )
scoped-name      = "@" scope "/" unscoped-name
scope            = ( LOWER / DIGIT ) 0*63( LOWER / DIGIT / "_" / "-" )

; === Artifact identifiers ===
skill-name       = identifier
command-name     = identifier
agent-name       = identifier
rule-name        = identifier
identifier       = LOWER 0*63( LOWER / DIGIT / "-" )

; === Qualified (namespaced) references ===
skill-ref        = [ package-name ":" ] skill-name
command-ref      = [ package-name ":" ] command-name
agent-ref        = [ package-name ":" ] agent-name

; === Version and constraints ===
semver           = numeric-id "." numeric-id "." numeric-id
                   [ "-" pre-release ] [ "+" build-metadata ]
numeric-id       = "0" / ( NON-ZERO-DIGIT *DIGIT )
pre-release      = pre-release-id *( "." pre-release-id )
pre-release-id   = 1*DIGIT / 1*( ALPHA / DIGIT / "-" )
build-metadata   = build-id *( "." build-id )
build-id         = 1*( ALPHA / DIGIT / "-" )

version-range    = exact-version / caret-range / tilde-range
                   / gte-range / gt-range / lte-range / lt-range
                   / compound-range / wildcard
exact-version    = semver
caret-range      = "^" semver
tilde-range      = "~" semver
gte-range        = ">=" semver
gt-range         = ">" semver
lte-range        = "<=" semver
lt-range         = "<" semver
compound-range   = ( gte-range / gt-range ) SP ( lte-range / lt-range )
wildcard         = "*"

; === Dist-tags ===
dist-tag         = 1*32( LOWER / DIGIT / "-" )
                   ; MUST NOT be a valid semver string

; === Vendor extensions ===
vendor-ext-key   = "x-" vendor-id
vendor-id        = 1*32( LOWER / DIGIT / "-" )

; === Filesystem mapping ===
fs-name          = unscoped-name / scope-fs-name
scope-fs-name    = scope "--" unscoped-name
                   ; @myorg/code-review → myorg--code-review

; === Core character classes ===
LOWER            = %x61-7A                   ; a-z
DIGIT            = %x30-39                   ; 0-9
NON-ZERO-DIGIT   = %x31-39                   ; 1-9
ALPHA            = %x41-5A / %x61-7A        ; A-Z / a-z
SP               = %x20                      ; space
```

All identifiers in this specification (package names, skill names, command names, etc.) MUST conform to the `identifier` production unless otherwise noted. The `package-name` production allows the additional `@scope/` prefix for scoped packages.
