---
name: backend
description: "TxGlobalAuth (GlobalAuth, GA) backend integration. Use when validating TxGlobalAuth JWT tokens via TxAuth Agent (gRPC) or HTTP introspect, decoding session/person data, deploying TxAuth agent, setting up gRPC token validation, building auth verify endpoints, or working with Kratos token introspection in Go, PHP, Node.js, NestJS, Express, Java, Python, or any backend."
---

# Global Auth - Backend Integration Skill

This skill covers backend integration with TxGlobalAuth — specifically JWT token validation, session data extraction, and anti-patterns that lead to broken integrations.

**Note**: TxGlobalAuth is a frontend widget. There is no backend SDK. Backend integration consists of validating JWTs issued by the widget and extracting user data from them.

**Source of truth for data structures**: Generated Go code from proto definitions lives in the Kratos repository at `globalauth/protobuf/modules/proto/`. The original `.proto` source files are maintained in a separate internal proto repository — the generated `.pb.go` files reference their source paths (e.g. `proto/common/person.proto`). Key generated files:
- `common/person/person.pb.go` — Person message (source: `proto/common/person.proto`)
- `txauth/common/session/session.pb.go` — Session, EdoxFirm, UserType (source: `proto/txauth/common/session/session.proto`)
- `txauth/common/provider/provider.pb.go` — AuthSource, AuthProvider (source: `proto/txauth/common/provider/provider.proto`)

**Proto file for TxAuth JwtAgent integration**: The proto definition for `txAuth.JwtAgent` service is located in the corporate Bitbucket repository at `projects/SER/repos/proto-repo/browse/grpc-txauth/src/main/proto/grpc/txauth/jwt_agent.proto`. Use this proto file to generate gRPC client stubs for token validation via the TxAuth agent.

## Probing Questions — Ask These First

**Before answering any backend integration question, you usually do not have enough context.** Most TxGlobalAuth backend questions ("how do I get the email", "why does my call return 403", "how do I store the user") have completely different answers depending on a small set of variables that the developer rarely volunteers up front. Ask actively — do not guess.

**The minimum context you need:**

1. **`appName` and authentication level** — is the product configured for Kratos L1, Kratos L2, AD, or Anonymous? The `appName` determines this. The JWT structure, which `person` fields are present, whether `kratosId` exists, and whether `get-identity` is needed at all — all depend on this. **A question about "user data from the JWT" cannot be answered without knowing the level.** See "Data Availability by Provider and Auth Level" below.

2. **Network position** — is the backend **inside the corporate network perimeter** (can reach TxAuth infrastructure / Consul) or **outside** (e.g. external cloud, no VPN)? This determines whether the correct approach is TxAuth Agent (gRPC, Approach 1) or HTTP introspect (Approach 2). Inside the perimeter using HTTP introspect is an anti-pattern — see below.

3. **What data the backend actually needs and why** — name/email "for display in the personal cabinet" has a very different correct answer than "as the primary key in our user table" or "for sending transactional email". The first should not even hit the backend (frontend `session.person` already has it for L2/AD). The second should use `kratosId` as the key, not the contact data. The third is the only case that genuinely needs `get-identity`.

4. **Where the data will be used** — display-only / business logic / audit log / matching / outbound communication. Each has a different correct pattern. In particular: **never use stored PII (login, email, phone) as a matching key** (see "Storing User Data — Trade-offs" below).

**Heuristic flow when a developer asks "how do I get email/name/phone on the backend":**

```
What is the appName / level?
├─ AD → person fields are in the token; introspect/agent returns them. No get-identity (no kratosId).
├─ Kratos L2 → all relevant person fields are in the token. introspect/agent is enough. No get-identity needed.
├─ Kratos L1 → email/name/phone are NOT in the token. You need:
│   ├─ Display only → use frontend session.person (already decoded in widget) — do not fetch from backend
│   ├─ Server-side use → introspect to get kratosId → call get-identity with kratosId
│   │   (get-identity is a SEPARATE S2S privilege from introspect — submit a ticket to project GA)
│   └─ Just verifying user has email/phone → use frontend requireEmail()/requirePhone() instead
└─ Anonymous → no user data exists; only deviceUUID
```

**If you cannot answer these questions yet, ask them — do not produce a generic answer that risks being wrong for the developer's actual configuration.**

## Token Validation — Required

**Every backend that receives a TxGlobalAuth JWT must validate it.** Never trust a token without validation — the frontend is an untrusted environment.

There are two approaches to validation. **Always prefer Approach 1 (TxAuth Agent via gRPC)** — Approach 2 (HTTP) exists only for backends that physically cannot reach TxAuth infrastructure:

