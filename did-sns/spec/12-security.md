# 12. Security Considerations

- **No PII on-chain.** All personal data is in encrypted Data Vaults. On-chain stores only cryptographic keys, flags, and hashes.
- **Class Key lock.** Platform-issued subdomains use `nameClass = Platform Authority` for cryptographic proof of issuance. Only the class key holder can create or transfer class-locked subdomains.
- **Bidirectional linking.** SNS→SAS (via `HAS_SAS` flag + UID) and SAS→SNS (via `did_reference` field) prevents attestation spoofing.
- **Deactivation detection.** Zero-owner check returns `deactivated: true`.
- **RPC trust model.** The resolver trusts the Solana RPC. For high-assurance use cases, verify against multiple providers or a local validator.
- **Key compromise.** Transfer the domain to a new key immediately. The DID identifier remains stable; only verification methods change.
- **Cryptographic separation.** Because SAS manages the proof state, even if someone compromises the SNS domain, they cannot forge the issuer-signed attestation.
- **Subdomain isolation.** Each subdomain is a separate on-chain account. Compromising one subdomain does not affect siblings or the parent domain.
- **Post-quantum readiness.** Current verification methods (Ed25519, secp256k1) are vulnerable to cryptographically-relevant quantum computers. See §12.1 below.
- **DID registration is not implicit.** Unlike methods where registration is automatic (e.g., `did:key`), `did:sns` requires an explicit on-chain write transaction. The resolver's minimal fallback (owner key only) is a degraded mode — verifiers SHOULD reject fallback-only documents for operations requiring service endpoints, encryption keys, or SAS attestation verification.

## 12.1 Post-Quantum Cryptography Migration

The current `did:sns` verification methods rely on Ed25519 (signatures) and secp256k1 (ECIES encryption), both of which are vulnerable to Shor's algorithm on a cryptographically-relevant quantum computer (CRQC). This section defines the migration path to NIST post-quantum standards.

### Threat Model

| Asset | Current Algorithm | Quantum Threat | Target Algorithm |
|---|---|---|---|
| DID authentication | Ed25519 (RFC 8032) | Shor's — key recovery | ML-DSA-44 (FIPS 204) |
| VP encryption | ECIES secp256k1 | Shor's — key recovery | ML-KEM-768 (FIPS 203) |
| Vault CEK wrapping | AES-256-GCM | Grover's — halved security | AES-256-GCM (128-bit PQ security, sufficient) |
| Blind indexes | HMAC-SHA256 | Grover's — halved security | HMAC-SHA256 (128-bit PQ security, sufficient) |
| SAS attestation signatures | Ed25519 | Shor's — key recovery | ML-DSA-44 (FIPS 204) |

> **Note:** AES-256 and HMAC-SHA256 provide 128-bit security against Grover's algorithm, which is considered sufficient. No migration is required for symmetric primitives. The urgent migration targets are asymmetric operations: signatures and key exchange.

### New Verification Method Types

| Key ID Suffix | Type | Purpose | Public Key Size |
|---|---|---|---|
| `#pq-auth-key` | `MlDsa44VerificationKey2025` | Post-quantum authentication and signing | 1,312 bytes |
| `#pq-kem-key` | `MlKem768VerificationKey2025` | Post-quantum key encapsulation for VP encryption | 1,184 bytes |

### Migration Strategy: Hybrid Mode

During the transition period, `did:sns` supports **hybrid verification** — both classical and post-quantum keys coexist in the DID Document. Verifiers SHOULD accept either key type; issuers SHOULD sign with both until the classical sunset date.

```
Phase 1 (Current)        Phase 2 (Hybrid)              Phase 3 (PQ-Only)
+--------------------+   +---------------------------+   +--------------------+
| #solana-key        |   | #solana-key (Ed25519)     |   | #pq-auth-key       |
|   Ed25519          |   | #pq-auth-key (ML-DSA-44)  |   |   ML-DSA-44        |
| #ecies-key         |   | #ecies-key (secp256k1)    |   | #pq-kem-key        |
|   secp256k1        |   | #pq-kem-key (ML-KEM-768)  |   |   ML-KEM-768       |
+--------------------+   +---------------------------+   +--------------------+
                          Dual signatures required         Classical keys removed
```

