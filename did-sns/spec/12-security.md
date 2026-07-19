# 12. Security Considerations

- **No PII on-chain.** All personal data is in encrypted Data Vaults. On-chain stores only cryptographic keys, flags, and hashes.
- **Class Key lock.** Platform-issued subdomains use `nameClass = Platform Authority` for cryptographic proof of issuance. Only the class key holder can create or transfer class-locked subdomains.
- **Bidirectional linking.** SNS→SAS (via `HAS_SAS` flag + UID) and SAS→SNS (via `did_reference` field) prevents attestation spoofing.
- **Deactivation detection.** Zero-owner check returns `deactivated: true`.
- **RPC trust model.** Resolution reads account state from a Solana RPC that the resolver trusts. A malicious or compromised RPC can return forged account data (attacker owner key or data buffer), causing the resolver to emit a DID Document with attacker-controlled keys — silent identity forgery. This is a first-class threat for high-assurance use, not an edge case. See [§12.6](#126-resolution-integrity--rpc-trust).
- **Key compromise.** Transfer the domain to a new key immediately. The DID identifier remains stable; only verification methods change.
- **Domain transfer / ownership continuity.** A name transfer (sale, marketplace, registrar action, or lapse-and-re-registration) lets the new owner inherit the DID identifier and is **indistinguishable at the DID layer from a legitimate key rotation** (§9.3; see §9's "Registration Persistence & Lapse"). Self-sovereign root domains have no built-in continuity anchor — trust rests on whoever holds the name *now*. Verifiers SHOULD anchor institutional trust in an attestation from an authority *independent of the current domain owner* plus a live GLEIF/LEI check ([§3.4](03-trust-model.md#34-ecosystem-trust-anchoring)), and SHOULD treat an unexpected owner change (detectable via `didDocumentMetadata.versionId`) as a re-verification trigger. Whether the spec should *mandate* an issuer-independent continuity proof for high-trust roots is an open normative question, tracked as a design decision.
- **Cryptographic separation.** Because SAS manages the proof state, an attacker who compromises *only* the SNS domain (owner/class key) cannot forge an attestation signed by an *independent* issuer authority. This separation holds **only when the issuer is distinct from the domain controller**. In the platform-managed model (§11, SAS-Locked), the platform is typically both the class-key holder and the attestation issuer, so a single platform compromise breaks the separation. See [§12.5](#125-malicious-or-compromised-controller--class-key-holder).
- **Controller / class-key power.** For class-locked subdomains the controller (class-key holder), not the named subject, can rewrite the subordinate's data buffer — rotating keys, replacing the encryption key, moving the SAS pointer, and repointing service endpoints. This is a first-class threat, not merely operational convenience. See [§12.5](#125-malicious-or-compromised-controller--class-key-holder).
- **Subdomain isolation.** Each subdomain is a separate on-chain account. Compromising one subdomain does not affect siblings or the parent domain.
- **Post-quantum readiness.** Current verification methods (Ed25519, secp256k1) are vulnerable to cryptographically-relevant quantum computers. See §12.1 below.
- **DID registration is not implicit.** Unlike methods where registration is automatic (e.g., `did:key`), `did:sns` requires an explicit on-chain write transaction. The resolver's minimal fallback (owner key only) is a degraded mode — verifiers **MUST** reject fallback-only documents for operations requiring authentication, service endpoints, encryption keys, or SAS attestation verification. The degraded state is machine-detectable: resolvers MUST set `didResolutionMetadata.degraded = true`, so this rejection can be enforced generically rather than relying on method-specific verifier logic. See [§9.2 Degraded Resolution Signalling](09-crud-operations.md#degraded-resolution-signalling).

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

> **Harvest-now-decrypt-later:** Encrypted VP payloads created today with ECIES are at risk of future quantum decryption. For high-sensitivity credentials (financial, medical), implementers SHOULD begin using ML-KEM-768 hybrid encryption in Phase 2, even before the full PQ transition.

## 12.2 Data Storage & Key Management

> [!IMPORTANT]
> **Vault architecture, key splitting, and social recovery are platform-level concerns, not part of the `did:sns` method specification.** The method defines what goes on-chain (160-byte metadata buffer with hashes and pointers — never PII). How platforms store, encrypt, and recover personal data off-chain is an implementation decision for each operator.
>
> The `did:sns` method's security contribution is:
> - **No PII on-chain** — only cryptographic commitments (hashes, public keys, attestation pointers)
> - **Deactivation** — transfer to zero address makes the DID irrecoverable
> - **Key rotation** — domain transfer updates the owner key without changing the DID
>
> Platforms implementing `did:sns` SHOULD implement encrypted storage with key splitting and social recovery for credential payloads, but the specific architecture (vault design, key hierarchy, recovery protocol) is outside this specification's scope.

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

## 12.5 Malicious or Compromised Controller / Class-Key Holder

For any class-locked subdomain (§11, SAS-Locked or Custom mode), write authority over the on-chain data buffer belongs to the **class-key holder** — the platform or tenant that provisioned the subdomain — not to the subject the DID names. This is the defining trust assumption of the hierarchical, delegated-control model, and it is the single most consequential threat for the focal use case (institutions issuing subdomains to their users). It is called out here because a verifier or user reasoning only from the resolved DID Document cannot see it.

### What a hostile or compromised class-key holder can do

Without breaking any cryptography, a class-key holder (or an attacker who compromises it) can rewrite a subordinate's buffer and thereby:

- **Rotate `#solana-key`** to a key it controls — then authenticate and produce `assertionMethod` signatures **as the subject**.
- **Replace the ECIES encryption key** — causing future Verifiable Presentations encrypted to the DID to be readable by the attacker (a forward-looking confidentiality break; payloads already encrypted to the prior key are unaffected).
- **Move the SAS attestation pointer** or, when the platform is *also* the issuer, **mint fresh attestations** for the subject — collapsing the issuer-vs-controller separation that the rest of §12 relies on.
- **Repoint the vault / VP / DIDComm service endpoints** to attacker-controlled infrastructure, intercepting credential-exchange traffic.

### What it cannot do

- **Forge an attestation signed by an *independent* issuer.** If the subject's trust-bearing attestation is signed by an authority distinct from the class-key holder, the class-key holder cannot forge or silently reissue it. This is the mitigation the model depends on — and the reason the issuer SHOULD be independent of the controller wherever the subject's assurance matters.
- **Retroactively invalidate signatures already made under a prior key.** Old signatures remain mathematically valid (see §9.3); the risk here is impersonation *going forward*, not repudiation of the past.
- **Reach across siblings or upward.** Compromising one class-locked subdomain does not grant control of siblings or the parent (subdomain isolation).

### Residual risk (stated plainly)

A class-locked Tier 1/2 subdomain is **platform-managed, not self-sovereign**. Its integrity is bounded by the honesty and operational security of the class-key holder. A single compromise of that holder is sufficient to impersonate every subject it controls at the DID layer, and — where the holder is also the issuer — to manufacture supporting attestations. The specification does not, by itself, prevent this; it constrains and makes it detectable, as follows.

### Mitigations

**Verifier-side (normative-strength guidance):**

- Verifiers SHOULD anchor trust in an attestation signed by an issuer **independent of the resolved subdomain's controller**, not in the subdomain's own `#solana-key` alone. Where issuer and controller are the same entity, the verifier SHOULD treat the identity assurance as no stronger than that single entity's trustworthiness and score it accordingly (see [§3.4](03-trust-model.md#34-ecosystem-trust-anchoring)).
- Verifiers SHOULD detect controller-initiated key changes via `didDocumentMetadata.versionId` / `versionTime` (§9.3) and treat an unexpected rotation as a signal requiring re-verification, not a silent event.

**Platform / class-key-holder-side (SHOULD):**

- Hold class keys in a multi-signature or HSM-backed configuration so that no single operator (or single compromised credential) can rewrite subject buffers unilaterally. A platform's own Key Management & Governance grade ([§3.4](03-trust-model.md#34-ecosystem-trust-anchoring)) SHOULD reflect this.
- Separate the attestation-issuer authority from the class-key/write authority so that a write-path compromise does not also yield issuance power.
- Log every buffer write as an auditable Solana transaction (this is inherent — §12.3) and monitor for writes not originating from an authorized change.

**Subject / user-side:**

- Subjects requiring genuine sole control SHOULD use **Tier 3 self-custodial** keys or an **Open-mode** domain (zero class key, §11), where no external party can rewrite the buffer. The tradeoff — loss of platform-managed recovery and convenience — is the subject's to make, and the tier model exists precisely to make it explicit.

### Open normative question

Whether the specification should *mandate* a subject-held co-authorization of buffer writes for class-locked subdomains (so that neither the platform nor the subject can rotate keys unilaterally) is deferred. It would require a custom on-chain authorization program beyond the native SNS class-key model (§11) and is tracked as a design decision rather than resolved here; the mitigations above are achievable within the current model.

## 12.6 Resolution Integrity & RPC Trust

`did:sns` resolution constructs the DID Document from a single Solana `getAccountInfo` response (§9.2, step 6). That response is **asserted** by the RPC endpoint; standard Solana JSON-RPC does not attach a consensus/Merkle proof that a light client can independently verify. A resolver that trusts one RPC therefore inherits that RPC as a single point of truth for every identity it resolves.

**Threat.** An RPC that is malicious, compromised, or subject to a network-level MITM can substitute the account's owner key (bytes 32–63) or its data buffer. The resolver builds a DID Document from the forged data, and any relying party authenticates the attacker as the subject. Because the substitution happens *below* the DID layer, nothing in the resolved document reveals it.

**Mitigations.**

- A resolver serving high-assurance verifiers **SHOULD** read the account from **multiple independent RPC providers and require agreement (quorum)** on the owner key and buffer bytes before emitting a document. A mismatch **MUST** be surfaced (e.g. `internalError`, or a resolution warning) rather than silently resolved from a single provider.
- Operators **MAY** run their own validator / RPC to remove third-party RPC trust entirely.
- Verifiers with the highest assurance needs **SHOULD NOT** rely on a single third-party public RPC for identity-bearing resolutions.

**Known limitation (stated honestly).** Solana does not today expose an easily verifiable light-client proof of account state to arbitrary clients, so "proof-carrying reads" are not yet a turnkey option — multi-provider quorum is the practical mitigation until a native account-proof mechanism is available. This limitation applies to *every* identity method anchored in Solana account state read over RPC, not to `did:sns` uniquely, but it MUST be accounted for in any high-assurance deployment.

---

[← §11 SAS Integration](11-sas-integration.md) | [Next: §13 Interoperability →](13-interoperability.md)
