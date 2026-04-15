# 4. Architectural Rationale

## 4.1 Why Aliases Over Account Numbers

Traditional financial identifiers — IBANs, SWIFT codes, routing numbers, wallet addresses — are opaque strings designed for machines. They are error-prone to share, impossible to remember, and carry no inherent trust information. SNS domains replace these with human-readable aliases:

| Traditional | `did:sns` Equivalent | UX |
|---|---|---|
| `CR05015201001027777777` (IBAN) | `alice.crbank.sol` | Shareable, memorable, resolvable |
| `BNCRCRSJ` (SWIFT) | `crbank.sol` | Institution identity, not just a routing code |
| `7nYB...3kPo` (Solana wallet) | `alice.platform.sol` | Human name, not a hash |
| `0x1a2b...9f8e` (Ethereum wallet) | `alice.chongkan.sol` | Human name, not a hash |

Any Web3 wallet that supports SNS resolution can send assets (stablecoins, tokens) directly to the domain's configured wallet address — no need to copy-paste public keys. The alias resolves to the wallet; the user shares a name, not a key.

> *The alias-anchored architecture described here applies equally to any blockchain name service. An Ethereum-based implementation using ENS would produce `did:ens:alice.crbank.eth` with the same trust model, privacy guarantees, and interoperability. See [§1 Abstract](01-abstract.md) — Cross-chain portability.*

### ISO 20022 Compliance

The global financial messaging infrastructure is migrating to ISO 20022 (SWIFT MT → MX, SEPA, FedNow, BIS Nexus). `did:sns` alias resolution produces structured identity data that maps directly to ISO 20022 fields:

| ISO 20022 Field | `did:sns` Source |
|---|---|
| `Dbtr/Nm` (Debtor Name) | Alias (`alice.crbank.sol`) or legal name from SAS attestation |
| `Dbtr/Id/OrgId/LEI` | LEI from Issuer DID's vLEI attestation |
| `Dbtr/Id/PrvtId` | KYC attestation UID (verifiable, not raw PII) |
| `CdtrAgt/FinInstnId/LEI` | Receiving institution's root domain LEI |
| `RgltryRptg` | SAS attestation chain provides FATF Travel Rule data |
| `PmtId/EndToEndId` | Transaction reference resolvable via DID service endpoint |

> [!IMPORTANT]
> **ISO 20022 mapping is a compliance layer built on top of `did:sns`, not embedded in the method.** The DID Document and SAS attestations provide the structured identity data. Mapping that data to ISO 20022 message fields is the responsibility of the platform or financial institution — the method provides the hooks (service endpoints, attestation fields), not the compliance logic itself.

> *This applies to any alias-anchored DID method following this spec (`did:ens`, etc.). The ISO 20022 mapping is method-agnostic — it depends on the attestation schema, not the underlying blockchain.*

### Payment Rails Comparison

| Traditional | Speed | Cost | `did:sns` Equivalent |
|---|---|---|---|
| ACH (US domestic) | 1–3 business days | $0.20–$1.50 | `alice.platform.sol` → seconds, near-zero |
| SEPA (EU domestic) | 1 business day | €0.20–€0.60 | `alice.crbank.sol` → seconds, near-zero |
| SWIFT (cross-border) | 1–5 business days | $15–$50 + intermediary fees | `alice.crbank.sol` → `bob.usbank.sol` → seconds, near-zero |
| BIS Nexus (multi-currency) | Real-time (where available) | Varies by corridor | Same alias, same resolution — local and cross-border identical |
| Wire transfer | Same day–2 days | $25–$50 | Replaced by alias resolution + stablecoin settlement |

> **What this enables:** Alias-anchored identity with compliance hooks opens payment corridors that previously required bilateral agreements and correspondent banking chains. Platforms and financial institutions build the compliance and settlement layers on top of `did:sns` resolution — the method provides the identity infrastructure, not the financial services themselves.

