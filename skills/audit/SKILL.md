---
name: audit
description: "TxGlobalAuth (GlobalAuth, GA) integration audit workflow. Use when reviewing an existing TxGlobalAuth frontend/backend integration, auditing widget setup and token exchange, checking identity and PII handling, verifying fail-closed or CORS/deployment posture, or producing a structured Global Auth integration audit report."
---

# Global Auth - Integration Audit Skill

Audit an existing TxGlobalAuth integration in the sequence below. Do not skip ahead: a wrong widget build, provider assumption, or validation path can invalidate later conclusions. Record evidence as file locations, configuration values, and runtime observations; distinguish verified behavior from unverified configuration.

## 1. Pre-Audit Gathering

Collect this context before judging code:

| Required context | Record |
|---|---|
| Product configuration | `appName`, GA environment, expected authentication level: Kratos L1, Kratos L2, AD, or Anonymous |
| Network position | Backend inside the corporate perimeter or outside it; this determines the permitted validator path |
| Technology shape | Frontend framework, backend language/framework, session mechanism, deployment type |
| Migration status | New integration or migration from a previous authentication system |
| Test access | Whether a registered `appName`, a real token, validator access, and GA environment configuration are available |

Read, or request, the smallest useful file set:

- HTML entrypoint and dynamic script-loading/configuration code.
- Widget initialization, token subscription, login, logout, and user-display code.
- Backend raw-token exchange endpoint and token validator adapter.
- Application session creation, authentication middleware, logout, and user persistence code.
- Environment/configuration documentation and deployment manifests or `Dockerfile`.
- Runtime defaults and user-visible configuration labels for `appName`, environment, origin, and port; flag contradictory defaults that make the documented/local auth preflight fail.

Build a route/handler inventory before writing findings. Include auth exchange variants, callback/redirect exchange variants, native/mobile/beacon exchange variants, magic-link/passwordless/session-mint routes, logout variants, session/bootstrap/status routes, debug/diagnostic routes, GraphQL/RPC/search routes, user lookup/linking routes, profile/preference mutation routes, profile render/export routes, support/impersonation/admin-session routes, export/download routes, proxy/fetch routes, and CSRF helper routes. Use this inventory to avoid letting a broad finding hide a distinct vulnerable endpoint. When a route creates an app session, links identities, exposes user existence, issues/returns CSRF material, or performs operator/support actions, decide whether it needs its own finding even if a broader "diagnostics", "client-controlled auth", or "protected endpoint authorization" finding exists.

State missing inputs in the report. Do not convert missing access to an assertion that the integration works.

## 2. Step 0 - Build Identity Verification

Perform this first because code written against an unsupported widget build cannot be meaningfully audited as the public API.

1. Inspect the configured script URL and deployed runtime configuration. Confirm the public wrapper path ends with `/auth-widget/@8.js`; treat an internal `global-auth.v8.js` build as a blocking finding.
2. Exercise or inspect the loaded export after initialization. Confirm it exposes `subscribeJWT`, `getJwt`, and `getTokenProvider`.
   Flag compatibility code that accepts private, legacy, or internal aliases such as `onToken`, `getJWT`, undocumented provider accessors, or direct Kratos/global-auth internals as a separate wrong-API finding even when the configured script URL is correct. Do not reduce this to "missing getTokenProvider"; accepting private aliases means the app can silently work against the wrong build.
3. Read only **Pre-Integration Health Check**, **Script Loading**, and **How to Choose and Document the CDN Host** in `global-auth:frontend` for the authoritative checks and host families.
4. Confirm the selected CDN hostname has a rationale beside configuration or in operating documentation. Compare the selected regional host with the audience region, operator/trust chain, and data-residency constraints. Flag a silent hardcoded host even when its wrapper path is correct.
5. Inspect dynamic wrapper loading failure behavior. Confirm load errors surface to the user/operator and cannot leave the application indefinitely waiting; for production-facing integrations require a bounded timeout or equivalent recovery path.
   If script integrity/SRI is configured, verify failures do not retry or fall back to the same external widget URL without integrity metadata. Treat integrity-stripping retries, downgrade fallbacks, or "try without SRI" recovery as a browser-delivery finding even when the wrapper path is correct.
6. Inspect every frontend fallback/default config path separately from the backend-injected runtime config. Flag fallback values that silently switch `env`, `appName`, CDN host, or CDN path away from the documented safe/runtime defaults, especially fallbacks to `prod`, a different `appName`, or `global-auth.v8.js`. Treat unsafe fallback `env` values as their own finding even when the CDN path is also wrong.
7. Inspect the `init()` control flow. Confirm initialization is awaited or otherwise chained before any TxGlobalAuth API call, subscription, auth button enablement, or state read. Flag "fire-and-forget" init, missing `await`, ignored timeout wrappers, and subscriptions registered before init settles as standalone findings even if the build URL is also wrong.

Stop logic-level review until a wrong-build finding is understood. Record runtime checks separately from static URL inspection.

## 3. API Surface Audit

Confirm that widget state has one authoritative path:

