# 1. Abstract

The `did:sns` method binds W3C Decentralized Identifiers to [Solana Name Service](https://sns.id) (SNS) domains. The method is **alias-anchored** — identity is tied to a human-readable alias (`alice.crbank.sol`), not to a cryptographic key. The alias persists across key rotations, wallet migrations, and custodial changes.

`did:sns` is **Web3 native but Web2 transparent**. The underlying infrastructure runs on Solana, but end users and tenants interact only with readable aliases — they never need to understand DIDs, W3C standards, or blockchain mechanics. A bank client sharing `alice.crbank.sol` with a fintech in another country triggers the same interoperable verification flow as a Web3 power user managing their own wallet keys. The specification eliminates friction between these worlds: **interoperability without complexity, compliance without lock-in**.

This specification is **operator-agnostic**: any SNS domain owner — a platform, a bank, a standards body, or an individual — can anchor DIDs under their namespace. A single user may hold multiple independent aliases across multiple issuers, each independently verifiable, each carrying its own trust level based on the issuer's attestations.

| Field | Value |
|---|---|
| Method Name | `sns` |
| Status | Provisional |
| Verifiable Data Registry | Solana (SPL Name Service + SAS) |
| Specification Version | v0.4.0 |
| Last Updated | 2026-03-12 |
| Editor | [Attestto](https://attestto.com) |
| Author | Eduardo Chongkan ([@chongkan](https://github.com/chongkan)) |
| Reference Implementation | [`@attestto/did-sns-resolver`](https://github.com/Attestto-com/did-sns-resolver) |

**Key properties:**

- **Alias-anchored** — identity persists across key rotations and wallet migrations; the alias is the stable identifier
- **Web2 transparent** — end users see only readable aliases; Web3 infrastructure is invisible at Tier 1/2
- **Hierarchical control** — subdomains inherit controller relationships from parent domains
- **Interoperable** — cross-institution, cross-border verification without bilateral integration
- **On-chain verifiable** — all resolution data anchored to Solana with sub-second finality
- **Issuer-signed proofs** — SAS attestations provide cryptographically verifiable claims, not self-asserted data
- **Compliance-first** — designed for regulated environments; no PII on-chain; jurisdiction-specific compliance mapping provided in [§5.3](05-privacy.md#53-regulatory-compliance-mapping)
- **Chain-replicable** — architecture portable to any blockchain with a name service; cross-chain compatibility under [active discussion](https://github.com/Attestto-com/did-sns-spec/discussions)

> **Cross-chain portability:** The `did:sns` architecture — alias-anchored identity with a metadata buffer and attestation layer — is not inherently Solana-specific. The same specification can be replicated on any blockchain with a compatible name service (e.g., ENS on Ethereum, Unstoppable Domains, etc.). Cross-chain compatibility and multi-registry resolution are active areas of development — see the [discussion section](https://github.com/Attestto-com/did-sns-spec/discussions) for proposals and feedback.

---

[Next: §2 Focal Use Case, Identity Tiers & Requirements →](02-focal-use-case.md)