### On-Chain Storage

ML-DSA-44 public keys (1,312 bytes) and ML-KEM-768 keys (1,184 bytes) exceed the SNS data buffer capacity. Post-quantum keys are stored in the **SAS attestation layer**, not the SNS data buffer:

- A `HAS_PQ` flag (bit `0x20`) in the SNS data buffer signals that PQ verification methods exist
- The SAS attestation includes `pq_auth_key` and `pq_kem_key` fields in its Borsh-serialized data payload
- The resolver, upon seeing `HAS_PQ`, fetches the SAS attestation and adds the PQ verification methods to the DID Document

### Timeline

| Phase | Trigger | Action |
|---|---|---|
| **Phase 1** (current) | Now | Classical-only. Reserve `HAS_PQ` flag bit. Spec defines PQ verification method types. |
| **Phase 2** (hybrid) | NIST finalizes FIPS 204/203 + Solana runtime supports PQ signature verification | Dual classical+PQ keys. Issuers sign with both. Verifiers accept either. |
| **Phase 3** (PQ-only) | CRQC threat is imminent or classical algorithms are deprecated by NIST | Classical keys removed. `#solana-key` and `#ecies-key` sunset. |

> **Harvest-now-decrypt-later:** Encrypted VP payloads created today with ECIES are at risk of future quantum decryption. For high-sensitivity credentials (financial, medical), implementers SHOULD begin using ML-KEM-768 hybrid encryption in Phase 2, even before the full PQ transition. The dual-key architecture (2-of-2 XOR VMK split, with 2-of-3 Shamir social recovery on the user share) provides an additional defense layer.

## 12.2 Vault Key Architecture & Social Recovery

Tier 3 (Self-Sovereign) credentials use a hierarchical key architecture to protect encrypted proof payloads. No single party — including any platform operator — can decrypt data alone.

### Key Hierarchy

```
                     Vault Master Key (VMK)
                     256-bit AES key
                            |
              +-------------+-------------+
              |         2-of-2 XOR        |
              |           Split           |
              v                           v
      +---------------+          +---------------+
      |   Share A     |          |   Share B     |
      |   (User)      |          |   (Platform)  |
      |               |          |               |
      | Stored in:    |          | Stored in:    |
      | Vault browser |          | PII Vault     |
      | extension     |          | (KEK-wrapped) |
      +-------+-------+          +---------------+
              |
              |  2-of-3 Shamir
              |  Social Recovery
              |
    +---------+---------+---------+
    |                   |         |
    v                   v         v
+--------+       +---------+  +-----------+
| Device |       | Cloud   |  | Guardian  |
| Share  |       | Backup  |  | Share     |
|        |       | Share   |  |           |
| Local  |       | E2E     |  | Trusted   |
| secure |       | encrypted| | contact   |
| storage|       | storage |  | (offline) |
+--------+       +---------+  +-----------+

Any 2 of 3 sub-shares reconstruct Share A
```

### Encryption Flow

```
Encrypt:                                 Decrypt:

1. Generate DEK (per-object)             1. User provides Share A
   DEK = random AES-256 key                 (from extension)

2. Encrypt payload                       2. Platform provides Share B
   ciphertext = AES-256-GCM(DEK, data)     (from PII Vault, KEK-unwrap)

3. Wrap DEK with VMK                     3. Reconstruct VMK
   wrapped_dek = AES-KEYWRAP(VMK, DEK)     VMK = Share_A XOR Share_B

4. Split VMK into shares                 4. Unwrap DEK
   Share_A = random(32)                     DEK = AES-KEYUNWRAP(VMK, wrapped_dek)
   Share_B = VMK XOR Share_A
                                         5. Decrypt payload
5. Store Share_A in extension               data = AES-256-GCM(DEK, ciphertext)
   Store Share_B in PII Vault
   (KEK-wrapped)                         Both shares required.
                                         Neither party can decrypt alone.
```

