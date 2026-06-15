---
name: audit
description: "TxGlobalAuth (GlobalAuth, GA) full-stack integration audit and code review. Use when reviewing frontend/backend TxGlobalAuth apps for auth-widget/API usage, JWT validation, token exchange, session handling, identity keys, PII, CORS, deployment posture, or writing Global Auth audit reports."
---

# Global Auth - Integration Audit Skill

Use this skill to review an existing TxGlobalAuth integration from cold start to written report. Follow the sections in order. Earlier checks can invalidate later conclusions: a wrong widget build, unsupported API surface, or fail-open token exchange changes the meaning of the rest of the review.

## 1. Pre-Audit Gathering

Collect the minimum context before judging code:

- `appName` and expected authentication level: Kratos L1, Kratos L2, AD, or Anonymous.
- Network position: backend inside the corporate perimeter or outside it. This determines whether TxAuth Agent gRPC or HTTP introspect is the expected validator path.
- Frontend framework and backend language/framework.
- Whether the integration is new or a migration from a previous authentication system.
- Test access: registered `appName`, real token, TxAuth Agent or introspection access, configured GA environment, and approved browser origin.

Read the smallest useful file set:

- HTML entrypoint and widget script loading/configuration code.
- Widget initialization, `subscribeJWT`, login, logout, and user display code.
- Backend token exchange endpoint and validator adapter.
- Application session creation, authentication middleware, logout, and user persistence code.
- Environment/configuration documentation and deployment files such as `Dockerfile`.

State missing inputs in the report. Missing access is a runtime limitation, not evidence that the integration works.

## 2. Step 0 - Build Identity Verification

Verify the widget build before logic review.

1. Confirm the configured public wrapper URL ends in `/auth-widget/@8.js`. A URL containing `global-auth.v8.js` is the internal UI module and is a blocking finding.
2. Confirm the initialized browser export exposes `subscribeJWT`, `getJwt`, and `getTokenProvider`. Compatibility with private or legacy aliases is a finding because it can hide the wrong build. Report fallback chains such as `subscribeJWT || subscribeToJwt`, `getJwt || getPublicSession`, `subscribeJwt`, `getJWT`, `getAuthenticationState`, or optional use of private aliases as their own finding, even when the wrong CDN URL is already reported.
3. Cross-reference `global-auth:frontend` sections `Pre-Integration Health Check`, `Script Loading`, `Internal Build API vs Official Wrapper API`, `CDN Sources (v8 - current)`, `How to Choose and Document the CDN Host`, and `TxGlobalAuth Export Type`.
4. Verify the CDN hostname choice is documented near configuration or in operational docs. The path is universal; the hostname is regional. Check the selected host against the product audience, operator/trust chain, and data-residency expectations. A silent hardcoded CDN host is a finding even when the path is correct; do not mark the CDN check complete just because the path is correct.
5. Confirm script load failure is visible to the user/operator. For production-facing integrations, require a bounded timeout or recovery path; a loader that relies only on `script.onerror` can leave the UI waiting through long browser network timeouts and should be reported. If `loadScript()` or equivalent has no timer, this remains a finding even when init is awaited. Call out the missing loader timeout explicitly in the finding text; do not bury it inside a generic wrong-build or initialization finding.
6. Confirm `init()` is awaited or otherwise chained before any TxGlobalAuth API call, subscription, auth button enablement, or state read. Inspect both event-listener registration and initial control state in HTML and JavaScript: login, logout, refresh, transfer, support, renew, and other auth-dependent buttons must be disabled or inert before successful widget initialization and API-surface verification. Do not list "handlers are installed after init" as a strength if the controls are visible/clickable before API verification, or if the code can proceed after a wrong/partial build because it skipped the exact public API verification from step 2. Report pre-verification clickable controls as their own finding or as an explicit subpoint with locations.
7. Inspect browser delivery hardening for the external widget path. Record whether the integration has a Content Security Policy that limits script sources to the selected GA CDN host and whether the external widget script has an explicit integrity/version-pinning decision. Missing CSP/SRI/version-pinning posture is a production-readiness finding even when the widget URL itself is otherwise correct; do not let a wrong-build finding hide the delivery-control gap.

Stop deeper review until any wrong-build finding is understood.

## 3. API Surface Audit

Confirm `subscribeJWT` is the single source of truth for auth state.