- Trace every write to authenticated-user state, application session state, and displayed identity.
- Confirm `subscribeJWT` drives first login, token refresh, account switch, auth-level/provider change, session loss, and initial page reload/re-mount reconciliation behavior. Inspect the same-user/same-level token-refresh branch specifically: it must not call the backend exchange, create a new application session, overwrite session cookies, or persist new authorization context unless a server-owned revalidation requirement is documented. Do not credit a generic `subscribeJWT` flow as correct until the refresh branch has been traced or runtime-tested with two callbacks for the same stable identity. For the same stable identity at a different auth level or provider, verify downgrade and provider-change cases clear or revalidate the application session before protected state remains usable; do not treat all same-person callbacks as harmless refreshes. Treat code that records variables such as `lastAuthLevel`, `lastProvider`, or similar bookkeeping without actually comparing them in the auth-state branch as evidence the downgrade/provider-change path may be missing. On account switch the old application session must be cleared before accepting a replacement.
- For every `subscribeJWT(null)` path, confirm backend app-session clearing is immediate and unconditional. Flag page-visibility/background guards, URL/config flags such as `ignoreNull`, "hidden tab" exceptions, deferred "pending sync" markers, or `/api/session` reconciliation paths that can leave an existing app session alive after GA reports guest/session loss. Report null-callback suppression as its own finding when a browser-controlled flag or branch can skip logout, not only as a general stale-state concern.
- Trace asynchronous exchange, session-status, and logout requests together. Confirm cancellation/sequence handling prevents a stale logout or exchange response from overwriting the current cookie or UI; require an explicit stale-response guard such as sequence comparison after exchange/logout responses and/or aborting prior in-flight requests before starting a replacement. Do not credit the mere presence of an `AbortController`, request counter, or `requestId` variable as sufficient; verify a stale check occurs after each awaited exchange/logout response and before applying session labels, cookies, cached identity, or other auth state. Flag unused request ids, pre-request-only checks, and logout/exchange handlers that apply responses after a newer auth transition. Inspect startup `/api/session`, bootstrap, or "restore existing app session" reads before widget initialization. They must be provisional and cleared or hidden if TxGlobalAuth init fails or returns `null`; flag code that displays or continues using an application session before GA becomes authoritative. Flag reconciliation reads whose result is immediately overwritten by the initial subscription callback, and where practical test a rapid `null` then token transition. For account switching, confirm the prior application session is cleared before any replacement exchange begins and remains cleared if that replacement fails. Treat this as a finding when code updates `lastPersonKey`, displayed identity, or pending auth state and then calls exchange without first clearing the backend cookie/session, because a failed replacement validation can leave the previous user's app session usable.
- Treat URL-carried or browser-storage-carried application session bootstrap as its own finding when it can issue, refresh, or display an app session before the widget has become authoritative. Do not merge this entirely into a general startup reconciliation issue when a backend route such as `/bootstrap?sid=...` can reissue a normal session cookie from the supplied value.
- Inventory every logout, revoke, disable, admin session-kill, account-switch, and webhook-driven session termination path. Confirm it invalidates all intended live sessions for the affected external identity/session family, not only the current cookie or a single session id, unless the product explicitly supports independent sessions and documents the residual risk. Inspect session creation/indexing as well as termination code: if sessions are stored only by session id and there is no external-identity/session-family index, normal logout/admin disable may be unable to revoke siblings even when a webhook loop happens to scan all sessions. For webhook-driven revocation, separately verify webhook authenticity and replay protection before trusting the supplied external identity/session id: require an approved signature/shared-secret/mTLS mechanism, event id replay checks, timestamp freshness, and a fail-closed path for unsigned or malformed events. Report unsigned webhook revocation as its own finding, not only as public diagnostics or process-local session storage. Report sibling-session gaps separately from generic process-local storage findings, and do not treat a server-to-server webhook revocation path as proof that browser logout or account-switch logout revokes sibling sessions.
- Confirm login/logout button handlers invoke widget behavior and let subscription results update state. Flag direct app-session logout/exchange calls from button handlers, including `sendBeacon()` shortcuts, before the corresponding `subscribeJWT` transition has occurred.
- Enumerate backend logout routes separately from frontend logout handlers. Flag any `GET` logout endpoint, method alias, image/beacon-compatible logout, or route that destroys an app session without the same CSRF/origin controls as the canonical POST logout. Do not collapse this into a generic frontend state-management finding; it is a separate auth-state mutation surface.
- Inspect progressive requirement chains such as `requireAgreements()`, `requireEmail()`, and `requirePhone()`. Required post-auth steps must be awaited or otherwise enforced before the app treats the flow as complete; flag fire-and-forget calls, optional-chained calls with no awaited result, detached promises, and `.catch()` handlers that silently continue after requirement failure. Report this even when the same frontend already has broader wrong-API or state-management findings, and name the exact requirement method that is not enforced.
- Flag competing auth-state paths, direct GA/Kratos browser requests, startup/bootstrap replay, or refresh handling that unnecessarily creates new application sessions. Inspect manual renew/retry/page-load handlers for `getJwt()`, `getJWT()`, local/session storage, `window.name`, postMessage data, cached tokens, or other stored bearer material followed by backend exchange; token refresh and page reload reconciliation should flow through `subscribeJWT`, not replay stored GA tokens. Treat `subscribeOnTokenRenewError` or similar error callbacks that fetch a token and re-exchange it outside the authoritative `subscribeJWT` transition as a separate finding, even when the app also has other duplicate exchange routes.
- Inspect frontend URL/config-triggered mutation helpers after login or initialization. Flags such as `profileEmail`, `profileRoles`, `profileAuthLevel`, `linkEmail`, `linkExternalId`, `supportUser`, `supportKey`, `trustDevice`, or similar values that automatically call profile, account-link, support impersonation, trusted-device, or admin/operator endpoints must be reported as a frontend-driven mutation finding in addition to the backend endpoint weakness.
- Inspect browser-controlled endpoint selection such as `exchangeEndpoint`, `logoutEndpoint`, `sessionEndpoint`, callback URLs, API base URLs, or query/config-selected auth routes. If the browser can redirect token exchange, logout, session bootstrap, profile mutation, account linking, support, or admin calls to an attacker-chosen or unintended endpoint, report it separately from duplicate exchange routes or open redirect findings.
- Inspect post-login redirects and continuation URLs such as `returnTo`, `next`, `redirect`, `callback`, and `RelayState`. Flag any browser-controlled value passed to `window.location`, `location.assign/replace`, link navigation, `openApp`, or backend redirect responses after authentication unless it is constrained to a server-owned same-origin path or exact allowlist. Open redirects after login or callback handling must be a standalone finding; do not bury them under "browser-controlled config" or "callback exchange" findings.
- Confirm login and token-renew errors are surfaced through the supported subscriptions. Also inspect whether these error handlers leave an existing application session usable after GA reports a failed login, failed renewal, expired token, provider change, or downgraded auth state; flag handlers that merely update UI/log text while explicitly retaining or restoring backend authenticated state without server-side revalidation.