- **Inside the network perimeter** → **Approach 1 (gRPC) is the standard.** Each project deploys its own TxAuth agent instance(s) — a local sidecar that validates tokens with sub-millisecond latency and zero external HTTP calls. If your backend is inside the perimeter and uses HTTP introspect, this should be migrated to TxAuth Agent.
- **Outside the network perimeter** (no network path to TxAuth infrastructure) → Approach 2 (HTTP). This is the **only** valid reason to use HTTP introspect.

### Approach 1: TxAuth Agent (gRPC Token Introspection)

Call the TxAuth agent service directly via gRPC. This is the lowest-level validation method — Approach 2 (HTTP) is actually a wrapper around this.

**gRPC service**: `grpc.txauth.JwtAgent`
**RPC method**: `Check`

**Request** (`JwtRequest`):
```protobuf
message JwtRequest {
    string token = 1;              // The JWT string from the frontend
    string expected_app_name = 2;  // Optional: restrict to specific product (see "Cross-Product Token Validation")
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

**Deployment model**: There is **no communal/shared TxAuth agent**. Each project must deploy its own agent instance(s). The agent runs as a local sidecar or co-located process on each backend instance. It connects upstream to the TxAuth server, caches validation results, and provides sub-millisecond local validation. This is why gRPC is the standard inside the perimeter — calls to the local agent have near-zero latency.

**How to get your own TxAuth agent deployed:**
1. **Submit a support ticket** to the technical department to provision TxAuth agent instances for your project
2. **Deploy a Consul agent** alongside your TxAuth agents — the TxAuth agent **must** register itself in Consul (this is an architecture requirement from the TxAuth Server team, used for agent monitoring). You will need a Consul token for registration.
3. **Configure your backend** to connect to your local agent's gRPC endpoint (typically `localhost:<port>` or a sidecar address within your deployment)

> **This is an infrastructure process, not just code.** Plan for agent provisioning early in the project timeline. The code integration is straightforward once agents are deployed — see "Integration Setup" below.

> For background reading: internal wiki pages on "TXAuth Agent (Linux)" cover agent installation details.

**Service discovery**: TxAuth agents register in Consul for monitoring purposes. To browse currently running agents: open Consul UI at `https://consul.entapp.work/ui/` → select the datacenter for your environment → search for `*-txauth-agent` (e.g., `prd-*-txauth-agent` for production, `dev-*` for development).

**Authentication**: No bearer tokens required for gRPC calls — the TxAuth agent relies on network-level isolation (internal network, Consul datacenter). The agent is not exposed to the public internet.

**Variants**: TxAuth supports multiple variants (`spc`, `mma`, `finam`) — different TxAuth server deployments for different product families. The variant determines which upstream TxAuth server the agent connects to. When deploying your agent, specify the correct `-dc` (datacenter) and `-remote-service` for your variant and environment (see table below). In PHP, wrapper libraries provide convenience methods: `CheckJwtByVariant(token, variant)` to validate against a specific variant, or `CheckJwt(token)` to check all configured variants. For the raw gRPC API, the variant is determined by which agent instance you connect to — each agent is configured for a specific variant/environment.

**When to use**: When your backend is **inside the network perimeter**. This works for any language with gRPC support (Go, Java, Node.js, Python, PHP, etc.) — generate client stubs from the proto file or use existing libraries. This approach supports `appName`-based cross-product token restriction via the `expected_app_name` field (see "Cross-Product Token Validation" below).

**Proto source**: The proto definition is in the corporate Bitbucket repository at `projects/SER/repos/proto-repo/browse/grpc-txauth/src/main/proto/grpc/txauth/jwt_agent.proto`. Generate gRPC client stubs from this file for your language.

#### Environment and Variant Reference

The table below is a **reference snapshot** (as of April 2026) — datacenter and remote-service values may change over time. Verify with the Tx Auth team or Consul UI before configuring a new deployment.

| Environment | TxGlobalAuth `env` | Variant | Datacenter (`-dc`) | Remote Service (`-remote-service`) |
|---|---|---|---|---|
| DEV, TST | `dev` / `tst` | `spc`, `mma`, `finam` | `dc-dev` | `dev-ftcore-txauth-server` |
| PrePROD (UAT) | `uat` | `spc` | `dc-pp` | `pp-ftr03-txauth-server` |
| PrePROD (UAT) | `uat` | `finam` | `dc-pp` | `pp-ftr01-txauth-server` |
| PrePROD (UAT) | `uat` | `mma` | `dc-dev` | `dev-ftcore-txauth-server` |
| PROD | `prod` | `spc` | `dc-ny` | `prd-ftrr03-txauth-server` |
| PROD | `prod` | `finam` | `dc-tt` | `prd-ftrr01-txauth-server` |
| PROD | `prod` | `mma` | `dc-fr` | `prd-ftrr02-txauth-server` |