- Trace every write to displayed identity, authenticated-user state, application session state, and local auth caches.
- Confirm login and logout button handlers call widget methods and let subscription callbacks update app state. They must not create or destroy app sessions directly before the widget state transition occurs.
- Confirm the `subscribeJWT` handler distinguishes first login, same-user token refresh, account switch, and `null` session loss. Report code that exchanges every non-null callback, because refresh should not create a new app session unless the product documents a server-owned revalidation need. Account switch must clear the prior backend app session before starting replacement exchange, or otherwise guarantee the old session cannot survive a failed replacement. `null` must clear the backend app session, and the clear/logout path needs the same stale-response protection as token exchange.
- In plain JavaScript or small starters, look for the absence of state such as `lastPersonKey`, `lastAuthLevel`, or equivalent server-derived identity tracking. A `subscribeJWT(response)` branch that always calls `exchangeToken(response.token, ...)` is an Important finding because refresh callbacks can create session churn and stale records.
- If exchange uses sequence guards, abort controllers, or stale-response checks, verify the backend logout/clear path uses equivalent protection. A delayed logout response must not overwrite or obscure a newer login/exchange state.
- Inspect async auth state transitions as races, not just linear flows. Look for delayed `setTimeout`, unawaited logout/clear calls, stale promise handlers, missing sequence/nonce checks, reused abort controllers, and UI updates that can run after a newer login, account switch, or token exchange. Report stale logout/clear UI or session overwrite risks separately when they can obscure or undo a newer authoritative auth state.
- Confirm progressive requirements such as `requireAgreements()`, `requireEmail()`, and `requirePhone()` are awaited or otherwise enforced before the app treats login as complete.
- Confirm there are no competing auth paths such as direct Kratos/global-auth browser requests, persisted raw GA tokens, raw-token replay from storage, BroadcastChannel/window-message token sharing, manual `getJwt()` exchange flows, or URL/message-driven auth state mutation.
- For any `postMessage`, `message`, BroadcastChannel, storage-event, mobile bridge, or opener/iframe auth path, verify both the source of the token and the sender context. A message token path without an exact origin/source allowlist and schema validation is a finding even if the broader "competing auth path" is already reported.
- Confirm login and token-renew error subscriptions do not silently retain, restore, or re-exchange the backend app session. Error handlers may surface UI state, but should not create a competing recovery path outside `subscribeJWT` unless a server-owned revalidation policy is documented.
- Confirm `subscribeJWT(null)` is never suppressed by browser state such as hidden tabs, offline mode, query flags, or local/session storage. Security state must be cleared even if rendering work is deferred; this is normally Important or higher when the backend app session can remain usable.
- Confirm auth-level downgrade and provider transition handling is explicit. If a user moves from L2 to L1, named to Anonymous, AD to Kratos, Kratos to AD, or any stronger-to-weaker state, the app must remove privileges that depended on the previous level/provider and clear or replace the backend session as needed. Do not treat downgrade callbacks as harmless same-user refreshes.
- Inspect post-login helpers tied to the auth callback. URL parameters must not automatically trigger authenticated profile, account-linking, trusted-session, settings, billing/payment/transfer, support, admin mutations, or unconstrained `next` navigation merely because GlobalAuth login completed.
- Search for navigation parameters such as `next`, `returnTo`, `redirect`, `continue`, `target`, and `callback`. They must be constrained to same-origin relative paths or a server-owned allowlist. Report open redirects and post-login navigation that can carry tokens, sessions, or users to attacker-controlled destinations.
- Treat pre-widget `/api/session` restoration as provisional. If the widget immediately overwrites it, call it dead startup state; if the widget fails or returns `null`, the app must not keep showing the old app session as authoritative.
- Flag URL-carried app-session bootstrap such as `sid`, session ids, handoff ids that are actually session ids, or restore links that set the normal app-session cookie before GlobalAuth is authoritative.
- Search frontend code and runtime config for GlobalAuth debug features such as `TxGlobalAuth.debug.enableLogging()`, `debug.screenNavigationEnable`, verbose token/session dumps, debug panels, or development-only auth traces. Debug tooling must be disabled or strongly gated outside local development and must not expose raw JWTs, person objects, identifiers, or widget navigation internals to normal users.

Cross-reference `global-auth:frontend` sections `Recommended Integration Architecture`, `Authentication Methods`, `Progressive Verification (require* methods)`, `JWT Token Management`, `subscribeJWT Handler - Required Cases`, `Event Subscriptions`, and `Diagnosing Existing Integrations`.

