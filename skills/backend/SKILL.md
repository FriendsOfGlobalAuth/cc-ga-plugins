---
name: backend
description: "TxGlobalAuth (GlobalAuth, GA) backend integration. Use when validating TxGlobalAuth JWT tokens on the server, decoding session/person data, building auth verify endpoints, or working with Kratos token introspection in Node.js, NestJS, Express, Go, or any backend."
---

# Global Auth - Backend Integration Skill

This skill covers backend integration with TxGlobalAuth — specifically JWT token validation, session data extraction, and anti-patterns that lead to broken integrations.

**Note**: TxGlobalAuth is a frontend widget. There is no backend SDK. Backend integration consists of validating JWTs issued by the widget and extracting user data from them.

**Source of truth for data structures**: Generated Go code from proto definitions lives in the Kratos repository at `globalauth/protobuf/modules/proto/`. The original `.proto` source files are maintained in a separate internal proto repository — the generated `.pb.go` files reference their source paths (e.g. `proto/common/person.proto`). Key generated files:
- `common/person/person.pb.go` — Person message (source: `proto/common/person.proto`)
- `txauth/common/session/session.pb.go` — Session, EdoxFirm, UserType (source: `proto/txauth/common/session/session.proto`)
- `txauth/common/provider/provider.pb.go` — AuthSource, AuthProvider (source: `proto/txauth/common/provider/provider.proto`)

**Proto file for TxAuth JwtAgent integration**: The proto definition for `txAuth.JwtAgent` service is located in the corporate Bitbucket repository at `projects/SER/repos/proto-repo/browse/grpc-txauth/src/main/proto/grpc/txauth/jwt_agent.proto`. Use this proto file to generate gRPC client stubs for token validation via the TxAuth agent.

## Token Validation — Required

**Every backend that receives a TxGlobalAuth JWT must validate it.** Never trust a token without validation — the frontend is an untrusted environment.

There are two approaches to validation. **The choice depends on network topology, not programming language:**

- **Inside the network perimeter** (backend has network access to TxAuth server) → use Approach 1 (gRPC). The TxAuth agent runs on each backend instance for minimal latency. **Always prefer this when available.**
- **Outside the network perimeter** (backend cannot reach TxAuth server) → use Approach 2 (HTTP). This is the only option for externally hosted backends.

### Approach 1: TxAuth Agent (gRPC Token Introspection)

Call the TxAuth agent service directly via gRPC. This is the lowest-level validation method — Approach 2 (HTTP) is actually a wrapper around this.

**gRPC service**: `grpc.txauth.JwtAgent`
**RPC method**: `Check`

**Request** (`JwtRequest`):
```protobuf
message JwtRequest {
    string token = 1;  // The JWT string from the frontend
}
```

**Response** (`JwtResponse`):
```protobuf
message JwtResponse {
    google.rpc.Status status = 1;              // Status code 0 = success
    session.Session session = 2;               // Decoded session with person data
    session.SessionContext context = 3;         // Session context (provider config, token metadata)
    provider.AuthProvider provider = 4;         // Authentication provider info
}
```

The `session.Session` contains the `person` field with the same structure described in "Session Structure by Token Type" below.

**Service discovery**: TxAuth agents are typically registered in Consul. The service name and datacenter are configured per variant (e.g., `dev-glaue1-txauth-agent` in datacenter `dc-dev`). Consult your infrastructure team for the service name for your environment.

**Deployment model**: The TxAuth agent is designed to run **on each backend instance** (sidecar or co-located process) for minimal latency. The agent connects to the TxAuth server and caches validation results. This is why gRPC is preferred inside the perimeter — calls to the local agent are sub-millisecond.

**Authentication**: No bearer tokens — gRPC calls to the TxAuth agent rely on network-level isolation (internal network, Consul datacenter). The agent is not exposed to the public internet.

**Variants**: TxAuth supports multiple variants (e.g., `finam`, `lime`) — different TxAuth deployments for different product families. Use `CheckJwtByVariant(token, variant)` to validate against a specific variant, or `CheckJwt(token)` to check all configured variants.

**When to use**: When your backend is **inside the network perimeter** with access to the TxAuth server/agent infrastructure. This works for any language with gRPC support (Go, Java, Node.js, Python, etc.) — generate client stubs from the proto file. This approach supports `appName`-based cross-product token restriction (see "Cross-Product Token Validation" below).

