# 9. CRUD Operations & Lifecycle

## 9.1 Create

A `did:sns` DID is created by registering an SNS domain on Solana and writing DID metadata to the domain's raw data buffer. The domain registration alone does **not** constitute a functional DID — the metadata write (step 2) is required.

| Step | Action | On-Chain Effect |
|---|---|---|
| 1 | Register SNS domain (Bonfida SDK / SNS CLI) | NameRegistry created; owner = registrant's Solana key |
| 2 | Write DID metadata to data buffer (**required**) | Magic `0x44494401` + flags + keys at offset 96 via `update` instruction |
| 3 | Create subdomain (optional) | Child NameRegistry with parent PDA as seed |
| 4 | Link SAS attestation (optional) | SAS UID written to buffer; `HAS_SAS` flag set |

> **DID registration requires an on-chain write.** The `did:sns` method stores DID metadata in the **raw data buffer** of the Name Registry Account (bytes 96+) using the Name Service Program's native `update` instruction — not the SNS Records Program. Without this explicit write, the resolver falls back to a degraded minimal document (owner key only) with no service endpoints, encryption keys, or SAS attestation link.

> **WARNING:** The degraded fallback (owner key only) MUST NOT be treated as a registered DID identity. Verifiers MUST reject fallback-only documents for any operation requiring authentication, encryption, credential verification, or SAS attestation validation. A domain without DID metadata in its raw data buffer is simply an SNS domain — not a `did:sns` identity.

### Subdomain Delegation Models

| Model | Who Provisions | Who Signs | Trust Level |
|---|---|---|---|
| **Direct** | Tenant via their wallet | Tenant | Full tenant control |
| **Delegated** | Platform on behalf of tenant | Platform (delegated authority) | Operational convenience |
| **Multi-Sig** | Either party initiates | 2-of-2 (tenant + platform) | Maximum security |

## 9.2 Read (Resolve)

Resolution fetches the NameRegistry account from Solana and constructs a W3C DID Document.

### Algorithm

```
resolve(did):
  1.  Parse DID → extract sns-name and optional network
  2.  Strip ".sol" suffix if present
  3.  Validate hierarchy depth (max 2 levels)
  4.  Hash domain: SHA-256("SPL Name Service" + name)
  5.  Derive PDA: FindProgramAddress([hash, classKey, parent], SNS_PROGRAM)
      • classKey = 32 zero bytes for standard domains (no class lock)
      • parent = ROOT_DOMAIN_ACCOUNT (58PwtjS...) for top-level domains
      • For subdomains: derive parent PDA first, then child with \0 prefix
  6.  Fetch AccountInfo from Solana RPC
  7.  If not found → return notFound error
  8.  Extract owner key (bytes 32-63)
  9.  If owner is zero address → return deactivated
 10.  Check data buffer (bytes 96+) for magic 0x44494401:
      a. Found → parse DID metadata (v1 or v2 schema)
      b. Not found → degraded fallback (owner key only, no services/encryption)
 11.  If v2 + HAS_SAS flag → follow SAS UID for issuer-signed claims
 12.  Set controller from parent domain hierarchy
 13.  Populate service endpoints
 14.  Return DidResolutionResult
```

### Resolution Priority

1. Check data buffer for `0x44494401` magic → parse DID metadata (v1 or v2)
2. If v2 + `HAS_SAS` → follow SAS attestation UID for issuer-signed claims
3. Fallback → degraded DID Document from owner key only (no services, no encryption)

### Error Codes

| Code | HTTP | Condition |
|---|---|---|
| `invalidDid` | 400 | Malformed syntax, wrong prefix, invalid characters |
| `notFound` | 404 | SNS domain not registered or account does not exist |
| `deactivated` | 200 | Owner is zero address; `didDocumentMetadata.deactivated = true` |
| `internalError` | 500 | RPC failure or resolver error |

## 9.3 Update

DID Documents are *derived* from on-chain state, not stored as-is. Updates happen at the Solana layer:

