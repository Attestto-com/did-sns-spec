# 3. Trust Model & Hierarchy

Trust in `did:sns` is established by **who holds the root domain and what attestations they carry**, not by any central registry. A root domain holder's trust level is determined by their LEI status, SAS attestations, and governance model. Subdomains inherit the trust context of their parent — a `did:sns` resolved under a verified bank's root carries the bank's institutional trust.

## 3.1 Trust Principles

1. **Root domain = trust anchor** — the root domain holder's DID, LEI, and SAS attestations establish the trust floor for all subdomains
2. **Trust is continuous** — not a one-time check; LEI status, attestation validity, and compliance standing are verified on every resolution
3. **Trust is contextual** — each operator and verifier sets its own acceptance rules, risk thresholds, and compliance requirements for external DIDs
4. **Trust is composable** — multiple independent attestations (KYC, LEI, vLEI, SAS) stack to build a trust profile

## 3.2 Hierarchy Models

### Model A — Institutional Root Domain (Recommended for Banks & Regulated Entities)

```
crbank.sol                          → Institutional root (LEI required, self-sovereign)
  ├── alice.crbank.sol              → End user (KYC by CRBank)
  ├── bob.crbank.sol                → End user (KYC by CRBank)
  └── ops.crbank.sol                → Service/department

Trust chain: alice → CRBank (LEI ✓) → GLEIF
Verifier resolves alice.crbank.sol → checks CRBank's LEI is active → trusts.
```

### Model B — Platform Root (Multi-Purpose)

```
platform.sol                        → Platform authority (LEI required)
  ├── alice.platform.sol            → Direct platform user (KYC by platform)
  ├── a7f3.platform.sol             → Pseudonymous user (KYC by platform, alias is user's choice)
  ├── payments.platform.sol         → Reserved subdomain (service endpoint)

sinpe.sol                           → Platform-held domain (country-specific, e.g., Costa Rica)
  └── alice.sinpe.sol               → User alias under country brand

chongkan.sol                        → Delegated domain (owner delegated management to platform)
  └── alice.chongkan.sol            → User under delegated namespace

school.sol                          → Delegated domain (educational institution)
  └── diploma.school.sol            → Credential endpoint

pet-shop.sol                        → Delegated domain (creative use cases)
  ├── max.pet-shop.sol              → Animal DID (pet identity, vaccination records)
  └── usd.pet-shop.sol              → Currency address (payment endpoint)

Trust chain varies: platform users → Platform (LEI ✓);
delegated domains → delegation agreement + platform attestation.
```

### Model C — Third-Party Spec Implementer

```
otherPlatform.sol                   → Independent operator (their own LEI, their own infrastructure)
  ├── carol.otherPlatform.sol       → End user (KYC by otherPlatform)
  └── dave.otherPlatform.sol        → End user

usBank.sol                          → US-based institution implementing did:sns independently
  └── eve.usBank.sol                → End user (KYC by usBank)

Trust chain: carol → otherPlatform (LEI ✓) → GLEIF
No dependency on any specific platform. Same spec, same verification flow, different operator.
```

### Model D — Scammers / Bad Actors

```
fakePlatform.sol                    → Purchased domain, no LEI, no valid attestations
  └── alice.fakePlatform.sol        → Claims to be "alice" — no issuer-signed KYC

Trust chain: BROKEN — fakePlatform has no LEI, no SAS attestation from a recognized authority.
Verifier resolves → no active LEI → no valid org attestation → REJECTED by any compliant operator.
The DID exists on-chain (anyone can register a .sol domain) but carries zero institutional trust.
```

## 3.3 Cross-Domain Trust

When a verifier encounters a DID from an unfamiliar root domain, the following verification steps are **recommended**:

1. **LEI validation** — is the root domain's LEI active and current via GLEIF API?
2. **SAS attestation check** — does the root domain have a valid attestation from a recognized issuer?
3. **Ecosystem trust credentials** — does the issuer hold compliance audit, interoperability, and data handling credentials? (see §3.4)
4. **Regulatory standing** — is the issuer licensed in a recognized jurisdiction?

Operators are NOT required to accept DIDs from all issuers. Each operator sets its own acceptance policy based on its regulatory obligations, risk appetite, and business rules. A bank in the EU may require eIDAS-qualified certificates in addition to LEI; a fintech in Costa Rica may accept LEI alone. Trust and risk scoring are per-operator responsibilities — the spec does not mandate a global trust score.

## 3.4 Ecosystem Trust Anchoring

Trust in a root domain holder is determined by **verifiable proofs embedded in their Issuer DID**, not by institutional type. A private entity with a Grade A compliance certification operates at equal or higher trust than a government entity lacking interoperability or protocol adherence.

### Issuer DID Credential Requirements

Root domain holders that wish to be recognized as trusted issuers MUST include ecosystem trust credentials in their DID's SAS attestation layer. The specification recognizes the following credential types:

| Credential Type | Grade | What It Certifies |
|---|---|---|
| **Spec & Protocol Compliance** | A / B / C | Technical implementation correctness + philosophical adherence to spec principles (open interoperability, user sovereignty, no vendor lock-in) |
| **Data Handling & Privacy** | A / B / C | PII vault architecture, encryption standards, data retention policies, compliance to global standards (GDPR, etc.) and local standards per jurisdiction |
| **Key Management & Governance** | A / B / C | Key custody model (HSM, multi-sig, recovery), governance structure (multisig, board, DAO), incident response, key rotation policies |
| **Interoperability Certification** | Pass / Fail | Cross-institution resolution, credential exchange, and verification tested against other compliant operators |
| **Regulatory License** | Per jurisdiction | Licensed to operate (banking, MSB, EMI, etc.) — jurisdiction-specific |
| **LEI + vLEI** | Active / Expired | Legal entity verified via GLEIF with authorized signers |

### Grading

- **Grade A** — full compliance, audited by spec maintainer or accredited body, all principles met
- **Grade B** — compliant with minor gaps, remediation plan in place
- **Grade C** — minimum viable compliance, significant gaps documented

### What the Audit Covers

The audit is not just technical — it ensures philosophical adherence to the spec's principles:

1. **Technical** — correct spec implementation, resolution works, metadata schema valid
2. **Philosophical** — adherence to spec principles: user sovereignty, interoperability-first, no vendor lock-in, open standard participation
3. **Data handling** — PII vault implementation, encryption at rest and in transit, 2-of-2 key split or equivalent, data retention and deletion policies
4. **Local compliance** — jurisdiction-specific requirements (GDPR in EU, Law 8968 in Costa Rica, LGPD in Brazil, CCPA in California, etc.)
5. **Key governance** — who holds the keys, what happens when keys are compromised, rotation schedule, recovery model
6. **Availability & reliability** — uptime commitments, resolver availability, service degradation handling

### How Verifiers Access This

When a verifier resolves an Issuer DID (e.g., `did:sns:crbank`), the trust credentials are available directly in the SAS attestation layer:

```
Verifier resolves: did:sns:crbank
  │
  ├── LEI: active (GLEIF API confirms)
  ├── Spec Compliance: Grade A (audited 2026-01-15, valid 12 months)
  ├── Data Handling: Grade A (audited 2026-01-15)
  ├── Key Governance: Grade B (remediation: HSM migration Q2 2026)
  ├── Interoperability: Pass (tested against 3 peer operators)
  └── Regulatory: Banking license CR, MSB license US

Verifier policy: require Grade B+ on all categories → ACCEPT
```

Each verifier evaluates the Issuer DID's credentials against its own acceptance policy. The spec defines the credential types and grading — operators decide what minimum grades they require.

### Operational Restrictions by Grade

| Data Handling Grade | Access to User Proofs | Resolution | Credential Issuance | Operating Model |
|---|---|---|---|---|
| **A** | Full — SD-JWT selective disclosure | Full | Independent | Self-operated or delegated |
| **B** | Limited — ZKP only, no raw claims | Full | Independent with monitoring | Self-operated with audit trail |
| **C** | **ZKP only** — zero access to raw user data | Full | **Must operate through a spec-compliant platform** | Managed model required — the platform handles PII vault, key management, and credential flows on behalf of the issuer |

### Why Grade C Requires a Managed Model

An issuer that cannot demonstrate adequate data handling (storing sensitive data in its own flows, binding user PII to internal processes, sharing data without consent) is a **data leak risk**. The ecosystem cannot prevent them from resolving DIDs — resolution is public — but it CAN restrict what they receive:

- **ZKP proofs only** — the issuer can verify "this person passed KYC Level 2" without ever seeing the underlying data (name, ID number, date of birth)
- **No SD-JWT with disclosed fields** — selective disclosure requires trust that the receiving party handles disclosed fields properly; Grade C issuers have not earned that trust
- **Managed operation** — the issuer MUST delegate credential issuance and PII handling to a spec-compliant platform (Grade A or B) that ensures user data never touches the issuer's non-compliant infrastructure

This creates a path for every institution to participate — even those with legacy systems or poor data practices — while **guaranteeing user data protection is never compromised** by the weakest link in the chain.

> **Example:** A bank with Grade C data handling can still issue `alice.bank.sol` and attest KYC — but the actual KYC data, PII vault, and credential signing are handled by the platform. The bank's systems receive only ZKP confirmations ("user is KYC'd") and can operate their banking services. User data never enters the bank's non-compliant pipeline.

> **Credential schemas** for ecosystem trust credentials are defined in the [Credential Schemas](../credentials/) section. Audits may be performed by the spec maintainer, accredited community participants, or recognized audit bodies. The [Community Charter](../charter/) defines the accreditation process.

---

[← §2 Focal Use Case](02-focal-use-case.md) | [Next: §4 Architectural Rationale →](04-architectural-rationale.md)