## 4.2 Trust Hierarchy Inspired by SSL Certificate Authorities

The Web's SSL/TLS trust model works because browsers trust a set of Certificate Authorities (CAs) who verify domain ownership. `did:sns` applies the same model to identity:

| SSL/TLS | `did:sns` |
|---|---|
| Root CA | Root domain holder with LEI + ecosystem trust credentials ([§3.4](03-trust-model.md#34-ecosystem-trust-anchoring)) |
| Intermediate CA | Platform or operator managing delegated domains |
| End-entity certificate | User's subdomain DID with KYC/KYB attestation |
| Certificate chain validation | Verifier walks subdomain → root → LEI → GLEIF |
| Certificate revocation (CRL/OCSP) | SAS attestation revocation + W3C Bitstring Status List |

Just as a browser trusts `https://alice.crbank.com` because it can verify the certificate chain up to a trusted CA, a verifier trusts `did:sns:alice.crbank` because it can verify the attestation chain up to a LEI-verified root domain.

> *This CA-inspired trust model is not Solana-specific. A `did:ens` implementation would use the same hierarchy: ENS root domain holder (LEI ✓) → subdomain user (KYC ✓) → verifier walks the chain.*

## 4.3 Separating Identity from Keys and Transactions

The core architectural decision: the **alias** (domain), the **keys** (cryptographic material), and the **transactions** (financial activity) are three independent layers:

| Layer | What | Where | Changes When |
|---|---|---|---|
| **Alias** | `alice.crbank.sol` | SNS domain registry | Never (permanent binding) |
| **Keys** | Ed25519, secp256k1, X509 | DID Document (derived from on-chain state) | Key rotation, tier upgrade, wallet migration |
| **Transactions** | Stablecoin transfers, credential presentations | Solana TX history, off-chain flows | Every interaction |

This separation provides **financial privacy**: even though the alias is public and resolvable, the wallet address it points to can be rotated, and the transaction layer operates independently. A verifier can confirm "this is Alice from CRBank" without seeing Alice's transaction history. A payment sender can resolve the alias to a wallet without learning Alice's KYC status.

> Traditional financial systems (SWIFT, IBAN) bind identity, routing, and transaction history into a single opaque code. Changing banks means changing your identifier. Compromised credentials mean compromised accounts. `did:sns` breaks this coupling — the alias stays, the keys rotate, the transactions are private.

## 4.4 The "Living DID"

Most blockchain-based DID methods couple *where data is found* with *how data is proved* into a single account. `did:sns` separates these:

| Layer | Role | Analogy |
|---|---|---|
| **SNS Domain** (`.sol`) | Public lookup registry — where you find the identity | The name on a bank folder |
| **SNS Data Buffer** (bytes 96+) | High-speed pointer to the proof | A sticker: "See Official Audit #88" |
| **SAS Attestation** (on-chain) | Issuer-signed, verifiable proof of claims | The stamped, notarized documents inside |

Traditional DIDs are static — if a user's KYC level changes, the entire credential must be reissued. With SAS as the proof layer, only the attestation updates. The `.sol` domain stays the same, but what it *proves* stays current in real-time.

### Resolution Architecture

```
Verifier / Client
    │
    │  did:sns:alice.crbank
    ▼
did:sns Resolver ──────── Solana RPC ──────── Solana
  1. Parse DID                                  SPL Name Service
  2. Derive PDA                                 NameRegistry accounts
  3. Build DID Document                         SAS attestations
    │
    ▼
DID Document
  id, controller
  verificationMethod[]
  authentication[]
  assertionMethod[]
  service[]
  snsMetadata (SAS)
```

The Universal Resolver driver (DIF) provides an HTTP bridge — any infrastructure running the [DIF Universal Resolver](https://dev.uniresolver.io/) can resolve `did:sns` identifiers out of the box.

---

[← §3 Trust Model](03-trust-model.md) | [Next: §5 Privacy Architecture →](05-privacy.md)