## 4. Token-Exchange Audit

Confirm the backend treats the GA token as a short-lived bearer that must be validated server-side and then replaced by an app-owned session.

- Confirm the backend never parses JWTs locally for trust decisions.
- Confirm validation goes through TxAuth Agent gRPC when inside the corporate perimeter, or HTTP introspect only when outside the perimeter with documented justification. HTTP introspect must use protected transport, the approved request shape, service authentication when required, and must not put raw GA tokens in URLs or query strings.
- For HTTP introspect integrations, verify service credentials are server-only secrets. Service tokens, API keys, client certificates, or equivalent auth material must not be hardcoded, committed, browser-exposed through config endpoints, baked into public images, printed in startup/config logs, or returned in diagnostic responses.
- Confirm the validator success contract requires strict booleans: `active is True` and `validated is True` before session creation. Truthy strings, objects, field presence, warning statuses, or locally normalized custom statuses are not enough. Check both fields; do not credit strict validation if either one is tested with generic truthiness.
- Inspect the exact condition used for validator acceptance and report truthiness patterns even when the validator adapter normally returns booleans. Examples include Python `if not result.get("active")`, JavaScript `if (body.active)`, PHP `if ($result['validated'])`, or any equivalent that would accept `"true"`, `1`, non-empty arrays/objects, or field presence. This must be a finding unless the code explicitly checks strict true values before creating an app session.
- Confirm the expected `appName` or equivalent product scope is passed to the validator or enforced by the validator adapter.
- Confirm validator-returned scope agrees with server-owned configuration when such fields exist. Returned `appName`, `audience`, `env`, `environment`, or `issuer` mismatches must fail closed before app-session creation, not become warnings in browser responses.
- Treat validator scope comparison as a two-sided check: browser/request scope must not drive validation, and validator-returned scope must be compared against server-owned expected values before any app session is created. If either side is missing, report it explicitly instead of grouping both under "browser-supplied scope."
- Confirm the validated token type, variant, provider, and authentication level are acceptable for the product action before app-session creation. A token can be valid but still wrong for the product: read-only tokens, Anonymous tokens, lower-than-required Kratos levels, AD tokens in a Kratos-only app, or tokens from the wrong variant/provider must not create the same session privileges as the intended mode.
- Confirm product scope is server-owned. The backend must not accept `expectedAppName`, `appName`, `env`, provider, tenant, org/account context, auth level, or similar validation-scope fields from the browser as authority for token validation or app-session creation.
- Inspect the exchange request schema, not just the fields currently consumed by the validator. Browser-sent fields such as `authLevel`, `clientAuthLevel`, `displayHint`, `person`, `selectedAccount`, `tenant`, `provider`, `expectedAppName`, `appName`, `env`, `ttl`, `rememberMe`, or account/org ids are findings when they are trusted, stored, echoed as authoritative state, or create ambiguity about which party owns validation and authorization. If the backend ignores a risky browser-sent field today, record it as a recommended cleanup unless it is part of a documented display-only contract with response minimization.
- Confirm `/api/auth/exchange` or its equivalent is the only consumer of raw GA tokens. Inspect request parsing, headers, form handlers, callbacks, beacons, mobile bridges, and query parameters for names such as `token`, `ga_token`, `jwt`, and `X-GA-Token`. Callback, beacon, form, text, header, mobile, or URL-query token exchange variants are findings unless they are intentionally supported and apply the same validator contract, origin/CSRF controls, rate limits, body limits, logging redaction, response minimization, and no URL-carried bearer tokens.
- Confirm the application session ID is app-owned, random, freshly issued after validation, and stored in an HttpOnly cookie or equivalent server-side session mechanism. Browser-supplied session IDs and reuse of pre-authentication cookie IDs at exchange are findings.
- Confirm app-session lifetime is server-owned and intentionally bounded. Inspect defaults such as `SESSION_TTL_SECONDS`, cookie `Max-Age`, remember-me policies, refresh windows, and idle/absolute timeout handling. Report long-lived starter defaults or app-session lifetimes that are not justified by a server-owned revalidation and revocation model. Browser-supplied `rememberMe`, TTL, expiry, freshness, device, or trusted-session flags must not extend the app session unless that same server-owned policy and revalidation model is documented.
- Inspect any validator-result cache. Cache keys, logs, metrics, diagnostics, and admin exports must not expose raw GA tokens; positive cache entries must not outlive revocation, account switch, auth-level changes, or app-scope changes in a way that bypasses fresh validation or session-state checks.
- Inspect application, access, reverse-proxy, framework, exception, and audit logs around token-bearing routes. Raw GA tokens, Authorization headers, request bodies, service credentials, full validator responses, and unredacted person/session objects must not be logged. Redaction must happen before generic request logging, error serialization, tracing, metrics labels, or debug dumps can capture token material.
- Include startup and configuration logging in the log review. Auth-facing services must not print service tokens, session secrets, API keys, validator URLs with credentials, raw environment dumps, or secret-derived config at boot; startup logs often bypass route-level redaction.
- Confirm exchange and session-status responses return only the minimal UI state needed. Live application session IDs, raw validator/session objects, validation-cache state, raw GA tokens, or internal authorization fields in JSON responses are findings unless there is a documented server-to-server diagnostic channel with redaction.
- Confirm auth and validator error responses are minimized. Browser-facing errors must not disclose raw upstream validator payloads, GA tokens, service tokens, agent hostnames, internal service names, Consul paths, stack traces, registry/appName internals, or environment-specific infrastructure details.
- Inspect failure branches as carefully as success responses. Shared secrets, expected CSRF tokens, expected support/admin/linking secrets, allowed app names, validator endpoints, stack traces, and "expected value" hints must not be returned to the browser. Report diagnostic detail in 4xx/5xx responses even if the same secret is also exposed through a success-path diagnostics endpoint.
- For reference starters, a pluggable validator adapter is acceptable when it fails closed and the docs clearly say the concrete gRPC stubs are product-specific. Record live validator coverage as not runtime-tested; do not report the missing concrete adapter as a core integration bug by itself.
- Separate adapter code from generated/runtime inputs. A missing adapter module that the app imports, Dockerfile copies, or README presents as included is a finding because the advertised validation path cannot run. Missing generated protobuf/gRPC stubs, real agent address, or real token access are normally runtime limitations for a starter; record them in Runtime Observations, not Findings, unless the repo explicitly claims those generated files are vendored or deployable as-is.
- Inspect the gRPC channel boundary. Insecure local channels are acceptable only with a documented same-host/private-network boundary, sidecar model, or compensating transport control; otherwise report the gap as production posture for the validator path.

