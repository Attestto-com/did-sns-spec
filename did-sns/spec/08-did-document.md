# 8. DID Document

## 8.1 Context

All `did:sns` DID Documents MUST include:

```json
"@context": [
  "https://www.w3.org/ns/did/v1",
  "https://w3id.org/security/suites/ed25519-2020/v1",
  "https://w3id.org/security/suites/secp256k1-2019/v1",
  "https://w3id.org/security/suites/x25519-2020/v1"
]
```

## 8.2 Example — Tier 1/2 Tenant User (Web2 Native / Anchored)

A bank-issued subdomain for a user who completed KYC. The user sees no Web3 features — signing uses their country's digital signature standard and the platform's biometric passkey flow (`#attestto-sign`). No wallet keys are exposed in the DID Document.

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/x509-2019/v1"
  ],
  "id": "did:sns:alice.crbank",
  "controller": "did:sns:crbank",
  "verificationMethod": [
    {
      "id": "did:sns:alice.crbank#firma-digital",
      "type": "X509Certificate2019",
      "controller": "did:sns:alice.crbank",
      "x509CertificateChain": ["MIIBxTCCAWugAwIBAgI..."]
    },
    {
      "id": "did:sns:alice.crbank#attestto-sign",
      "type": "ConditionalProof2022",
      "controller": "did:sns:crbank",
      "conditionWeightedThreshold": 1,
      "conditionOperator": "or",
      "condition": [
        { "type": "WebAuthnAuthentication", "origin": "https://sign.crbank.com" },
        { "type": "BiometricAuthentication", "origin": "https://sign.crbank.com/mobile" }
      ]
    }
  ],
  "authentication": [
    "did:sns:alice.crbank#firma-digital",
    "did:sns:alice.crbank#attestto-sign"
  ],
  "assertionMethod": [
    "did:sns:alice.crbank#firma-digital",
    "did:sns:alice.crbank#attestto-sign"
  ],
  "service": [
    {
      "id": "did:sns:alice.crbank#vault",
      "type": "EncryptedDataVault",
      "serviceEndpoint": "https://vault.crbank.com/v1/alice"
    },
    {
      "id": "did:sns:alice.crbank#vp-endpoint",
      "type": "VerifiablePresentationService",
      "serviceEndpoint": "https://api.crbank.com/v1/presentations/alice"
    }
  ]
}
```

**Note:** No wallet keys, no `keyAgreement`, no `capabilityInvocation`. The user's signing keys are their country's digital certificate (e.g., Firma Digital in Costa Rica, X.509 in the EU) and the platform's passkey-based biometric flow. The user may not be aware that an on-chain DID exists — they simply sign documents and share credentials through the tenant's UX.

## 8.2b Example — Tier 3 Tenant User (Full Web3 / SSI)

The same bank-issued subdomain, but the user has opted into full Web3 features. The DID Document now includes self-custodial wallet keys (Solana + Ethereum) alongside the country signature and platform passkey. The user controls which key signs each transaction.

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1",
    "https://w3id.org/security/suites/secp256k1-2019/v1",
    "https://w3id.org/security/suites/x25519-2020/v1",
    "https://w3id.org/security/suites/x509-2019/v1"
  ],
  "id": "did:sns:alice.crbank",
  "controller": "did:sns:crbank",
  "verificationMethod": [
    {
      "id": "did:sns:alice.crbank#firma-digital",
      "type": "X509Certificate2019",
      "controller": "did:sns:alice.crbank",
      "x509CertificateChain": ["MIIBxTCCAWugAwIBAgI..."]
    },
    {
      "id": "did:sns:alice.crbank#attestto-sign",
      "type": "ConditionalProof2022",
      "controller": "did:sns:crbank",
      "conditionWeightedThreshold": 1,
      "conditionOperator": "or",
      "condition": [
        { "type": "WebAuthnAuthentication", "origin": "https://sign.crbank.com" },
        { "type": "BiometricAuthentication", "origin": "https://sign.crbank.com/mobile" }
      ]
    },
    {
      "id": "did:sns:alice.crbank#solana-key",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:sns:alice.crbank",
      "publicKeyMultibase": "z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
    },
    {
      "id": "did:sns:alice.crbank#eth-key",
      "type": "EcdsaSecp256k1VerificationKey2019",
      "controller": "did:sns:alice.crbank",
      "publicKeyMultibase": "zQ3shP2mWsZYWgUELrzg..."
    },
    {
      "id": "did:sns:alice.crbank#key-agreement",
      "type": "X25519KeyAgreementKey2020",
      "controller": "did:sns:alice.crbank",
      "publicKeyMultibase": "z6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc"
    }
  ],
  "authentication": [
    "did:sns:alice.crbank#firma-digital",
    "did:sns:alice.crbank#attestto-sign",
    "did:sns:alice.crbank#solana-key",
    "did:sns:alice.crbank#eth-key"
  ],
  "assertionMethod": [
    "did:sns:alice.crbank#firma-digital",
    "did:sns:alice.crbank#attestto-sign",
    "did:sns:alice.crbank#solana-key",
    "did:sns:alice.crbank#eth-key"
  ],
  "keyAgreement": ["did:sns:alice.crbank#key-agreement"],
  "capabilityInvocation": ["did:sns:alice.crbank#solana-key"],
  "capabilityDelegation": ["did:sns:alice.crbank#solana-key"],
  "service": [
    {
      "id": "did:sns:alice.crbank#vault",
      "type": "EncryptedDataVault",
      "serviceEndpoint": "https://vault.crbank.com/v1/alice"
    },
    {
      "id": "did:sns:alice.crbank#vp-endpoint",
      "type": "VerifiablePresentationService",
      "serviceEndpoint": "https://api.crbank.com/v1/presentations/alice"
    },
    {
      "id": "did:sns:alice.crbank#didcomm",
      "type": "DIDCommMessaging",
      "serviceEndpoint": "https://relay.crbank.com/v1/didcomm",
      "accept": ["didcomm/v2"]
    }
  ],
  "alsoKnownAs": [
    "https://search.gleif.org/#/record/5493001KJTIIGC8Y1R12"
  ]
}
```

