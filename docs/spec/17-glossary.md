## 18. Glossary

| Term | Definition |
|------|-----------|
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