Cross-reference `global-auth:backend` sections `Token Validation — Required`, `Approach 1: TxAuth Agent (gRPC Token Introspection)`, `Approach 2: Kratos JWT Introspection (HTTP)`, ``Cross-Product Token Validation (`appName`)``, `Anti-Patterns`, and `Common Integration Pattern: Token Exchange`.

## 5. Identity-Key Audit

Confirm the integration stores the stable provider-appropriate identity key.

- For Kratos users, prefer `person.kratosId` or `kratos_id` as the stable cross-level identity key. Do not use `person.id` as the durable cross-level Kratos key.
- For AD users, use `person.id` as the AD GUID when `kratosId` is absent.
- For Anonymous users, treat `person.id` as a device UUID and classify using the validator `provider` field where available.
- Inspect provider parsing and auth-level classification order. Provider dictionaries may use fields such as `name`, `provider`, `type`, `source`, or `authProvider`; if an Anonymous token without a guaranteed `isAnonymous` flag is classified as any named account level such as AD, Kratos, L1/L2, customer, or generic `named`, report it. Check both paths where `person.id` exists and paths where provider alone says anonymous.
- Verify the classifier actually receives and uses the validator `provider` value where Anonymous/AD/Kratos distinctions matter. A helper that only receives `session` and ignores provider is suspect even if it checks optional `isAnonymous` flags. Report classifiers that treat the mere presence of `person.id` as AD, L1/L2, client, customer, or another named/high-auth state without first excluding Anonymous/provider-specific cases.
- Confirm the app does not use email, login, phone, display name, or other PII as a durable identity key.
- Confirm local user/account records are namespaced by the server-owned product and identity context needed by the app, such as `appName`/environment/provider/account realm, so stable ids from different products or providers cannot collide. Report missing namespacing as its own identity/account-binding issue even when the same code also has PII-key or authorization findings.
- Verify the namespace is present in the actual persistence key or unique constraint, not only in a session field or display object. Look for maps keyed only by email, person id, login, account id, or external id; upserts that omit app/environment/provider/realm; and cross-product support/admin linking paths. Missing namespace remains a distinct finding even when the app also uses the wrong identity key.
- Confirm Anonymous or guest-like identities are separated from named account records unless the product explicitly documents a linking model. Do not let anonymous device ids become durable customer-account records by default.
- Confirm auth level and provider changes are not mistaken for harmless same-user refreshes when the app has level-specific or provider-specific authorization.