**Note:** Tier 3 adds `#solana-key` (Ed25519), `#eth-key` (secp256k1), `keyAgreement` for DIDComm, and `capabilityInvocation`/`capabilityDelegation` for on-chain governance (DAO voting, multisig treasury). The country certificate and platform passkey remain available — the user chooses which method to use per transaction. `alsoKnownAs` binds external identifiers (LEI records, `did:web`, etc.) to the DID — it does not link to other `did:sns` subdomains of the same user.

## 8.2c Example — Tier 3 Platform User (Direct, Full Web3)

A platform-issued subdomain for a power user with full self-sovereign control. Same key structure as 8.2b but under a platform root domain.

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://w3id.org/security/suites/ed25519-2020/v1",
    "https://w3id.org/security/suites/secp256k1-2019/v1",
    "https://w3id.org/security/suites/x25519-2020/v1"
  ],
  "id": "did:sns:alice.platform",
  "controller": "did:sns:platform",
  "verificationMethod": [
    {
      "id": "did:sns:alice.platform#attestto-sign",
      "type": "ConditionalProof2022",
      "controller": "did:sns:platform",
      "conditionWeightedThreshold": 1,
      "conditionOperator": "or",
      "condition": [
        { "type": "WebAuthnAuthentication", "origin": "https://sign.platform.com" },
        { "type": "BiometricAuthentication", "origin": "https://sign.platform.com/mobile" }
      ]
    },
    {
      "id": "did:sns:alice.platform#solana-key",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:sns:alice.platform",
      "publicKeyMultibase": "z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK"
    },
    {
      "id": "did:sns:alice.platform#eth-key",
      "type": "EcdsaSecp256k1VerificationKey2019",
      "controller": "did:sns:alice.platform",
      "publicKeyMultibase": "zQ3shP2mWsZYWgUELrzg..."
    },
    {
      "id": "did:sns:alice.platform#key-agreement",
      "type": "X25519KeyAgreementKey2020",
      "controller": "did:sns:alice.platform",
      "publicKeyMultibase": "z6LSbysY2xFMRpGMhb7tFTLMpeuPRaqaWM1yECx2AtzE3KCc"
    }
  ],
  "authentication": [
    "did:sns:alice.platform#attestto-sign",
    "did:sns:alice.platform#solana-key",
    "did:sns:alice.platform#eth-key"
  ],
  "assertionMethod": [
    "did:sns:alice.platform#attestto-sign",
    "did:sns:alice.platform#solana-key",
    "did:sns:alice.platform#eth-key"
  ],
  "keyAgreement": ["did:sns:alice.platform#key-agreement"],
  "capabilityInvocation": ["did:sns:alice.platform#solana-key"],
  "capabilityDelegation": ["did:sns:alice.platform#solana-key"],
  "service": [
    {
      "id": "did:sns:alice.platform#vault",
      "type": "EncryptedDataVault",
      "serviceEndpoint": "https://vault.platform.com/v1/alice"
    },
    {
      "id": "did:sns:alice.platform#vp-endpoint",
      "type": "VerifiablePresentationService",
      "serviceEndpoint": "https://api.platform.com/v1/presentations/alice"
    },
    {
      "id": "did:sns:alice.platform#didcomm",
      "type": "DIDCommMessaging",
      "serviceEndpoint": "https://relay.platform.com/v1/didcomm",
      "accept": ["didcomm/v2"]
    }
  ]
}
```

**Note:** Platform users at Tier 3 have the same Web3 capabilities as tenant users but under the platform's root domain. No country-specific certificate is shown here — the platform user authenticates via passkey biometric + self-custodial wallets. Country certificates can be added if the user's jurisdiction supports them.

## 8.3 Example — Tenant Root Domain (Self-Sovereign)

```json
{
  "id": "did:sns:crbank",
  "verificationMethod": [{
    "id": "did:sns:crbank#solana-key",
    "type": "Ed25519VerificationKey2020",
    "controller": "did:sns:crbank",
    "publicKeyMultibase": "z6MkBankPublicKeyBase58encoded..."
  }],
  "authentication": ["did:sns:crbank#solana-key"],
  "assertionMethod": ["did:sns:crbank#solana-key"],
  "service": [{
    "id": "did:sns:crbank#lei-resolver",
    "type": "GleifLookupService",
    "serviceEndpoint": {
      "lei": "9845008661B99CC9FD07",
      "entityName": "CR Bank",
      "jurisdiction": "CR",
      "gleifApiUrl": "https://api.gleif.org/api/v1/lei-records/9845008661B99CC9FD07"
    }
  }],
  "alsoKnownAs": [
    "https://search.gleif.org/#/record/9845008661B99CC9FD07",
    "did:web:crbank.cr"
  ]
}
```

**Note:** No `controller` field — top-level domains are self-sovereign. Trust is established via `alsoKnownAs` and LEI bridge, not hierarchy.

## 8.4 Example — Tenant Client (Under Bank Root)

```json
{
  "id": "did:sns:alice.crbank",
  "controller": "did:sns:crbank",
  "verificationMethod": [{
    "id": "did:sns:alice.crbank#solana-key",
    "type": "Ed25519VerificationKey2020",
    "controller": "did:sns:alice.crbank",
    "publicKeyMultibase": "z6MkAlicePublicKey..."
  }],
  "authentication": ["did:sns:alice.crbank#solana-key"],
  "assertionMethod": ["did:sns:alice.crbank#solana-key"],
  "service": [{
    "id": "did:sns:alice.crbank#vault",
    "type": "EncryptedDataVault",
    "serviceEndpoint": "https://api.crbank.cr/v1/vault/alice"
  }]
}
```

**Note:** `controller` points to the tenant root. Service endpoints point to the tenant's infrastructure (whitelabel).

## 8.5 Verification Methods

| Key ID Suffix | Type | Tiers | Purpose |
|---|---|---|---|
| `#firma-digital` | `X509Certificate2019` | 1, 2, 3 | Country-specific digital certificate (e.g., Firma Digital CR, eIDAS EU) — legally binding signature |
| `#attestto-sign` | `ConditionalProof2022` | 1, 2, 3 | Platform passkey + biometric flow (QR → phone → biometric) — default signing method |
| `#solana-key` | `Ed25519VerificationKey2020` | 3 | Solana self-custodial wallet — on-chain governance, multisig treasury, SBT minting |
| `#eth-key` | `EcdsaSecp256k1VerificationKey2019` | 3 | Ethereum self-custodial wallet — cross-chain identity, EVM DeFi, ERC-based credentials |
| `#key-agreement` | `X25519KeyAgreementKey2020` | 3 | DIDComm v2 key agreement — P2P encrypted channel for credential presentation |