| Update Action | On-Chain Operation | DID Document Effect |
|---|---|---|
| Rotate keys | Transfer SNS domain to new owner | `#solana-key` changes to new owner's key |
| Update metadata | Write to data buffer (offset 96+) | Flags, ECIES key, SAS UID updated |
| Link SAS attestation | Set `HAS_SAS` flag + write UID | Issuer-signed claims in resolution metadata |
| Transfer subdomain | Reassign subdomain owner | New key in verification methods |

> **Alias portability:** When domain ownership transfers, the DID identifier remains stable. Only the verification methods change. This is a key advantage over account-anchored methods.
>
> **Security implication:** The new domain owner inherits the DID identifier and can write fresh metadata under it. Verifiers relying solely on the minimal DID Document (without SAS attestation) cannot detect ownership changes. For identity continuity assurance, verifiers MUST validate the SAS attestation chain, which is bound to the original issuer — not the domain owner. See [§12 Security Considerations](12-security.md).

### Transfer Security Model

A domain transfer (key rotation) changes the controlling key but does **not** retroactively invalidate signatures made under the previous key.

```
Timeline:
                                      Transfer
Key A (original owner)                event                Key B (new owner)
--------[=========================]----||----[=========================]--------
        ^                        ^    ||    ^                         ^
    Signs doc #1            Signs doc #2    Signs doc #3          Signs doc #4
    (valid forever)         (valid forever) (valid, new key)      (valid, new key)
                                      ||
                                      || DID Document now shows Key B
                                      || Key A is no longer resolvable
                                      || via current DID resolution
```

| Asset | Effect of Transfer | Implication |
|---|---|---|
| **Old signatures** | Mathematically valid, but Key A is no longer in the DID Document | Verifiers resolving the DID *today* cannot verify old signatures without historical key lookup |
| **SAS attestations** | Unaffected — signed by the *issuer* authority, not the domain owner | Attestations survive domain transfers. Issuer can independently revoke. |
| **SBT (Soulbound Token)** | Stays with the *old wallet* (non-transferable by definition) | Domain transfer creates a mismatch: DID points to Key B, SBT remains on Key A's wallet. Issuer must mint a new SBT for Key B. |
| **Smart contracts** | Contracts referencing the DID identifier still resolve, but to Key B | The controlling party changes. Contracts should verify the *key*, not just the DID, for authorization checks. |
| **Encrypted vault objects** | VMK shares remain with the original holder (Share A) and platform (Share B) | Transfer does not grant the new owner access to encrypted data. Re-encryption under a new VMK is required for handover. |

> **WARNING:** After a domain transfer, the previous owner's key is no longer resolvable via current DID resolution. Old signatures become unverifiable through the DID layer alone. Smart contracts and legal documents MUST embed the signer's public key (not just the DID) in signature metadata to remain independently verifiable. SBTs create a permanent mismatch — the issuer must mint a new SBT for the new owner.

### Versioned Resolution

