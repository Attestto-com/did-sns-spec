# Attestto Credential Schemas — v1

W3C Verifiable Credential JSON-LD contexts for financial compliance and identity verification workflows. Each schema defines a full-disclosure credential type and a selective-disclosure (SD-JWT) companion type for privacy-preserving presentations.

**Namespace:** `https://spec.attestto.com/credentials/v1/`

---

## Schemas

| File | Credential Type | SD Companion | Fields |
|---|---|---|---|
| `identity-verification.jsonld` | `IdentityVerificationCredential` | `IdentityVerificationSelectiveDisclosure` | 21 |
| `source-of-funds.jsonld` | `SourceOfFundsCredential` | `SourceOfFundsSelectiveDisclosure` | 12 |
| `proof-of-address.jsonld` | `ProofOfAddressCredential` | `ProofOfAddressSelectiveDisclosure` | 13 |
| `edd-clearance.jsonld` | `EddClearanceCredential` | `EddClearanceSelectiveDisclosure` | 14 |

---

## 1. Identity Verification (`IdentityVerificationCredential`)

Personal KYC credential issued after identity document verification + liveness check.

| Field | Type | Description | Example |
|---|---|---|---|
| `fullName` | `xsd:string` | Legal full name as on identity document | `"María García López"` |
| `dateOfBirth` | `xsd:date` | Date of birth | `"1990-05-14"` |
| `nationality` | `xsd:string` | ISO 3166-1 alpha-2 country code | `"CR"` |
| `nationalId` | `xsd:string` | Government-issued national ID number | `"1-0234-0567"` |
| `documentType` | `xsd:string` | Type of identity document presented | `"passport"`, `"national_id"`, `"drivers_license"` |
| `documentNumber` | `xsd:string` | Document serial/number | `"PA1234567"` |
| `documentCountry` | `xsd:string` | Issuing country (ISO 3166-1 alpha-2) | `"CR"` |
| `documentExpiry` | `xsd:date` | Document expiration date | `"2030-12-31"` |
| `documentFrontHash` | `xsd:string` | SHA-256 hash of document front image | `"a3f2..."` |
| `documentBackHash` | `xsd:string` | SHA-256 hash of document back image | `"b7e1..."` |
| `selfieHash` | `xsd:string` | SHA-256 hash of liveness selfie image | `"c9d4..."` |
| `phoneNumber` | `xsd:string` | Verified phone number (E.164 format) | `"+50688887777"` |
| `verificationLevel` | `xsd:string` | KYC tier achieved | `"enhanced"`, `"basic"`, `"standard"` |
| `verificationMethod` | `xsd:string` | How verification was performed | `"automated_ocr_biometric"` |
| `livenessCheckPassed` | `xsd:boolean` | Whether liveness detection succeeded | `true` |
| `documentAuthenticity` | `xsd:string` | Authenticity assessment result | `"genuine"`, `"suspected_forgery"` |
| `sanctionsScreeningPassed` | `xsd:boolean` | Passed sanctions list check | `true` |
| `pepScreeningPassed` | `xsd:boolean` | Passed PEP (Politically Exposed Person) check | `true` |
| `jurisdiction` | `xsd:string` | Regulatory jurisdiction of verification | `"CR"` |
| `regulatoryFramework` | `xsd:string` | Applicable regulation | `"Ley 8204"`, `"AMLD5"` |
| `verifiedAt` | `xsd:dateTime` | Timestamp of verification completion | `"2026-03-12T14:30:00Z"` |

### Selective Disclosure Predicates

| Field | Type | Description |
|---|---|---|
| `disclosedFields` | `@set` | Which full fields the holder chose to reveal |
| `meetsMinimumAge` | `xsd:boolean` | Prove age ≥ threshold without revealing DOB |
| `minimumAgeRequired` | `xsd:integer` | The age threshold being tested |
| `countryInAllowList` | `xsd:boolean` | Prove nationality is in an accepted list |
| `meetsMinimumLevel` | `xsd:boolean` | Prove KYC level ≥ required tier |
| `requiredLevel` | `xsd:string` | The tier being tested against |
| `sanctionsCleared` | `xsd:boolean` | Prove sanctions screening passed |
| `isDocumentValid` | `xsd:boolean` | Prove document is genuine + not expired |