Tier 1/2 DIDs expose only `#firma-digital` and/or `#attestto-sign`. Tier 3 adds self-custodial wallet keys and DIDComm key agreement. When present, `#solana-key` MUST correspond to the SNS domain owner's Solana public key (bytes 32–63 of the NameRegistry header).

## 8.6 Service Endpoints

| Type | Description |
|---|---|
| `EncryptedDataVault` | PII Vault with per-object encryption (VMK+DEK) |
| `VerifiablePresentationService` | VP request/response with SD-JWT selective disclosure |
| `DIDCommMessaging` | DIDComm v2 relay for P2P credential presentation |
| `GleifLookupService` | LEI-to-entity mapping (auto-populated when vLEI bridge attestation exists) |
| `BitstringStatusList` | W3C Bitstring Status List for real-time credential revocation |

## 8.7 Controller Hierarchy

| Domain Type | Example | Controller |
|---|---|---|
| Top-level | `did:sns:platform` | Self-sovereign (omitted or self) |
| Tenant root | `did:sns:crbank` | Self-sovereign (omitted or self) |
| Platform subdomain | `did:sns:alice.platform` | `did:sns:platform` |
| Tenant client | `did:sns:alice.crbank` | `did:sns:crbank` |

---

[← §7 DID Syntax](07-did-syntax.md) | [Next: §9 CRUD Operations →](09-crud-operations.md)