Cross-reference `global-auth:backend` sections `Session Structure by Token Type`, `person.id Semantics — Critical`, `Anonymous Classification Using Validator Provider`, `Data Availability by Provider and Auth Level`, and `Storing User Data — Trade-offs`.

## 6. PII Surface Audit

Find every code path that reads, stores, logs, renders, or returns person data such as `person.email`, `person.name`, `person.login`, and `person.verifiedPhone`.

For each use:

- Consult the Data Availability tables in `global-auth:frontend` and `global-auth:backend` to confirm whether the field is available for the product provider/auth level.
- Classify the use as display, diagnostics, matching, authorization, persistence, or export.
- Display can be acceptable when it is minimal and intentional. Identity comparison, account matching, authorization, or durable storage based on PII is an anti-pattern unless the product has a documented reason.
- Inspect app-session models, internal session dataclasses, cache values, serialized cookies, server-side session stores, and database upserts for copied `email`, `name`, `lastname`, `login`, `phone`, display names, or raw `person` objects. Storing PII in the application session is persistence, not harmless display, because it becomes a durable identity surface and is often returned by status/debug endpoints.
- Browser logs and debug panels should not render raw session/person objects in production-facing flows.
- Public lookup or diagnostics endpoints must not search or disclose account existence by email, login, phone, display name, raw external id, auth level, or provider unless authenticated, authorized, and intentionally documented.

Cross-reference `global-auth:frontend` section `Data Availability by Provider and Auth Level` and `global-auth:backend` sections `Data Availability by Provider and Auth Level`, `Storing User Data — Trade-offs`, and `Data Not Available from TxGlobalAuth`.

## 7. Fail-Closed Audit

Verify the backend refuses to authenticate when the validator path is missing, degraded, or explicitly local-only.

- Missing TxAuth Agent or introspection configuration must refuse exchange. It must not fall back to local JWT parsing or unsigned token acceptance.
- Dev bypasses, mock validators, and fake tokens must be impossible to enable in production-like environments.
- Validator errors, malformed responses, missing session data, and negative validation must not create app sessions.
- Startup/readiness should make production-incomplete auth configuration visible. At minimum, report if production can start with placeholder `appName`, missing validator, unsafe HTTP introspection, dev bypasses, validation caching that can bypass freshness, wrong widget path, insecure cookies, or undocumented origin/environment settings.

Cross-reference `global-auth:backend` sections `Token Validation — Required`, `Do NOT Parse the JWT Manually for User Data`, `Do NOT Skip Validation for "Trusted" Frontends`, and `Reviewing Existing Backend Integrations`.

## 8. CORS / Environment Audit

Verify the browser origin, GA environment, and CDN/runtime configuration agree.

- Confirm the app's local and deployed origins are approved for the selected GA environment. If the origin is not listed, report the practical failure mode: the widget loads but GA backend requests fail browser CORS checks.
- Compare documented local ports against the approved dev-origin list when that list is available. Check application defaults, Docker `ENV`/`EXPOSE`, run commands, and README URLs together. A starter defaulting to an unapproved port is a starter finding, not just an unverified environment note. In the reviewed starter, known dev-allowlisted local origins included `http://localhost`, `http://127.0.0.1`, `http://127.0.0.1:8080`, `http://127.0.0.1:8081`, `http://localhost:3000`, `http://localhost:5000`, and `http://localhost:8080`; use project-owned GA environment documentation as the authority for current values.
- Confirm documentation tells developers whether to request allowlist inclusion or run on an already-approved local port/origin.
- When the origin is missing, recommend either changing the local default to an already-approved origin such as `127.0.0.1:8080` when that matches the product's environment, or requesting allowlist inclusion for the exact scheme, host, and port through the Global Auth team.
- Confirm browser-visible config cannot silently switch `env`, `appName`, CDN host, or CDN path away from the documented runtime values.
- Confirm visible product/app text that claims to be configurable, such as page title, hero copy, or displayed app name, is driven from the same server-owned config. Hardcoded placeholder product names are polish findings when they confuse app registration or operator validation.
- Confirm production and development examples do not imply that a CDN reachability problem is the same as a GA backend CORS allowlist problem.
- For app APIs using the GA-backed session cookie, credentialed CORS must be same-origin-only or exact allowlist based. Origin reflection and broad preflight methods or sensitive headers are findings unless explicitly justified by a registered cross-origin product flow.