---

## 2. Source of Funds (`SourceOfFundsCredential`)

Attests to verified income sources and amounts for AML/CFT compliance.

| Field | Type | Description | Example |
|---|---|---|---|
| `totalVerifiedAmount` | `xsd:decimal` | Total verified funds amount | `150000.00` |
| `currency` | `xsd:string` | Currency code (ISO 4217) | `"USD"` |
| `verificationPeriodStart` | `xsd:date` | Start of verified period | `"2025-01-01"` |
| `verificationPeriodEnd` | `xsd:date` | End of verified period | `"2025-12-31"` |
| `sourceBreakdown` | object | Per-source amounts (nested) | — |
| ↳ `employment` | `xsd:decimal` | Employment income | `80000.00` |
| ↳ `business` | `xsd:decimal` | Business income | `30000.00` |
| ↳ `investment` | `xsd:decimal` | Investment returns | `20000.00` |
| ↳ `rental` | `xsd:decimal` | Rental income | `10000.00` |
| ↳ `pension` | `xsd:decimal` | Pension/retirement | `0.00` |
| ↳ `government` | `xsd:decimal` | Government benefits | `0.00` |
| ↳ `inheritance` | `xsd:decimal` | Inheritance | `0.00` |
| ↳ `sale_of_assets` | `xsd:decimal` | Asset sales | `10000.00` |
| ↳ `crypto` | `xsd:decimal` | Crypto-related income | `0.00` |
| ↳ `other` | `xsd:decimal` | Other sources | `0.00` |
| `confidenceScore` | `xsd:integer` | Verification confidence (0-100) | `85` |
| `riskLevel` | `xsd:string` | Risk assessment result | `"low"`, `"medium"`, `"high"` |
| `verificationMethod` | `xsd:string` | How SoF was verified | `"bank_statements_ocr"` |
| `bankingSources` | `xsd:integer` | Number of bank sources verified | `3` |
| `jurisdiction` | `xsd:string` | Regulatory jurisdiction | `"CR"` |
| `regulatoryFramework` | `xsd:string` | Applicable regulation | `"FATF R.10"` |

### Selective Disclosure Predicates

| Field | Type | Description |
|---|---|---|
| `disclosedFields` | `@set` | Which full fields the holder chose to reveal |
| `amountRange` | object | Prove total is within a range (min/max) |
| `meetsThreshold` | `xsd:boolean` | Prove total ≥ a required threshold |
| `thresholdAmount` | `xsd:decimal` | The threshold being tested |

---

## 3. Proof of Address (`ProofOfAddressCredential`)

Attests to a verified residential or business address.

| Field | Type | Description | Example |
|---|---|---|---|
| `addressType` | `xsd:string` | Type of address | `"residential"`, `"business"` |
| `country` | `xsd:string` | Country (ISO 3166-1 alpha-2) | `"CR"` |
| `region` | `xsd:string` | State/province/region | `"San José"` |
| `locality` | `xsd:string` | City or town | `"Escazú"` |
| `postalCode` | `xsd:string` | Postal/ZIP code | `"10201"` |
| `streetAddress` | `xsd:string` | Street address | `"Calle 5, Edificio ABC"` |
| `documentType` | `xsd:string` | Type of document used as proof | `"utility_bill"`, `"bank_statement"` |
| `documentIssuer` | `xsd:string` | Who issued the proof document | `"ICE Electricidad"` |
| `documentDate` | `xsd:date` | Date on the proof document | `"2026-02-15"` |
| `verificationMethod` | `xsd:string` | How address was verified | `"document_ocr"`, `"database_lookup"` |
| `verifiedAt` | `xsd:dateTime` | Verification timestamp | `"2026-03-01T10:00:00Z"` |
| `residencySince` | `xsd:date` | How long at this address | `"2020-06-01"` |
| `jurisdiction` | `xsd:string` | Regulatory jurisdiction | `"CR"` |

### Selective Disclosure Predicates

