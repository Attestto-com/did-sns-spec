# did:sns — DID Method Specification

**W3C Decentralized Identifier Method for Solana Name Service**

The `did:sns` method binds [W3C Decentralized Identifiers](https://www.w3.org/TR/did-core/) to human-readable [Solana Name Service](https://sns.id) (SNS) domains — making identities portable, resolvable, and bankable.

```
did:sns:attestto.sol              # Platform root domain
did:sns:alice.attestto.sol        # Platform-issued subdomain
did:sns:crbank.sol                # Tenant root domain
did:sns:alice.crbank.sol          # Tenant client subdomain
did:sns:devnet:alice.attestto.sol # Devnet network qualifier
```

## Specification

| Document | URL |
|---|---|
| Method Specification | [spec.attestto.com/did-sns](https://spec.attestto.com/did-sns) |
| Implementation Report | [spec.attestto.com/did-sns/report](https://spec.attestto.com/did-sns/report) |
| Community Charter | [spec.attestto.com/did-sns/charter](https://spec.attestto.com/did-sns/charter) |
| JSON-LD Context | [spec.attestto.com/v1/sns.jsonld](https://spec.attestto.com/v1/sns.jsonld) |
| Credential Schemas | [spec.attestto.com/credentials](https://spec.attestto.com/did-sns/credentials/) |
| W3C Registry Entry | [sns.json](./sns.json) |

## Status

| Field | Value |
|---|---|
| Registry Status | Provisional |
| Specification Version | v0.3.0 |
| Implementations | 1 (Attestto resolver) |
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

- **Name-anchored identity** — human-readable `.sol` domains as DID identifiers
- **Controller hierarchy** — tenant root domains issue client subdomains with cryptographic proof of issuance
- **SAS integration** — Solana Attestation Service for issuer-signed on-chain claims (vLEI, KYC, compliance)
- **No PII on-chain** — all personal data in encrypted Data Vaults; on-chain stores only SHA-256 commitments
- **Selective disclosure** — SD-JWT per-field disclosure with salted hashes
- **W3C conformant** — DID Core v1.0, DID Resolution v0.3, JSON-LD context

## Contributing

See the [Community Charter](https://spec.attestto.com/did-sns/charter) for governance, decision process, and how to contribute.

- **Implement a resolver** in your language/framework
- **Propose extensions** via GitHub issues
- **Report gaps** for regulatory compliance in your jurisdiction

## Contact

| Channel | Address |
|---|---|
| Standards Group | [standards@attestto.com](mailto:standards@attestto.com) |
| Website | [attestto.com](https://attestto.com) |

## License

Published under the [W3C Software and Document License](https://www.w3.org/Consortium/Legal/2015/copyright-software-and-document).