> **Note**: The `mma` variant in PrePROD (UAT) uses the DEV TxAuth server — there is no dedicated UAT instance for this variant.

> **Tip**: To find currently running agents in Consul UI (`https://consul.entapp.work/ui/`), select the datacenter from the table above and search for `*-txauth-agent`.

#### Integration Setup

Proto modules with generated gRPC client stubs are published as **zip archives in the corporate Artifactory**. There is currently no package manager integration (no `go get`, no `npm install`) — download, unpack, and configure locally. The exact Artifactory URL depends on the team's infrastructure region — ask the Tx Auth team or check your project's existing Artifactory configuration.

**Go integration** (reference example):

1. Download and unpack proto modules. The artifact to look for is `grpc-txauth-golang-{VERSION}-golang.zip` in Artifactory. The version (e.g. `3.0.4190` as of April 2026) will change over time — check Artifactory for the latest available version.
```bash
# Example — adjust PROTO_URL to your Artifactory instance and latest version
PROTO_VERSION="3.0.4190"
PROTO_URL="https://<your-artifactory>/path/to/grpc-txauth-golang/${PROTO_VERSION}/grpc-txauth-golang-${PROTO_VERSION}-golang.zip"

mkdir -p modules
curl -s "$PROTO_URL" --output protos.zip
unzip -o protos.zip -d modules/
rm protos.zip
```

2. Configure `go.mod` with replace directives pointing to local modules:
```go
replace grpc => ./modules/grpc
replace proto => ./modules/proto

require (
    google.golang.org/grpc v1.58.3
    google.golang.org/protobuf v1.31.0
)
```

3. Create gRPC client and call `Check`:
```go
import (
    "context"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    agent "grpc/txauth/agent"
)

func ValidateToken(ctx context.Context, agentAddr, jwt string) (*agent.JwtResponse, error) {
    conn, err := grpc.NewClient(agentAddr,
        grpc.WithTransportCredentials(insecure.NewCredentials()),
    )
    if err != nil {
        return nil, err
    }
    defer conn.Close()

    client := agent.NewJwtAgentClient(conn)
    return client.Check(ctx, &agent.JwtRequest{
        Token:           jwt,
        ExpectedAppName: "your-app-name",  // recommended — restricts to your product
    })
}
```

4. Validate the response:
```go
resp, err := ValidateToken(ctx, "localhost:4811", jwtToken)
if err != nil {
    // gRPC error — agent unreachable, network issue
    return err
}
if resp.GetStatus() != nil && resp.GetStatus().Code != 0 {
    // Token rejected — invalid, expired, or wrong appName
    return fmt.Errorf("token rejected: %s", resp.GetStatus().Message)
}

// Token valid — extract session data
person := resp.GetSession().GetPerson()
userType := resp.GetSession().GetUserType()
```

The module directory structure after unpacking:
```
modules/
├── grpc/txauth/
│   └── agent/          # JwtAgent service stubs
└── proto/
    ├── common/
    │   ├── account/    # Account message
    │   └── person/     # Person message
    └── txauth/common/
        ├── session/    # Session, SessionContext
        └── provider/   # AuthProvider
```

**Other languages** (PHP, Node.js, Java, Python, etc.): Some languages may have internal wrapper libraries available — check with the Tx Auth team for your stack. For any language with gRPC support, you can generate client stubs from the proto file (`jwt_agent.proto`) using the standard `protoc` compiler with language-specific plugins. The proto source is in the corporate Bitbucket at `projects/SER/repos/proto-repo/browse/grpc-txauth/src/main/proto/grpc/txauth/jwt_agent.proto`.

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