Use project-owned GA environment docs or configuration as evidence when available. If the allowlist cannot be inspected, state that the origin registration was not verified.

## 9. Deployment-Posture Audit

Check and report, as production-readiness findings when relevant:

- CSRF or exact Origin/Referer protection on app-session creating, destroying, and browser-facing state-changing routes that use the GA-backed app session.
- CSRF tokens, when used, must be unguessable and bound to the session, origin, or a short rotation window. Public process-wide/static tokens, or tokens accepted before Origin/Referer checks, do not provide meaningful protection for GA-backed session mutations.
- Request body-size limits on token-bearing exchange routes. Report the absence of a route-specific limit even when the route also has larger validation failures; do not let fail-open validation obscure resource-exhaustion and logging-risk controls.
- Rate limiting and body-size limits are separate checks. Do not collapse them into one finding unless both are actually implemented or both are missing and clearly evidenced.
- Rate limiting on token exchange, keyed by a trusted client identity. Report a missing rate limit as a separate deployment-posture finding or explicit runtime observation, even if token validation itself is already Critical. Inspect the exact key derivation code. If it reads `X-Forwarded-For`, `Forwarded`, `X-Real-IP`, or similar client-controllable proxy headers without a documented trusted proxy that strips and rewrites them, report that as a finding even when a rate limiter exists. Do not list the rate limiter as "done right" unless the key source is trusted or clearly limited to local-only starter scope.
- Origin/CSRF helpers must build the expected origin from trusted server configuration or trusted proxy metadata. Client-supplied `X-Forwarded-Host` or `X-Forwarded-Proto` are findings unless the deployment boundary is documented.
- Secure, HttpOnly, SameSite, and narrow Domain/Path cookie posture for app sessions.
- App-session cookie names should be product-specific enough to avoid collisions with sibling apps on the same host/path. Generic names such as `session` are production findings unless host/path isolation is documented.
- Durable session storage and revocation posture for restarts, multiple instances, logout, and account switch. Process-local dictionaries, in-memory maps, or per-instance stores are production findings for GA-backed app sessions even when acceptable for a starter.
- Protected product endpoints should derive org/account selection, required provider/auth level, role, and entitlement from the server-owned app session plus product data. Browser headers, query parameters, or request-body flags may request a selection, but must not authorize it.
- Audit protected product routes after login, not only the token exchange endpoint. Inspect profile, account, portfolio, billing, transfer, admin, support, export, invite, linking, settings, and mutation endpoints for continued reliance on browser-supplied account ids, auth levels, roles, tenant ids, provider names, or display/person fields. A correct exchange endpoint does not prove downstream authorization is correct.
- Account-linking, support-linking, account-merge, invite-accept, and identity-association routes must prove control of the target identity and apply server-side ownership, freshness/step-up, and replay controls. A browser-supplied target id, email, account id, URL parameter, shared support secret, or browser-visible secret is not proof. Report support/admin linking helpers separately when they can bind an email or account without a server-owned identity proof, even if a broader "browser-supplied authorization" finding already exists.
- Check logout and revocation scope against the product's session model. Determine whether logout clears only the current browser session, all app sessions for the GA identity, all sessions for the selected account, or stale sessions created during earlier exchanges. Missing or undocumented revocation semantics are findings when multi-device access, admin disablement, account switch, or high-risk actions depend on timely session invalidation.
- Non-root containers, health/readiness checks, no secrets or PII in images, minimal image context, and sane local bind examples. Inspect `COPY .` or broad context copies in auth-facing images because they can ship local artifacts, generated reports, caches, or secrets unless `.dockerignore` is proven sufficient.
- Dependency installation posture for deployable images: use cache-conscious install flags where appropriate, avoid retaining package-manager caches, and document exceptions. For Python images, record `pip install` without `--no-cache-dir` as a low-severity deployment-posture issue when the image is intended to be deployed.
- Pin deployable base images or document the update policy; floating runtime tags are production posture findings for auth-facing starters. Also inspect server banners on auth endpoints. Framework/runtime version disclosure is usually low severity, but it should be recorded when visible and easy to suppress.
- CSP/script-host and SRI/version-pinning decisions for production-facing external widget delivery.

