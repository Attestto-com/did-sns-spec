# 13. Interoperability

`did:sns` is designed to be usable outside any single platform. Every layer binds to an existing standard or protocol so that identities, credentials, and presentations work with third-party infrastructure without custom code.

| Layer | Standard | Integration |
|---|---|---|
| **DID Resolution** | W3C DID Core v1.1 + DID Resolution v0.3 | Full conformance. Universal Resolver driver published — any infrastructure running the [DIF Universal Resolver](https://dev.uniresolver.io/) can resolve `did:sns` identifiers out of the box. |
| **Credentials** | SD-JWT-VC ([draft-ietf-oauth-sd-jwt](https://datatracker.ietf.org/doc/draft-ietf-oauth-selective-disclosure-jwt/)) | Per-field selective disclosure with individual salt and hash. Verifiers don't need to know about Solana — they verify a standard JWT with any JOSE library. |
| **Agent Frameworks** | [Credo-TS 0.6](https://credo.js.org/) (formerly Aries Framework JS) | `did:sns` registered as a resolver module. Any Credo-based agent (government, bank, healthcare) can issue credentials to and verify presentations from `did:sns` identities without custom code. |
| **Credential API** | [W3C Credential Management API](https://www.w3.org/TR/credential-management-1/) | Browser extension bridges `navigator.credentials.get()` / `.store()` to the vault. Websites supporting WebAuthn/FIDO2 can request verifiable presentations through the same browser API. |
| **Presentation Exchange** | [DIDComm v2](https://identity.foundation/didcomm-messaging/spec/) Present Proof 3.0 | P2P encrypted channel between holder and verifier. Works with any DIDComm v2 implementation. |
| **Status Lists** | [W3C Bitstring Status List v1.0](https://www.w3.org/TR/vc-bitstring-status-list/) | Revocation and suspension published at stable URLs. Any W3C-compliant verifier can check credential status without calling any specific platform's API. |
| **Organizational Identity** | [GLEIF vLEI](https://www.gleif.org/en/lei-solutions/gleifs-digital-strategy-for-the-lei/introducing-the-verifiable-lei) (OOR credentials) | vLEI Bridge maps GLEIF-issued LEI credentials to SAS attestations. A verifier who trusts GLEIF's root of trust can verify `did:sns` organizational DIDs. |
| **Key Types** | Ed25519 (RFC 8032), secp256k1 (ECIES), ML-DSA-44 / ML-KEM-768 (FIPS 203/204) | Standard key types — no proprietary cryptography. Post-quantum hybrid mode defined for the transition period. See [§12.1](12-security.md#121-post-quantum-cryptography-migration). |
| **JSON-LD Context** | [W3C JSON-LD 1.1](https://www.w3.org/TR/json-ld11/) | Published at `spec.attestto.com/v1/sns.jsonld`. Any JSON-LD processor can expand `did:sns` documents without platform-specific tooling. |
| **DNS Discovery** | DNS TXT records | `_did.alice.com TXT "did=did:sns:alice.crbank"` — Web2 domains can alias to `did:sns` identifiers. Verifiers resolve via standard DNS before hitting Solana. |

## Universal Resolver: Convenience, Not Infrastructure

The [DIF Universal Resolver](https://dev.uniresolver.io/) is a shared HTTP service that wraps many DID method drivers behind a single REST API. It is a **convenience bridge** for testing, prototyping, and low-volume verification — not a production dependency.

| Use Case | Recommended Approach |
|---|---|
| **Testing & prototyping** | Use the public Universal Resolver instance at `dev.uniresolver.io` |
| **Low-volume verification** | Self-host a Universal Resolver instance with the `did:sns` driver |
| **Production / high-volume** | Use the open-source resolver library ([`@attestto/did-sns-resolver`](https://github.com/Attestto-com/did-sns-resolver)) directly against your own Solana RPC node |
| **Maximum reliability** | Run your own Solana validator or use multiple RPC providers for redundancy |

> **Do not depend on the public Universal Resolver instance for production.** It is a community-maintained reference endpoint. Production deployments should run their own resolver or query Solana RPC directly using the open-source resolver library (MIT-licensed).

## Verification Without Any Specific Platform

A party who has never heard of any specific `did:sns` operator can verify a `did:sns` credential end-to-end: run the Universal Resolver (or use the open-source library directly), get a standard DID Document, verify the SD-JWT with any JOSE library, and check the status list at a public URL. They never need to interact with any specific platform, install proprietary software, or call a proprietary API. The same applies in reverse — a `did:sns` holder can present credentials to a verifier running Credo, ACA-Py, Walt.id, or any DIDComm v2 implementation.

---

[← §12 Security](12-security.md) | [Next: §14 References →](14-references.md)