**Proto source**: The proto definition is in the corporate Bitbucket repository at `projects/SER/repos/proto-repo/browse/grpc-txauth/src/main/proto/grpc/txauth/jwt_agent.proto`. Generate gRPC client stubs from this file for your language.

### Approach 2: Kratos JWT Introspection (HTTP)

Validate the token through the Kratos HTTP introspection endpoint. Use this when the backend is **outside the network perimeter** and cannot reach the TxAuth agent directly.

> **Architecture note**: The HTTP `/api/jwt/introspect` endpoint is hosted by Kratos itself and internally calls the gRPC TxAuth agent (Approach 1). It abstracts away service discovery, Consul, and gRPC transport — at the cost of higher latency compared to a local agent.

**Endpoint**: `POST https://ga.{domain}/api/jwt/introspect`

Production URLs by CDN region:
- CIS: `https://ga.finam.ru/api/jwt/introspect`
- International: `https://ga.lime.co/api/jwt/introspect`

Ask your infrastructure team which base URL to use — it must match the environment where the frontend widget is configured.

**Request**:
```http
POST /api/jwt/introspect HTTP/1.1
Content-Type: application/json
Accept: application/json
Authorization: Bearer {SERVICE_TOKEN}

{ "token": "<JWT string from frontend>" }
```

- `SERVICE_TOKEN` — a server-to-server (S2S) token that authenticates your backend to the introspection service. Obtain it from your infrastructure/DevOps team. Store it as a secret (environment variable, vault) — never commit to source control.

**Response** (success):
```json
{
    "active": true,
    "person": {
        "id": "55ca4536-...",
        "kratos_id": "e9b9aba4-...",
        "name": "John",
        "lastname": "Doe",
        "email": "jdoe@example.com",
        "login": "jdoe"
    }
}
```

> **Naming convention difference**: The introspection endpoint returns `kratos_id` (snake_case). The frontend widget's `session.person` uses `kratosId` (camelCase). These are the same value — just different naming conventions between the HTTP API and the JavaScript widget.

**Response** (invalid/expired token):
```json
{
    "active": false
}
```

**Validation logic**:
1. Check HTTP status — non-2xx means introspection service is unreachable
2. Check `active === true` — if `false` or missing, reject the token
3. Check `person` object exists — if missing, reject
4. Extract identity from `person` (see "person.id Semantics" below)

**Error handling**: Set a reasonable timeout (5–10 seconds). If the introspection service is down, fail closed (reject the request) — do not fall back to local JWT parsing.

**Environment variables** (typical naming):
```
GA_INTROSPECT_URL=https://ga.lime.co/api/jwt/introspect
GA_SERVICE_TOKEN=<obtain from infrastructure team>
```

Both approaches validate the token signature and expiration, decompress the `sess` field, and return the decoded session data including `person`. The difference is network topology: gRPC (Approach 1) for backends inside the perimeter with TxAuth agent access, HTTP (Approach 2) for backends outside the perimeter.

## JWT Structure — What the Token Actually Contains

### Top-Level Claims

All TxGlobalAuth JWTs share the same top-level claim structure regardless of provider:

```
area, parent, scontext, zipped, created, renewExp, sess, iss,
keyId, firebase, secrets, provider, scope, tstep, spinReq, exp, spinExp, jti
```

**There is no `email`, `first_name`, `last_name`, `sub`, or `user_id` at the top level.** This is not a standard OIDC token. User data lives inside the `sess` field.

Key top-level claims:
- `provider` — provider name string (e.g. `"AD_OFFICE"`, `"KRATOS"`, or a product-specific L2 provider name)
- `scontext` — session context with provider config, token type, expiration settings
- `scope` — token scope (may be empty `{}` for L1, may contain content flags for L2)
- `firebase` — nested Firebase JWT (present for AD and Kratos L1, empty string for L2)
- `zipped` — always `true`, indicates `sess` is gzip-compressed

### The `sess` Field Is a Compressed JSON Object

The JWT has `"zipped": true` — the `sess` payload is gzip-compressed. When you decode the JWT on the backend (e.g., `jwt.decode()` in Node.js), the `sess` field appears as a binary/base64 blob, **not** as a structured object.

