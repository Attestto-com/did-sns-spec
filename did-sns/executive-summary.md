# did:sns — Executive Summary

**Alias-anchored W3C Decentralized Identifiers for cross-border interoperability, privacy, and compliance**

## The Problem

Regulated institutions — fintechs, banks, stablecoin issuers — face an impossible trade-off: Ethereum (ENS) is the gold standard for ecosystem reach, but it's a privacy nightmare (fully transparent transaction history) and identities can "expire" if leases aren't renewed. Solana offers speed and low cost, but identity standards are fragmented. Meanwhile, compliance teams need KYC/KYB proof, users need privacy, and every chain maintains its own identity silo.

The result: duplicated onboarding costs per chain, PII scattered across counterparties, stale compliance data, and no portable identity between partners.

## The Solution: Dual-DID Architecture

`did:sns` introduces a **Dual-DID architecture** where a permanent, privacy-preserving Solana identity serves as the **primary credential anchor**, while linked identities on Ethereum and other chains extend the holder's reach — without duplicating credentials or sacrificing privacy.

```
did:sns:alice.attestto.sol          ← Primary anchor (credentials, vault, consent)
  ├── alsoKnownAs: did:ens:alice.eth        ← Ethereum ecosystem reach
  ├── alsoKnownAs: did:pkh:solana:CKg5...   ← Universal key-proof layer
  └── alsoKnownAs: did:web:alice.com         ← Traditional web binding
```

**Primary DID (Solana):** Holds the credentials, 2-of-N Shamir key splitting, and post-quantum security. Permanent (rent-exempt).

**Linked DID (Ethereum):** Acts as the public face for ecosystem reach.

**The Bridge:** Bidirectional `alsoKnownAs` linking allows an Ethereum user to present a Solana-backed credential without ever leaving the ETH ecosystem.

## Privacy by Design — 7 Layers

The DID Document contains **zero personal data**. Every layer of the architecture is engineered to minimize what a verifier, an on-chain observer, or even the platform itself can learn about the holder.

1. **No PII on-chain.** The Solana ledger stores only cryptographic commitments (SHA-256 hashes, public keys, attestation pointers). All personal data lives in encrypted Data Vaults protected by a 2-of-N Shamir key split. Neither the platform alone nor a single device can access the data.

2. **Selective disclosure via SD-JWT.** Credentials use per-field salted hashes (IETF SD-JWT). A holder can prove "I am over 18" or "my country is in the EU" without revealing their date of birth or exact nationality. The verifier sees only what they asked for.

3. **Pairwise identifiers.** Each verifier relationship gets a unique subdomain DID derived from `SHA-256(verifierDID + holderSecret)`. Two verifiers cannot collude to correlate the same holder.

4. **Consent-gated proof access.** No credential is ever shared without explicit holder consent. Field-by-field consent, time-bounded and revocable.

5. **Dual-key encryption.** A per-user Vault Master Key (VMK) is split via 2-of-N Shamir secret sharing across the user's devices, the platform, and optional guardians. Reconstruction requires a minimum threshold of shares.

6. **Crypto-shredding for right to erasure.** Deleting the platform's key share renders all vault objects permanently inaccessible. Satisfies GDPR Article 17 and Costa Rica Law 8968.

7. **Post-quantum forward secrecy.** Hybrid migration path: ML-DSA-44 (FIPS 204) alongside Ed25519, and ML-KEM-768 alongside X25519.

## Value Proposition for Fintechs

- **One credential, every chain.** Issue a KYC credential once to `did:sns`. The user presents it on Ethereum, Solana, or any chain with a linked DID — no re-verification, no duplicate onboarding, no per-chain compliance cost.

- **Selective disclosure by default.** Your user proves "I am KYC Level 2 in a FATF-compliant jurisdiction" without revealing their name, address, or document scans. SD-JWT per-field hashing — the verifier sees only what they asked for.

- **Portable compliance across partners.** A user onboarded by Bank A can present the same credential to Fintech B. The credential is anchored to a permanent Solana DID — it survives the user switching banks, wallets, or chains.

## Value Proposition for Stablecoin Issuers

- **Travel Rule without the privacy leak.** FATF requires originator/beneficiary data on transfers above the threshold. With SD-JWT, you disclose exactly the required fields to the counterparty — not your user's entire identity profile.

- **Cross-chain mint/burn with one identity.** A user minting USDC on Solana and bridging to Ethereum keeps the same verified identity across both chains. The Ethereum side resolves `did:ens` → `alsoKnownAs` → `did:sns` → credential. One identity, two ecosystems, zero re-onboarding.

- **Revocation in real time.** If a user's compliance status changes (sanctions hit, KYC expires, EDD required), the credential is revoked via W3C Bitstring Status List. Every verifier checking that credential — on any chain — sees the revocation instantly. No stale KYC.

- **Audit trail without custodial liability.** The credential proves what your compliance team attested to and when. But the user's PII sits in their own encrypted vault (2-of-N Shamir) — you never hold the raw data after verification. Reduces your data breach surface to zero.

- **ISO 20022 alignment.** The `did:sns` identity model maps directly to ISO 20022 party identification structures — the same standard that SWIFT, Fedwire, TARGET2, and BCCR's SINPE use for cross-border payments. A `did:sns` DID resolves to structured party data (legal name, LEI, jurisdiction) that slots into `pacs.008` and `pacs.009` messages without translation layers. Stablecoin transfers carrying a `did:sns` identity can bridge into traditional payment rails with compliance data already in the format banks expect.

## The Numbers That Matter

| Today (per-chain KYC) | With Dual-DID |
|---|---|
| $5-15 per user per chain for KYC | One verification, every chain |
| 3-7 days re-onboarding per partner | Instant credential presentation |
| Full PII stored per counterparty | Zero PII held — selective disclosure only |
| Manual sanctions re-screening | Real-time revocation propagation |
| Chain-specific identity silos | One portable identity, chain-agnostic |

## The Bottom Line

**Identity becomes infrastructure, not overhead.** Issue once, verify everywhere, revoke instantly, and never hold PII you don't need to.

The specification meets the requirements of GDPR, Costa Rica Law 8968, FATF Travel Rule, and ISO 20022 simultaneously — proving that regulatory compliance and user privacy are not in conflict.

## Learn More

| Resource | Link |
|---|---|
| Full Specification (14 sections) | [spec.attestto.com/did-sns](https://spec.attestto.com/did-sns) |
| Privacy Architecture | [Section 5 — Privacy](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/05-privacy.md) |
| Interoperability | [Section 13 — Interoperability](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/13-interoperability.md) |
| Cross-Chain Proposal | [W3C did-extensions #680](https://github.com/w3c/did-extensions/issues/680) |
| Implementation Report | [spec.attestto.com/did-sns/report](https://spec.attestto.com/did-sns/report) |
| Reference Implementation | [`@attestto/did-sns-resolver`](https://github.com/Attestto-com/did-sns-resolver) |