- `SERVICE_TOKEN` — a server-to-server (S2S) token that authenticates your backend. Each S2S method (introspect, get-identity, etc.) is a **separate privilege** — the token only grants access to methods explicitly approved for your product. To request a token or additional method access, submit a ticket to the **GA project** or contact the **Global Auth team**. Store the token as a secret (environment variable, vault) — never commit to source control.

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
GA_SERVICE_TOKEN=<request from Global Auth team via GA project ticket>
```

Both approaches validate the token signature and expiration, decompress the `sess` field, and return the decoded session data including `person`. The difference is network topology: gRPC (Approach 1) for backends inside the perimeter with TxAuth agent access, HTTP (Approach 2) for backends outside the perimeter.

## Introspect vs Get-Identity — What Each Returns

**Both `introspect` and `get-identity` are S2S (server-to-server) endpoints** — called from your backend with a `SERVICE_TOKEN`, never from the browser. Each grants a separate privilege.

> **Do not confuse `get-identity` (S2S, this skill) with `getIdentifiers()` (WEB, in-browser widget method).** They have nothing to do with each other:
> - `get-identity` — backend S2S Kratos API. Returns full identity record (all traits, all verified emails/phones, **unmasked**). Requires `SERVICE_TOKEN` and a separately-granted privilege. Documented in this skill.
> - `getIdentifiers()` — frontend widget method (`TxGlobalAuth.getIdentifiers()`). Returns only verified emails/phones, **masked** (`j***@example.com`), for in-app display. No service token, runs in the browser. Documented in `global-auth:frontend` skill.
>
> They are not interchangeable, not comparable, and serve different audiences. If a developer says "I'll just use `getIdentifiers` from the backend" — that is a category error.

**Introspect returns the data that is already in the token** — validated and decompressed, but nothing more. If `person.email` is absent from the JWT (e.g. Kratos L1 — see "Data Availability" below), introspect will also return no email. Introspect does not query the Kratos database — it only validates and unpacks the token.

**To obtain full user data** (email, phone, name, etc.) when the token doesn't contain it, use the **Kratos S2S API `get-identity`** endpoint. This queries the Kratos identity database directly and returns the complete identity record regardless of what's in the token.

| | Token Introspect (Approach 1 / 2) | Kratos S2S API `get-identity` |
|---|---|---|
| **Input** | JWT string | `person.kratosId` (**only** — not `person.id`) |
| **Available for** | All providers | **Kratos only** (AD and Anonymous have no `kratosId`) |
| **Returns** | Data from token (validated, decompressed) | Full identity from Kratos DB |
| **Email (Kratos L1)** | **absent** (not in token) | array of 0+ verified emails |
| **Phone (Kratos L1)** | **absent** (not in token) | array of 0+ verified phones |
| **Email/Phone (Kratos L2)** | scalar, may be present | array of 0+ verified values |
| **Validates token** | yes (signature, expiry, scope) | no (identity lookup, not token validation) |
| **Use for** | Authentication, session validation | User registration, profile enrichment, contact data |

**Typical flow when you need data not in the token:**
1. Validate JWT via introspect → get `person.kratosId`
2. **Verify `kratosId` is present** — it is absent for AD and Anonymous tokens. If absent, `get-identity` cannot be called.
3. Call Kratos S2S API `get-identity` with `kratosId` → get full identity (emails, phones, logins, etc.)
4. Use the identity data for registration, profile creation, or backend logic

> **Anti-pattern**: Passing `person.id` to `get-identity` instead of `person.kratosId`. For Kratos L2 tokens, `person.id` is a client account GUID — not a Kratos identity ID. The call will fail or return wrong data.

**Access**: `get-identity` uses Bearer token authentication like introspect, but it is a **separate privilege**. Each S2S method is individually granted — having introspect access does not imply `get-identity` access. By default, access is denied. To obtain credentials or request access to additional S2S methods, submit a ticket to the **GA project** or contact the **Global Auth team** directly.

> **Common mistake**: Assuming introspect will return email or phone for all token types. If your backend needs contact data and the product uses Kratos L1, introspect alone is insufficient — you must add a `get-identity` call. Note that `get-identity` returns **arrays** (a user may have multiple verified emails or phones), while the token contains at most one scalar value per field. See "Data Availability by Provider and Auth Level" below.

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

> **Anti-pattern — manual JWT parsing**: Do NOT decode the JWT yourself (`jwt.decode()`, base64 decode + gunzip) and read `sess.person` fields directly. Even if you decompress successfully, this bypasses **all validation**: no signature check, no expiration check, no `appName` scope restriction. A tampered, expired, or stolen token will pass your code silently. The internal `sess` format is also not a stable API — field names, compression, and structure may change without notice. Always use TxAuth Agent (gRPC) or HTTP introspect for token validation and data extraction.

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

**For most current products**, the Kratos L1 token is minimal:

```json
{
    "person": {
        "id": "e9b9aba4-...",
        "kratosId": "e9b9aba4-..."
    },
    "permissions": [ ... ],
    "segments": ["employees"],
    "subscriptionTariff": "PRO"
}
```

- `person.id` **equals** `person.kratosId` — both are the Kratos identity UUID
- `person` fields — **very minimal**: typically only `id` and `kratosId`. **Email, name, lastname, login, phone are absent.**
- `userType` — may be absent
- `permissions`, `segments`, `subscriptionTariff` — may be present

**Kratos L1 special (deprecated)**: Some legacy products use an older provider configuration where `person.email` IS present in L1 tokens. If you encounter existing code that reads `person.email` from L1 tokens successfully, this is likely the case. New integrations should not depend on this behavior.

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

**`person.id` is not a single concept — it is a key under which different identifiers are exposed depending on the provider.** This is the most common source of integration bugs.

| Provider Type | What `person.id` actually is | `person.kratosId` |
|---------------|------------------------------|-------------------|
| AD | LDAP/AD GUID (objectGUID) | absent |
| Kratos L1 | **The same value as `kratosId`** — both fields hold the Kratos identity UUID | Kratos UUID |
| Kratos L2 | A separate identifier — client/trading account GUID (internally referred to as "Global ID" because it is stored in the Global DB) | Kratos UUID |

**This is not "person.id changes value between L1 and L2".** For the same user, the L1 token's `person.id` carries the Kratos UUID, and the L2 token's `person.id` carries a different identifier entirely (the client account GUID). The field name is the same; the semantics underneath are different.

**Implications:**

- `person.kratosId` (camelCase in decoded token / `kratos_id` snake_case in introspection API response) is the **stable user identity** across authentication levels and the recommended primary key for any Kratos user. It is **not 100% immutable** — rare identity-merge events (e.g. when a Kratos identity is merged with an existing one) can change it — but it is the most stable identifier the ecosystem provides. Treat absolute immutability as a property no system can guarantee.
- The L2 `person.id` (client account GUID / "Global ID") can also change in rare cases (account merges, re-issuance), and its semantics have nothing to do with the L1 `person.id` of the same user.
- For AD users, `person.id` (AD GUID) is the only stable identifier available — `kratosId` is absent. AD users live outside the Kratos identity system entirely.

**Recommended primary key choice:**

| User type | Primary key |
|-----------|-------------|
| Kratos users (L1 or L2) | `person.kratosId` (camelCase in JWT / `kratos_id` snake_case in introspect response) |
| AD users | `person.id` (AD GUID) |
| Anonymous (device) | `person.id` (device UUID) — but no human identity is implied |

**Anti-framing to avoid in code, comments, and conversation**: do not call `person.id` "the global ID of the user" without qualification. The phrase is technically correct only for L2 (where the value comes from the Global DB), and it actively misleads readers into using `person.id` as a stable cross-level identity. Be explicit: "`person.id` at L1 = `kratosId`; at L2 = client account GUID; for stable cross-level matching use `kratosId`".

### Data Availability by Provider and Auth Level

**Not all `person` fields are populated for all providers and auth levels.** Before accessing any field, check whether it is available for the product's token type. A field existing in the proto definition does not mean it has a value.

| Field | Kratos L1 special (deprecated) | Kratos L1 | Kratos L2 | AD | Anonymous |
|-------|------------------------|-------------------|-----------|-----|-----------|
| `person.id` | = `kratosId` (same value) | = `kratosId` (same value) | `clientGUID` (different identifier — see below) | AD GUID | deviceUUID |
| `person.kratosId` | present | present | present | **absent** | **absent** |
| `person.email` | **present**, unmasked | **absent** | may be present, unmasked | present, unmasked | **absent** |
| `person.name` | may be absent | may be absent | present | present | **absent** |
| `person.lastname` | may be absent | may be absent | present | present | **absent** |
| `person.login` | absent | absent | may be present | sAMAccountName | **absent** |
| `person.verifiedPhone` | absent | absent | may be present, unmasked | may be present, unmasked | **absent** |

> **Note on masking**: Values inside the JWT and introspection response are **never masked** — `person.email` returns `john.doe@example.com`, not `j***@example.com`. Masking is performed only by the **frontend** `getIdentifiers()` widget method (see `global-auth:frontend` skill) — that is a separate WEB endpoint for in-app display, not part of S2S backend flow.

**Key takeaways for backend developers:**
- **Kratos L1** — the most common mode for consumer products — has **no email, no name, no login** in the token. If your backend needs the email (e.g. for user registration), you must obtain it through introspect (to get `kratosId`) → then `get-identity` (to get full identity data including all emails/phones).
- **Kratos L1 special (deprecated)** — a legacy provider configuration where `person.email` IS present in the L1 token. If existing backend code reads email from L1 tokens successfully, the product may be using this deprecated provider. New integrations should NOT depend on this behavior.
- **`person.id` semantics differ between L1 and L2** — at L1, `person.id` and `kratosId` are literally the **same value** (the Kratos UUID); at L2, `person.id` becomes a **different identifier** (a client/trading account GUID — internally called the "Global ID" because it is stored in the Global DB). This is **not "the same field changing value"** — they are semantically different identifiers exposed under the same key. Code that stores `person.id` from L1 will mismatch the same user's `person.id` at L2.
- **Use `kratosId` as the stable key for Kratos users** — it is the same identity from L1 to L2 and is the most stable identifier in the ecosystem. It is not 100% immutable (rare identity-merge events can change it), but no identifier in any system is — `kratosId` is the best available.
- **Token fields are scalars, Kratos stores collections**: `person.email` and `person.verifiedPhone` in the token/introspection response are single values, but a Kratos identity may have **multiple** emails and phones. The token only contains one (if any). To get the full list, use `get-identity` — it returns all verified identifiers.
- **Reading email/phone from introspection result** is fragile — the field may be absent depending on provider and auth level (see table above). If your backend relies on `person.email` from the introspection response, verify it is present for the product's provider type. If the field is absent but needed, use `get-identity` with `kratosId`.

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

When validating via TxAuth Agent, the `expected_app_name` field in `JwtRequest` (raw gRPC) or the `appName` parameter in PHP wrapper methods (`CheckJwt` / `CheckJwtByVariant`) restricts token acceptance to your specific product. If a token was issued for a different product, validation will fail. This covers various scenarios — from 2FA policy differences between products to simply ensuring your API only accepts tokens issued for it.

**Recommendation**: If you see a `Check` / `CheckJwt` / `CheckJwtByVariant` call without `expected_app_name` / `appName` — suggest adding it. Historically, this validation did not exist and all tokens within the ecosystem were accepted regardless of which product issued them. The `expected_app_name` parameter is a newer security feature that limits token scope to the intended product. Unless the API intentionally serves multiple products (a shared backend accepting tokens from several frontends), `expected_app_name` should be set.

## Anti-Patterns

### `sess` Is NOT a Kratos Session Token

**Wrong**: Sending `sess` to Kratos `/sessions/whoami` endpoint as a session token.

`sess` is a compressed JSON object containing the full session with person data. It is not a Kratos session identifier. Sending it to whoami will fail or return unexpected results.

### Do NOT Parse the JWT Manually for User Data

**Wrong**: Any form of manual JWT parsing to extract user data:
- Looking for `email`, `first_name`, `sub`, or similar standard claims at the top level (they don't exist — this is not an OIDC token)
- Using `jwt.decode()` + gunzip to decompress `sess` and read `person` fields directly
- Base64-decoding JWT segments and extracting fields with regex or string manipulation

All of these bypass signature validation, expiration checks, and `appName` scope restriction. A tampered, expired, or stolen token will be accepted silently. The internal `sess` format is not a stable API — field names, compression, and structure may change.

**Always** use TxAuth Agent (gRPC) or HTTP introspect (see Approach 1 and 2 above) to validate and extract data from tokens.

### Do NOT Assume Session Fields Are Always Present

**Wrong**: Accessing `sess.person.name`, `sess.userType`, `sess.firms`, etc. without checking if they exist.

The session structure varies dramatically by provider and auth level. Kratos L1 tokens may contain **only** `person.id` and `person.kratosId` — with no email, name, lastname, login, phone, userType, firms, or other fields. Always treat every field except `person.id` as optional. See "Data Availability by Provider and Auth Level" above.

### Do NOT Use person.id as a Stable Cross-Level Identifier

**Wrong**: Storing `person.id` from an L1 token and expecting it to match `person.id` from an L2 token for the same user.

`person.id` is a key under which **different identifiers** are exposed depending on provider and auth level — not a single value that "changes":
- At L1, `person.id` carries the **same value** as `kratosId` (the Kratos identity UUID).
- At L2, `person.id` carries a **separate identifier** — the client/trading account GUID (internally called "Global ID" because it lives in the Global DB).
- For AD users, `person.id` is the AD GUID; `kratosId` is absent.

Use `person.kratosId` (camelCase in JWT / `kratos_id` snake_case in introspection response) as the stable cross-level identity for Kratos users. Use `person.id` only for AD users (where `kratosId` doesn't exist).

### Do NOT Call person.id "the Global ID of the User"

**Wrong**: Comments, docs, or conversation framing `person.id` as "the global ID of the user" without qualifying which provider/level you mean.

The phrase is internally accurate **only at L2** (where the value comes from the Global DB) and actively misleads readers into using `person.id` as a stable cross-level identity. Even experienced engineers slip into this framing because the L2 backend names it that way. The framing causes:
- Storing `person.id` as the user primary key, which breaks on L1↔L2 transitions
- Treating `person.id` as a global cross-product identity, which it is not
- Confusing readers about which identifier is actually stable

Be explicit instead: "`person.id` at L1 is the same value as `kratosId`; at L2 it is the client account GUID; for stable cross-level matching use `kratosId`."

### Do NOT Rely on Kratos whoami as Primary Data Source

The Kratos whoami endpoint (`/sessions/whoami`) may be unreachable from your backend due to:
- IP filtering (Docker containers, cloud environments, non-VPN networks)
- Geographic restrictions
- Session token format mismatches

Use whoami only as a supplementary check, not as the primary source of user data. The token validation approaches above are the reliable path.

### Do NOT Skip Validation for "Trusted" Frontends

Even in internal enterprise applications, always validate the JWT. The token arrives from the browser — it can be tampered with, replayed, or expired. Validation is not optional.

### Do NOT Use /sessions/token as a Token Validation Shortcut

**Wrong**: Calling `POST https://ga.{domain}/sessions/token` from your backend (or frontend) to validate tokens or resolve user data instead of using proper introspection (Approach 1 or 2 above).

