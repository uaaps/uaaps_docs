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

### 14.5 Provenance Attestation (SLSA-Aligned)

Beyond signing (which proves *who* published), provenance proves *how* a package was built — from which source, at which commit, using which build system. UAAPS defines four provenance levels aligned with the [SLSA framework](https://slsa.dev/).

#### Provenance Levels

| Level | Name | Requirement | Verification |
|-------|------|-------------|-------------|
| **L0** | Checksum | SHA-256 hash of archive (always applied) | `aam install` verifies hash automatically |
| **L1** | Signed | Author identity verified via Sigstore OIDC or GPG | `aam install` verifies signature per trust policy |
| **L2** | Provenance | Build provenance attestation linking archive to source repo, commit, and build system | `aam audit --provenance` verifies attestation chain |
| **L3** | Reproducible | Deterministic build from source — anyone can rebuild and obtain the same archive hash | `aam audit --reproducible` rebuilds from source and compares hash |

Each level subsumes all requirements of lower levels (L3 includes L2 includes L1 includes L0).

#### Provenance Attestation Format

Provenance attestations follow the [in-toto attestation framework](https://github.com/in-toto/attestation) with a UAAPS-specific predicate type.

```json
{
  "_type": "https://in-toto.io/Statement/v1",
  "subject": [
    {
      "name": "@myorg/code-review-1.3.0.aam",
      "digest": { "sha256": "abc123..." }
    }
  ],
  "predicateType": "https://uaaps.github.io/provenance/v1",
  "predicate": {
    "builder": {
      "id": "https://github.com/actions/runner"
    },
    "buildType": "https://uaaps.github.io/build/aam-pack/v1",
    "invocation": {
      "configSource": {
        "uri": "git+https://github.com/myorg/code-review@refs/tags/v1.3.0",
        "digest": { "sha1": "def456..." },
        "entryPoint": ".github/workflows/publish.yml"
      }
    },
    "metadata": {
      "buildStartedOn": "2026-02-22T14:30:00Z",
      "buildFinishedOn": "2026-02-22T14:30:45Z",
      "completeness": {
        "parameters": true,
        "environment": true,
        "materials": true
      },
      "reproducible": false
    },
    "materials": [
      {
        "uri": "git+https://github.com/myorg/code-review@refs/tags/v1.3.0",
        "digest": { "sha1": "def456..." }
      }
    ]
  }
}
```

#### Provenance Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `predicateType` | `string` | **Yes** | MUST be `https://uaaps.github.io/provenance/v1`. |
| `predicate.builder.id` | `string` | **Yes** | URI identifying the build system (e.g. GitHub Actions, GitLab CI). |
| `predicate.buildType` | `string` | **Yes** | URI identifying the build process type. |
| `predicate.invocation.configSource.uri` | `string` | **Yes** | Git URI + ref of the source repository. |
| `predicate.invocation.configSource.digest` | `object` | **Yes** | Cryptographic digest of the source commit. |
| `predicate.metadata.reproducible` | `boolean` | No | Whether the build is deterministic. Required for L3. |
| `predicate.materials` | `array` | **Yes** | All input materials (source repos, dependencies) with digests. |

#### CLI Commands

```bash
# Generate provenance attestation during publish
aam pkg publish --provenance

# Verify provenance of an installed package
aam audit --provenance @myorg/code-review

# Attempt reproducible build verification (L3)
aam audit --reproducible @myorg/code-review

# View provenance attestation
aam inspect provenance @myorg/code-review@1.3.0
```

#### Verification Policy

The provenance level requirement is configured alongside signing policy in `config.yaml`:

```yaml
security:
  require_provenance: L1            # L0 | L1 | L2 | L3
  on_provenance_failure: warn       # warn | error | ignore
```

Consumers MUST ignore unrecognized fields in the provenance attestation to support forward-compatible predicate evolution.