Consult **Recommended Integration Architecture**, **JWT Token Management**, and **Diagnosing Existing Integrations** in `global-auth:frontend`. Cite those sections rather than reproducing API guidance in the report.

## 4. Token-Exchange Audit

Follow the raw GA token from browser callback to backend validation and session creation.

1. Confirm a single exchange boundary consumes the raw GA token and that subsequent application requests use the application's own session mechanism. Enumerate every route and request location that can carry the raw GA token, including JSON body, query string, callback/redirect query parameters such as `token` or `ga_token`, form body, text/plain body, cookies, `Authorization` / custom headers, local storage replay, and message-driven exchange. Flag duplicate or fallback bearer-token transports because they increase proxy/log/header exposure and create competing exchange paths. Treat callback handlers that create app sessions directly from URL token material as token-exchange routes, not harmless redirect plumbing. For each duplicate exchange route, compare its controls against the canonical exchange route one by one: CSRF/origin policy, body-size limit, parser/schema validation, rate limiting, logging/redaction, stale-response semantics, and whether it can be triggered as a simple form/beacon request. Report missing rate limiting, missing body limits, and simple-request exposure as separate findings when present, not only as generic duplicate transport risk. Inspect body-limit bypass flags or modes such as `bulk`, `import`, `native`, `mobile`, `beacon`, `text/plain`, or special headers that skip the normal cap; report bypassable body limits as a body-size finding even if the canonical route has a size check. Explicitly flag any backend fallback that treats the application session cookie or app-session id as a GA token source; app-session identifiers and GA bearer tokens must not be interchangeable.
2. Search validator code and dependencies for local JWT decode, base64/decompression, signature-free claim reads, or direct trust of frontend-decoded person data. Flag any such acceptance path.
3. Confirm an inside-perimeter backend uses TxAuth Agent through gRPC. Verify the gRPC transport setup itself: flag `grpc.insecure_channel`, missing TLS/mTLS where the deployment requires authenticated transport, unauthenticated sidecar addresses exposed beyond the trusted local boundary, or validator addresses controlled by browser/runtime request input. Accept HTTP introspect only when an outside-perimeter network constraint is documented. If HTTP introspect is used, require HTTPS or an equivalently protected internal transport; flag cleartext `http://` defaults as a transport finding separate from local-decode fallback. If an HTTP or adapter response feeds a product-scoped application session, compare the fields the application trusts against the documented endpoint contract; plain HTTP introspection must not be credited with `validated` or `appName` scope fields it does not document. Inspect the exact HTTP introspection request shape as well: flag raw GA tokens sent in query strings, redirect-following fetches, or other transports that expose bearer material to logs, proxies, caches, or browser history-like surfaces. Report query-string raw-token introspection as a distinct bearer-exposure finding even when the same validator also has a broader "HTTP fallback" or "local decode fallback" finding.
4. Confirm successful validation uses authoritative success evidence from the selected validator contract before creating a session. Where an approved adapter contract exposes `active=True` and `validated=True`, require both as strict booleans, not truthy strings/numbers/objects; where plain HTTP introspection documents only `active`, require a separately approved app-scope enforcement path before treating the exchange as product-scoped.
5. Confirm validation scopes tokens with `expected_app_name` or its wrapper equivalent, unless a deliberately multi-product API documents why it accepts several products.
   Confirm the exact expected app name sent to the validator is the server-owned product configuration value. Flag lowercasing, prefix/suffix injection, wildcarding, environment-variable transforms, aliases, or other normalization that can validate a different app scope than the configured product. Confirm the default expected app name is derived from the same server-owned product configuration as `appName`, or that any hardcoded/different value is explicitly documented as intentional. Flag drift where changing `GA_APP_NAME` / runtime app configuration would leave validation scoped to another product.
