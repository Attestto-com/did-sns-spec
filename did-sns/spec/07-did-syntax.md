# 7. DID Syntax

The method name is `sns`. A DID using this method MUST begin with `did:sns:`. Per DID Core, the method name MUST be lowercase.

## 7.1 ABNF Grammar

```
did-sns       = "did:sns:" [ network ":" ] sns-name
network       = "mainnet" / "devnet" / "testnet"
sns-name      = label *( "." label )
label         = 1*( ALPHA / DIGIT / "-" )
```

> **Reserved network labels (disambiguation).** The grammar is ambiguous for a bare token that matches a network name (e.g. `did:sns:testnet` could parse as the top-level name `testnet`, or as the network `testnet` with an empty name). To resolve this deterministically across implementations:
>
> - The tokens `mainnet`, `devnet`, and `testnet` are **reserved as network selectors**. When the first colon-delimited segment after `did:sns:` equals one of these tokens **and is followed by another `:` and a non-empty `sns-name`**, a resolver **MUST** parse it as the `network` component (e.g. `did:sns:devnet:alice.crbank`).
> - Consequently these three tokens **MUST NOT** be registered or resolved as top-level `sns-name` labels, and a resolver **MUST** reject `did:sns:mainnet`, `did:sns:devnet`, and `did:sns:testnet` (a network token with no name) as `invalidDid`.
> - When no `network` component is present, `mainnet` is implied.

**Examples:**

| DID | Description |
|---|---|
| `did:sns:alice` | Top-level domain (mainnet implied) |
| `did:sns:alice.crbank` | Subdomain under tenant root |
| `did:sns:devnet:alice.crbank` | Explicit devnet resolution |
| `did:sns:ops.crbank` | Service/department subdomain |

## 7.2 Hierarchy Depth

SNS supports root domains and one level of subdomains. `did:sns` resolution follows this constraint — **two levels maximum** (root + one subdomain).

| Depth | Example | Supported |
|---|---|---|
| 0 (root) | `did:sns:crbank` | Yes |
| 1 (subdomain) | `did:sns:alice.crbank` | Yes |
| 2+ (nested) | `did:sns:dept.alice.crbank` | No — rejected by resolver |

## 7.3 Naming Strategies

| Strategy | Structure | Use Case |
|---|---|---|
| **Tenant Root Domain** (recommended) | `alice.crbank.sol` | Tenant owns root domain; cleanest hierarchy |
| **Flat Namespacing** | `crbank-alice.platform.sol` | All under platform domain; hyphen-separated tenant prefix |

---

[← §6 W3C Coverage](06-w3c-coverage.md) | [Next: §8 DID Document →](08-did-document.md)