**Do not attempt to read `sess` fields without decompression.** Use one of the validation approaches above — they handle decompression and return structured data.

### The `firebase` Field Is a Nested JWT

The `firebase` claim contains a separate Firebase JWT. When present (non-empty), its only useful field is `uid`, which equals `person.id`. Present for AD and Kratos L1 tokens, empty string for L2 tokens.

## Session Structure by Token Type

The decompressed `sess` object varies significantly between provider types and authentication levels. **Do not assume all fields are always present — most fields outside of `person.id` are optional and may be absent depending on the provider and auth level.**

### AD Tokens (`AD_OFFICE`, `AD_OFFICE_OTP`, `AD_OFFICE_SMS`, `AD_WTE`)

```json
{
    "person": {
        "id": "55ca4536-...",
        "name": "John",
        "lastname": "Doe",
        "middlename": "...",
        "email": "jdoe@corp.example.com",
        "login": "jdoe"
    },
    "userType": "EMPLOYEE_USER_TYPE"
}
```

- `person.id` — LDAP/AD GUID (likely objectGUID). **Not a Kratos ID.**
- `person.kratosId` — absent
- `person` fields — typically rich (name, lastname, email, login). Exact set may vary across AD configurations.
- `userType` — `"EMPLOYEE_USER_TYPE"`
- No `permissions`, `firms`, `segments`, or trading-related data

AD provider variants:
- `AD_OFFICE` — standard AD (username + password)
- `AD_OFFICE_OTP` — AD with OTP as second factor
- `AD_OFFICE_SMS` — AD with SMS as second factor
- `AD_WTE` — separate domain controller (no 2FA at this time)

### Kratos L1 Tokens (`KRATOS` and similar L1 providers)

```json
{
    "person": {
        "id": "e9b9aba4-...",
        "email": "user@example.com",
        "kratosId": "e9b9aba4-..."
    },
    "permissions": [ ... ],
    "segments": ["employees"],
    "subscriptionTariff": "PRO"
}
```

- `person.id` **equals** `person.kratosId` — both are the Kratos identity UUID
- `person` fields — **minimal**: often only `id`, `email`, `kratosId`. Name, lastname, login, phone may be **absent**.
- `userType` — may be absent
- `permissions`, `segments`, `subscriptionTariff` — may be present

Multiple L1 provider names exist depending on product configuration and region. All share the same session structure.

### Kratos L2 Tokens (product- and region-specific L2 providers)

```json
{
    "person": {
        "id": "4337d6dc-...",
        "name": "John",
        "lastname": "Doe",
        "middlename": "...",
        "email": "user@example.com",
        "login": "jdoe",
        "verifiedPhone": "+1(312) 555-0690",
        "kratosId": "e9b9aba4-..."
    },
    "userType": "COMMON_USER_TYPE",
    "authenticationId": "123249",
    "permissions": [ ... ],
    "firms": [
        { "id": 1, "ratingLegacy": "...", "managerId": "176786", "manager": "..." }
    ],
    "crmId": "156cd68c-...",
    "edoxVerified": true,
    "segments": ["employees"],
    "subscriptionTariff": "PRO"
}
```

- `person.id` **differs from** `person.kratosId` — `id` is a trading/client account identifier, `kratosId` is the Kratos identity UUID
- `person` fields — rich (name, lastname, email, login, phone, middlename)
- `userType` — `"COMMON_USER_TYPE"` for regular clients
- `firms` — EDOX firm associations (trading accounts)
- `authenticationId`, `crmId`, `edoxVerified` — trading/client account metadata
- L2 providers require 2FA — `scontext.provider.tfaRequired` is a **provider-level configuration flag** meaning "this provider requires 2FA". It is not a token state. If you receive a valid L2 token, 2FA has already been completed — the widget does not issue L2 tokens until 2FA passes. See **"Determining 2FA Status from the JWT"** section below for how to programmatically check 2FA completion on the backend.

Multiple L2 provider names exist depending on product and region. All share the same session structure.

### person.id Semantics — Critical

**The meaning of `person.id` changes depending on the provider:**