Name the GlobalAuth-specific reason each item matters, then defer detailed remediation to an appropriate security or DevOps review unless the user explicitly asks for a full hardening pass.

For starter-style apps that run without external credentials, run lightweight local checks when feasible: missing-validator exchange fails closed, any dev bypass is disabled in prod, malformed cookies do not 500, static path traversal is blocked, and Docker/non-root/healthcheck claims are verified or marked static-only. If the widget cannot be exercised end-to-end, state the exact missing input such as registered `appName`, approved origin, real token, or validator access.

## 10. Report Template

### Completion Pass - Omission Guardrails

Before finalizing the report, perform a short coverage pass:

- Re-scan the findings against the source checklist in sections 2-9 and ensure each concrete issue is either reported, explicitly grouped with enough detail and evidence locations, or recorded as not applicable.
- Do not let Critical findings hide Important/Recommended checks that still matter for remediation planning. Wrong widget build and fail-open validation do not eliminate the need to mention missing loader timeouts, pre-init control enablement, downstream authorization routes, support/account linking, cookie posture, body limits, rate limits, diagnostics, and deployable image posture.
- When grouping related findings, name the individual risky mechanisms in the finding body. For example, "deployment hardening is incomplete" should list floating base image, root user, missing healthcheck, image-baked secrets, broad `COPY .`, and package-cache retention when those are present.
- Run an edge-case sweep over non-happy-path auth behavior before writing the final verdict:
  - browser loading and delivery controls: loader timeout/recovery, CSP, SRI/version-pinning, and documented CDN host choice;
  - asynchronous state races: stale logout/clear responses, delayed callbacks, account-switch replacement failures, and hidden/offline/null-session suppression;
  - ambient browser inputs: `postMessage` origin/source checks, storage replay, URL-triggered mutations, URL-carried session IDs, and open redirect parameters;
  - validator boundaries: browser-request scope ownership and validator-returned scope mismatch against server-owned expected app/environment/issuer/provider/auth level;
  - durable identity/account binding: provider-aware stable keys, Anonymous separation, persistence namespace/unique constraints, and support/admin linking paths;
  - disclosure surfaces: success and error responses, startup logs, diagnostics, admin exports, expected-token/secret hints, and browser debug panels.

Produce the audit report using this structure. Put verified strengths before findings. Order findings by severity. Keep the report focused on the sections below unless the user explicitly asks for supplemental detail.

```markdown
# Code Review - <Application>

**Reviewer**: <name or agent>
**Date**: <YYYY-MM-DD>
**Scope**: full integration audit (frontend, backend, deployment) against TxGlobalAuth integration best practices.
**Verdict**: <merge/deployment-readiness summary>

This document is self-contained: it includes the architecture, the reference best-practice criteria each finding is checked against, the runtime observations, and a concrete priority order. A reader does not need to chase external skills, configs, or repos to follow the reasoning.

---

## 1. Summary

<Overall outcome, key blockers, and what was or was not runtime-tested. Lead with whether the integration is fundamentally correct before listing defects.>

## 2. Architecture

<Concise observed browser, widget, exchange, validator, application-session, and deployment flow. A small flow diagram is fine.>

## 3. Reference Criteria

<Self-contained criteria applied from global-auth:frontend, global-auth:backend, and any explicitly invoked security/deployment review. Include enough of the contract for a reader to understand the findings without opening the skill files.>

## 4. What's Done Right

<Verified choices with evidence locations and why they matter. Do not list a behavior as done right if a sibling route, fallback path, or runtime default violates the same control; qualify it or omit it.>

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

<Commands/tests run, widget behavior, validator negatives, local CORS/appName observations, path traversal/session-cookie probes when performed, Docker/container observations, and runtime checks that could not be run. Separate real GA validation from dev-bypass behavior.>

## 8. Priority Order

<Ordered remediation list or statement that no open remediation remains.>

## 9. Final Verdict

<Merge/deployment-readiness conclusion and any required validation still outstanding. For reference starters, say whether the core integration is ready as a starter while naming the production work that remains; avoid implying production readiness merely because no Critical findings remain.>
```
