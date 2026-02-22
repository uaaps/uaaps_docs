## 17. Security Considerations

1. **Only install packages from trusted sources** — skills can execute code and access the filesystem.
2. **Audit all scripts** before enabling — look for network calls, file access, and unexpected operations.
3. **Use `allowed-tools`** to restrict skill capabilities where possible.
4. **Hook commands** run as the user — apply least-privilege principles.
5. **MCP servers** should not expose secrets; use environment variable injection.
6. **Environment Variables**: Packages must explicitly declare required environment variables in the `env` manifest field. Host platforms should prompt users securely for these values rather than storing them in plaintext.
7. **Permissions Model**: Platforms should enforce the `permissions` manifest field, restricting filesystem, network, and shell access to declared scopes.
8. **Sandbox** untrusted packages before production deployment.
9. **Verify lock file integrity** — `package.agent.lock` hashes prevent supply-chain tampering.
10. **Run `aam audit`** regularly to check dependencies for known vulnerabilities.
11. **System dependency auto-install** never uses `sudo` without explicit `--allow-sudo` flag.
12. **Transitive dependencies** should be audited — `aam tree` reveals the full graph.