6. Confirm the backend enforces the product's required authentication provider/level from validated validator output before creating its session; frontend button choice or local configuration labels do not enforce L1/L2/AD/Anonymous requirements.
7. Confirm validation success checks are type-strict where the validator contract is boolean. Flag truthiness checks such as `if result["validated"]` that would accept strings, numbers, or other non-boolean values as validation success. Inspect adapter status/error handling as part of this check: enumerate every status code or status name the adapter accepts as success, not just the obvious failure branch. Non-success validator status codes, errors, warnings, degraded/partial-success codes, or unknown codes must fail closed and must not be converted to success based on code sets such as `{0, 10}`, status text such as "warning", or outage/degraded-mode labels. Report an extra accepted nonzero/custom validator status as a distinct validation-contract finding even when another fail-open degraded-mode finding is already present. Also record adapter transport and timeout defaults in the finding set: insecure gRPC channels, missing TLS/mTLS, browser/config-controlled validator addresses, and long request-path validation timeouts must not disappear into a generic "fallback validator" finding.
8. Inspect every request field sent with the exchange payload. The backend must not trust client-supplied timestamps, identity context, auth level, provider, tenant/firm/account id, roles, entitlements, agreement/compliance state, session id, freshness windows, or other browser metadata for validation, authorization, session identity, or token lifetime decisions; such fields may be logged only as diagnostics after server-side validation. Also inspect whether these values are sourced from or written back to browser storage (`localStorage`, `sessionStorage`, cookies, `window.name`) and then replayed into future exchanges; persisted browser authorization context must not drive backend authorization.
9. Inspect validator-result caches, memoization layers, and session bootstrap caches. Search for cache maps, TTL constants, decorators, LRU helpers, and names such as `validation_cache`, `tokenCache`, `memoizedValidation`, `cache_ttl`, `VALIDATION_CACHE_SECONDS`, or similar. Flag raw GA tokens used as cache keys or stored in memory/logs, and flag positive validation caching unless it has a documented validator-owned revocation/expiry model, a very short bounded lifetime, and revalidation behavior that cannot outlive token revocation, account switch, auth-level change, or `appName` scope changes. When a validation cache is present, record its key material, TTL/duration, positive/negative behavior, and whether revocation is checked before or after cache hits; produce a standalone cache finding when raw tokens are keys, TTL is long, or revocation checks happen after cache return. Do not let a generic fail-open validator finding replace a concrete cache-duration/order finding, and do not treat "export exposes cache keys" as covering the underlying validation-cache safety issue. Separately trace every revocation marker (`revokedTokens`, `revokedExternalIds`, logout grace maps, webhook revocation lists, disabled users) back into the validation/session-lookup path; report markers that are only logged, only appended, or checked after a positive cache hit as ineffective revocation, not merely as weak webhook/logout behavior.
10. Inspect validator client timeouts, retry loops, and circuit-breaker behavior. TxAuth Agent calls should use a short bounded timeout appropriate for local sidecar validation and fail closed without tying up request threads; flag long defaults, unbounded waits, broad retry storms, or user-request-controlled timeout values unless an operational SLO and worker-capacity model justify them.

Consult **Token Validation — Required**, **Cross-Product Token Validation (`appName`)**, **Anti-Patterns**, and **Common Integration Pattern: Token Exchange** in `global-auth:backend`.

## 5. Identity-Key Audit

Trace the persisted external identifier and all user lookup or linking operations:

- For Kratos authentication, confirm the stable external identity key is `kratosId`.
- For AD authentication, confirm the external identity key is `person.id`.
- For Anonymous support, confirm classification is based on the validated provider, not the mere presence of `person.id`. Inspect provider dictionaries for every common field name the integration may receive (`name`, `provider`, `type`, `source`, `authProvider`, etc.); flag helpers that read only one or two fields and can misclassify anonymous/device providers as AD or named users.
- Confirm anonymous, guest, device, or other non-person sessions are stored separately from durable named-user account records unless the product explicitly supports anonymous account persistence. Flag anonymous sessions written into the same user table, profile store, account-link graph, analytics identity, or entitlement record as Kratos/AD users without a separate namespace, lifecycle, and retention policy.
- Confirm persisted user/account records are scoped by the provider family and product/environment boundary that owns the identity. A bare normalized external id, email/login-derived key, or shared `external_id` namespace can collide across Kratos, AD, Anonymous, multiple `appName` values, tenants, or GA environments; flag missing namespace/scope separately from mutable-PII matching.
- Flag email, login, phone, or other mutable PII used for identity matching. Inspect the complete fallback chain, not only the first preferred key: a helper that prefers `kratosId` / `person.id` is still a finding if it can persist or link users by `email`, `login`, phone, display name, or other mutable PII when stable identifiers are absent.
- Inspect identity-adjacent lookup and linking endpoints separately from session creation. Public or broadly reachable `/user`, `/users`, `/user-lookup`, `/account`, `/account-link`, or similar routes must not disclose whether a user exists by `email`/`login`, list known user keys, leak stored provider/auth-level/session metadata, or allow linking an arbitrary `targetExternalId` without proving control of that identity, checking provider/account ownership, and enforcing recent reauthentication for the link action. Account linking is a high-risk identity mutation: if a route accepts a browser-supplied `targetExternalId` or equivalent, record it as a concrete finding even if the same endpoint is also covered by generic "required auth level/provider not enforced" or "arbitrary profile mutation" findings. Treat public user-enumeration routes as separate from generic public diagnostics/export findings when they expose existence, keys, email/login lookup, or identity metadata.
- Trace whether URL/runtime config values such as `linkExternalId`, `targetExternalId`, `profileEmail`, or similar browser-controlled parameters automatically trigger lookup or linking requests after login. Treat URL-driven identity linking as a separate finding even when the underlying backend link route is also weak.
- For each account lookup/link route, state whether the route is public, merely authenticated, or privileged, and record whether it uses stable provider-owned identifiers or mutable PII. Do not let a generic "identity storage is unscoped" finding replace concrete findings for public enumeration, metadata disclosure, or arbitrary account-link mutation.
- Inventory PII copied into application user records and require a stated business/retention justification for each field.

