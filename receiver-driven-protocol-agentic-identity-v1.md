# A Receiver-Driven Protocol for Agentic Identity (x402 for Identity)

## Abstract

As AI agents evolve from internal enterprise automations to cross-platform economic actors navigating the open web, the infrastructure for establishing trust has failed to keep pace. Current frameworks attempt to solve agentic identity through preemptive, fragmented developer registrations, creating siloed ecosystems and insurmountable friction. This whitepaper proposes a paradigm shift: a parameter-driven protocol where host servers dynamically dictate identity and authorization requirements at the exact moment of interaction, leveraging HTTP `401 Unauthorized` and `WWW-Authenticate` headers paired with agent-aggregated, Base64-encoded Verifiable Credentials.

## Introduction: The Identity Bottleneck

The agentic economy remains fundamentally gated by trust. Historically, there has been no universal way to verify a human or business, forcing organizations to rely on fragmented vendors for individual needs.

The industry's current trajectory replicates this flaw by demanding preemptive developer registration on proprietary portals, such as those operated by Mastercard, Visa, Vouched, and Skyfire. This model introduces severe limitations:

* **Developer Fragmentation:** Creators must maintain independent registrations across multiple networks to operate globally.
* **Receiver Overhead:** Instead of a federated trust standard, merchants or receivers must maintain well-known URLs or local registries for the specific agent platforms they want to allow through.

To scale, the agentic web must abandon static, pushed registrations. It requires a protocol where identity requirements are pulled dynamically by the receiver at the moment of interaction.

## The Use-Case Spectrum & Parameterized Demands

Agentic risk is highly non-uniform. Because there will be more use cases, these are just some examples of fragmented use cases and fragmented risks:

* **Unrestricted Content Scraping:** An agent fetching content which is currently not restricted requires zero end-user or business verification.
* **Gated Content Access:** An agent fetching content which is accessible only via email sign in may require an Enterprise Single Sign-On (SSO) token from the end user, alongside a legally binding declaration from the business that the data will not be repurposed maliciously.
* **High-Value/Age-Gated E-Commerce:** An agent purchasing restricted goods (e.g., laptops with chargeback risk, or age-gated products like cigarettes) requires user SSO, proof that there is a human in the loop, age verification, and a cryptographically verified business entity (KYB) of the entity who made the agent to allocate liability.
* **Regulated Financial Operations:** An agent opening a bank account on behalf of an end user represents the highest risk tier, demanding full payload disclosure of user KYC and face biometrics, alongside the agent's KYB. The receiving agent must also prove its own regional regulatory compliance (e.g., GDPR, financial certifications) before the user's agent can trust it.

Because these nuances cannot be flattened into rigid identity tiers (like NIST's IALs), the protocol must allow receivers to request specific, standalone parameters on demand, analogous to how the `x402` protocol dynamically dictates required payment addresses, currencies, and amounts.

## The Proposed Protocol Flow & Specifications

The protocol operates as a dynamic handshake where the host server dictates terms, and the agent acts as the primary orchestrator to aggregate and present the necessary proofs.

To ensure system stability, predictability, and universal understanding, the protocol relies on two architectural pillars:

1. **A Governed Parameter Schema:** To prevent an arbitrary, chaotic proliferation of parameters, keys (e.g., `require_user_sso`, `require_is_human_above_18`) must be defined by mutual agreement across the ecosystem. They will be maintained in a centrally governed, append-only schema registry (similar to the IANA HTTP header registry) so that all agents natively understand what a requested parameter means.
2. **Strict JSON Encapsulation:** The protocol enforces a strict `require_` (in the challenge) and `fulfill_` (in the response) mapping, encapsulated as a Base64-encoded JSON object.

### Step 1: The Initial Request

The agent attempts an action (e.g., downloading a licensed dataset) without providing identity context.

```http
GET /api/v2/research-datasets/402 HTTP/1.1
Host: data-provider.com
User-Agent: AI-Agent/2.0
```

### Step 2: The Receiver-Driven Challenge (`401 Unauthorized`)

The server denies the request, responding with an HTTP `401 Unauthorized` status. It includes a custom `WWW-Authenticate: Agent-Identity` header outlining the exact parameter menu required, defining trusted issuers directly at the parameter level.

```http
HTTP/1.1 401 Unauthorized
Date: Fri, 05 Jun 2026 13:50:00 GMT
WWW-Authenticate: Agent-Identity 
  realm="research_docs",
  require_user_sso="true; trusted_issuers='okta, pingidentity, entra, auth0, forgerock, onelogin'",
  require_developer_kyb="true; trusted_issuers='socure, onfido, sumsub, trulioo, jumio, stripe, veriff, idnow'",
  require_declaration="The bearer of this token agrees that data retrieved from this endpoint will not be ingested into LLM training sets or used for public scraping replication."
Content-Type: application/json

{
  "error": "identity_required",
  "message": "Dynamic agent credential aggregation required for this endpoint."
}
```

### Step 3: Agent-Led Token Aggregation (Explicit JSON Packaging)

The agent parses the challenge. Programmatically communicating with the respective identity issuers, it constructs a structured JSON object. Crucially, the agent explicitly defines which issuer fulfilled the request, allowing the receiver to verify the token without guesswork:

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

The agent then applies standard Base64 encoding to this JSON object to ensure safe, unbroken transmission over HTTP.

### Step 4: Resolution and Ledger Verification

The agent resubmits the request, attaching the Base64-encoded JSON string in the `Authorization` header.

```http
GET /api/v2/research-datasets/402 HTTP/1.1
Host: data-provider.com
Authorization: Agent-Identity eyJmdWxmaWxsX3VzZXJfc3NvIjp7Imlzc3VlciI6Im9rdGEiLC...
Accept: application/json
```

The receiver decodes the JSON string. Reading the explicitly declared `issuer` for each parameter, it extracts the corresponding public keys from a shared decentralized ledger or the issuer's well-known URL. It mathematically verifies the signatures within the bundle without requiring a live, centralized runtime call to the identity vendors, processing the transaction instantly.

## Challenges for the Community Group

This draft presents the foundational handshake for a receiver-driven protocol. And, it would be great to collaborate on the following open implementation challenges:

1. **Schema Governance:** Establishing a centrally governed, append-only registry (similar to the IANA HTTP header registry) to define standard parameter keys (e.g., `require_is_human`).
2. **HTTP Header Size Limits:** Designing fallback mechanisms (such as token exchange endpoints) for scenarios where bundled credentials exceed standard 8KB server header limits.
3. **Session Security:** Standardizing anti-replay mechanisms, such as cryptographic nonces bound to the receiver's challenge.