To verify signatures made under a previous key, verifiers SHOULD use `didDocumentMetadata.versionId` to request the DID Document at the time of signing. The resolver MAY support a `versionTime` query parameter (per [DID Core §3.2.1](https://www.w3.org/TR/did-core/#did-parameters)) to return the historical document state.

> **Recommendation for contracts and legal documents:** When signing legally binding documents, embed the signer's public key (not just the DID) in the signature metadata. This ensures the signature remains independently verifiable even after domain transfers, without depending on historical DID resolution.

## 9.4 Deactivate

A DID is deactivated by transferring ownership to the system program (all-zero address):

```json
{
  "@context": "https://w3id.org/did-resolution/v1",
  "didDocument": null,
  "didResolutionMetadata": { "contentType": "application/did+ld+json" },
  "didDocumentMetadata": { "deactivated": true }
}
```

> **WARNING — IRREVERSIBLE:** Transferring a domain to the zero address is permanent. The SNS program does not support reclaiming a zero-owner domain. All associated SAS attestations, service endpoints, and credential references become unresolvable. Deactivate only when the identity must be permanently retired.

## 9.5 DID Lifecycle: Issue Once, Evolve Forever

A `did:sns` identifier is created **once** and never re-issued. The DID persists across key rotations, wallet migrations, custodial changes, KYC level upgrades, credential issuance, and organizational restructuring. Only permanent deactivation (§9.4) ends a DID's life — and even then the identifier remains resolvable (with `deactivated: true`) to preserve audit history.

This "issue once, evolve forever" model is a deliberate architectural choice that contrasts with DID methods where identity changes require re-issuance or accumulate an ever-growing log of historical states.

### Lifecycle Phases

| Phase | Trigger | DID Identifier | What Changes | What Stays |
|---|---|---|---|---|
| **Creation** | SNS domain registration + DID metadata write | Minted | — | — |
| **Attestation** | SAS attestation linked (KYC, vLEI, etc.) | Unchanged | SAS UID in buffer, `HAS_SAS` flag set | DID, domain, owner key |
| **Key Rotation** | Domain transferred to new wallet | Unchanged | `#solana-key` verification method | DID, domain name, SAS attestation, service endpoints |
| **Credential Update** | KYC level change, LEI verification, status change | Unchanged | New SAS attestation PDA (old one closed) | DID, domain, all keys, service endpoints |
| **Tier Upgrade** | User graduates from Tier 1 → 2 → 3 | Unchanged | Flags (IS_TIER3, HAS_SBT), new verification methods | DID, domain, existing keys |
| **Encryption Key Rotation** | ECIES key compromised or scheduled rotation | Unchanged | ECIES public key in buffer | DID, domain, owner key, SAS attestation |
| **Organizational Transfer** | User moves from `alice.bankA.sol` to `alice.bankB.sol` | **New DID** | New domain = new DID (linked via `alsoKnownAs`) | SAS attestation can reference previous DID |
| **Suspension** | Compliance hold, fraud investigation | Unchanged | Active flag cleared; SAS attestation closed | DID remains resolvable but non-functional |
| **Recovery** | Identity re-verified after suspension | Unchanged | Active flag restored; new SAS attestation created | DID, domain (same identifier through the entire cycle) |
| **Deactivation** | Permanent retirement (owner → zero address) | Frozen | `deactivated: true` in metadata | DID resolvable for audit; cannot be used for new operations |

### Contrast with Other Lifecycle Models

| Model | Methods | Lifecycle Behavior | PQ Scaling Impact |
|---|---|---|---|
| **Ephemeral (one key = one DID)** | `did:key` | No rotation possible. Key compromise = new DID. No continuity. | None (no history) |
| **Log-based (append-only history)** | `did:webvh`, `did:peer` | DID persists but accumulates full change log. Every rotation adds an entry. Resolver must replay the entire log. | **Severe** — PQ signatures (2–17 KB each) accumulate in every log entry. 5-year DID: 1–4 MB log (see [§12.3](12-security.md#123-stateless-resolution--no-did-log)). |
| **Document-hosted (fetch current)** | `did:web` | DID persists. Current state only (no built-in history). Relies on web server availability. | Low (only current doc) |
| **Mutable state (overwrite in place)** | **`did:sns`** | DID persists. Current state overwritten in fixed-size buffer. History delegated to blockchain's native transaction log. O(1) resolution always. | **None** — buffer is 160 bytes regardless of key/sig sizes. PQ keys stored in SAS layer, referenced by hash. |

### Why "Issue Once" Matters for Regulated Environments

In banking, compliance, and cross-border payments, identifier stability is a regulatory requirement. A user's identity must survive:

- **Key compromise recovery** — the user gets a new wallet, but their identity (and all linked credentials, SAS attestations, and audit history) must continue under the same identifier
- **Custodial transitions** — a user moves from one bank to another, or the platform migrates infrastructure. The DID must not change because external parties hold references to it
- **Cryptographic evolution** — when the industry transitions from Ed25519 to ML-DSA-44, the DID must survive the key migration. With `did:sns`, this is a buffer update, not a DID re-issuance
- **Compliance continuity** — auditors need a stable identifier to trace the full history of an entity

> **Design principle:** The DID identifier is an *alias*, not a *key*. Aliases persist; keys rotate. The SNS domain (`alice.crbank.sol`) is the stable anchor. Everything underneath — keys, attestations, credentials, encryption methods — evolves independently without disturbing the identifier. This is the "Living DID" concept: the identity is always current, never re-issued, and never accumulates historical baggage in its resolution path.

## 9.6 Root Domains as Organizational DIDs

Root domains (`crbank.sol`, `platform.sol`) are **first-class DIDs**, not merely namespace containers. A root domain DID represents the *organization* itself, while subdomains represent individuals, departments, or services within that organization.

### Organizational vs Individual DIDs

| Aspect | Root Domain (Organization) | Subdomain (Individual) |
|---|---|---|
| **Represents** | A legal entity (bank, fintech, DAO, government body) | A natural person, employee, or service endpoint |
| **Onboarding** | KYB (Know Your Business) — LEI, articles of incorporation, UBO verification | KYC (Know Your Customer) — identity document, liveness, biometric |
| **SAS attestation** | vLEI OOR (Organizational Role) with LEI code, jurisdiction, authorized signers | DID Identity Attestation with KYC level, tier, role |
| **Issuer role** | **Can be an issuer** — the org DID signs SAS attestations for its subdomains | Consumer only — receives attestations, does not issue them |
| **Key custody** | Institutional wallet (HSM, multi-sig, or managed by platform) | Personal wallet, passkey, or platform-managed |
| **Subdomain provisioning** | Owner of the namespace — creates subdomains for members | Cannot create sub-subdomains (max depth = 2) |

### Trust Chain Verification

When a verifier resolves `did:sns:alice.crbank`, they can walk the trust chain upward to verify the issuing organization:

```
Verifier resolves: did:sns:alice.crbank
    │
    ├─ 1. Fetch SNS PDA for alice.crbank.sol
    │     → owner = Alice's wallet, HAS_SAS = true
    │
    ├─ 2. Fetch SAS attestation (Alice's identity)
    │     → signer = CRBank's authority key
    │     → claims: KYC Level 2, jurisdiction: CR
    │
    ├─ 3. Resolve parent: did:sns:crbank
    │     → controller = crbank (self-sovereign)
    │     → Verify: CRBank's authority key matches #solana-key
    │
    ├─ 4. Fetch SAS attestation (CRBank's org identity)
    │     → signer = Platform authority or recognized auditor
    │     → claims: LEI = 5299009QI7..., vLEI OOR verified
    │
    └─ 5. (Optional) Verify LEI via GLEIF API
          → Trust anchor — LEI active, entity status confirmed
```

Each step in the chain is independently verifiable from on-chain data. The verifier does not need to trust any intermediary — they verify the cryptographic signatures at each level.

### Organizational Domain Ownership Transfer (Acquisition / Merger)

When a company is acquired, merged, or restructured, the root domain may change owners. This is fundamentally different from an individual key rotation because it represents a **change in the legal entity** controlling the namespace.

| Scenario | What Happens | Impact on Subdomains |
|---|---|---|
| **Acquisition** (MegaBank acquires CRBank) | `crbank.sol` ownership transfers to MegaBank's wallet. New SAS attestation issued under MegaBank's authority. | All subdomains retain their DIDs. The `controller` chain now leads to MegaBank. |
| **Merger** (CRBank + FinCo → NewBank) | New domain `newbank.sol` registered. Old domains deactivated or kept as aliases via `alsoKnownAs`. | Subdomains migrated: `alice.crbank.sol` gets `alice.newbank.sol` (new DID). Old DID linked via `alsoKnownAs`. |
| **Rebrand** (CRBank → CRDigital) | New domain `crdigital.sol` registered. `crbank.sol` kept active with `alsoKnownAs`. | Both domains resolve during transition. |
| **Regulatory seizure** | Domain ownership transferred to regulator's wallet. | Client DIDs survive — regulator becomes namespace controller. Vault data protected by 2-of-2 XOR key-split. |

> **Key difference from individual transfers:** When an individual's wallet key rotates, the same person retains the identity. When an org domain changes owners, the *legal entity* behind the namespace changes. Regulators must be able to trace the acquisition in the on-chain history. Client consent may be required before re-attestation. The 2-of-2 XOR vault architecture ensures the acquiring entity cannot access client vault data without each client's individual consent.

---

[← §8 DID Document](08-did-document.md) | [Next: §10 On-Chain Metadata Schema →](10-metadata-schema.md)