Consult **person.id Semantics — Critical**, **Anonymous Classification Using Validator Provider**, and **Storing User Data — Trade-offs** in `global-auth:backend`.

## 6. PII Surface Audit

Search frontend, backend, logging, persistence, analytics, and templates for reads of:

```text
person.email
person.name
person.lastname
person.lastName
person.login
person.verifiedPhone
```

Build an inventory for each occurrence:

| Field/location | Provider and level | Available at that level? | Purpose | Finding? |
|---|---|---|---|---|
| `<evidence>` | `<Kratos L1/L2, AD, Anonymous>` | `<verified/absent/unknown>` | `<display, outbound communication, matching, persistence, logging>` | `<result>` |

Consult **Data Availability by Provider and Auth Level** in `global-auth:frontend` or `global-auth:backend`; load only that named section. Treat display as potentially acceptable subject to product privacy rules. Treat identity comparison or account matching on PII as an integration anti-pattern. Record unneeded DOM, browser storage (`localStorage`, `sessionStorage`, `window.name`), browser console, debug panel, `<pre>` log, telemetry, or app log disclosure as a finding even if the value is otherwise available; verify masking/redaction helpers actually remove the sensitive fields before display.
Also search for derived display/profile fields populated from those PII values, such as `display_name`, `displayName`, `fullName`, `profileName`, DOM `dataset` assignments, and app-session response fields. Inventory and justify them even when the original `person.*` read is not returned directly.
Inspect authenticated profile render/export surfaces that consume those stored fields, including profile cards, HTML previews, CSV exports, and spreadsheet/download endpoints. Flag stored HTML rendering without output encoding, unsafe embedding of profile values into JavaScript or `postMessage`, and CSV exports that write unneutralized cells beginning with formula characters, because auth-adjacent profile data often becomes a durable stored XSS or formula-injection surface even when the initial write came from a seemingly benign runtime-config path. Also inspect filenames and response headers for profile/export downloads; email- or display-name-derived filenames should be sanitized and should not leak unnecessary PII. If route inventory contains concrete sinks such as `/profile-card`, `/api/profile.csv`, `/profile/export`, or similar, decide each sink explicitly; these must appear as explicit findings when vulnerable. Do not count a URL-driven profile update, arbitrary profile-key finding, or broad whole-session serialization finding as covering the downstream render/export sink.

## 7. Fail-Closed Audit

Verify failure behavior by inspection and, where practical, negative runtime tests:

- Remove or withhold validator configuration and call the exchange endpoint. Confirm it refuses the token and creates no authenticated application session.
- Make the validator unreachable or return an inactive/unvalidated result. Confirm rejection and no fallback to locally parsed JWT data.
- Inspect exchange preconditions for client-supplied clocks, timestamps, auth windows, or freshness hints. Token freshness and expiry must come from validator output or trusted server time, not browser-provided request fields.
- Inspect exchange payloads for client-controlled persistence flags such as `remember`, `keepSignedIn`, `sessionTtl`, or `expiresAt`. Browser input must not extend the application-session lifetime unless the server has an explicit product policy, maximum lifetime, and revalidation model independent of the request value.
- Send missing, malformed, and oversized exchange request bodies where practical. Confirm parser failures are handled and request size is bounded before token parsing or validator work.
- If account switching is supported, make the replacement exchange fail and confirm the previously authenticated application session does not remain usable through the intended browser flow.
- Locate development bypasses, mock validators, feature flags, or environment switches. Confirm production configuration cannot enable them.
- Inspect alternate non-GA sign-in/session-mint routes such as magic links, passwordless links, callback helpers, invite acceptance, "native login", support bootstrap, or admin session creation. Any route that creates an authenticated app session without authoritative TxGlobalAuth validation, server-owned provider/auth-level policy, and replay/freshness controls must be a standalone finding even when broader fail-open validation or support/admin findings exist.
- Inspect session-adjacent mutation endpoints such as profile, preferences, settings, device registration, remembered accounts, and feature flags. Flag arbitrary client-controlled keys merged into server-side user/session objects without an allowlist, schema validation, ownership check, and clear separation between display preferences and authorization/security state. Also inspect frontend URL parameters such as `pref.*`, `setting.*`, `flag.*`, or similar values that are replayed into these endpoints.
- Check that errors expose actionable operator information without returning raw token or PII content.
- Enumerate CSRF token issuance and bootstrap helper endpoints such as `/api/csrf-token`, `/api/csrf`, `/csrf`, or similar support routes. Flag publicly readable static process-wide CSRF tokens, tokens reused across users/sessions, or token checks that are accepted before origin/referrer validation. Inspect any CSRF cookie set by the helper and record `Secure`, `HttpOnly`, `SameSite`, `Domain`, `Path`, and whether JavaScript can read it; readable/non-`Secure`/missing-`SameSite` CSRF cookies must be reported separately when they weaken the CSRF model. If the application both exposes a static CSRF token and accepts missing/static tokens, report the readable helper endpoint separately from the weak CSRF check; mentioning it only in a route inventory or broad diagnostics finding is not sufficient.
- Inspect export/download/admin-ish endpoints such as `/api/export`, `/debug/sessions`, `/debug/config`, `/api/audit`, `/api/user`, GraphQL/RPC search endpoints, or similar support routes. Static shared header keys, default operator keys, unauthenticated access, or schema/introspection-style queries are not enough protection for session inventories, validator state, user records, or recently revoked session data. Report weak export protection separately from public diagnostics when the endpoint returns bulk sensitive data. Report user-enumeration/search/GraphQL endpoints separately when they expose known user keys, existence by email/login, whole sessions, schema introspection, or stored auth/session metadata.
- Inspect auth/session status responses, token-exchange success responses, validation caches, browser-visible debug logs, frontend log widgets, server logs, and telemetry for application session identifiers, internal session IDs, cookie values, raw GA tokens, raw validator payloads, `Authorization` headers, or other bearer/session secrets. These must be redacted or avoided even when PII fields are absent; returning a live application session ID in JSON is a finding even if the cookie is HttpOnly, and rendering it into a frontend debug log is also a finding. Inspect exchange, `/api/session`, bootstrap, profile, and protected-data endpoints for whole-session serialization patterns such as `asdict(session)`, `session.__dict__`, or raw ORM/session model JSON; flag them when they expose session IDs, expiry internals, token-derived identity, provider/auth-level fields, external identifiers, or authorization fields. Redacting only the session id is not enough if the response still returns token-derived application-session internals that the browser does not need.

