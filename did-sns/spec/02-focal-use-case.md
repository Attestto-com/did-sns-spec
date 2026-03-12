# 2. Focal Use Case, Identity Tiers & Requirements

`did:sns` is designed for **regulated multi-issuer identity** — an ecosystem where banks, fintechs, platforms, and standards bodies independently issue DIDs for their users under their own domains, all following the same specification to guarantee interoperability.

## The Scenario

A user has accounts at three banks and a platform. Each issuer independently performs KYC/KYB and issues a `did:sns` subdomain under its own root domain:

- `alice.crbank.sol` — issued by CRBank after KYC
- `alice.megabank.sol` — issued by MegaBank after KYC
- `alice.fintech.sol` — issued by a fintech after KYB
- `alice.platform.sol` + `a7f3.platform.sol` — platform subdomains (pseudonym or real name, user's choice)
- `alice.chongkan.sol` — user's own custom domain, purchased or rented, delegated to a platform for management

Each DID is **independent** — signed with whatever keys the issuing institution uses, carrying its own SAS attestations (KYC level, LEI status, credentials). The same physical person holds all five DIDs, provable via ZKP (national ID or SSN+country) without revealing the underlying identity data. Each bank's credentials are distinct — a liquidity VC from CRBank reflects different debt levels, risk profiles, and jurisdictional rules than one from MegaBank.

Tenant subdomains provide **branding, UX, and trust mapping** — easy to share, easy to remember, and resolvable by any other institution following the same spec. Users share the aliases they choose, and ecosystem accounts can transact, send, receive, and resolve to external domains — even cross-chain domains. Tier 1/2 users may not be aware of the on-chain layer at all.

## Trust Model

Trust is established by the **root domain holder having a DID in its root**, issued by a recognized authority. The authority may be a platform operator, a standards body (`identity.sol`, `w3c.sol`), or any entity with proper governance (e.g., multisig wallet holding the root domain). Root domain holders can delegate management to a platform that enforces the spec, or implement it independently.

**External domains** (e.g., from the SNS marketplace) that are not delegated to or managed by a compliant operator carry no inherent trust record. However, external domain owners can use any compliant operator (e.g., Attestto) to perform KYC/KYB and issue a DID to their domains or subdomains, without having to delegate the domain itself. Any operator can resolve and validate the DID presented from an external domain, but each operator and tenant reserves the right to set its own allowed features, compliance risk levels, and acceptance rules based on local regulations.

> **Note:** Domain delegation mechanics (how a domain owner delegates management to an operator) are outside the scope of this specification. SNS provides native domain delegation; each operator implements its own delegation flow, business rules, and monetization model.

## Identity Tiers

| Tier | Name | Who | Blockchain | UX | Capabilities |
|---|---|---|---|---|---|
| 1 | Web2 Native | Persons | Anchored (invisible) | No Web3 surface — user unaware of on-chain layer | Platform-managed identity, alias sharing, cross-institution resolution |
| 2 | Web2 Anchored | Persons | Anchored (known) | User understands the model, uses platform custody | Tier 1 + attestation anchoring, verifiable credentials, platform-managed keys |
| 3 | Full Web3 / SSI | Persons only | Anchored (self-custodial) | Full Web3 — mints, exports DID to own wallet | Tier 2 + self-custodial wallets, DAO governance, multisig, Web3 ecosystem access on top of Web2 |
| Org | Business / Issuer | Legal entities | Anchored | Root domain holder | Subdomain provisioning, SAS attestation authority. **MUST have active, non-expired LEI.** |

> **LEI Requirement for Organizations:** All business accounts and recognized issuers MUST attach a valid, active LEI (Legal Entity Identifier) to their root domain DID via GLEIF vLEI bridge attestation. Expired or inactive LEIs invalidate the issuer's trust chain. Third-party verifiers can query GLEIF APIs to obtain additional proofs, entity status, and risk/trust assessments independently.

## Requirements Derived from This Use Case

| # | Requirement | W3C Features Used | did:sns Mechanism |
|---|---|---|---|
| R1 | **Multi-issuer namespace control** — each institution owns its root domain and issues subdomains independently; any root domain holder can delegate management to a platform | Delegation of control, Decentralized | SNS hierarchical subdomains; `controller` field points to root; delegation is open to any domain holder |
| R2 | **Identity survives key rotation** — wallet changes, device loss, or custodial migration must not break any of the user's DIDs | Key rotation, Survives provider relationship | Alias-anchored design; SNS domain transfer updates owner key without changing DID identifier |
| R3 | **Issuer-signed proofs, not self-asserted** — each issuer independently signs attestations with its own keys; different issuers may use different signature methods | Authentication, Associated crypto, Legally-enabled | SAS attestations signed by issuer's keypair; bidirectional SNS↔SAS linking prevents spoofing |
| R4 | **Selective disclosure per presentation** — share only the fields required for a specific verification (e.g., "KYC Level 2" without revealing date of birth) | Privacy preserving, Limit trackable data trails | SD-JWT per-field disclosure with individual salt+hash; verifier sees only requested claims |
| R5 | **No PII on the public ledger** — financial regulation (GDPR, etc.) prohibits storing personal data on a public blockchain | Privacy preserving | On-chain stores only hashes and pointers; all PII in encrypted vaults with 2-of-2 XOR key-split |
| R6 | **Cross-border, cross-institution verification** — a credential issued by a Costa Rican bank must be verifiable by a European fintech without bilateral integration | Inter-jurisdictional, No call home, No vendor lock-in | Universal Resolver + standard SD-JWT + public status lists; verifier needs no proprietary software or Solana tooling |
| R7 | **Real-time credential state** — when a client's compliance status changes (KYC upgrade, suspension, LEI renewal), the identity must reflect it immediately without reissuing the DID | Service endpoint discovery, Key rotation | "Living DID" architecture — SAS attestation updates in place; SNS buffer pointer stays current; no credential reissuance |
| R8 | **Organizational identity with legal standing** — each issuing institution needs its own verifiable DID (root domain with LEI/vLEI) that anchors trust for all subdomains it issues | Legally-enabled, Authentication, Human-centered interop | Root domains as organizational DIDs; GLEIF vLEI bridge maps LEI credentials to SAS attestations |
| R9 | **Audit trail without log bloat** — regulators need full history but the system must not accumulate unbounded data per identity | Crypto future-proof, Sustainability | Stateless 160-byte buffer (O(1) resolution); full audit via Solana transaction history, not a DID Log |
| R10 | **Post-quantum resilience** — financial identities have 10–30 year lifespans; they must survive the cryptographic transition | Crypto future-proof | ML-DSA-44 / ML-KEM-768 hybrid mode defined; PQ keys stored in SAS layer, referenced by hash in fixed buffer |
| R11 | **Permanent identity binding** — once a DID is issued via KYC/KYB, the identity record is permanently bound to the subject; only access and capabilities can expire, not the identity itself | Can't be admin denied, Sustainability | DID remains resolvable and auditable even after credential suspension; closing a bank account deactivates the tenant alias but the identity binding persists |
| R12 | **Privacy-preserving transfers** — payment flows must be untraceable to unauthorized third parties while remaining transparent to authorized auditors and regulators | Privacy preserving, Limit trackable data trails | Off-chain privacy layer beyond Solana's confidential transfer protocol; compliant operators provide authorized auditor access |
| R13 | **Open specification** — any third party can implement the spec independently using their own domains, producing the same trust guarantees without depending on any single platform | No vendor lock-in, Registry agnostic, Decentralized | Open-source spec and reference implementation (MIT-licensed); third-party implementers use their own root domains and enforce the same opinionated flows |

> **Why these requirements converge:** The `did:sns` specification was not designed to check off a requirements list. The philosophy — eliminate friction, guarantee interoperability, protect user data, and remain open — naturally produced an architecture that satisfies all thirteen requirements simultaneously: multi-issuer control (R1, R3, R8) without sacrificing user portability (R2, R6), privacy compliance (R4, R5, R12) without losing auditability (R9), permanent identity binding (R11) with regulatory suspension capability, real-time state (R7) without log bloat (R9) or quantum vulnerability (R10), and an open standard (R13) that any institution can implement independently.

---

[← §1 Abstract](01-abstract.md) | [Next: §3 Trust Model & Hierarchy →](03-trust-model.md)
