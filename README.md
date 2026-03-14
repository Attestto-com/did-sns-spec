# did:sns — DID Method Specification

**Alias-anchored W3C Decentralized Identifiers for cross-border interoperability, privacy, and compliance**

The `did:sns` method binds [W3C Decentralized Identifiers](https://www.w3.org/TR/did-core/) to human-readable `.sol` domain aliases — enabling institutions, fintechs, and regulated entities to issue, verify, and present identity credentials across jurisdictions and blockchains. Web3 native, Web2 transparent — end users interact with readable aliases while the specification guarantees interoperability, selective disclosure, and regulatory compliance out of the box.

```
did:sns:alice.crbank.sol          # Tenant client subdomain
did:sns:crbank.sol                # Tenant root domain (organizational DID)
did:sns:alice.attestto.sol        # Platform-issued subdomain
did:sns:attestto.sol              # Platform root domain
did:sns:devnet:alice.attestto.sol # Devnet network qualifier
```

## Privacy by Design

Most blockchain-based DID methods expose the holder to correlation, surveillance, and data leakage — the DID itself becomes a tracking handle. `did:sns` takes the opposite approach: **the DID Document contains zero personal data**, and every layer of the architecture is engineered to minimize what a verifier, an on-chain observer, or even the platform itself can learn about the holder.

**7 privacy layers, working together:**

1. **No PII on-chain.** The Solana ledger stores only cryptographic commitments (SHA-256 hashes, public keys, attestation pointers). All personal data lives in encrypted Data Vaults protected by a 2-of-N Shamir key split — decryption requires a threshold of shares to combine. Neither the platform alone nor a single device can access the data.

2. **Selective disclosure via SD-JWT.** Credentials use per-field salted hashes (IETF SD-JWT). A holder can prove "I am over 18" or "my country is in the EU" without revealing their date of birth or exact nationality. The verifier receives only the claims they requested — nothing more.

3. **Pairwise identifiers.** Each verifier relationship gets a unique subdomain DID derived from `SHA-256(verifierDID + holderSecret)`. Two verifiers cannot collude to correlate the same holder — they each see a different DID, a different service endpoint, and a different presentation.

4. **Consent-gated proof access.** No credential is ever shared without explicit holder consent. The Vault Extension presents a field-by-field consent screen before assembling any Verifiable Presentation. Consent is logged, time-bounded, and revocable.

5. **Dual-key encryption (Vault architecture).** A per-user Vault Master Key (VMK) is split via 2-of-N Shamir secret sharing: shares are distributed across the user's devices, the platform, and optional guardians. Reconstruction requires a minimum threshold of shares — the platform alone cannot decrypt, a stolen device alone cannot decrypt, and social recovery is possible without a single point of failure.

6. **Crypto-shredding for right to erasure.** Deleting Share B renders all vault objects permanently inaccessible — the AES-256-GCM keys that protect individual objects can never be reconstructed. This satisfies GDPR Article 17 and Costa Rica Law 8968 without needing to locate and delete every copy of every credential.

7. **Post-quantum forward secrecy.** The spec defines a hybrid migration path: ML-DSA-44 (FIPS 204) signatures alongside Ed25519, and ML-KEM-768 key encapsulation alongside X25519. Today's credentials remain verifiable after quantum computers arrive — and today's encrypted vault data remains confidential.

**The result:** A holder's `did:sns` identity is publicly resolvable (verifiers can discover service endpoints and public keys), but the holder retains full control over *what* is disclosed, *to whom*, and *for how long*. The specification meets the privacy requirements of GDPR, Costa Rica Law 8968, and FATF Travel Rule simultaneously — proving that regulatory compliance and user privacy are not in conflict.

For the full privacy architecture, see [Section 5 — Privacy Architecture](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/05-privacy.md).

## Cross-Chain Interoperability

`did:sns` is not an island. The specification defines a **dual-DID architecture** where `did:sns` serves as the primary credential anchor while linked identities on other chains extend the holder's reach across ecosystems — without duplicating credentials or sacrificing the privacy guarantees above.

**How it works:**

The W3C `alsoKnownAs` property creates bidirectional links between a `did:sns` identity and identities on other chains. Both DID Documents must reference each other — a unilateral claim is rejected. This prevents identity hijacking if a linked name (e.g., an ENS domain) expires and is re-registered by someone else.

```
did:sns:alice.attestto.sol          ← Primary anchor (credentials, vault, consent)
  ├── alsoKnownAs: did:ens:alice.eth        ← Ethereum ecosystem reach
  ├── alsoKnownAs: did:pkh:solana:CKg5...   ← Universal key-proof layer
  └── alsoKnownAs: did:web:alice.com         ← Traditional web binding
```

**Supported cross-chain DID methods:**

| Linked Method | Chain | Link Mechanism | What It Adds |
|---|---|---|---|
| [`did:ens`](https://github.com/veramolabs/did-ens-spec) | Ethereum | ENS TEXT record `org.w3c.did.alsoKnownAs` | Ethereum ecosystem identity, Safe multi-sig control, ENS subdomain pairwise DIDs |
| [`did:pkh`](https://github.com/w3c-ccg/did-pkh) | Any (generative) | Companion registry or verifier hints | Universal blockchain address → DID wrapper, same Ed25519 key backing |
| `did:web` | DNS | `.well-known/did-configuration.json` | Traditional web domain binding |
| `did:sol` | Solana | `equivalentId` (same key material) | Direct Solana account-anchored equivalence |

**Cross-chain verification flow:**

1. Resolve `did:sns` (primary) → extract `alsoKnownAs` entries
2. Resolve each linked DID → confirm it references the `did:sns` back (bidirectional check)
3. For lease-based methods (ENS): check name expiry — expired links are flagged, grace-period links are degraded
4. Verify the credential's SD-JWT signature against the **issuing DID's key** (`did:sns`), not the linked identity's key
5. The linked identity proves ecosystem membership; the `did:sns` identity proves the credential

**Design principles:**

- **Credentials never leave `did:sns`.** A credential issued to `did:sns:alice.attestto.sol` is presented via the linked identity but verified against the primary anchor. If the linked identity is compromised, the credential remains valid — only the cross-chain link is severed.
- **Pairwise isolation crosses chains.** When presenting via `did:ens`, a pairwise ENS subdomain (`<hash>.alice.eth`) is generated per verifier — the same anti-correlation guarantee as native `did:sns` pairwise subdomains.
- **No chain dependency for security.** Vault encryption, consent gating, and revocation all anchor to Solana via `did:sns`. Linked chains provide reach, not trust.

We actively contribute upstream to align cross-chain specs with this architecture — see PRs to [did:ens](https://github.com/veramolabs/did-ens-spec/pulls?q=author%3Achongkan) and [did:pkh](https://github.com/w3c-ccg/did-pkh/pulls?q=author%3Achongkan), tracked at [w3c/did-extensions#680](https://github.com/w3c/did-extensions/issues/680).

For the full interoperability specification, see [Section 13 — Interoperability](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/13-interoperability.md).

## Specification

The spec is organized in 14 sections — human-readable context first, standards coverage in the middle, deep technical detail at the bottom.

| # | Section | Description |
|---|---------|-------------|
| 1 | [Abstract](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/01-abstract.md) | What is `did:sns` — alias-anchored, Web3 native, Web2 transparent, chain-replicable |
| 2 | [Focal Use Case & Identity Tiers](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/02-focal-use-case.md) | Multi-issuer regulated identity, Tier 1/2/3/Org, 13 requirements (R1–R13) |
| 3 | [Trust Model & Hierarchy](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/03-trust-model.md) | Models A–D, cross-domain trust, ecosystem trust anchoring & A/B/C grading |
| 4 | [Architectural Rationale](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/04-architectural-rationale.md) | Aliases vs IBANs/SWIFT, SSL CA-inspired trust, ISO 20022, the Living DID |
| 5 | [Privacy Architecture](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/05-privacy.md) | 7 privacy layers, correlation mitigations, regulatory compliance mapping |
| 6 | [W3C Coverage](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/06-w3c-coverage.md) | 21/22 features covered — feature matrix, benefit alignment grid |
| 7 | [DID Syntax](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/07-did-syntax.md) | ABNF grammar, hierarchy depth, naming strategies |
| 8 | [DID Document](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/08-did-document.md) | Examples (Tier 1/2, Tier 3, Root Domain), verification methods, services |
| 9 | [CRUD Operations](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/09-crud-operations.md) | Create, Resolve, Update, Deactivate, DID Lifecycle, Root Domains as Org DIDs |
| 10 | [Metadata Schema](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/10-metadata-schema.md) | 160-byte v2 buffer layout, flags bitfield |
| 11 | [SAS Integration](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/11-sas-integration.md) | SNS↔SAS linking, attestation hierarchy, class key security model |
| 12 | [Security](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/12-security.md) | Post-quantum migration, vault key architecture, stateless resolution, schema migration |
| 13 | [Interoperability](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/13-interoperability.md) | Universal Resolver, SD-JWT, Credo-TS, DIDComm v2, vLEI, DNS discovery |
| 14 | [References](https://github.com/Attestto-com/did-sns-spec/blob/main/did-sns/spec/14-references.md) | W3C, IETF, NIST, Solana, GLEIF, ISO standards |

**Landing page:** [spec.attestto.com/did-sns](https://spec.attestto.com/did-sns)

## Resources

| Document | URL |
|---|---|
| Implementation Report | [spec.attestto.com/did-sns/report](https://spec.attestto.com/did-sns/report) |
| Community Charter | [spec.attestto.com/did-sns/charter](https://spec.attestto.com/did-sns/charter) |
| IP Affirmation | [spec.attestto.com/did-sns/ip-affirmation](https://spec.attestto.com/did-sns/ip-affirmation) |
| JSON-LD Context | [spec.attestto.com/v1/sns.jsonld](https://spec.attestto.com/v1/sns.jsonld) |
| Credential Schemas | [spec.attestto.com/did-sns/credentials](https://spec.attestto.com/did-sns/credentials/) |
| W3C Registry Entry | [sns.json](./sns.json) |
| Reference Implementation | [`@attestto/did-sns-resolver`](https://github.com/Attestto-com/did-sns-resolver) |
| npm Package | [`@attestto/did-sns-resolver`](https://www.npmjs.com/package/@attestto/did-sns-resolver) |

## Status

| Field | Value |
|---|---|
| Registry Status | Provisional |
| Specification Version | v0.4.0 |
| Implementations | 1 ([`@attestto/did-sns-resolver`](https://github.com/Attestto-com/did-sns-resolver)) |
| Test Coverage | 186 tests, 0 failures |
| Verifiable Data Registry | Solana (SPL Name Service + SAS) |

## Financial Credential Schemas

JSON-LD contexts for W3C Verifiable Credentials in financial compliance workflows, designed for use with `did:sns` identities and SD-JWT selective disclosure.

| Schema | Context URL | Regulatory Alignment |
|---|---|---|
| Identity Verification (KYC) | [`/credentials/v1/identity-verification.jsonld`](https://spec.attestto.com/did-sns/credentials/v1/identity-verification.jsonld) | FATF R.10, EU AMLD6 Art. 13, CR Law 8204 Art. 15 |
| Source of Funds (SoF) | [`/credentials/v1/source-of-funds.jsonld`](https://spec.attestto.com/did-sns/credentials/v1/source-of-funds.jsonld) | FATF R.10, EU AMLD6 Art. 13, CR Law 8204 Art. 16 |
| Proof of Address (PoA) | [`/credentials/v1/proof-of-address.jsonld`](https://spec.attestto.com/did-sns/credentials/v1/proof-of-address.jsonld) | FATF R.10, EU AMLD6 Art. 13(1)(a), CR SUGEF 12-21 |
| EDD Clearance | [`/credentials/v1/edd-clearance.jsonld`](https://spec.attestto.com/did-sns/credentials/v1/edd-clearance.jsonld) | FATF R.19, EU AMLD6 Art. 18, CR Law 8204 Art. 17 |

Each schema includes a selective disclosure companion type — verifiers can confirm a user meets a financial threshold or holds a valid EDD clearance without seeing the underlying documents or exact amounts.

## W3C Requirements Coverage

`did:sns` covers **21 of 22** W3C DID requirements ([full analysis](did-sns/spec/06-w3c-coverage.md)). Requirements are classified into tiers based on how many of the [18 W3C use cases](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#uc) reference them. Structured data: [`did-sns/data/`](did-sns/data/).

### Core — expected of any serious DID method

| ID | Requirement | did:sns |
|---|---|---|
| R1 | Authentication / Proof of Control | Covered |
| R22 | Human-Centered Interoperability | Covered |

### Common — important for most implementations

| ID | Requirement | did:sns |
|---|---|---|
| R2 | Decentralized / Self-Issued | Covered |
| R3 | Guaranteed Unique Identifier | Covered |
| R5 | Associated Cryptographic Material | Covered |
| R7 | Service Endpoint Discovery | Covered |
| R8 | Privacy Preserving | Covered |
| R10 | Inter-Jurisdictional | Covered |
| R13 | No Vendor Lock-In | Covered |
| R18 | Survives Relationship with Service Provider | Covered |

### Specialized — needed for specific domains

| ID | Requirement | did:sns |
|---|---|---|
| R4 | No Call Home | Covered |
| R6 | Streamlined Key Rotation | Covered |
| R9 | Delegation of Control | Covered |
| R11 | Cannot Be Administratively Denied | Covered |
| R12 | Minimized Rents | Covered |
| R14 | Preempt / Limit Trackable Data Trails | Covered |
| R15 | Cryptographic Future-Proof | Covered |
| R16 | Survives Issuing Organization Mortality | Covered |
| R17 | Survives Deployment End-of-Life | Covered |
| R19 | Cryptographic Authentication & Communication | Covered |
| R20 | Registry Agnostic | **Partial** |
| R21 | Legally-Enabled Identity | Covered |

> R20 (Registry Agnostic) is partial — `did:sns` is bound to Solana/SNS, but the architecture is chain-replicable and consumers are registry-agnostic via the Universal Resolver driver.

## Use Case Alignment

`did:sns` maps to the following [W3C focal use cases](https://www.w3.org/TR/2021/NOTE-did-use-cases-20210317/#uc). Structured data: [`did-sns/data/use-cases.json`](did-sns/data/use-cases.json).

| Use Case | Category | Requirements | did:sns Coverage |
|---|---|---|---|
| Enterprise Identifiers | Finance & Commerce | 17 reqs | 17/17 (100%) |
| Life-long Credentials | Education & Workforce | 19 reqs | 19/19 (100%) |
| Prescriptions | Healthcare | 9 reqs | 9/9 (100%) |
| Digital Executor | Government & Legal | 18 reqs | 18/18 (100%) |
| Portable Credentials | Finance & Commerce | 6 reqs | 6/6 (100%) |
| Secure Communication | Privacy & Communication | 15 reqs | 14/15 (93%) |

Additionally supports non-focal use cases: Online Shopper, Vehicle Assemblies, Confidential Customer Engagement, eIDAS Public Authority Credentials, Digital Permanent Resident Card, and 7 others. See [`did-sns/data/use-cases.json`](did-sns/data/use-cases.json) for the full list.

## Key Features

- **Alias-anchored identity** — human-readable `.sol` domains as DID identifiers
- **Web3 native, Web2 transparent** — blockchain infrastructure invisible to end users at Tier 1/2
- **Controller hierarchy** — root domains issue client subdomains with cryptographic proof of issuance
- **SAS integration** — Solana Attestation Service for issuer-signed on-chain claims (vLEI, KYC, compliance)
- **No PII on-chain** — all personal data in encrypted Data Vaults; on-chain stores only SHA-256 commitments
- **Selective disclosure** — SD-JWT per-field disclosure with salted hashes
- **Post-quantum ready** — ML-DSA-44 / ML-KEM-768 hybrid migration path defined
- **Chain-replicable** — architecture portable to ENS (Ethereum), Unstoppable Domains, and other naming services
- **W3C conformant** — DID Core v1.1, DID Resolution v0.3, JSON-LD context

## Contributing

See the [Community Charter](https://spec.attestto.com/did-sns/charter) for governance, decision process, and how to contribute.

- **Implement a resolver** in your language/framework
- **Propose extensions** via GitHub issues
- **Discuss cross-chain compatibility** in [GitHub Discussions](https://github.com/Attestto-com/did-sns-spec/discussions)
- **Report gaps** for regulatory compliance in your jurisdiction

## Contact

| Channel | Address |
|---|---|
| Author | Eduardo Chongkan ([@chongkan](https://github.com/chongkan)) |
| Standards Group | [standards@attestto.com](mailto:standards@attestto.com) |
| Website | [attestto.com](https://attestto.com) |

## License

Published under the [W3C Software and Document License](https://www.w3.org/Consortium/Legal/2015/copyright-software-and-document).