Use the failure requirements in **Token Validation — Required** and **Anti-Patterns** in `global-auth:backend` as the integration authority.

## 8. CORS / Environment Audit

Verify browser-to-GA compatibility separately from CDN reachability. The wrapper loading successfully does not prove that widget requests from the application's origin are permitted.

1. Record the exact browser origin, including scheme and port, and the selected GA environment.
2. Obtain the GA backend environment configuration or confirmation from its owner for that environment. Inspect the CORS origin list key named `allowed_origins` (or its exported camel-case representation, `allowedOrigins`) and cite the configuration source in the report. Do not substitute the application's own CORS settings for the GA backend allowlist.
3. Confirm the exact application origin appears in that list. For local DEV testing, `http://127.0.0.1:8080` and `http://localhost:8080` are known commonly pre-allowed origins; still confirm against the target environment configuration because allowlists can change.
4. Confirm `appName`, environment, and CDN/environment pairing are consistent. Use **Pre-Integration Health Check** in `global-auth:frontend` for this preflight.
5. If an origin is absent, either request its inclusion from the Global Auth owner or change local development to an already approved origin/port. Record which decision was made; do not diagnose the failure as CDN unreachability without evidence.
6. Separately inspect the application's own CORS/preflight behavior for auth/session endpoints. Flag reflected arbitrary `Origin` with `Access-Control-Allow-Credentials: true`, permissive `OPTIONS` responses, broad `Access-Control-Allow-Methods` / `Access-Control-Allow-Headers`, or credentialed cross-origin reads of `/api/session`, `/config.js`, exchange, logout, or session bootstrap responses unless guarded by a server-owned exact-origin allowlist.

## 9. Deployment-Posture Audit

Inspect deployment posture through the integration-specific consequences below. Use a dedicated `security-audit` or `devops-architect` review for general implementation detail rather than expanding this skill into a generic security checklist.