### Social Recovery Protocol

If the user loses access to their Vault extension (device loss, browser reset), Share A can be reconstructed from social recovery sub-shares using 2-of-3 Shamir Secret Sharing:

| Sub-Share | Storage | Access |
|---|---|---|
| **Device share** | Browser extension local storage (encrypted) | Automatic — available while device is active |
| **Cloud backup share** | End-to-end encrypted cloud storage | User authenticates to cloud provider |
| **Guardian share** | Held by a trusted contact (offline or separate device) | Guardian provides share upon identity verification |

### Recovery Scenarios

| Scenario | Shares Available | Recovery |
|---|---|---|
| Lost phone, have laptop | Device (laptop) + Cloud | Automatic — 2 of 3 |
| Lost all devices | Cloud + Guardian | Guardian-assisted — 2 of 3 |
| Lost device + no cloud | Guardian + new device enrollment | Guardian-assisted — requires re-enrollment |
| Lost all 3 sub-shares | None | Unrecoverable — crypto-shredding equivalent |

### Crypto-Shredding

Deleting Share B from the PII Vault renders all encrypted objects permanently inaccessible, regardless of Share A availability. This satisfies GDPR and other data protection regulations' right to erasure without requiring on-chain data deletion — the on-chain pointers become meaningless without the decryption keys.

> **WARNING — IRREVERSIBLE:** Crypto-shredding (deleting Share B) is a permanent, unrecoverable operation. All vault objects encrypted under that VMK become permanently inaccessible. Similarly, losing all 3 social recovery sub-shares for Share A is equivalent to crypto-shredding — there is no backdoor, no master key, and no platform override. This is by design.

> **Threat model:** An attacker who compromises only the platform obtains Share B (KEK-wrapped) but cannot decrypt without Share A. An attacker who compromises only the user's device obtains Share A but cannot decrypt without Share B. Compromising both requires breaching the platform *and* the user's device simultaneously.

## 12.3 Stateless Resolution — No DID Log

Some DID methods (notably `did:webvh`) maintain an **append-only DID Log** — a JSON Lines file where every key rotation, metadata update, and witness proof adds a new entry. To resolve the current DID Document, a verifier must fetch the entire log and replay every entry. This design creates compounding scalability concerns, particularly in a post-quantum future.

`did:sns` uses a fundamentally different model: a **mutable state buffer** (160 bytes) that is overwritten in place on the Solana blockchain. There is no log to accumulate, no replay to perform, and no witness proofs to store.

### Why This Matters

| Concern | DID Log Methods | `did:sns` |
|---|---|---|
| **Resolution cost** | O(n) — grows linearly with identity age | **O(1)** — always 2 RPC calls regardless of identity age |
| **Storage growth** | Unbounded — the log can never shrink | **Fixed 160 bytes** — buffer is overwritten, not appended |
| **Witness proofs** | Stored in log or separate file | **Not applicable** — Solana's PoS consensus provides native witnessing |
| **Compression effectiveness** | ~30–50% on JSON, minimal on PQ signatures | **Not needed** — 160 bytes is already smaller than a single PQ signature |

### Post-Quantum Signature Size Impact on DID Logs

| Scheme | Public Key | Signature | vs. Ed25519 (32B key, 64B sig) |
|---|---|---|---|
| ML-DSA-65 (Dilithium) | 1,952 B | 3,309 B | ~60x key, ~52x sig |
| SLH-DSA-128f (SPHINCS+) | 32 B | 17,088 B | Same key, **267x sig** |
| FALCON-512 | 897 B | 690 B | ~28x key, ~11x sig |

