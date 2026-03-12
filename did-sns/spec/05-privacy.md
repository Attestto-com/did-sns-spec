# 5. Privacy Architecture

Privacy in `did:sns` is not a single feature — it is a layered architecture where each layer addresses a different threat model. The design principle: **the minimum possible data is exposed at each interaction, and no single party holds enough information to reconstruct a user's full identity or activity.**

## 5.1 Privacy Layers

| Layer | Mechanism | What It Protects |
|---|---|---|
| **On-chain minimization** | NameRegistry stores only keys, flags, and hashes — never names, emails, or personal data | Prevents PII exposure from public ledger |
| **Selective disclosure** | SD-JWT ([RFC 9449](https://datatracker.ietf.org/doc/draft-ietf-oauth-selective-disclosure-jwt/)) with per-field salt and hash | User reveals only the fields required per presentation (e.g., "KYC Level 2" without date of birth) |
| **ZKP predicates** | BBS+ signatures for unlinkable proofs | Prove properties ("age ≥ 18", "jurisdiction = EU") without revealing the underlying value |
| **Consent-gated access** | 2-of-2 XOR key-split (VMK architecture) | Neither the platform nor the user alone can decrypt proof data; both must participate |
| **Alias independence** | Each DID is independent; user controls which to share | Prevents cross-issuer correlation — sharing `alice.crbank.sol` reveals nothing about `alice.fintech.sol` |
| **Transaction privacy** | Alias resolves to wallet; wallet can rotate; transaction layer is independent | Payment sender resolves alias to wallet without learning KYC status; verifier confirms identity without seeing transaction history |
| **LEI privacy** | LEI numbers stored as SHA-256 hashes in SAS attestations | Verifiers hash a known LEI to compare; on-chain data doesn't reveal the LEI itself |

## 5.2 Correlation Risk & Mitigations

SNS aliases are public and resolvable. This creates correlation risk — if a user shares `alice.crbank.sol` with multiple parties, those parties can correlate interactions. Mitigations:

1. **Multiple aliases** — users hold independent DIDs across issuers; share different aliases in different contexts
2. **Pseudonymous aliases** — platform subdomains can be pseudonymous (`a7f3.platform.sol`), user's choice
3. **ZKP cross-DID proofs** — prove "all these DIDs belong to the same person" without revealing which DIDs, via ZKP on national ID or biometric hash
4. **Fresh subdomains** — for maximum unlinkability, users can request a new subdomain per relationship

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