This is an **internal Kratos endpoint** used by the TxGlobalAuth widget internally. It is not a public API and not a substitute for token introspection:
- **Will break** when IP/UserAgent binding is enforced (planned security hardening)
- Does not perform full token validation (signature, expiration, scope)
- Returns data in a format that differs from the introspection response
- Not designed for S2S calls — no service token authentication, no `appName` restriction

**If you encounter direct calls to `/sessions/token` in existing code** (frontend or backend), this is typically a workaround for obtaining data (usually email) that isn't in the JWT for the product's provider type (see "Data Availability by Provider and Auth Level"). The correct backend approach: validate JWT via introspect → call Kratos S2S API (`get-identity`) for full user data.

### Do NOT Use HTTP Introspect When Inside the Network Perimeter

**Wrong**: Using Approach 2 (HTTP `POST /api/jwt/introspect`) from a backend that has network access to TxAuth infrastructure.

HTTP introspect is itself a wrapper around the gRPC TxAuth agent — it adds an extra network hop through the Kratos HTTP gateway, introduces an external dependency, requires a `SERVICE_TOKEN`, and offers no additional validation capability over a local agent. If your backend is inside the perimeter, deploy your own TxAuth agent and use gRPC (Approach 1).

**If you encounter HTTP introspect in existing code for a perimeter-internal backend**, flag it for migration to TxAuth Agent. The migration path: request agent deployment via support ticket → deploy agent + Consul agent → configure gRPC client → replace HTTP calls with gRPC `Check`.

