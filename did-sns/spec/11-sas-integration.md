# 11. SAS Integration (Proof Layer)

The [Solana Attestation Service](https://attest.solana.com/docs) (SAS) provides the issuer-signed proof layer. The SNS data buffer points to a SAS attestation; the resolver follows the pointer to fetch verifiable claims.

```
SNS Data Buffer                         SAS Attestation (on-chain)
+-------------------------+  follows UID  +-----------------------------+
| Magic + Version         | -----------> | credential: Issuer PDA      |
| Flags (HAS_SAS = 0x10) |              | schema: DID Identity v1     |
| SAS Attestation UID ----|              | signer: Authorized key      |
| ECIES Public Key        |              | data:                       |
| Vault Endpoint Hash     |              |   lei_hash (SHA-256)        |
+-------------------------+              |   role_level (1-4)          |
       ^ owned by                        |   jurisdiction (ISO 3166)   |
  Domain owner                           |   zkp_proof_hash            |
  (or platform)                          |   expires_at                |
                                         | tokenAccount (SBT)          |
                                         +-----------------------------+
                                                ^ signed by
                                          Authorized Issuer
```

## SAS Hierarchy

```
Credential (Issuer Authority)
  +-- Schema (DID Identity Attestation v1)
        +-- Attestation (per-user, per-domain)
              |-- data: Borsh-serialized identity claims
              |-- expiry: Unix timestamp
              |-- signer: Authorized signer key
              +-- tokenAccount: Optional Token-2022 SBT
```

## Why Two Layers?

| Concern | SNS Buffer | SAS Attestation |
|---|---|---|
| **Who writes** | Domain owner | Authorized issuer (SAS program) |
| **Trust level** | Self-asserted | Issuer-signed, verifiable |
| **Cost** | Included in rent deposit | Separate PDA rent + tx fee |
| **Speed** | Single RPC call | Second RPC call (follow UID) |
| **Data** | Compact pointer + encryption key | Rich identity claims |

## Class Key Security Model

The subdomain's Class Key (bytes 64–95) controls who can update the data buffer:

| Mode | Class Key | Who Can Update Buffer | Use Case |
|---|---|---|---|
| **Open** | Zero (default) | Domain owner only | Self-sovereign users |
| **SAS-Locked** | SAS Program ID | SAS program via CPI | Platform-managed identities |
| **Custom** | Custom program | Custom program via CPI | Tenant governance programs |

> **Security consequence of SAS-Locked / Custom mode.** When a subdomain is class-locked, the **class-key holder — not the named subject — controls every write to that subdomain's data buffer**: rotating `#solana-key`, replacing the ECIES encryption key (which lets the holder read VP payloads encrypted to the DID from that point forward), moving the SAS attestation pointer, and repointing service endpoints. A class-locked subdomain is therefore **platform-managed, not self-sovereign** — its integrity depends on the honesty and operational security of the class-key holder. See [§12.5 Malicious or Compromised Controller / Class-Key Holder](12-security.md#125-malicious-or-compromised-controller--class-key-holder) for the full threat model and mitigations. The **Open** mode (zero class key) is the self-sovereign alternative, where only the domain owner can write.

---

[← §10 Metadata Schema](10-metadata-schema.md) | [Next: §12 Security Considerations →](12-security.md)