| Check | TxGlobalAuth integration lens |
|---|---|
| Non-root container and healthcheck | Keep the exchange service available without granting unnecessary runtime privilege; health should expose loss of the request-serving process. |
| No PII in image or logs | Token-derived identity and developer secrets must not be baked into artifacts or written to routine logs. |
| Secrets configuration | TxAuth/HTTP introspect credentials, signing/session secrets, static CSRF values, admin/export keys, support/impersonation keys, webhook secrets, and validator bypass toggles must come from approved secret management, not source, image layers, browser-visible runtime config, query strings, or client-overridable config. Flag hardcoded defaults such as `starter-secret`, `admin`, `support`, `secret`, `csrf`, or similar even in demos unless they are impossible to enable in production. |
| CSRF defense | Protect the token-exchange and logout endpoints because they create or terminate the application's authenticated session. Enumerate all routes and methods that mutate auth state; flag GET logout, beacon/simple-request logout shortcuts, method aliases, or any state-changing path that bypasses the normal same-origin/CSRF checks. Inspect how the expected origin is constructed; flag checks that trust client-supplied `X-Forwarded-Host`, `Forwarded`, `X-Forwarded-Proto`, or similar proxy headers unless the app is behind a documented trusted proxy boundary that strips and rewrites them. |
| Application CORS/preflight | Auth/session endpoints must not reflect arbitrary origins with credentials or accept broad preflights. Require an exact server-owned allowlist for credentialed CORS, and treat permissive `OPTIONS` or response-wide CORS reflection as an integration finding even when GA backend CORS is separately correct. |
| Sensitive-response caching | Auth/session bootstrap responses and session-dependent HTML containing CSRF or identity state should be non-cacheable (`Cache-Control: no-store` or equivalent). |
| Cookie transport enforcement | Confirm `Secure` is enforced, not merely documented, for HTTPS/production delivery of application-session and auth-CSRF cookies. Record the actual default/config gate such as `COOKIE_SECURE`, framework secure-cookie settings, reverse-proxy HTTPS assumptions, and production startup behavior; do not mark cookie posture as healthy merely because `HttpOnly` and `SameSite=Lax` are present. If `Secure` is optional, defaults off, or relies on an operator remembering to set an env var with no production fail-safe, report it as a finding. Confirm application-session cookies are `HttpOnly`; flag missing `HttpOnly` as a distinct finding because it makes the session identifier readable to JavaScript even if other cookie attributes are discussed. Confirm `SameSite=None` is used only when cross-site delivery is required and is paired with enforced `Secure`; otherwise prefer `Lax` or `Strict`. Confirm cookie `Domain` is absent for host-only cookies or is a narrow server-owned deployment value; flag domains derived from the request `Host` header, forwarded headers, query/config controlled by the browser, or broad parent domains that share app sessions across unrelated hosts. Confirm the cookie name is product-specific enough to avoid collisions with sibling apps on the same host/path; flag generic names such as `session`, `sid`, or `auth` when the deployment may share a domain or path with other services. Inspect logout/expiration cookies separately and flag mismatched clearing attributes such as missing `Secure`, `HttpOnly`, `SameSite`, `Domain`, or path consistency when the original cookie used them, because failed clearing can leave a live app session cookie behind. Do the same attribute comparison for companion CSRF cookies; a session-cookie finding does not automatically cover a separately readable or uncleared CSRF cookie. |
| Body-size limit | Bound every exchange request containing a token before reading/parsing it, including JSON, form, multipart, text/plain, beacon, alternate mobile, and compatibility exchange routes. Do not credit a canonical JSON body limit for sibling token-exchange routes unless each route applies an equivalent pre-parse limit. |
| Rate limiting | Bound abusive exchange attempts because each accepted attempt invokes validation infrastructure. Apply this check to every token-exchange route and raw-token fallback path, not only the canonical JSON exchange endpoint. If keyed by IP, confirm it uses a trusted connection/proxy-derived client identity and does not trust client-supplied forwarding/IP headers such as `X-Forwarded-For`, `X-Real-IP`, `CF-Connecting-IP`, or `Forwarded` unless a trusted proxy layer sanitizes them. If no rate limit exists, report that; if a rate limit exists but is bypassable or keyed by spoofable headers, report spoofable/bypassable rate-limit identity as a separate exchange-abuse finding and include the header/source used. Always inspect the rate-limit key expression/source when a rate-limit map, bucket, middleware, or helper exists; do not credit the mere presence of rate limiting. Do not bury it under CORS, proxy-origin, or forwarded-header CSRF findings. |
| Application-session storage | Verify the application's replacement for the GA token fits the intended restart, multi-instance, expiry, and revocation posture; identify process-local memory as development/single-instance only unless explicitly accepted. Confirm session identifiers are random per session and not derived from identity, timestamps, deterministic salts, token data, or other predictable inputs. For logout, revocation, account switch, and admin disable flows, inspect whether all live sessions for the same external identity/session family are tracked and invalidated as intended; flag designs that revoke only the current session id while sibling sessions remain active without an explicit product policy. |
| Application-session lifetime | Compare server-side expiry, pruning, cookie `Max-Age`/`Expires`, remember-me settings, validator/token expiry fields, and any grace windows. Flag app sessions that remain accepted after their recorded expiry, whose cookie lifetime exceeds the server-authenticated session lifetime, or whose extended lifetime is selected by browser input without a documented, server-owned, revalidated renewal model. Also inspect adapter/validator-derived fields such as `session.expiresAt`, `session.exp`, `expires_at`, `ttl`, or `maxAge`; flag application sessions whose lifetime is copied from those fields without a server-owned maximum cap, sanity bounds, and consistency with the validator's authoritative token expiry contract. Inspect read-only helpers such as `/api/session`, `/bootstrap`, or cache reconciliation code for sliding-expiry-on-read behavior and post-logout grace maps/lists that keep a recently revoked session temporarily acceptable. |
| Protected endpoint authorization | Inventory session-protected product endpoints beyond exchange/session/logout. Confirm sensitive or account-specific endpoints enforce the product's required provider, auth level, tenant/account ownership, and authorization checks from server-side session state or backend data; flag endpoints that accept any authenticated app session where L2/client-level, AD role, firm/account ownership, or explicit entitlement is required. For high-risk actions such as transfers, withdrawals, profile credential changes, trusted-device enrollment, agreement acceptance, account linking, support impersonation, admin session creation, or operator overrides, separately verify recent step-up/transaction-specific reauthentication, server-side amount/currency/destination/account-ownership validation, operator identity/role proof, and freshness windows; do not treat a stored L2 auth level, trusted-device flag, static support key, or `X-Support-Mode` style header as sufficient without a documented policy. Report support/impersonation routes that mint admin/support sessions without GlobalAuth validation as their own finding, not merely as a generic secret/config or protected-endpoint issue. Report missing freshness/recent-reauth checks and missing server-side business validation as separate findings when both are present, even if the same endpoint also has a trusted-device or auth-level bypass. |
| Idempotency and replay controls | Inspect idempotency keys, replay caches, webhook event ids, and client retry identifiers on auth-sensitive or financial actions. Keys must be scoped to the authenticated user/session, route/action, payload fingerprint, and reasonable expiry; flag global client-supplied idempotency keys whose cached result can be replayed or disclosed across users, sessions, tenants, or actions. |
| Trusted device posture | If the app records trusted devices, remembered devices, or device exemptions, verify enrollment requires recent step-up and server-bound device proof, entries expire or can be revoked, and logout/account switch/revocation clears or revalidates device state as intended. Flag device IDs supplied by URL/body alone, trusted-device bypasses of L2 for high-risk actions, and trusted devices not tied to provider/session context. Report trusted-device lifecycle gaps as their own finding when entries never expire, are not provider/session bound, or survive logout/account switch/revocation, even if enrollment and authorization-bypass issues are also present. |
| Content proxying and user-supplied fetches | Inspect authenticated preview/proxy endpoints such as avatar, import, attachment preview, URL metadata, PDF/image fetch, or webhook test utilities. Flag arbitrary user-supplied server-side fetches as SSRF. Separately flag proxy responses that forward upstream `Content-Type`, redirects, cache headers, or response bodies into the application origin without an allowlist, content sniffing prevention, image-only validation, size limits, and active-content blocking; do not collapse active-content serving into a generic SSRF finding. |
| Browser delivery controls | For a production-facing external widget script, record the CSP/script-host policy and any integrity or version-pinning decision; refer broad header design to a security review. If SRI/integrity is configured, verify load-error handling does not retry without integrity, strip `crossOrigin`, switch to an unpinned mirror, or otherwise downgrade the protection. Report integrity-stripping retry/downgrade behavior as its own finding even when the same integration already has a wrong widget path, wrong API surface, or init race. |
| Public diagnostics | Inspect health, readiness, config, bootstrap, user lookup, GraphQL/RPC, and diagnostic endpoints. Public health/config should avoid exposing auth integration internals such as `appName`, `expected_app_name`, validator module/function names, agent addresses, cookie names/domains, selected GA environment, secrets-adjacent paths, static CSRF tokens, admin/export keys, support keys, webhook secrets, magic-link/session-mint toggles, or session configuration unless access-controlled and operationally justified. Browser-visible runtime config that exposes operator/support/webhook/CSRF secrets or lets query parameters override those values must be reported as a concrete config-secret finding, even if the same app also has public audit/export diagnostics. |
| Production startup and health gates | Inspect startup behavior and health/readiness checks for production-like environments. The app should refuse to start, or at least mark readiness failed, when expected app name is empty, fallback/local validation is enabled, widget path is internal, secure cookies are disabled for HTTPS/prod, placeholder secrets are present, or validator infrastructure is unavailable. Report missing startup/readiness gates separately from the underlying misconfiguration when health checks can pass while auth is fail-open or misconfigured. |
| Runtime/image hygiene | Record unpinned base images or routine server-version disclosure as broader deployment hardening follow-ups when present, without displacing integration findings. |

