# 6. W3C Use Cases & Requirements Coverage

The W3C [DID Use Cases and Requirements](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/) note defines 22 features that DID methods should address, mapped against five benefit categories ([Feature/Benefit Grid](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#featureBenefitGrid)). The table below shows how `did:sns` covers each feature.

## 6.1 Feature Coverage Matrix

| W3C Feature | did:sns | How |
|---|---|---|
| [Authentication / proof of control](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#authenticate) | **Covered** | Ed25519 signature over the DID Document; SAS attestations bind issuer-signed proofs to the identifier. |
| [Decentralized / self-issued](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#decentralized) | **Covered** | SNS domain registration on Solana requires no central authority. Any wallet can register a `.sol` domain and anchor a DID. |
| [Guaranteed unique identifier](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#unique) | **Covered** | SNS domains are globally unique on Solana. The PDA derivation algorithm guarantees collision-free identifiers. |
| [No call home](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#noCallHome) | **Covered** | Resolution reads on-chain state via any Solana RPC. Verification uses standard JOSE libraries — no need to contact the issuer or any specific platform. |
| [Associated cryptographic material](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#associatedCrypto) | **Covered** | DID Document includes Ed25519, secp256k1, and ML-DSA verification methods tightly bound to the identifier. |
| [Streamlined key rotation](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#keyRotation) | **Covered** | Core design principle — SNS domain transfer updates the owner keypair while the DID identifier stays the same. No credential reissuance required. |
| [Service endpoint discovery](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#serviceEndpoint) | **Covered** | DID Document `service` array supports arbitrary endpoints (DIDComm, credential status, linked domains). |
| [Privacy preserving](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#privacyPreserving) | **Covered** | Layered privacy architecture: (1) users choose their own subdomain names; (2) each DID is independent — the user controls which to share; (3) off-chain privacy layer keeps all PII off the public ledger; (4) SD-JWT per-field selective disclosure; (5) on-chain data contains only hashes and pointers; (6) ZKP proofs can link multiple DIDs without revealing identity data. See [§5](05-privacy.md). |
| [Delegation of control](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#delegation) | **Covered** | Hierarchical subdomain model — parent domain owner controls child subdomains. `controller` field in DID Document supports multi-party delegation. |
| [Inter-jurisdictional](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#interJ) | **Covered** | Solana is a permissionless global network. DID resolution and credential verification work identically regardless of jurisdiction. |
| [Can't be administratively denied](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#noDenial) | **Covered** | The DID binding (subject ↔ subdomain) is **permanent** — once issued via KYC/KYB, the identity record cannot be reassigned or deleted. What expires is *access and capabilities*, not the identity itself. Operators can suspend credentials per regulatory requirements, but the DID remains resolvable and auditable. |
| [Minimized rents](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#minRent) | **Covered** | Three identity tiers with progressive cost: **Tier 1** — zero blockchain cost; **Tier 2** — platform/tenant absorbs; **Tier 3** — optional SBT minting fee (the only user-facing on-chain cost). No per-transaction resolution fees. |
| [No vendor lock-in](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#noVendorLock) | **Covered** | Universal Resolver driver, standard SD-JWT credentials, DIDComm v2, and public status lists. No proprietary software required — any compliant implementation works. See [§13](13-interoperability.md). |
| [Preempt / limit trackable data trails](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#limitTrack) | **Covered** | On-chain stores only hashes and pointers — no PII. Each DID is independent; the user decides which to present. Pseudonymous aliases available. SD-JWT selective disclosure available at the credential layer (requires issuer stack). |
| [Cryptographic future-proof](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#cryptoFuture) | **Covered** | Post-quantum migration path defined with ML-DSA-44 / ML-KEM-768 (FIPS 203/204). Hybrid mode during transition period. See [§12.1](12-security.md#121-post-quantum-cryptography-migration). |
| [Survives issuing organization mortality](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#orgMort) | **Covered** | DID data is anchored on Solana. If any operator ceases to exist, the on-chain state and open-source resolver code (MIT-licensed) remain independently verifiable by any party. |
| [Survives deployment end-of-life](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#deployEnd) | **Covered** | On-chain persistence. Any Solana RPC node can serve resolution data regardless of the requesting party's software lifecycle. |
| [Survives relationship with service provider](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#survivesRel) | **Covered** | Alias-anchored design — the DID persists across custodial changes. Domain transfer changes the controller without changing the identifier. |
| [Cryptographic authentication & communication](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#cryptoAuthComm) | **Covered** | DIDComm v2 P2P encrypted channels. Ed25519 authentication + X25519 key agreement for secure communication. |
| [Registry agnostic](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#regAgnostic) | **Partial** | Bound to Solana/SNS as the verifiable data registry. However, the architecture is chain-replicable (see [§1](01-abstract.md)) and the Universal Resolver driver means consumers are registry-agnostic. |
| [Legally-enabled identity](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#legallyEnabled) | **Covered** | GLEIF vLEI bridge for organizational identity. SAS attestations provide issuer-signed proofs suitable for regulated contexts. ISO 20022 compliance mapping. See [§4.1](04-architectural-rationale.md#41-why-aliases-over-account-numbers). |
| [Human-centered interoperability](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#humanInterop) | **Covered** | Human-readable SNS aliases (`alice.crbank.sol`) instead of opaque hashes. DNS TXT aliasing bridges Web2 domain names to `did:sns` identifiers. |

> **Summary:** `did:sns` fully covers **21 of 22** W3C features. The single partial coverage (Registry Agnostic) is an inherent property of binding to Solana/SNS as the verifiable data registry, mitigated by the chain-replicable architecture and Universal Resolver driver which makes *consumers* registry-agnostic.

## 6.2 Benefit Alignment Grid

The W3C note groups features into five benefit categories. The grid below mirrors the [official Feature/Benefit Grid](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#featureBenefitGrid) and marks which benefits `did:sns` delivers for each feature it covers.

| Feature | Anti-censor | Anti-exploitation | Ease of use | Privacy | Sustainability | did:sns |
|---|---|---|---|---|---|---|
| Authentication / proof of control | ✓ | ✓ | | | | ✓ |
| Decentralized / self-issued | ✓ | ✓ | ✓ | ✓ | | ✓ |
| Guaranteed unique identifier | ✓ | ✓ | | | | ✓ |
| No call home | ✓ | ✓ | ✓ | ✓ | | ✓ |
| Associated cryptographic material | ✓ | ✓ | ✓ | | | ✓ |
| Streamlined key rotation | ✓ | ✓ | ✓ | ✓ | | ✓ |
| Service endpoint discovery | | | ✓ | | | ✓ |
| Privacy preserving | | | | ✓ | ✓ | ✓ |
| Delegation of control | ✓ | | | | | ✓ |
| Inter-jurisdictional | ✓ | ✓ | | | | ✓ |
| Can't be administratively denied | ✓ | | | | | ✓ |
| Minimized rents | | ✓ | | | | ✓ |
| No vendor lock-in | ✓ | ✓ | | | | ✓ |
| Preempt / limit trackable data trails | ✓ | ✓ | ✓ | ✓ | | ✓ |
| Cryptographic future-proof | | | | | ✓ | ✓ |
| Survives issuing org mortality | | | | | ✓ | ✓ |
| Survives deployment end-of-life | | | | | ✓ | ✓ |
| Survives relationship with provider | | | | | ✓ | ✓ |
| Crypto authentication & communication | ✓ | ✓ | | | | ✓ |
| Registry agnostic | ✓ | ✓ | ✓ | | | ~ |
| Legally-enabled identity | ✓ | ✓ | ✓ | | | ✓ |
| Human-centered interoperability | ✓ | ✓ | ✓ | | | ✓ |

**Legend:** ✓ = W3C-defined benefit alignment | ✓ (did:sns column) = fully covered | ~ = partially covered

---

[← §5 Privacy](05-privacy.md) | [Next: §7 DID Syntax →](07-did-syntax.md)
