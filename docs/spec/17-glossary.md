## 18. Glossary

| Term | Definition |
|------|-----------|
| **AAM** | Agent Artifact Manager — the CLI tool (`aam`) for installing, publishing, testing, and managing UAAPS packages. |
| **Command** | A markdown file triggered by user typing `/name` or referenced as a parameterized template with `{{variable}}` interpolation. User or agent-invoked. |
| **Skill** | A folder with `SKILL.md` that teaches an agent a specific capability. Model-invoked. |
| **Agent** | A specialized persona definition with `agent.yaml` + `system-prompt.md`. Model or user-invoked. |
| **Rule** | Persistent context (coding standards, conventions). Applied by scope rules. |
| **Hook** | A shell command triggered by a lifecycle event. Always executed, zero token cost. |
| **MCP Server** | An external tool server using the Model Context Protocol. |
| **AGENTS.md** | Universal, freeform instruction file for any agent. |
| **Progressive Disclosure** | Loading strategy: metadata → instructions → resources, on-demand. |
| **Dependency** | Another agent package required for this package to function. |
| **Peer Dependency** | A package that must be provided by the host environment, not auto-installed. |
| **Optional Dependency** | A package that enhances functionality but isn't required. |
| **System Dependency** | An OS binary, runtime, or package-manager package required on the host. |
| **Lock File** | `package.agent.lock` — pins exact resolved versions for reproducible installs. |
| **Hoisting** | Flattening shared dependencies to the top of the install tree to reduce duplication. |
| **Resolution Override** | A `resolutions` entry that forces a specific version for a transitive dependency. |
| **Pre-flight Check** | Verification that system dependencies exist before package installation. |
| **SemVer** | Semantic Versioning 2.0 — `MAJOR.MINOR.PATCH` with defined compatibility rules. |
| **DAG** | Directed Acyclic Graph — the dependency tree structure. Cycles are forbidden. |
| **Permissions** | Declared scopes (fs, network, shell) that restrict what a package can do. |
| **Env** | Environment variables and secrets required by the package to function. |
| **Scope** | The `@org` namespace prefix in `@org/package-name` for organizational grouping. |
| **Archive** | A `.aam` gzipped tar file containing a distributable package. |
| **Bundle** | A pre-compiled, platform-specific archive requiring no install-time resolution. |
| **Dist-tag** | A named alias for a specific package version (e.g., `stable`, `bank-approved`). |
| **Policy Gate** | A client-side governance rule enforced before install or publish operations. |
| **Yank** | Soft-delete of a published version. Yanked versions are hidden from search and new resolution but remain downloadable for existing lock files. See §12.7. |
| **Deprecation** | Advisory state marking a version as superseded. Deprecated versions remain fully functional but emit warnings. See §12.7. |
| **Provenance** | Cryptographic attestation linking a package archive to its source repository, commit, and build system. See §14.5. |
| **Conformance Level** | A tiered compliance claim (Level 1–3) indicating which UAAPS features a platform implements. See §1. |
| **Workspace** | A monorepo configuration where multiple UAAPS packages share a single dependency tree and lock file. See §13.14. |
| **Capability Negotiation** | Runtime mechanism for packages to discover what features the host platform supports. See §10. |
| **Resolver Version** | A manifest field (`resolverVersion`) that selects which dependency resolution algorithm to apply. See §13.4. |
| **Permission Grant** | An explicit escalation allowing a dependency to use capabilities beyond the root package's permission boundary. See §17.3. |
| **Threat Model** | A structured analysis of threat actors, attack surfaces, and mitigations for the UAAPS ecosystem. See §17.2. |
| **ABNF** | Augmented Backus-Naur Form (RFC 5234) — formal grammar notation used to define UAAPS syntax. See §1. |
| **SLSA** | Supply-chain Levels for Software Artifacts — a framework for graduated provenance requirements. See §14.5. |
