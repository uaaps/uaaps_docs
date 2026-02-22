## 14. Package Signing & Verification

AAM supports multiple levels of integrity and authenticity verification for packages.

### 14.1 Signing Methods

| Method | Description | When to Use |
|--------|-------------|-------------|
| **Checksum** | SHA-256 hash of archive, always applied | Automatic — all packages |
| **Sigstore** | Keyless, identity-based via OIDC | Recommended for public packages |
| **GPG** | Traditional key-based signing | Teams with existing GPG infrastructure |
| **Registry Attestation** | Server-side signatures | Automated trust for verified registries |

### 14.2 Signing Flow

```
Author                          Registry                         User
  │  aam pkg publish --sign       │                               │
  ├──────────────────────────────►│                               │
  │  1. Calculate SHA-256         │                               │
  │  2. Sign with Sigstore / GPG  │  Verify signature             │
  │  3. Upload archive+signature  │  Store attestation            │
  │                               │         aam install pkg       │
  │                               │◄──────────────────────────────┤
  │                               │  1. Download archive          │
  │                               ├──────────────────────────────►│
  │                               │  2. Verify checksum           │
  │                               │  3. Verify signature          │
  │                               │  4. Check trust policy        │
```

### 14.3 Verification Policy

Configure in global `~/.aam/config.yaml` or project `.aam/config.yaml`:

```yaml
security:
  require_checksum: true           # Always enforced (non-configurable)
  require_signature: false         # Require signed packages
  trusted_identities:              # Sigstore OIDC identities to trust
    - "*@myorg.com"
  trusted_keys:                    # GPG key fingerprints to trust
    - "ABCD1234..."
  on_signature_failure: warn       # warn | error | ignore
```

### 14.4 Lock File Integrity

The `package.agent.lock` stores SHA-256 hashes for every resolved package. On `--frozen` install, hashes are re-verified — any mismatch aborts with an error, preventing supply-chain tampering.