For a DID Log method with 3 witnesses per entry:

| Scenario | Classical (Ed25519) | ML-DSA-65 | SLH-DSA-128f |
|---|---|---|---|
| Single log entry | ~288 B | ~18 KB | ~69 KB |
| 5-year DID (60 rotations) | ~17 KB | **~1.1 MB** | **~4.1 MB** |

### How `did:sns` Avoids This Entirely

1. **No accumulated signatures.** The buffer contains the *current* state only. Key rotations overwrite the previous key.
2. **Signatures live in Solana transactions.** Authorization signatures are part of the Solana transaction, validated at block time and stored in the blockchain's native transaction log.
3. **PQ keys are hashed, not stored inline.** The `HAS_PQ` flag signals PQ keys exist. Full PQ public keys are stored in the SAS attestation; the buffer stores only a hash reference.
4. **History on demand, not by default.** Auditors query Solana's `getSignaturesForAddress` + `getTransaction` APIs. History retrieval cost is borne only by auditors who need it.

> **Key insight:** DID Log methods conflate *current state* with *history*. The log IS the DID — you must replay it to get the current document. `did:sns` separates these completely: current state = SNS buffer (160 bytes, O(1) read); history = Solana transaction log (queried only when auditors need provenance).

### Audit Trail Without a DID Log

`did:sns` provides equivalent auditability through Solana's native transaction history:

1. **Every buffer mutation is a signed transaction.** Solana records the signer, timestamp, slot, and instruction data for every `update` call.
2. **SAS attestation history is independently auditable.** Each attestation create/close is a separate on-chain transaction.
3. **No replay required for verification.** An auditor can verify the current state directly and independently inspect history.
4. **Validator-attested timestamps.** Solana slot times are attested by the supermajority of stake-weighted validators.
5. **Signer-level filtering.** Auditors can reconstruct history filtered by actor.

> **Key advantage over DID Logs:** This audit history is *optional* — only accessed when an auditor explicitly needs it. Normal resolution (the 99.9% case) never touches the transaction log. With DID Log methods, the entire history is *mandatory* on every resolution.

## 12.4 Schema Version Migration Path

The SNS data buffer schema is versioned (byte 4). Migration from v2 to a future v3 schema follows a defined path:

1. **Buffer reallocation.** The Name Service Program supports `reallocate` to resize a Name Registry Account's data buffer. If PQ keys require more than the current 25-byte reserved block, the buffer can be expanded without changing the domain address or DID identifier.
2. **Version bump.** The version byte changes from `0x02` to `0x03`. Resolvers MUST check the version byte and apply the corresponding schema layout. Unknown versions MUST be rejected with `VERSION_NOT_SUPPORTED`.
3. **Backward compatibility.** Fields at offsets 0–5 (Magic, Version, Flags) remain stable across all versions.

### PQ Key Storage Approaches

| Approach | Buffer Change | Pros | Cons |
|---|---|---|---|
| **A: Hash-in-buffer, full-key-in-SAS** | No size change; PQ key hash in existing ECIES slot | Buffer stays 160B; PQ key retrieved from SAS only when needed | Extra RPC call for PQ key retrieval |
| **B: Expanded buffer with PQ key slot** | +64–128B for PQ compressed key | Single-read resolution; no SAS dependency for key | Larger buffer; rent cost increase (~0.001 SOL) |

> **Preferred approach:** Approach A (hash-in-buffer) is recommended for the initial PQ transition because it maintains O(1) resolution cost and fixed buffer size. Approach B may be adopted if PQ key verification becomes a hot path requiring single-read resolution.

**Migration rollout:** Schema migration is performed per-identity by the operator's Identity Manager wallet. There is no global migration event — each subdomain is upgraded individually when its SAS attestation is next updated. The `versionId` in `didDocumentMetadata` increments on migration.

---

[← §11 SAS Integration](11-sas-integration.md) | [Next: §13 Interoperability →](13-interoperability.md)
