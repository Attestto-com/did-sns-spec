# did:sns — DID Method Specification

**Alias-anchored W3C Decentralized Identifiers for Solana Name Service**

The `did:sns` method binds [W3C Decentralized Identifiers](https://www.w3.org/TR/did-core/) to human-readable [Solana Name Service](https://sns.id) (SNS) domains. Web3 native, Web2 transparent — end users interact with readable aliases while the specification guarantees interoperability, privacy, and compliance across institutions, jurisdictions, and blockchains.

```
did:sns:alice.crbank.sol          # Tenant client subdomain
did:sns:crbank.sol                # Tenant root domain (organizational DID)
did:sns:alice.attestto.sol        # Platform-issued subdomain
did:sns:attestto.sol              # Platform root domain
did:sns:devnet:alice.attestto.sol # Devnet network qualifier
```

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
| Source of Funds (SoF) | [`/credentials/v1/source-of-funds.jsonld`](https://spec.attestto.com/did-sns/credentials/v1/source-of-funds.jsonld) | FATF R.10, EU AMLD6 Art. 13, CR Law 8204 Art. 16 |
| Proof of Address (PoA) | [`/credentials/v1/proof-of-address.jsonld`](https://spec.attestto.com/did-sns/credentials/v1/proof-of-address.jsonld) | FATF R.10, EU AMLD6 Art. 13(1)(a), CR SUGEF 12-21 |
| EDD Clearance | [`/credentials/v1/edd-clearance.jsonld`](https://spec.attestto.com/did-sns/credentials/v1/edd-clearance.jsonld) | FATF R.19, EU AMLD6 Art. 18, CR Law 8204 Art. 17 |

Each schema includes a selective disclosure companion type — verifiers can confirm a user meets a financial threshold or holds a valid EDD clearance without seeing the underlying documents or exact amounts.

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
