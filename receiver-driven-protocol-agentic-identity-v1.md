# A Receiver-Driven Protocol for Agentic Identity
### *x402 for Identity — V1 Draft*

> **Status:** Draft · Submitted for discussion to the W3C AI Agent Protocol Community Group
> **Author:** Disha G.
> **Date:** June 2026

## Abstract

As AI agents evolve from internal enterprise automations to cross-platform economic actors navigating the open web, the infrastructure for establishing trust has failed to keep pace. Current frameworks attempt to solve agentic identity through preemptive, fragmented developer registrations, creating siloed ecosystems and insurmountable friction.

This whitepaper proposes a paradigm shift: a **parameter-driven protocol** where host servers dynamically dictate identity and authorization requirements at the exact moment of interaction, leveraging HTTP `401 Unauthorized` and `WWW-Authenticate` headers paired with agent-aggregated, Base64-encoded Verifiable Credentials.

## 1. Introduction: The Identity Bottleneck

The agentic economy remains fundamentally gated by trust. Historically, there has been no universal way to verify a human or business, forcing organizations to rely on fragmented vendors for individual needs.

The industry's current trajectory replicates this flaw by demanding preemptive developer registration on proprietary portals such as those operated by Mastercard, Visa, Vouched, and Skyfire. This model introduces severe limitations.

**Developer Fragmentation:** Creators must maintain independent registrations across multiple networks to operate globally.

**Receiver Overhead:** Instead of a federated trust standard, merchants or receivers must maintain well-known URLs or local registries for the specific agent platforms they want to allow through.

To scale, the agentic web must abandon static, pushed registrations. It requires a protocol where identity requirements are **pulled dynamically** by the receiver at the moment of interaction.

## 2. The Use-Case Spectrum & Parameterized Demands

Agentic risk is highly non-uniform. The following examples illustrate the spectrum:

| Use Case | Risk Level | Required Parameters |
|---|---|---|
| Unrestricted content scraping | None | Zero verification |
| Gated content access | Low | Enterprise SSO + usage declaration |
| High-value / age-gated e-commerce | Medium | User SSO + human-in-loop proof + age verification + KYB |
| Regulated financial operations | High | Full KYC + face biometrics + KYB + regional compliance proof |

Because these nuances cannot be flattened into rigid identity tiers (like NIST IALs), the protocol must allow receivers to request **specific, standalone parameters on demand** — analogous to how the `x402` protocol dynamically dictates required payment addresses, currencies, and amounts.

## 3. Protocol Flow & Specifications

The protocol operates as a dynamic handshake where the host server dictates terms, and the agent acts as the primary orchestrator to aggregate and present the necessary proofs.

Two architectural pillars ensure stability and interoperability:

1. **A Governed Parameter Schema:** Parameter keys (e.g., `require_user_sso`, `require_is_human_above_18`) are defined by mutual ecosystem agreement and maintained in a centrally governed, append-only schema registry — similar to the IANA HTTP header registry — so all agents natively understand each requested parameter.

2. **Strict JSON Encapsulation:** The protocol enforces a strict `require_` (challenge) → `fulfill_` (response) mapping, encapsulated as a Base64-encoded JSON object.

### Step 1: Initial Request

The agent attempts an action without providing identity context.

```http
GET /api/v2/research-datasets/402 HTTP/1.1
Host: data-provider.com
User-Agent: AI-Agent/2.0
```

### Step 2: Receiver-Driven Challenge (`401 Unauthorized`)

The server denies the request with HTTP `401 Unauthorized`, including a `WWW-Authenticate: Agent-Identity` header that outlines the exact parameter menu required. Trusted issuers are defined **at the parameter level**.

```http
HTTP/1.1 401 Unauthorized
Date: Fri, 05 Jun 2026 13:50:00 GMT
WWW-Authenticate: Agent-Identity
  realm="research_docs",
  require_user_sso="true; trusted_issuers='okta, pingidentity, entra, auth0, forgerock, onelogin'",
  require_developer_kyb="true; trusted_issuers='socure, onfido, sumsub, trulioo, jumio, stripe, veriff, idnow'",
  require_declaration="The bearer of this token agrees that data retrieved from this endpoint
    will not be ingested into LLM training sets or used for public scraping replication."
Content-Type: application/json

{
  "error": "identity_required",
  "message": "Dynamic agent credential aggregation required for this endpoint."
}
```

### Step 3: Agent-Led Token Aggregation

The agent parses the challenge and programmatically communicates with the respective identity issuers. It constructs a structured JSON object, explicitly declaring which issuer fulfilled each requirement — allowing the receiver to verify tokens without guesswork.

```json
{
  "fulfill_user_sso": {
    "issuer": "okta",
    "token": "eyJhbGciOiJSUzI1NiIs..."
  },
  "fulfill_developer_kyb": {
    "issuer": "sumsub",
    "token": "eyJhbGciOiJFUzI1NiIs..."
  },
  "fulfill_declaration_agreement": "agreed",
  "fulfill_declaration_signature": "MEYCIQcxe...",
  "agent_did": "did:web:agent.enterprise.com"
}
```

The agent applies standard Base64 encoding to this JSON object for safe transmission over HTTP.

### Step 4: Resolution and Ledger Verification

The agent resubmits the request with the encoded credential bundle in the `Authorization` header.

```http
GET /api/v2/research-datasets/402 HTTP/1.1
Host: data-provider.com
Authorization: Agent-Identity eyJmdWxmaWxsX3VzZXJfc3NvIjp7Imlzc3VlciI6Im9rdGEiLC...
Accept: application/json
```

The receiver decodes the JSON string, reads the explicitly declared `issuer` for each parameter, and extracts the corresponding public keys from a shared decentralized ledger or the issuer's `.well-known` URL. It then **mathematically verifies the signatures** without requiring a live, centralized runtime call to the identity vendors.

## 4. Open Challenges for the Community Group

This draft presents the foundational handshake. Collaboration is invited on the following open implementation questions:

1. **Schema Governance** — Establishing a centrally governed, append-only registry (similar to the IANA HTTP header registry) to define and version standard parameter keys such as `require_is_human`.

2. **HTTP Header Size Limits** — Designing fallback mechanisms (e.g., a `challenges_url` token-exchange endpoint) for scenarios where bundled credentials exceed standard 8KB server header limits.

3. **Session Security** — Standardizing anti-replay mechanisms, such as cryptographic nonces bound to the receiver's challenge.

## 5. Conclusion & Next Steps

By anchoring agentic identity in a receiver-driven protocol, the web can maintain native security boundaries without succumbing to the fragmentation of proprietary registries. A centrally maintained parameter schema and standardized JSON structures eliminate parsing risks while enabling extreme granularity in trust delegation.

We invite members of the **W3C AI Agent Protocol Community Group** to collaborate on standardizing the append-only `Agent-Identity` schema registry, establishing a governance framework for federated ledger roots, and defining nonce-binding and anti-replay standards for the credential bundle.

## References

[x402 Payment Protocol](https://x402.org) — architectural analogy for receiver-driven payment negotiation

[IANA HTTP Authentication Scheme Registry](https://www.iana.org/assignments/http-authschemes/)

[W3C Verifiable Credentials Data Model](https://www.w3.org/TR/vc-data-model/)

[W3C Decentralized Identifiers (DIDs) v1.0](https://www.w3.org/TR/did-core/)

[IETF WIMSE Working Group](https://datatracker.ietf.org/wg/wimse/about/)

[IETF draft-klrc-aiagent-auth-00](https://datatracker.ietf.org/doc/draft-klrc-aiagent-auth/)

*Licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Contributions welcome via pull request.*
