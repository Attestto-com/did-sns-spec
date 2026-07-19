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
| **Selective disclosure** | SD-JWT ([IETF draft-ietf-oauth-selective-disclosure-jwt](https://datatracker.ietf.org/doc/draft-ietf-oauth-selective-disclosure-jwt/)) with per-field salt and hash | User reveals only the fields required per presentation (e.g., "KYC Level 2" without date of birth) | **Credential layer (requires issuer stack)** — did:sns defines the identity; selective disclosure applies to credentials issued to that identity |
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
## 5.2 Correlation Risk & Mitigations

SNS aliases are public and resolvable. This creates correlation risk — if a user shares `alice.crbank.sol` with multiple parties, those parties can correlate interactions. Mitigations:

1. **Multiple aliases** — users hold independent DIDs across issuers; share different aliases in different contexts
2. **Pseudonymous aliases** — platform subdomains can be pseudonymous (`a7f3.platform.sol`), user's choice
3. **Fresh subdomains** — for maximum unlinkability, users can request a new subdomain per relationship

## 5.3 Cross-Chain Wallet Linking: Deliberately Excluded

> [!CAUTION]
> **`did:sns` does not link or expose wallet addresses across chains.** Exposing wallet addresses from other chains (e.g., via `alsoKnownAs` CAIP-10 entries) in the DID Document would break the core privacy property — any resolver could see the wallet address and trace its full transaction history. This is the exact problem `did:sns` is designed to prevent.
>
> Cross-chain interoperability for payments and transfers is handled by dedicated services at the transaction layer (e.g., Circle CCTP, bridge protocols), not at the identity layer. `did:sns` resolves aliases to DID Documents — it is a resolution layer, not a wallet linking service.

> [!NOTE]
> **Contrast with `did:pkh` (CAIP-10):** In `did:pkh`, the wallet address *is* the identity — public and permanently linkable. Every transaction under that address is part of the identity's history, forever. `did:sns` inverts this: the alias is the identity, the wallet is hidden behind it by the platform.

## 5.4 Regulatory Compliance

> [!IMPORTANT]
> **Regulatory compliance is a separate layer built on top of `did:sns`, not embedded in the method.** The method's contribution to compliance is: no PII on-chain, pseudonymous aliases, and service endpoints where compliance data can attach. The compliance logic itself (FATF Travel Rule, AML screening, ISO 20022 mapping) is implemented by platforms and issuers using the hooks defined in [§2 Use Cases](02-focal-use-case.md).

| Regulation | did:sns contribution (method level) | Platform/issuer responsibility |
|---|---|---|
| **GDPR** (Art. 17) | No PII on-chain; DID deactivation makes pointers meaningless | Vault content deletion, consent management, crypto-shredding |
| **Law 8968** (Costa Rica) | Same as GDPR — no PII on-chain | Platform manages data access, ARCO rights |
| **FATF Travel Rule** | Service endpoint `TravelRuleService` for data exchange | Originator/beneficiary data exchange between platforms |
| **ISO 20022** | Service endpoint `ISO20022PartyId` for party mapping | Financial message formatting and routing |
| **AML / Sanctions** | Alias resolves to platform address (not user wallet) | Sender screening before accepting funds (Circle blacklist, OFAC, etc.) |

---

[← §4 Architectural Rationale](04-architectural-rationale.md) | [Next: §6 W3C Coverage →](06-w3c-coverage.md)
