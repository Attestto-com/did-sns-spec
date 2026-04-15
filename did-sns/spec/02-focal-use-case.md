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

Tenant subdomains provide **branding, UX, and trust mapping** — easy to share, easy to remember, and resolvable by any other institution following the same spec. Users share the aliases they choose. Tier 1/2 users may not be aware of the on-chain layer at all.

## Use Case: Stablecoin Payments via Human-Readable Aliases

A user holds stablecoin accounts across multiple institutions. Instead of sharing unfriendly blockchain addresses, IBANs, or routing/account numbers, the user shares human-readable aliases:

```
Traditional:
  "Send $500 to CR05015201001027777777"              (IBAN — 22 chars, error-prone)
  "Send $500 to 7Xf3kP9qR2vN8mT4..."                (Solana address — 44 chars, unreadable)
  "Send $500 to routing 021000021, account 123456789" (US — two numbers, easily confused)

With did:sns:
  "Send $500 to bob.bank1.sol"                        (human-readable, per-institution)
```

Bob holds separate aliases per institution:

- `bob.bank1.sol` → Bank 1 stablecoin wallet (regulated, KYC'd)
- `bob.bank2.sol` → Bank 2 stablecoin wallet (different jurisdiction)
- `bob.personal.sol` → Personal self-custodial wallet

Each alias resolves to a different wallet address. The wallet address is never shared directly — it lives behind the alias, managed by the institution. Bob shares the alias he chooses per context, and the counterparty resolves it at transaction time to get the current receiving address.

### Compliance Layer (separate from did:sns, attached via hooks)

`did:sns` defines the identity layer. Payment compliance operates on top of it — the spec does not embed compliance logic, but defines where it attaches:

**Pre-transaction compliance flow:**

> [!IMPORTANT]
> **The alias resolves to the platform's receiving address, never to the user's personal wallet.** Bob's actual wallet is internal to the platform — it is never exposed on-chain and never shared with the counterparty. The platform handles internal routing as a ledger operation.
>
> **Compliance screening checks the SENDER first** — before accepting any funds into the platform wallet. Tainted funds must be rejected before they touch the platform's infrastructure, not after.

```
1. Sender initiates: "Send $500 USDC to bob.bank1.sol"

2. Receiver's platform resolves the sender's identity:
   - Check the SENDER's wallet address against Circle's USDC blacklist (frozen addresses)
   - Check against OFAC SDN list, EU sanctions list, local AML watchlists
   - Check the sender's institution LEI status via GLEIF API (if institutional sender)
   - If any check fails → transaction REJECTED before funds touch platform wallet

3. Sender's platform resolves did:sns:bob.bank1 →
   - Gets DID Document
   - Receives the PLATFORM's receiving address (not Bob's personal wallet)
   - Checks SAS attestation: jurisdiction, regulatory status, LEI of receiving institution
   - Checks service endpoint: "TravelRuleService" → URL for compliance data exchange

4. Travel Rule exchange (if threshold exceeded):
   - Sender's platform sends originator info to receiver's TravelRuleService endpoint
   - Receiver's platform validates and acknowledges
   - Both platforms retain records per jurisdictional requirements

5. Transaction executes:
   - Stablecoin transfer from sender → platform's receiving address
   - Platform internally credits Bob's account (ledger operation, not on-chain)
   - Bob's personal wallet address is never exposed to the sender or the blockchain
   - Bob's alias remains permanent regardless of internal wallet changes
```

**What did:sns provides for this flow:**

| Component | Source | Purpose |
|---|---|---|
| Human-readable destination | `bob.bank1.sol` (alias) | UX — no addresses shared |
| Platform receiving address | DID Document → service endpoint (never the user's personal wallet) | Transaction routing to platform |
| Receiving institution identity | SAS attestation → LEI hash | Institutional verification |
| Travel Rule endpoint | DID Document → service: `TravelRuleService` | Compliance data exchange |
| ISO 20022 party mapping | DID Document → service: `ISO20022PartyId` | Financial messaging interop |
| Sender screening input | Sender's wallet address (checked BEFORE accepting funds) | Check against Circle blacklist, OFAC, EU lists |

**What did:sns does NOT embed (separate compliance layer):**

- FATF Travel Rule logic (jurisdiction-specific, evolves independently)
- AML/KYC verification procedures (institution-specific)
- Sanctions list maintenance (OFAC, EU, national lists)
- Circle USDC blacklist checking (Circle's API, changes in real-time)
- ISO 20022 message formatting (financial messaging standard, maintained by SWIFT/ISO)
- Transaction monitoring and suspicious activity reporting

This separation ensures the DID spec doesn't need to change when compliance rules change. The compliance layer reads from the identity layer and attaches its own logic.

### An Alternative to Traditional Payment Rails

When used as recommended — alias-to-platform resolution, sender screening before acceptance, Travel Rule via service endpoints, institutional LEI verification — this flow provides a **compliant alternative to traditional correspondent banking and SWIFT messaging** for stablecoin-denominated cross-border payments:

| | SWIFT / Correspondent Banking | did:sns + Stablecoins (as recommended) |
|---|---|---|
| **Destination identifier** | IBAN + BIC/SWIFT code | `bob.bank1.sol` (human-readable alias) |
| **Routing** | Correspondent chain (2-5 intermediaries) | Direct: sender platform → receiver platform (1 hop) |
| **Settlement** | T+1 to T+3 (days) | Near-instant (Solana finality ~400ms) |
| **Cost** | $15–$50+ per transfer (intermediary fees) | Stablecoin transfer fee (<$0.01) + platform fees |
| **Sender screening** | Each intermediary screens independently (redundant) | Sender screened once BEFORE funds leave — platform wallet stays clean |
| **Travel Rule** | SWIFT MT103/MT202 fields | Service endpoint exchange (same data, modern transport) |
| **Institutional identity** | BIC code (opaque, no cryptographic verification) | LEI + SAS attestation (cryptographically verifiable via GLEIF API) |
| **Recipient wallet exposure** | Account number shared with all intermediaries | Never exposed — alias resolves to platform, internal routing only |
| **Currency conversion** | FX conversion at each intermediary hop (spread + fees compound) | Single conversion at entry/exit; stablecoin travels the rail without intermediate FX |
| **Audit trail** | Fragmented across intermediaries | On-chain (immutable) + Travel Rule logs (both platforms) |
| **Hours of operation** | Banking hours, cut-off times, weekends | 24/7/365 |
| **ISO 20022 compatibility** | Native (migrating from MT to MX) | Mapped via service endpoint — DID Document carries ISO 20022 party identification |

> [!NOTE]
> **This is not a theoretical comparison.** Stablecoin rails (Circle USDC, EURC) already process billions in cross-border value. What they lack is a standardized identity layer that satisfies compliance requirements without exposing user wallet addresses. `did:sns` provides that layer — human-readable aliases, institutional verification, Travel Rule hooks, and sender screening — while keeping the user's actual wallet private and the platform's wallet clean.

### Why This Matters

The combination of human-readable aliases + compliance hooks means:

1. **Users** share `bob.bank1.sol` instead of wallet addresses or IBANs — simpler, memorable, works across borders. Their actual wallet is never exposed.
2. **Platforms** screen the sender's wallet BEFORE accepting funds — tainted money never touches the platform infrastructure. Then resolve the receiving alias to get institution identity, Travel Rule endpoint, and LEI from a single DID resolution.
3. **Regulators** get the audit trail they require — the DID is permanent, the SAS attestations are on-chain, the Travel Rule exchange is logged by both platforms.
4. **Bad actors** are caught at step 2 — the sender's wallet is checked against Circle's USDC blacklist, OFAC, and sanctions lists BEFORE any funds move. The platform wallet stays clean.

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
| R5 | **No PII on the public ledger** — financial regulation (GDPR, etc.) prohibits storing personal data on a public blockchain | Privacy preserving | On-chain stores only hashes and pointers; PII storage is the responsibility of the platform/issuer (not defined by the method) |
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
