## 16. Validation Rules

| Rule | Constraint |
|------|-----------|
| Package `name` | `[a-z0-9-]`, max 64 chars |
| Package `version` | Valid SemVer 2.0 |
| Skill `description` | Max 1024 chars, must describe WHAT + WHEN |
| SKILL.md body | Recommended < 5,000 tokens / 500 lines |
| Frontmatter | Valid YAML, no tabs |
| Scripts | Must be self-contained or document dependencies |
| Hook commands | Must handle JSON on stdin |
| File references | Forward slashes only, relative paths |
| Dependency versions | Valid SemVer range syntax (`^`, `~`, `>=`, exact) |
| Dependency graph | Must be acyclic — circular dependencies are an error |
| Peer dependencies | Must be satisfied by root or ancestor package |
| System dependencies | Pre-flight check must pass (or explicit `--skip-checks`) |
| Lock file integrity | Hash must match on `--frozen` install |
| Namespace uniqueness | No two skills with same `name` within a single package |
| Resolution overrides | `resolutions` entries must reference packages in the tree |