Report these checks under deployment posture, identifying any broader security review needed.

## 10. Report Template

Produce the audit report using this skeleton and these section headers. Put positive verified behavior before findings, and order findings by severity.

```markdown
# TxGlobalAuth Integration Audit - <Application>

**Date**: <YYYY-MM-DD>
**Auditor**: <name or agent>
**Scope**: full integration audit (frontend, backend, deployment) against TxGlobalAuth integration best practices.

## 1. Summary

<Overall outcome, key blockers, and what was or was not runtime-tested.>

## 2. Architecture

<Observed browser, widget, exchange, validator, application-session, and deployment flow.>

## 3. Reference Criteria

<Criteria applied from global-auth:frontend, global-auth:backend, and any explicitly invoked security/deployment review.>

## 4. What's Done Right

<Verified choices with evidence locations and why they matter.>

## 5. Findings

### Critical (must fix before further integration work)

#### 5.1 <Finding title>

**Location**: `<file/config/runtime evidence>`

<Problem, impact, and required remediation.>

### Important (fix before production)

#### 5.2 <Finding title>

**Location**: `<file/config/runtime evidence>`

<Problem, impact, and required remediation.>

### Recommended (polish)

#### 5.3 <Finding title>

**Location**: `<file/config/runtime evidence>`

<Problem, impact, and recommendation.>

## 6. Findings Addressed In-Flight

<Include this section only when changes were made during the audit; record the resolved finding and evidence of the fix.>

## 7. Runtime Observations

<Static-only limitations, commands/tests run, widget behavior, validator negatives, and deployment observations.>

## 8. Priority Order

<Ordered remediation list or statement that no open remediation remains.>

## 9. Final Verdict

<Deployment-readiness conclusion and any required validation still outstanding.>
```

For a dogfood audit of the starter project that supplied this workflow, run all ten sections against its current state and compare classifications with its `REVIEW.md`: findings already repaired should appear as verified strengths or resolved observations, while any still-present behavior should remain in Findings.