| Provider Type | `person.id` | `person.kratosId` |
|---------------|-------------|-------------------|
| AD | LDAP/AD GUID | absent |
| Kratos L1 | = kratosId (same value) | Kratos UUID |
| Kratos L2 | Trading/client account ID | Kratos UUID |

**`person.kratosId` (camelCase in decoded token / `kratos_id` snake_case in introspection API response) is the stable user identity** across authentication levels. Think of it as the primary identity — everyone who authenticates through Kratos gets one. `person.id` in L2 is a secondary identifier (trading account) that the user acquires when they become a client. For AD users, `person.id` (AD GUID) is the only identifier since they exist outside the Kratos identity system.

When correlating users across sessions or linking to internal databases:
- For Kratos users (L1 and L2) — use `person.kratosId` as the stable identity. `person.id` may change between L1 and L2 for the same user.
- For AD users — use `person.id` (the AD GUID). These users don't have a `kratosId`.

## Determining 2FA Status from the JWT

The JWT contains a top-level claim `tfa` that indicates the **current state** of two-factor authentication:

- **`tfa` claim is present** (not null) → 2FA is **required but not yet completed**. The token was issued at a stage where the user still needs to pass a second factor. In normal operation, the TxGlobalAuth widget does **not** expose this intermediate token to the consuming application — it completes 2FA internally before issuing the final token via callbacks. If your backend receives a token with `tfa` present, treat it as an anomaly.
- **`tfa` claim is absent** (null/missing) → 2FA has been **successfully completed** (or was not required by this provider). The token is fully authenticated.

The internal structure of `tfa` (channel, address, etc.) is used by the widget itself to render the 2FA input UI — it is not relevant for backend integrators.

**How to determine if a token was issued with 2FA:**

Check `scontext.provider.tfaRequired` — if `true`, the token was issued under a provider configured to require 2FA. A small number of users may have individual exemptions from 2FA, but this is transparent to the consuming product — the token is valid regardless.

**Important**: `tfaRequired` reflects the provider configuration for the **specific environment**. On DEV/TST environments, 2FA is often disabled even for providers that require it in production. Do not rely on `tfaRequired` values from non-production environments when reasoning about production security.

**If you need cross-product 2FA enforcement** (e.g., ensuring a token from a non-2FA product is not accepted by a 2FA-requiring product), this is an advanced scenario — **consult the Global Auth support team**. The standard validation via TxAuth Agent with `appName` already covers most such cases.

### Cross-Product Token Validation (`appName`)

When validating via TxAuth Agent (gRPC `CheckJwt` / `CheckJwtByVariant`), the `appName` parameter restricts token acceptance to your specific product. If a token was issued for a different product, validation will fail. This covers various scenarios — from 2FA policy differences between products to simply ensuring your API only accepts tokens issued for it.

**Recommendation**: If you see a `CheckJwt` / `CheckJwtByVariant` call without `appName` — suggest adding it. Historically, `appName` validation did not exist and all tokens within the ecosystem were accepted regardless of which product issued them. The `appName` parameter is a newer security feature that limits token scope to the intended product. Unless the API intentionally serves multiple products (a shared backend accepting tokens from several frontends), `appName` should be set.

## Anti-Patterns

### `sess` Is NOT a Kratos Session Token

**Wrong**: Sending `sess` to Kratos `/sessions/whoami` endpoint as a session token.

`sess` is a compressed JSON object containing the full session with person data. It is not a Kratos session identifier. Sending it to whoami will fail or return unexpected results.

### Do NOT Parse JWT Claims for User Data

**Wrong**: Decoding the JWT and looking for `email`, `first_name`, `sub`, or similar standard claims at the top level.

TxGlobalAuth JWTs do not follow standard OIDC claim naming. User data is inside `sess.person` (compressed). Use the validation service to extract it.

### Do NOT Assume Session Fields Are Always Present

**Wrong**: Accessing `sess.person.name`, `sess.userType`, `sess.firms`, etc. without checking if they exist.

The session structure varies dramatically by provider and auth level. Kratos L1 tokens may contain only `person.id`, `person.email`, and `person.kratosId` — with no name, lastname, login, phone, userType, firms, or other fields. Always treat every field except `person.id` as optional.

### Do NOT Use person.id as a Stable Cross-Level Identifier

