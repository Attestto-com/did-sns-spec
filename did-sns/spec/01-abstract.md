# 1. Abstract

`did:sns` gives users a **human-readable alias** — already resolvable by Solana wallets — and formalizes it as a W3C Decentralized Identifier. The core design separates three things that traditional systems conflate:

> [!NOTE]
> **The three separations that define did:sns:**
>
> 1. **Alias ≠ key.** The identity is `alice.crbank.sol` — a human-readable name. The cryptographic key lives in the DID Document and can be **rotated without changing the identity**. A compromised key doesn't mean a lost identity.
>
> 2. **Identity ≠ transaction history.** When the key rotates, the old key's on-chain transaction history stays with the old key. The alias resolves to the *current* key — past transactions under previous keys are **not linkable to the current identity** through DID resolution. The identity is permanent; the traceable keys are disposable.
>
> 3. **Alias ≠ single issuer.** A user can hold independent aliases across multiple institutions (`alice.crbank.sol`, `alice.fintech.sol`). Each is independently controlled, independently verifiable, and reveals nothing about the others. Sharing one alias with a counterparty does not expose the rest.

These separations are not theoretical — they work today. SNS domains are already resolved by Phantom, Solflare, and other Solana wallets. `did:sns` formalizes this into the W3C ecosystem so any DID-aware system — not just Solana wallets — can resolve the alias, get the current public key, and verify credentials, without being locked to any specific wallet, platform, or blockchain.

The specification is **operator-agnostic**: any SNS domain owner — a platform, a bank, a standards body, or an individual — can anchor DIDs under their namespace. The underlying infrastructure runs on Solana, but end users interact only with readable aliases — they never need to understand DIDs, W3C standards, or blockchain mechanics.

| Field | Value |
|---|---|
| Method Name | `sns` |
| Status | Provisional |
| Verifiable Data Registry | Solana (SPL Name Service + SAS) |
| Specification Version | v0.4.0 |
| Last Updated | 2026-04-14 |
| Editor | [Attestto](https://attestto.com) |
| Author | Eduardo Chongkan ([@chongkan](https://github.com/chongkan)) |
| Reference Implementation | [`@attestto/did-sns-resolver`](https://github.com/Attestto-com/did-sns-resolver) |

**Key properties:**

- **Human-readable** — `alice.crbank.sol`, not `7Xf3kP9...` — already resolvable by Solana wallets
- **Key rotation without identity loss** — compromise a key, rotate it, your alias and identity persist
- **Transaction history detachment** — old keys' on-chain activity doesn't follow you after rotation; the alias resolves to the current key only
- **Permanent identity** — the alias is the anchor; keys, wallets, platforms are replaceable layers underneath
- **Hierarchical control** — subdomains inherit controller relationships from parent domains (`alice.crbank.sol` controlled by `crbank.sol`)
- **Web2 transparent** — end users see only readable aliases; blockchain infrastructure is invisible
- **On-chain verifiable** — all resolution data anchored to Solana with sub-second finality
- **Issuer-signed proofs** — SAS attestations provide cryptographically verifiable claims (requires SAS infrastructure — see [§5](05-privacy.md))
- **Compliance-first** — no PII on-chain; jurisdiction-specific compliance mapping in [§5.3](05-privacy.md#53-regulatory-compliance-mapping)
- **Chain-replicable** — architecture portable to any blockchain with a name service

> **Cross-chain portability:** The `did:sns` architecture — alias-anchored identity with a metadata buffer and attestation layer — is not inherently Solana-specific. The same specification can be replicated on any blockchain with a compatible name service (e.g., ENS on Ethereum, Unstoppable Domains, etc.). Cross-chain compatibility and multi-registry resolution are active areas of development — see the [discussion section](https://github.com/Attestto-com/did-sns-spec/discussions) for proposals and feedback.

---

[Next: §2 Focal Use Case, Identity Tiers & Requirements →](02-focal-use-case.md)