> **The only valid use of HTTP introspect**: backends hosted outside the corporate network perimeter that cannot reach TxAuth infrastructure at all (e.g., external cloud deployments without VPN).

## Reviewing Existing Backend Integrations

When reviewing or auditing an existing TxGlobalAuth backend integration, check for these issues in order of priority:

1. **Validation approach**: Is the backend using TxAuth Agent (gRPC) or HTTP introspect?
   - Inside the perimeter but using HTTP introspect → recommend migration to TxAuth Agent (see anti-pattern above)
   - Outside the perimeter using HTTP introspect → correct, but verify `SERVICE_TOKEN` is properly stored as a secret (not hardcoded)

2. **`expected_app_name` / `appName` usage**: Is the `Check` call passing `expected_app_name` (gRPC) or the equivalent `appName` parameter (PHP wrappers)?
   - If absent → recommend adding it to restrict token scope to the specific product
   - Exception: shared backends that intentionally serve multiple products from a single API

3. **Token handling**: Is the code manually parsing/decoding the JWT?
   - Any `jwt.decode()`, base64 decode, or gunzip on the token → flag as anti-pattern, replace with proper validation via TxAuth Agent or HTTP introspect

4. **Identity key**: What field is used as the stable user identity?
   - `person.id` used for Kratos users → recommend switching to `person.kratosId` (`kratos_id` in HTTP introspect response) for stability across L1/L2 transitions
   - `person.id` used for AD users → correct (no `kratosId` available for AD)