| Field | Type | Description |
|---|---|---|
| `disclosedFields` | `@set` | Which full fields the holder chose to reveal |
| `countryMatch` | `xsd:boolean` | Prove country matches a required value |
| `regionMatch` | `xsd:boolean` | Prove region matches a required value |
| `isCurrentAddress` | `xsd:boolean` | Prove address is current (document < 3 months old) |

---

## 4. EDD Clearance (`EddClearanceCredential`)

Enhanced Due Diligence clearance for high-value or high-risk transactions.

| Field | Type | Description | Example |
|---|---|---|---|
| `tierLabel` | `xsd:string` | EDD tier achieved | `"T3"`, `"T4"` |
| `clearedCumulativeThreshold` | `xsd:decimal` | Max cumulative amount cleared | `500000.00` |
| `clearedSingleTxThreshold` | `xsd:decimal` | Max single transaction cleared | `100000.00` |
| `currency` | `xsd:string` | Currency (ISO 4217) | `"USD"` |
| `evaluationPeriod` | `xsd:string` | Validity period | `"30d"`, `"90d"` |
| `documentsProvided` | `@set` | List of documents submitted | `["bank_statement", "tax_return"]` |
| `proofOfAddressVerified` | `xsd:boolean` | PoA credential verified | `true` |
| `sourceOfFundsVerified` | `xsd:boolean` | SoF credential verified | `true` |
| `enhancedScreeningPassed` | `xsd:boolean` | Enhanced AML screening passed | `true` |
| `officerApproval` | object | Compliance officer sign-off (nested) | — |
| ↳ `approved` | `xsd:boolean` | Whether officer approved | `true` |
| ↳ `approverDid` | `xsd:string` | DID of approving officer | `"did:sns:attestto.officer.1"` |
| ↳ `approvedAt` | `xsd:dateTime` | Approval timestamp | `"2026-03-10T16:00:00Z"` |
| `riskAssessment` | object | Risk evaluation (nested) | — |
| ↳ `level` | `xsd:string` | Risk level | `"medium"` |
| ↳ `score` | `xsd:integer` | Risk score (0-100) | `45` |
| ↳ `factors` | `@set` | Contributing risk factors | `["high_value", "cross_border"]` |
| `jurisdiction` | `xsd:string` | Regulatory jurisdiction | `"CR"` |
| `regulatoryFramework` | `xsd:string` | Applicable regulation | `"Ley 8204"` |
| `clearanceValidFrom` | `xsd:dateTime` | Clearance start | `"2026-03-10T00:00:00Z"` |
| `clearanceValidUntil` | `xsd:dateTime` | Clearance expiry | `"2026-06-10T00:00:00Z"` |
| `relatedCredentials` | `@set` | Linked SoF/PoA credentials | — |
| ↳ `credentialId` | `xsd:string` | ID of related credential | `"urn:uuid:..."` |
| ↳ `credentialType` | `xsd:string` | Type of related credential | `"SourceOfFundsCredential"` |

### Selective Disclosure Predicates

| Field | Type | Description |
|---|---|---|
| `disclosedFields` | `@set` | Which full fields the holder chose to reveal |
| `meetsMinimumTier` | `xsd:boolean` | Prove EDD tier ≥ required tier |
| `requiredTier` | `xsd:string` | The tier being tested against |
| `isCurrent` | `xsd:boolean` | Prove clearance has not expired |

---

## Design Principles

1. **Binary data as hashes** — Document images and biometrics are stored as SHA-256 hashes, not raw blobs. The credential attests that the issuer verified the original.

2. **SD-JWT companion types** — Each credential has a `*SelectiveDisclosure` type with boolean predicates (e.g., `meetsMinimumAge`, `meetsThreshold`) enabling zero-knowledge-style proofs without revealing underlying data.

3. **Jurisdiction awareness** — Every credential carries `jurisdiction` and (where applicable) `regulatoryFramework` fields to support cross-border compliance routing.

4. **Credential chaining** — EDD Clearance links to prerequisite SoF and PoA credentials via `relatedCredentials`, enabling downstream verifiers to trace the full compliance chain.