**Wrong**: Storing `person.id` from an L1 token and expecting it to match `person.id` from an L2 token.

For the same user, `person.id` in L1 equals their kratosId, but `person.id` in L2 may be a different identifier (trading account). Use `person.kratosId` as the stable identity.

### Do NOT Rely on Kratos whoami as Primary Data Source

The Kratos whoami endpoint (`/sessions/whoami`) may be unreachable from your backend due to:
- IP filtering (Docker containers, cloud environments, non-VPN networks)
- Geographic restrictions
- Session token format mismatches

Use whoami only as a supplementary check, not as the primary source of user data. The token validation approaches above are the reliable path.

### Do NOT Skip Validation for "Trusted" Frontends

Even in internal enterprise applications, always validate the JWT. The token arrives from the browser — it can be tampered with, replayed, or expired. Validation is not optional.

## Common Integration Pattern: Token Exchange

The standard pattern for integrating TxGlobalAuth with a backend is **token exchange** — the frontend sends the GA token, the backend validates it, extracts user data, and issues its own session (e.g., an internal JWT or a server-side session).

### Flow

```
Frontend                          Backend                         TxAuth Agent
   │                                │                                │
   │  POST /api/auth/login          │                                │
   │  { token: "ga-jwt-string" }    │                                │
   │ ──────────────────────────────>│                                │
   │                                │  Validate GA token             │
   │                                │ ──────────────────────────────>│
   │                                │  { active: true, person: {...}}│
   │                                │ <──────────────────────────────│
   │                                │                                │
   │                                │  Extract person.id, person.login, etc.
   │                                │  Find or create internal user
   │                                │  Issue internal session/JWT
   │                                │                                │
   │  { internalToken: "...",       │                                │
   │    user: { id, name, ... } }   │                                │
   │ <──────────────────────────────│                                │
```

### Backend Pseudocode (framework-agnostic)

```
function handleLogin(request):
    gaToken = request.body.token
    if not gaToken:
        return 401 "Missing token"

    // Step 1: Validate GA token via introspection
    introspectionResult = validateGAToken(gaToken)  // see Approach 1 or 2 above
    if not introspectionResult.active:
        return 401 "Invalid or expired token"

    // Step 2: Extract identity
    person = introspectionResult.person
    // For Kratos users: person.kratos_id is the stable identity
    // For AD users: person.id (AD GUID) is the identity
    // Note: introspection API uses snake_case (kratos_id), not camelCase (kratosId)
    stableId = person.kratos_id ?? person.id

    // Step 3: Find or create internal user
    internalUser = findUserByExternalId(stableId)
    if not internalUser:
        internalUser = createUser({
            externalId: stableId,
            email: person.email,
            name: person.name,
            login: person.login
        })

    // Step 4: Issue internal session
    internalToken = issueJWT({ userId: internalUser.id, ... })
    return { token: internalToken, user: internalUser }
```

### Key Points

- **The frontend never stores or forwards the raw GA token for subsequent API calls** — after the exchange, it uses the internal token issued by your backend.
- **The exchange endpoint is the only place** where the GA token is validated. All other backend endpoints validate the internal token using your own middleware.
- **User matching**: Use `person.kratosId` (Kratos) or `person.id` (AD) as the external identity key. See "person.id Semantics" above.
- **Do not skip the exchange**: Passing the raw GA token to every API call and validating it every time is inefficient and couples your entire backend to the TxAuth introspection service.

## Receiving Data from the Frontend

The frontend widget decodes the compressed `sess` field client-side and provides structured `session.person` data in its callbacks (see `global-auth:frontend` skill).

For **non-critical display data** (names, email for UI display), the frontend may pass `session.person` fields alongside the token for convenience. This is acceptable because:
- The token itself is validated independently on the backend
- Display fields like name/lastname are not used for authorization decisions
- The widget has already decompressed the data client-side

For **authorization decisions** (permissions, access control, user identity), always extract data from the validated token on the backend side.

## Data Not Available from TxGlobalAuth

Regardless of provider type, the following are **never** included in TxGlobalAuth tokens:
- Job title / position
- Department / organizational unit
- Office location
- Manager hierarchy
- Employee expertise / skills

These require direct LDAP/AD queries or separate HR system integration on the backend.