5. **Field assumptions**: Does the code assume `person.email`, `person.name`, or other fields are always present without checking?
   - Check against the "Data Availability by Provider and Auth Level" table above
   - If the product uses Kratos L1 — most `person` fields are absent in the token. Code must handle this or supplement with `get-identity`

6. **Direct `/sessions/token` calls**: Is the code calling the internal Kratos endpoint directly?
   - Flag for replacement with proper validation + `get-identity` if additional user data is needed

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

    // Step 3: Find or create internal user — store ONLY the external identity key
    //         Do NOT copy email/name/login/phone into your DB unless you have a
    //         specific, justified reason — see "Storing User Data — Trade-offs"
    internalUser = findUserByExternalId(stableId)
    if not internalUser:
        internalUser = createUser({ externalId: stableId })
        // Display data (name, email) — fetch live from token / get-identity when
        // needed, or accept it from the frontend for non-critical display surfaces

    // Step 4: Issue internal session
    internalToken = issueJWT({ userId: internalUser.id, ... })
    return { token: internalToken, user: internalUser }
```

### Key Points

- **The frontend never stores or forwards the raw GA token for subsequent API calls** — after the exchange, it uses the internal token issued by your backend.
- **The exchange endpoint is the only place** where the GA token is validated. All other backend endpoints validate the internal token using your own middleware.
- **User matching**: Use `person.kratosId` (Kratos) or `person.id` (AD) as the external identity key. See "person.id Semantics" above.
- **Store the identity key, not the contact data** — see "Storing User Data — Trade-offs" below for why duplicating email/name/login into your own DB is usually a bad idea.
- **Do not skip the exchange**: Passing the raw GA token to every API call and validating it every time is inefficient and couples your entire backend to the TxAuth introspection service.

## Storing User Data — Trade-offs

**The default position should be: do NOT copy user PII (email, name, login, phone) into your own database.** TxGlobalAuth / Kratos / AD is the source of truth. Your system is a consumer, not an owner, of identity data. Copying it creates more problems than it solves.

**Store only**:
- `kratosId` (Kratos users) or `person.id` (AD users) — as the external identity key
- Internal data your product genuinely owns (preferences, business state, audit records keyed by `kratosId`)

**Fetch live when needed**:
- Display name in UI → from frontend `session.person` (already decoded by widget) or backend `get-identity`
- Outbound email/SMS → resolve via `get-identity` at send time, do not rely on a stored copy

### Why Storing PII Is Usually Wrong

1. **Update timing problem.** When does your stored copy refresh? Most systems refresh "on next login" — which means your copy is stale between logins. Users change email, phone, name regularly; you will have wrong data and won't know it. Refreshing on every request defeats the point of caching at all.

2. **Matching by stored PII is dangerous — never do this.** Using stored `email`, `login`, or `phone` as a matching key against incoming auth events is an anti-pattern across all providers:
    - **AD `login` (sAMAccountName)** can change (renames, mergers, marriage). Same human, different login.
    - **AD `email`** is a free-text field in AD — not validated, can be set to anything an admin (or an attacker with directory write access) chooses. Using it as an identity anchor is an authorization vulnerability.
    - **Kratos `email`/`phone`** can be added, removed, or replaced via the widget — they are not stable.
    - The only safe match keys are `kratosId` (Kratos) or `person.id` AD GUID (AD). Both are opaque and not user-modifiable.

3. **Architectural drift over time.** Even if you store PII purely as a "display hint" today, a future engineer joining the project will see those columns populated and reasonably assume they are authoritative. They will write features that depend on the stored email being correct, send marketing email to it, or use the stored login in audit logs as proof of identity. **Display hints decay into authoritative facts.** Removing them later is hard because by then real code depends on them.

4. **PII surface area and compliance.** Every additional copy of personal data is a copy you are responsible for protecting, expiring, and deleting on request. The fewer copies, the smaller the attack surface and the less compliance burden.

### When Storing PII Is Acceptable

There are legitimate cases — but they should be deliberate, not default:

- **Admin tooling / analytics dashboards** where displaying "Jane Doe" is genuinely more useful than `kratos_id=4f3a...`. Acceptable, but mark the data as a snapshot (with a timestamp), never use it for matching, and refresh it on every observation rather than caching long-term.
- **First-contact data** captured at registration that the identity provider does not preserve (e.g. social-login avatar, social profile name from Facebook/Google). Store it, but tag it as "registration-time snapshot from provider X", not as the user's current identity.
- **Audit / compliance records** where you must record what email/name/login was used at the time of an action. Store it as a frozen historical fact tied to the audit event, not as the user's current contact data.

In all these cases, the stored data is a **secondary copy** annotated with provenance and time. The **primary source remains TxGlobalAuth/Kratos/AD**, looked up live when current data is needed.

### When You See Stored PII in Existing Code

If you encounter a `users` table with `email`, `name`, `login` columns populated from token data:

1. Check whether anything actually reads those columns. Often they were added defensively and nothing depends on them — drop them.
2. Check whether anything **matches** on those columns (joins, lookups). If yes, this is a bug — replace with `kratosId` / `person.id` matching.
3. If display tooling depends on them, add a refresh-on-read path via `get-identity` and stop relying on the stored copy as truth.

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
