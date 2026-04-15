# 5. Privacy Architecture

Privacy in `did:sns` is a layered architecture where each layer addresses a different threat model. The design principle: **the minimum possible data is exposed at each interaction, and no single party holds enough information to reconstruct a user's full identity or activity.**

> [!IMPORTANT]
> **Transparency on privacy scope:** Some privacy properties are inherent to the `did:sns` method itself. Others require additional infrastructure (issuer platforms, SAS attestation layer, credential issuance stack) to be effective. The table below clearly marks which layer provides each property. Implementers should understand that **the DID method alone provides pseudonymity and on-chain minimization — not full privacy**. Full selective disclosure, ZKP predicates, and consent-gated access require a credential issuance and presentation stack built on top of `did:sns`.

## 5.1 Privacy Layers

| Layer | Mechanism | What It Protects | Provided by |
|---|---|---|---|
| **On-chain minimization** | NameRegistry stores only keys, flags, and hashes — never names, emails, or personal data | Prevents PII exposure from public ledger | **did:sns method (inherent)** |
| **Alias pseudonymity** | SNS domains are public but pseudonymous — `a7f3.platform.sol` reveals nothing about the holder's real identity | Identity not exposed on-chain unless voluntarily linked | **did:sns method (inherent)** |
| **Alias independence** | Each DID is independent; user controls which to share | Prevents cross-issuer correlation — sharing `alice.crbank.sol` reveals nothing about `alice.fintech.sol` | **did:sns method (inherent)** |
| **Transaction privacy** | Alias resolves to wallet; wallet can rotate; transaction layer is independent | Payment sender resolves alias to wallet without learning KYC status | **did:sns method (inherent)** — but wallet rotation requires platform support |
| **Selective disclosure** | SD-JWT ([RFC 9449](https://datatracker.ietf.org/doc/draft-ietf-oauth-selective-disclosure-jwt/)) with per-field salt and hash | User reveals only the fields required per presentation (e.g., "KYC Level 2" without date of birth) | **Credential layer (requires issuer stack)** — did:sns defines the identity; selective disclosure applies to credentials issued to that identity |
| **ZKP predicates** | BBS+ signatures for unlinkable proofs | Prove properties ("age ≥ 18", "jurisdiction = EU") without revealing the underlying value | **Credential layer (requires issuer stack + BBS+ implementation)** — not yet implemented in reference stack |
| **Consent-gated access** | 2-of-2 XOR key-split (VMK architecture) | Neither the platform nor the user alone can decrypt proof data; both must participate | **Platform infrastructure (requires vault implementation)** — defined in spec but requires operational infrastructure |
| **LEI privacy** | LEI numbers stored as SHA-256 hashes in SAS attestations | Verifiers hash a known LEI to compare; on-chain data doesn't reveal the LEI itself | **SAS layer (requires attestation infrastructure)** |

### What did:sns provides standalone (no additional infrastructure)

1. **No PII on-chain** — the Solana ledger never contains personal data; only keys, flags, and hashes
2. **Pseudonymous aliases** — SNS domains do not inherently reveal the holder's identity
3. **Independent aliases** — multiple DIDs are unlinkable unless the user chooses to link them
4. **Wallet rotation** — the DID persists across key/wallet changes, preventing historical transaction correlation to current identity
5. **Public alias correlation risk** — SNS domains are publicly resolvable. If a user shares `alice.crbank.sol` with multiple parties, those parties CAN correlate interactions. This is a fundamental property of human-readable aliases on public ledgers.

### What requires additional infrastructure

1. **Selective disclosure (SD-JWT)** — requires a credential issuer that issues SD-JWT credentials to the DID holder
2. **ZKP predicates (BBS+)** — requires BBS+ signature implementation in the issuance and presentation stack
3. **Consent-gated vault access** — requires platform-operated encrypted vault infrastructure with 2-of-2 key split
4. **SAS attestations** — require the Solana Attestation Service program and authorized issuers
5. **Cross-chain binding** — requires SAS attestation infrastructure for opt-in CAIP-10 linking

## 5.2 Correlation Risk & Mitigations

SNS aliases are public and resolvable. This creates correlation risk — if a user shares `alice.crbank.sol` with multiple parties, those parties can correlate interactions. Mitigations:

1. **Multiple aliases** — users hold independent DIDs across issuers; share different aliases in different contexts
2. **Pseudonymous aliases** — platform subdomains can be pseudonymous (`a7f3.platform.sol`), user's choice
3. **ZKP cross-DID proofs** — prove "all these DIDs belong to the same person" without revealing which DIDs, via ZKP on national ID or biometric hash
4. **Fresh subdomains** — for maximum unlinkability, users can request a new subdomain per relationship

## 5.4 Cross-Chain Privacy Model

`did:sns` uses `.sol` as the **privacy anchor**. Binding other chain accounts (Ethereum, Bitcoin, etc.) to a `did:sns` identity is **strictly opt-in** — nothing about a user's cross-chain wallets is exposed unless they explicitly create a SAS attestation.

### Default State: Privacy by Inaction

A user who holds an Ethereum wallet and a `did:sns` identity has no cross-chain linkage unless they take explicit action. Resolvers who query `did:sns:alice.attestto` learn nothing about `0x...` addresses the user may hold.

> [!NOTE]
> **Contrast with `did:pkh` (CAIP-10):** In `did:pkh`, the wallet address *is* the identity — public and permanently linkable. Every transaction under that address is part of the identity's history, forever. `did:sns` inverts this: the alias is the identity, the wallet is hidden behind it, and cross-chain binding is opt-in and revocable.

### Opt-In Binding: SAS Attestation

When a user chooses to link another chain:

1. User signs a SAS attestation on Solana proving ownership of the target chain account (CAIP-10 format)
2. Platform writes the attestation on-chain
3. `did_document_service` emits the CAIP-10 address in `alsoKnownAs` on the next DID Document resolution
4. Verifiers who receive the DID Document can now confirm cross-chain ownership

The user can **revoke the SAS attestation at any time** — the CAIP-10 entry disappears from the next DID Document resolution. This is not possible with address-based DID methods.

### Resolution Privacy: Always Through `did:sns`

Cross-chain resolution MUST go **through** `did:sns`, not around it. The resolver chain is:

```
Cross-chain verifier
  → discovers CAIP-10 address in did:sns alsoKnownAs (opt-in only)
    → resolves did:sns identity
      → privacy controls, SD-JWT selective disclosure, consent-gated access apply
```

A verifier who has only a raw Ethereum address and attempts to skip DID resolution loses all privacy protections — they see only what is on-chain on Ethereum, with no access to the holder's selective disclosure controls, credential status lists, or consent gate. This creates a clear incentive for verifiers to prefer `did:sns` resolution over raw chain lookups.

### Privacy Comparison

| Approach | Cross-chain exposure | Revocable? | Privacy controls |
|---|---|---|---|
| Raw CAIP-10 / `did:pkh` | Always public (address is the ID) | ❌ No | ❌ None |
| `did:sns` unlinked | No cross-chain data exposed | N/A | ✅ Full |
| `did:sns` + SAS binding | User-chosen chains only | ✅ Yes | ✅ Full |

## 5.3 Regulatory Compliance Mapping

| Regulation | Jurisdiction | How `did:sns` Complies |
|---|---|---|
| **GDPR** (Art. 17 — Right to erasure) | EU | Delete vault content + deactivate DID. On-chain pointers become meaningless without encrypted payload. Dual-key encryption = pseudonymized data beyond any single custodian's control |
| **Law 8968** (Protección de Datos Personales) | Costa Rica | Same as GDPR — 2-of-2 XOR key-split ensures data is pseudonymized. Platform alone cannot access user data |
| **LGPD** | Brazil | Same erasure model; consent-gated access aligns with consent requirements |
| **CCPA** | California, US | Right to delete satisfied by vault content deletion; right to know satisfied by user's access to their own vault |
| **FATF Travel Rule** | Global | Originator/beneficiary data embedded in DID attestation layer; transmitted with transaction, not stored publicly |
| **PCI DSS** | Global | No payment card data in DID layer; stablecoin settlement bypasses card networks entirely |

> *This table will expand as the ecosystem grows. Implementers in new jurisdictions should contribute their local compliance mapping. See the [discussion section](https://github.com/Attestto-com/did-sns-spec/discussions) for jurisdiction-specific proposals.*

> *As with all alias-anchored DID methods, these privacy guarantees apply equally to `did:ens` or any method following this specification.*

---

[← §4 Architectural Rationale](04-architectural-rationale.md) | [Next: §6 W3C Coverage →](06-w3c-coverage.md)
