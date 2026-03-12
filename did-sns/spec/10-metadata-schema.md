# 10. On-Chain Metadata Schema

DID metadata is stored in the SNS account data buffer (bytes 96+), identified by magic bytes `0x44494401`.

```
SNS NameRegistry Account:
+--------------------------------------------------+
| HEADER (96 bytes -- fixed by SNS program)        |
|  +-- Parent Key      (bytes 0-31)    32 bytes    |
|  +-- Owner Key       (bytes 32-63)   32 bytes    |
|  +-- Class Key       (bytes 64-95)   32 bytes    |
+--------------------------------------------------+
| DATA BUFFER (bytes 96+ -- DID metadata)          |
|  +-- Magic + Version + Flags + SAS UID           |
|      + ECIES key + Vault hash + Doc hash         |
+--------------------------------------------------+
```

## v2 Schema Layout (160 bytes)

| Field | Offset | Size | Description |
|---|---|---|---|
| Magic | 0 | 4 | `0x44494401` ("DID\x01") |
| Version | 4 | 1 | Schema version (`0x02` for SAS-linked) |
| Flags | 5 | 1 | Bitfield (see below) |
| SAS UID | 6 | 32 | SAS attestation PDA address |
| ECIES Key | 38 | 33 | Compressed secp256k1 public key for encryption |
| Vault Hash | 71 | 32 | SHA-256 of vault service endpoint URL |
| Doc Hash | 103 | 32 | SHA-256 of canonical DID Document |
| Reserved | 135 | 25 | Zero-filled (future use) |

## Flags Bitfield

| Bit | Flag | Meaning |
|---|---|---|
| `0x01` | `ACTIVE` | Domain is active (liveness flag) |
| `0x02` | `HAS_SBT` | Soul-Bound Token minted |
| `0x04` | `IS_TIER3` | Self-Sovereign ID tier |
| `0x08` | `HAS_LEI` | LEI/vLEI credential linked |
| `0x10` | `HAS_SAS` | SAS attestation linked (proof layer) |
| `0x20` | `HAS_PQ` | Post-quantum keys in SAS attestation (reserved, Phase 2) |

**Note:** Domain liveness is determined by the owner field (bytes 32–63), not a flag. A zero-address owner means the domain is deactivated. The `HAS_PQ` flag is reserved for the post-quantum migration (see [§12.1](12-security.md#121-post-quantum-cryptography-migration)).

---

[← §9 CRUD Operations](09-crud-operations.md) | [Next: §11 SAS Integration →](11-sas-integration.md)
