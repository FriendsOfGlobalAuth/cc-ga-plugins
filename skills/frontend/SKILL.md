---
name: frontend
description: "TxGlobalAuth (GlobalAuth, GA) JavaScript widget integration. Use when working with TxGlobalAuth, auth-widget, Kratos authentication, JWT token management, or embedding authentication UI in frontend projects."
---

# Global Auth - Frontend Integration Skill

This skill provides expert knowledge of the TxGlobalAuth authentication widget for frontend integration.

**Important**: All TxGlobalAuth methods are available only after successful `init()`. Always structure code so that all API calls happen inside the `.then()` callback of `init()` or after `await init()`.

## Pre-Integration Health Check

**Before writing or reviewing any TxGlobalAuth code, verify the integration baseline.** Most integration bugs stem from loading the wrong build or calling non-existent methods. Run these checks first — they take 30 seconds and prevent hours of debugging.

**Step 0 — Verify the widget build:**

1. **Check the CDN URL** in the HTML/script loader. The path must end with `/auth-widget/@8.js` — the hostname varies by CDN region (there are multiple CIS and international CDN hosts), but the path is always the same.
   - Correct pattern: `https://{cdn-host}/auth-widget/@8.js`
   - If you see `global-auth.v8.js` in the URL (any host) — **STOP**. This is the internal UI module, not the official wrapper. See "Widget Architecture" below.

2. **Check the API surface** in the deployed environment (browser console):
   ```javascript
   typeof TxGlobalAuth.subscribeJWT    // Must be 'function'
   typeof TxGlobalAuth.getJwt          // Must be 'function'
   typeof TxGlobalAuth.getTokenProvider // Must be 'function'
   ```
   If any returns `'undefined'` — the wrong build is loaded. The internal build has a completely different API (`subscribeToJwt`, `getPublicSession`, etc.).

3. **Check the export type**:
   ```javascript
   typeof TxGlobalAuth                 // May be 'function' or 'object' — both are valid
   ```
   The official build may export `TxGlobalAuth` as a constructor function (not a plain object). Type checks in existing code must accept both — see "Script Loading" below.

**Only proceed with code review or refactoring after all three checks pass.** If Step 0 fails, fix the CDN URL first — all other issues may be symptoms of the wrong build.

## Integration Rules

**Before writing any init code**, you MUST know the `appName`. First check the existing codebase for a TxGlobalAuth.init() call — if found, reuse that `appName`. If there is no existing init in the project and the user hasn't specified one, ask directly: "Which `appName` should I use for TxGlobalAuth init?" Do not guess or use a default value — the `appName` determines the authentication mode (Kratos, AD, or Anonymous) and using the wrong one will produce non-functional code.

**CORS and local development**: The `prod` environment does **not** allow `localhost` origins — opening an HTML file directly or serving from `localhost` will cause CORS errors. Only `dev`, `tst`, and `uat` environments allow localhost. When the user is building a demo or developing locally, advise them to use a non-prod `env` value (e.g. `dev` or `tst`). Also note that non-prod environments have their own CDN script URLs — ask the developer for the correct URL if the environment is not `prod`.

## Widget Overview

TxGlobalAuth is a JavaScript authentication widget loaded from CDN that provides modal/inline authentication UI, JWT token management, multi-provider support, and progressive user verification flows.

### Three Authentication Modes

The widget operates in one of three modes, determined by `appName` configuration:

| Mode | Method | Description |
|------|--------|-------------|
| **Kratos** | `authenticate()` | Standard authentication with email/phone + code/password and OIDC. Most common mode. |
| **Active Directory** | `authorize()` | Corporate AD/LDAP authentication. Used in internal enterprise products. |
| **Anonymous (Device)** | `authorizeAnonymously()` | Device-level token without user identity. Used for anonymous tracking. |

The mode is determined by the `appName` — each product has a preconfigured mode. When integrating, **ask the developer which `appName` to use** — this determines the available authentication mode and other defaults (palette, lang, etc.).

### Authentication Levels

| Level | Description | Check Method |
|-------|-------------|--------------|
| **Guest** | No token. Not authenticated. | None of the `isAuthenticated*` methods return `true` |
| **Anonymous** | Device token only, no user identity. | `TxGlobalAuth.isAuthenticatedAnonymous()` |
| **L1 (Account)** | Authenticated user (Kratos or AD). | `TxGlobalAuth.isAuthenticatedUserAccount()` |
| **L2 (Client)** | L1 + second factor verified (e.g. 2FA). | `TxGlobalAuth.isAuthenticatedUserClient()` |

Note: `isAuthenticatedUserAccount()` returns `true` for both Kratos L1 and AD-authenticated users.

### Data Availability by Provider and Auth Level

**Not all `person` fields are populated for all providers and auth levels.** If you see code accessing `person.email` or `person.name` — check whether the field is actually available for the product's provider type. A field being in the TypeScript type does not mean it has a value.

| Field | Kratos L1 special (deprecated) | Kratos L1 | Kratos L2 | AD | Anonymous |
|-------|------------------------|-------------------|-----------|-----|-----------|
| `person.id` | kratosId | kratosId | clientGUID (changes!) | AD GUID | deviceUUID |
| `person.kratosId` | present | present | present | **absent** | **absent** |
| `person.email` | **present** | **absent** | may be present | present | **absent** |
| `person.name` | may be absent | may be absent | present | present | **absent** |
| `person.lastname` | may be absent | may be absent | present | present | **absent** |
| `person.login` | absent | absent | may be present | sAMAccountName | **absent** |
| `person.verifiedPhone` | absent | absent | may be present | may be present | **absent** |

**Key takeaways:**
- **Kratos L1** — the most common mode for consumer products — has **no email, no name, no login** in the token. Only `person.id` and `person.kratosId` are reliable. If you need a verified email, use `requireEmail()` — it checks Kratos identity first and only prompts the user if the email is missing; after it resolves, the identity is updated and `getIdentifiers()` returns the email. Alternatively, obtain the email through the backend via `get-identity`.
- **Kratos L1 special (deprecated)** — a legacy provider configuration where `person.email` IS present in the L1 token. If you encounter an integration that reads email from L1 tokens and it works — the product may be using this deprecated provider. New integrations should NOT depend on this behavior.
- **person.id changes meaning** between L1 and L2 for Kratos users — see "person.id vs person.kratosId" in the `global-auth:backend` skill.
- **AD users** have no `kratosId` — use `person.id` (AD GUID) as the identifier.
- **Token fields are scalars, Kratos stores collections**: `person.email` and `person.verifiedPhone` in the token are single values, but a Kratos identity may have **multiple** emails and phones. The token only contains one (if any). To check whether a user has a verified email or phone **at all**, use `getIdentifiers()` (returns masked values) or `getVerificationInfo()` — do not rely on token fields for presence checks.
- **Reading email/phone from token fields** is not strictly wrong if the value happens to be present, but it is **fragile** — the field may be absent depending on provider and auth level (see table above). If you see code reading `person.email` or `person.verifiedPhone` from the token, **raise the question**: is this field guaranteed to be present for this product's provider type? If not, consider `requireEmail()` / `requirePhone()` (idempotent — resolves instantly if the identity already has the data, otherwise prompts and saves to Kratos) or a backend `get-identity` call.

### Key Integration Concepts

**1. `subscribeJWT` is the single source of truth**: All auth state management must go through the `subscribeJWT` callback — it is the **only** place where you read and update the current user, token, and session. The callback fires immediately on subscription (with the current token or `null`), and then on every change: login, logout, token refresh, session changes from other tabs. **Handle both cases**: when `response` has a token (user is authenticated) and when it is `null` (user logged out or session expired). Button handlers for login/logout should only call `authenticate()`/`authorize()`/`logout()` — they must **not** update app state directly. The state update happens reactively when `subscribeJWT` fires in response.

> **Anti-pattern**: Do not split auth state management between `subscribeJWT` (for login) and `subscribeGlobal('logout')` (for logout). This leads to stale UI when logout happens externally (other tab, console, session expiry). `subscribeJWT` already covers logout — the callback fires with `null`.

**2. Existing app migration — logout is the #1 integration mistake**: When integrating TxGlobalAuth into an existing application, find **every place** where the app currently clears auth state on logout (clearing cookies, localStorage, resetting stores, redirecting) and replace them with `TxGlobalAuth.logout()`. The old direct-cleanup approach breaks cross-tab logout, external session management, and the reactive `subscribeJWT` architecture. If you leave the old logout handlers alongside TxGlobalAuth, the app will have two competing logout mechanisms and the UI will get out of sync.

## Recommended Integration Architecture

The methods described below are individual building blocks. This section shows how to assemble them into a working integration. **Read this first** — it prevents the most common architectural mistakes.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  ROOT LEVEL — Global Auth Provider (persists across pages)  │
│                                                             │
│  1. Load CDN script                                         │
│  2. TxGlobalAuth.init({ env, appName })                     │
│  3. subscribeJWT(callback) — THE ONLY place that reads      │
│     and updates auth state in your app                      │
└─────────────────────┬───────────────────────────────────────┘
                      │ auth state flows DOWN reactively
          ┌───────────┴───────────┐
          ▼                       ▼
┌──────────────────┐   ┌──────────────────────┐
│    LOGIN PAGE    │   │   LOGOUT BUTTON      │
│                  │   │                      │
│ Calls only:      │   │ Calls only:          │
│ authenticate()   │   │ TxGlobalAuth.logout()│
│ or authorize()   │   │                      │
│                  │   │ Does NOT clear state  │
│ Does NOT update  │   │ directly — state      │
│ app state — the  │   │ clears reactively     │
│ subscribeJWT     │   │ when subscribeJWT     │
│ callback in the  │   │ fires with null       │
│ provider handles │   │                      │
│ everything       │   │                      │
└──────────────────┘   └──────────────────────┘
```

**Key rule**: Auth state flows in one direction — from `subscribeJWT` callback down to the rest of the app. Login and logout buttons only trigger widget methods; they never touch app state directly.

### Why This Architecture

- **subscribeJWT at root level** survives page navigation. If placed on the login page, the subscription dies when the user navigates away.
- **Logout via `TxGlobalAuth.logout()` only** ensures that cross-tab logout, console logout, and server-side session revocation all work — they all flow through `subscribeJWT(null)`.
- **Login page has no state logic** because `subscribeJWT` already fires when `authenticate()`/`authorize()` completes — duplicating the state update creates race conditions.

### Common Migration Mistake

When integrating into an **existing** application that already has authentication:

1. **Find all existing logout handlers** — search for cookie clearing, localStorage removal, store resets, redirect-on-logout logic
2. **Replace them** with `TxGlobalAuth.logout()` — the reactive cleanup happens via `subscribeJWT(null)`
3. **Move auth state initialization** to the root-level `subscribeJWT` callback — not the login page
4. **Remove any direct state manipulation** from login/logout button handlers

If you skip step 1-2, the app will have two logout mechanisms that conflict with each other.

**3. Progressive requirement chains**: For actions that require a specific auth level, build a chain of `authenticate()` → `require*()` calls in the action handler. Every method in the chain is idempotent — if the requirement is already met, it resolves instantly. For example, a "Post comment" button handler should call `authenticate()` → `requireAgreements()` → `requireEmail()` → then submit the comment with the JWT token. This way, guests get the full onboarding flow, while authenticated users skip straight to submission.

## Script Loading

### Widget Architecture: Two-Layer Build System

TxGlobalAuth has a **two-layer architecture**. Understanding this prevents the single most common integration mistake:

| Layer | File | What it is |
|-------|------|------------|
| **Official wrapper** | `auth-widget/@8.js` | Public API facade. Adds `subscribeJWT`, `getJwt`, `getTokenProvider`, and all documented methods. **This is what you must load.** |
| **Internal UI module** | `global-auth.v8.js` | Internal rendering engine with a **private, undocumented API** — no backward compatibility guarantees for external consumers. **NEVER load this directly.** |

The wrapper (`auth-widget/@8.js`) internally loads the UI module itself — there is no reason to reference `global-auth.v8.js` directly, and doing so bypasses all public API guarantees.

> **Critical**: If you see `global-auth.v8.js` in a `<script>` tag or dynamic loader — the project is using the internal build directly. This is not a supported configuration. The internal module's methods can be renamed, removed, or changed at any version bump without notice. The entire public API described in this skill is unavailable. Fix the CDN URL before doing anything else.

### Internal Build API vs Official Wrapper API

If you encounter these method names in existing code, the project is loading the **internal build** (`global-auth.v8.js`) instead of the official wrapper (`auth-widget/@8.js`):

| Internal build (wrong) | Official wrapper (correct) | Notes |
|------------------------|---------------------------|-------|
| `subscribeToJwt` | `subscribeJWT` | Different casing AND different name |
| `getPublicSession` | `getJwt` | Completely different method name |
| `getAuthenticationState` | `isAuthenticatedUserAccount` | Different name and possibly different return type |
| No `getTokenProvider` | `getTokenProvider()` | Method does not exist in internal build |

**If you see `subscribeToJwt`, `getPublicSession`, or `getAuthenticationState` in code — do NOT treat them as typos or "fictional methods".** They are private methods of the internal build — not part of the public API and not covered by backward compatibility guarantees. They can be renamed or removed in any release. The only fix is to change the CDN URL to the official wrapper and update all method calls to the public API.

**Fallback chains like `getJwt || getJWT || getToken` in existing code** are a symptom of developers not knowing which build they loaded, trying to find a working method by trial and error. Do not remove these chains without first fixing the CDN URL — the chain may be the only thing keeping the integration partially functional.

### CDN Sources (v8 - current)

The official wrapper is always at the path `/auth-widget/@8.js`. The hostname varies by CDN region — there are multiple CDN hosts for CIS (5 hosts) and international (3 hosts) deployments. Two common examples:

```html
<!-- CIS CDN (one of several hosts) -->
<script src="https://libs-cdn.finam.ru/auth-widget/@8.js"></script>

<!-- International CDN (one of several hosts) -->
<script src="https://libs-cdn.lime.co/auth-widget/@8.js"></script>
```

**CDN selection**: The project should already have a CDN URL configured — check the existing codebase. If starting from scratch, ask the developer which CDN host to use for their product and region.

**How to verify**: The URL path must end with `/auth-widget/@8.js`. The hostname doesn't matter for correctness — all CDN hosts serve the same build. What matters is that the path points to the **wrapper** (`auth-widget/@8.js`), not the **internal module** (`global-auth.v8.js`).

> **Warning about other URLs**: URLs containing `global-auth.v8.js` in the path (on any host, e.g. `ga.lime.co/globalauth/ga/global-auth.v8.js`) point to the **internal UI module**, not the official wrapper. The widget UI may appear to work, but the JavaScript API is completely different, private, and **may change in any release** without backward compatibility.

### TxGlobalAuth Export Type

The official build may export `TxGlobalAuth` as a **constructor function**, not a plain object. Code that checks `typeof window.TxGlobalAuth` must accept both:

```javascript
// WRONG — rejects function exports
if (typeof window.TxGlobalAuth !== 'object') { /* error */ }

// CORRECT — accepts both function and object exports
if (typeof window.TxGlobalAuth !== 'function' && typeof window.TxGlobalAuth !== 'object') { /* error */ }

// ALSO CORRECT — check for existence and specific methods
if (window.TxGlobalAuth && typeof window.TxGlobalAuth.init === 'function') { /* ready */ }
```

### Deferred Loading

Use any CDN source URL from above:

```html
<script defer onload="onLoadTxGlobalAuth()" onerror="onErrorTxGlobalAuth()" src="..."></script>
```

Always provide an `onerror` handler for script loading failures.

## Initialization

### Basic Init

Only `env` and `appName` are required. Everything else (palette, lang, theme) is preconfigured per `appName` internally and can be overridden if needed.

```javascript
await TxGlobalAuth.init({
    env: 'prod',
    appName: 'myApp'
});
```

**Ask the developer** which `appName` to use for their product.

### Known Values

**env**: `dev`, `tst`, `uat`, `prod`

**palette** (override if needed): `lime`, `purple`, etc. — see official documentation for full list.

### Advanced: txAuth Provider Configuration

For products requiring custom Kratos provider configuration or multi-provider setups (e.g. cross-platform L2 authentication), consult official documentation or contact the support team. This is an advanced configuration that most integrations do not need.

## Authentication Methods

### Kratos Mode
```javascript
// Open auth dialog — widget automatically shows the best screen
// Resolves with ILoginResponse (contains token, session, provider, etc.)
// Rejects if user closes the dialog
TxGlobalAuth.authenticate();

// Pre-fill user data: email
TxGlobalAuth.authenticate({ initialData: { email: 'user@example.com' } });

// Pre-fill user data: phone
TxGlobalAuth.authenticate({ initialData: { phone: '+13125550690' } });

// Pre-fill user data: phone and email
TxGlobalAuth.authenticate({ initialData: { phone: '+13125550690', email: 'user@example.com' } });


// Inline mount (embed in page instead of modal)
TxGlobalAuth.authenticate({
    mountInline: {
        prepareContainer: () => document.getElementById('auth-root'),
        removeContainer: () => { /* cleanup when widget unmounts */ }
    }
});
// prepareContainer is called only if UI needs to be shown
// removeContainer is called only if prepareContainer was called, before the promise resolves
```

**Safe to call when already authenticated** — if the user is already logged in, `authenticate()` resolves immediately without showing any UI.

### Active Directory Mode
```javascript
// Resolves with ILoginResponse (same structure as authenticate())
const result = await TxGlobalAuth.authorize();
// result.session.person — user data (login, name, email, etc.)
// result.token — JWT string
```

Only available for `appName` configurations that use AD. Will not work in Kratos-mode products and vice versa.

**Idempotent**: If the user already has an active AD session, `authorize()` resolves immediately without showing any UI — same behavior as `authenticate()` in Kratos mode. This is useful for restoring state on page reload.

**After `authorize()`**, the full API works the same as after `authenticate()`: `subscribeJWT` fires with the token, `isAuthenticatedUserAccount()` returns `true`, `getTokenProvider()` returns the token and session data. The only differences are:
- `getTokenProvider().getKratosId()` returns an empty string (AD users don't have a Kratos ID)
- `getIdentifiers()` does not return data for AD sessions (currently Kratos-only)
- `session.person.login` contains the AD sAMAccountName (e.g. `"dvorobiev"`)
- `session.person.kratosId` is absent

See **AD Provider Session Data** below for the full `session.person` structure.

### Anonymous (Device) Mode
```javascript
TxGlobalAuth.authorizeAnonymously();
```

Only available for `appName` configurations that include a Device provider. Compatible with products with AD Mode and Kratos Mode.

### Logout
```javascript
await TxGlobalAuth.logout();                        // Full logout (shows confirmation dialog)
// Resolves when user confirms, rejects if user cancels or error

await TxGlobalAuth.loseClientAuthorization();       // Drop from L2 to L1 only (no UI)
```

### State Checks
```javascript
TxGlobalAuth.isAuthenticatedAnonymous();      // Anonymous device token
TxGlobalAuth.isAuthenticatedUserAccount();    // L1 (Kratos or AD)
TxGlobalAuth.isAuthenticatedUserClient();     // L2 (after 2FA and only for clients)
```

**Legacy**: `TxGlobalAuth.isLoggedIn*` — if encountered in existing code, replace with `isAuthenticated*`.

## Progressive Verification (require* methods)

Chain these to ensure the user meets requirements before proceeding. Each method is **idempotent** — if the requirement is already satisfied, it resolves immediately without showing UI.

```javascript
await TxGlobalAuth.authenticate();
await TxGlobalAuth.requireAgreements();
// Rejects with error.payload?.unsignedAgreements (array of unsigned agreement keys)

await TxGlobalAuth.requirePassword();

await TxGlobalAuth.requirePhone({ initialData: { phone: '+4...' } });
await TxGlobalAuth.requireEmail({ initialData: { email: '...' } });
// Both resolve with { hasVerifiedEmail, hasVerifiedPhone }
// Reject with error.payload?.verificationInfo on failure
//
// How these work:
// 1. Check Kratos identity traits — does the user already have a verified email/phone?
// 2. If YES → resolve immediately, no UI shown
// 3. If NO → show input form → validate → save to Kratos identity → resolve
//
// After resolve, the Kratos identity is updated — getIdentifiers() will now return
// the verified email/phone (masked). This is an identity mutation, not just a prompt.
```

### Profile Data with Validation
```javascript
await TxGlobalAuth.requireUserProfileData({
    requirements: { firstName: true, lastName: false }, // true = required, false = optional, undefined = not shown
    initialData: { firstName: 'John' },
    validator: (key, value) => {
        // Throw to mark field invalid — thrown string is shown as error in UI
        if (key === 'firstName' && value.length < 2) {
            throw 'Name too short';
        }
    }
});
// Resolves with profile data object, e.g. { firstName: 'John', lastName: 'Doe' }
// Rejects if user closes dialog without completing required fields
```

Available profile fields: `avatar`, `firstName`, `lastName`, `gender` (`'male'` | `'female'`), `phone`.

**Note**: The widget does not persist profile data to its own database. The product is responsible for storing the returned data.

### Options

**`disableSkip`**: Pass `disableSkip: true` to any `require*()` or `authenticate()` method to prevent the user from skipping the step.

**Optional fields in requirePhone/requireEmail**: By default these are required. Pass `requirements: { phone: false }` to make the field optional (shown but not enforced).

### Promise Rejection and Close Button

All `authenticate()` and `require*()` methods return a Promise that **rejects when the user closes the dialog** (clicks the X button). Handle rejections appropriately:

```javascript
try {
    await TxGlobalAuth.authenticate();
    await TxGlobalAuth.requireAgreements();
} catch (error) {
    // User closed the dialog — this is normal, not an error
    console.log('User dismissed the dialog');
}
```

The close button (X) behavior:
- `authenticate()` — close button is shown by default
- `require*()` methods — close button is hidden by default (can be enabled)

### Deprecated

`requireUserIdentifiers()` — use `requirePhone()` and `requireEmail()` separately instead.

### Permission Denied Flow

In a multi-product ecosystem, being authenticated does not mean the user is authorized for a specific product. Use `requireUnauthorizedChoice` to show a unified "access denied" screen with product-specific messaging. The dialog explains why access is denied, optionally offers a way to resolve the issue, and allows switching accounts (which silently logs out the current session — the dialog itself serves as confirmation of intent).

```javascript
const result = await TxGlobalAuth.requireUnauthorizedChoice({
    title: 'Access denied',
    description: 'Your account does not have access to this service',
    buttonTitle: 'Create account'    // optional — shows custom action button
});

// result.ok — true if user interacted, false if dismissed
// result.button — 'default' (switch account) or 'custom' (clicked buttonTitle)
```

The dialog shows a default "switch account" button (which silently logs out) and optionally a custom action button. Rejects if the user is not authenticated.

### Advanced: Platform-Specific L2 Authorization

Methods like `requireMmaAuthorization()` and `requireLimeAuthorization()` enable L1→L2 transitions within specific products. This is an advanced feature available only for select `appName` configurations. Consult official documentation or contact the support team.

## JWT Token Management

### Subscribe to Token Changes

The callback fires **immediately** on subscription with the current state (token or `null` if not authenticated), and then on every subsequent token change (login, logout, refresh, changes from other tabs).

```javascript
const unsubscribe = TxGlobalAuth.subscribeJWT((response) => {
    if (response) {
        // Authenticated — update app state with user data
        // response.token     - JWT string
        // response.session   - Session object
        // response.session.person   - Person data (name, email, login, etc.)
        // response.session.accounts - Account array
        // response.provider  - Provider ID
    } else {
        // Not authenticated (guest, logged out, or session expired)
        // Clear user state in your app here
    }
});

// Later (only if needed for some reason): unsubscribe();
```

**This callback is your single source of truth** — handle both branches. Do not manage auth state elsewhere (see Key Integration Concepts above).

**React**: Use `subscribeJWT` as the sole auth state driver. Clean up on unmount:
```jsx
useEffect(() => {
    const unsub = TxGlobalAuth.subscribeJWT((response) => {
        if (response) {
            setUser(response.session.person);
            setToken(response.token);
        } else {
            setUser(null);
            setToken(null);
        }
    });
    return () => unsub();
}, []);
```

### subscribeJWT Handler — Required Cases

The callback fires in several distinct scenarios. Your handler must distinguish between them:

| Callback receives | Current app state | What happened | Required action |
|---|---|---|---|
| `response` (token) | Not authenticated | **First login** | Exchange token with backend, set user session |
| `response` (token) | Authenticated, same `person.id` | **Token refresh** | Update stored token if needed, no session change |
| `response` (token) | Authenticated, different `person.id` | **Account switch** | Clear current session, exchange new token with backend |
| `null` | Authenticated | **Session lost** (logout, expiry, revocation) | Clear local session, redirect to login |
| `null` | Not authenticated | **Already logged out** | No action needed |
| `response` (token, immediate on subscription) | Authenticated | **Page reload / re-mount** | Sync identity tracker with token, no backend exchange needed |

**Identity tracking**: To distinguish "token refresh" from "account switch", compare `person.id` from the new token against the `person.id` you stored from the previous callback. Use `person.id` (always present, immutable within a session) — not `person.login` or `person.email`, which may be absent for some providers.

> **Important nuance**: `person.id` is stable for detecting account switches within a session, but its **meaning changes across auth levels** — at L1 it equals `kratosId`, at L2 it becomes a client account GUID. If you persist `person.id` to your backend for long-term user tracking, use `person.kratosId` instead (stable across L1→L2 transitions). See "Data Availability by Provider and Auth Level" above and the `global-auth:backend` skill for details.

**React implementation sketch** (extends the basic example above):

```javascript
const personIdRef = useRef(null);

useEffect(() => {
    const unsub = TxGlobalAuth.subscribeJWT((response) => {
        if (response) {
            const newPersonId = response.session.person.id;
            if (!personIdRef.current) {
                // First login or page reload — exchange token with backend
                personIdRef.current = newPersonId;
                exchangeToken(response.token);
            } else if (personIdRef.current !== newPersonId) {
                // Account switch — clear old session, exchange new token
                personIdRef.current = newPersonId;
                clearSession();
                exchangeToken(response.token);
            }
            // else: token refresh — same person, update token only if stored
        } else {
            // Session lost — clear everything
            personIdRef.current = null;
            clearSession();
        }
    });
    return () => unsub();
}, []);
```

### Token Provider
```javascript
const tp = TxGlobalAuth.getTokenProvider();
tp.getResponse().token;   // JWT string
tp.getKratosId();         // Kratos user ID (empty string for AD providers)
tp.getPersonId();         // Person ID (works for all providers)
tp.getResponse().session; // Full session (works for all providers)
```

**AD note**: For AD-authenticated users, use `tp.getPersonId()` or `tp.getResponse().session.person` to identify the user. `tp.getKratosId()` is only meaningful for Kratos providers.

### Force Renewal

Call before long-running server operations where the JWT may expire during processing. If the full request chain (including backend microservice calls with the client JWT) is expected to complete within 20–30 seconds, force renewal is unnecessary.

```javascript
await TxGlobalAuth.forceRenew();
// Now safe to start a long-running operation with a fresh token
const token = TxGlobalAuth.getTokenProvider().getResponse().token;
```

### Token Scope Management

`TxGlobalAuth.updateTokenScope({ scope, variant? })` updates the JWT scope at runtime and requests a new token with the target scope. This is an advanced feature — consult official documentation for available scope values and variant options before implementing.

### Additional Token Utilities
```javascript
TxGlobalAuth.getJwt();                    // Parsed JWT object, or null
const { userId } = await TxGlobalAuth.getGlobalVisitor(); // Kratos userId, or null if not authenticated
const { token } = await TxGlobalAuth.getReadOnlyToken();  // Long-lived readonly JWT (rejects if not authorized)
```

## AD Provider Session Data

For AD-type providers (`AD_OFFICE`, `AD_OFFICE_OTP`, `AD_OFFICE_SMS`, `AD_WTE`, etc.), the JWT response contains user identity in `session.person`. The exact set of fields may vary across AD configurations, but the typical structure is:

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` | User GUID (likely LDAP/AD objectGUID — not a Kratos ID) |
| `name` | `string` | First name |
| `lastname` | `string` | Last name |
| `middlename` | `string` | Patronymic (may be absent in some AD configurations) |
| `email` | `string` | Corporate email |
| `login` | `string` | AD sAMAccountName |
| `verifiedPhone` | `string` | Phone number (may be empty) |
| `gender` | `number` | Gender |

`session.userType` is `"EMPLOYEE_USER_TYPE"` for AD-authenticated users.

**Identity tracking field**: Use `person.id` to track which user is currently authenticated **within a single session** (e.g., to detect account switches in `subscribeJWT`). Do not use `person.login` or `person.email` — these may be absent for some provider types (e.g., Kratos L1 has no login or email).

> **Warning**: `person.id` is **not stable across authentication levels**. For the same Kratos user, `person.id` equals `kratosId` at L1, but changes to a client/trading account GUID at L2. If your app stores `person.id` and the user later upgrades to L2, the stored ID will no longer match. **Use `person.kratosId` as the stable primary key for Kratos users.** For AD users, `person.id` (AD GUID) is stable — but `kratosId` is absent. See the `global-auth:backend` skill — section "person.id Semantics" and "Data Availability by Provider and Auth Level".

**AD provider variants**:
- `AD_OFFICE` — standard AD authentication (username + password)
- `AD_OFFICE_OTP` — AD with OTP as second factor
- `AD_OFFICE_SMS` — AD with SMS as second factor
- `AD_WTE` — separate domain controller (no 2FA at this time)

**What is NOT available from any TxGlobalAuth provider**: job title, department, office location, manager, expertise/skills. These require direct LDAP/AD queries on the backend.

**Backend token validation**: The JWT payload contains `session.person` in a compressed (`zipped: true`) form. The backend must validate and decode the token — see the `global-auth:backend` skill for approaches.

## Event Subscriptions

### Initialization
```javascript
const unsubscribe = TxGlobalAuth.subscribeInitialized((widget) => {
    // widget — the TxGlobalAuth instance
    // Fires when widget completes initialization
    // If called after init() already resolved, fires synchronously
});
```

### Error Monitoring

Important for production — detect silent failures that may affect user sessions:

```javascript
// Login errors
TxGlobalAuth.subscribeOnLoginError((error) => {
    console.error('Login failed:', error);
    // Show user-facing error or send to monitoring
});

// Token renewal errors — user may lose session silently
TxGlobalAuth.subscribeOnTokenRenewError((error) => {
    console.error('Token renewal failed:', error);
    // Critical: user session may expire without warning
});
```

### Global Event Bus

`subscribeGlobal` fires on **widget UI events and authentication state changes**. These events are **not directly tied to JWT changes** — for reactive token/session state, use `subscribeJWT` (see Key Integration Concepts). Use `subscribeGlobal` for UI lifecycle hooks, analytics, and reacting to specific user actions in the widget.

```javascript
const unsubscribe = TxGlobalAuth.subscribeGlobal(eventName, callback, callOnce?);
// callOnce: optional boolean — if true, callback fires only once, then auto-unsubscribes
```

**Authentication events** (not tied to JWT):

| Event | Callback | Description |
|-------|----------|-------------|
| `authorization` | `() => void` | User successfully logged in via widget UI. Not fired on token refresh or background sync. |
| `registration` | `() => void` | User registered via widget UI. |
| `logout` | `() => void` | User lost authentication: via `logout()` call, UI logout flow, or account switch. |
| `authenticationChange` | `(isAuthorized: boolean) => void` | Authentication status changed. Not tied to JWT — tracks the global auth state. |
| `visitorChange` | `({ userId: string \| null }) => void` | Global authenticated user changed. Not tied to JWT. |
| `backgroundVisitorChange` | `({ userId: string \| null }) => void` | Background sync (cookies/localStorage) detected a user change. |

**User action events**:

| Event | Callback | Description |
|-------|----------|-------------|
| `agreementAccepted` | `(agreementKey: string) => void` | User accepted an agreement via widget UI. |
| `passwordChange` | `() => void` | User changed password via widget UI. |
| `unverifiedIdentifierEntered` | `(payload) => void` | User submitted a form with identifiers (login, registration, add identifier). Fires per field. Value may be masked. |
| `finish` | `() => void` | Widget process completed. Not the same as unmount — use only if you understand the distinction. |

**UI lifecycle events**:

| Event | Callback | Description |
|-------|----------|-------------|
| `beforeMount` | `() => void` | Fires synchronously after `mountInline.prepareContainer()`, before widget mounts. |
| `afterMount` | `() => void` | Fires synchronously after widget mounts. |
| `beforeUnmount` | `() => void` | Fires synchronously before widget unmounts. |
| `afterUnmount` | `() => void` | Fires synchronously after unmount and `mountInline.removeContainer()`. |

```javascript
// Example: analytics tracking
TxGlobalAuth.subscribeGlobal('authorization', () => {
    analytics.track('user_logged_in');
});

TxGlobalAuth.subscribeGlobal('agreementAccepted', (key) => {
    analytics.track('agreement_accepted', { key });
});

// Example: inline mount lifecycle
TxGlobalAuth.subscribeGlobal('afterMount', () => {
    console.log('Widget UI is now visible');
});
```

> **Do not use `subscribeGlobal` for auth state management.** The `logout` event fires when the widget processes a logout — but it does not cover all cases where the user loses their session (e.g. token expiry, server-side revocation). For reliable auth state, use `subscribeJWT` which fires with `null` on any session loss. See **Key Integration Concepts** above.

## User Profile

```javascript
// Show profile edit dialog (always shown, even if data already exists)
const data = await TxGlobalAuth.showUserProfileData({
    requirements: { firstName: true, lastName: false },
    initialData: { firstName: 'John' },
    disableSkip: true
});
// data: { firstName: '...', lastName: '...' }

// Set profile data programmatically
TxGlobalAuth.setUserProfileData({ firstName: 'John', lastName: 'Doe' });

// Get profile data (returns previously set data, or empty fields if none)
const data = await TxGlobalAuth.getUserProfileData();

// Get verification status
const verInfo = TxGlobalAuth.getVerificationInfo();

// Get user identifiers — Kratos-only, returns MASKED data
const { emails, phones, logins } = await TxGlobalAuth.getIdentifiers();
// emails example: ["j***@example.com"] — masked, NOT the full email address
```

> **Warning — `getIdentifiers()` returns MASKED identifiers**: Values are partially hidden (e.g. `j***@example.com`, `+1***0690`). These are **for display purposes only** — do NOT use them for registration, identity verification, backend lookups, or any business logic that requires the actual value.
>
> **Anti-pattern**: `getIdentifiers().emails[0]` to obtain user email for backend registration — this returns a masked string, not the real email.
>
> **`getIdentifiers()` currently works only for Kratos providers** — for AD sessions, retrieve user data from `subscribeJWT` callback's `response.session.person` or from `getTokenProvider().getResponse().session.person` instead.
>
> `getUserProfileData`, `getVerificationInfo`, and `getIdentifiers` are under active refactoring consideration. Their behavior may change.

## UI Management

Additional dialog methods for products with settings or account management pages:

```javascript
await TxGlobalAuth.showPasswordChange();   // Open password change dialog
await TxGlobalAuth.showSessions();         // Show active sessions management
await TxGlobalAuth.showSettings();         // Show user settings (emails, phones, social accounts, password)
await TxGlobalAuth.showAgreements();       // Show agreements (informational — does not enforce acceptance)
// All resolve when the dialog is closed. All accept mountInline option.
```

## Client Creation (Demo Accounts)

```javascript
await TxGlobalAuth.createClient({
    type: 'etna',    // or 'edox'
    source: 0,       // platform source
    initialData: {
        firstName: '...', lastName: '...',
        phone: '...', email: '...'
    }
});
```

## App Navigation with OTT (One-Time Token)

Use `openApp` to navigate between products while preserving the current authentication level.

```javascript
TxGlobalAuth.openApp({
    url: 'https://app.example.com/path',
    target: '_self'   // or '_blank'
});
```

**Why use openApp instead of a regular link?**
- Cross-domain cookie sharing only transfers L1. If the user is at L2, a regular link to another domain will require re-authentication (2FA).
- `openApp` generates a one-time token that transparently transfers the current authentication level (including L2) to the target product.
- Use for cross-domain navigation. For same-domain navigation, regular links are fine.

## Dynamic UI Control

These methods override the `appName` defaults:

```javascript
TxGlobalAuth.setLang('en');
TxGlobalAuth.setTheme('dark');
TxGlobalAuth.setPalette('lime');
TxGlobalAuth.dispose();  // Destroy widget instance
```

## Debug Tools

```javascript
TxGlobalAuth.debug.enableLogging();
TxGlobalAuth.debug.screenNavigationEnable = true;
```

## TypeScript Declarations

Reference types for the TxGlobalAuth public API. Use these when declaring `window.TxGlobalAuth` or typing callbacks. This is a subset of the most commonly used methods — consult the widget source for the full API.

```typescript
// --- Init options ---
type Env = 'dev' | 'tst' | 'uat' | 'prod';
type Lang = 'ru' | 'en';
type Theme = 'light' | 'dark';
type Palette = 'finam' | 'j2t' | 'lime' | 'limex' | 'purple' | 'takeProfit' | 'j2tGlobal';
type Variant = 'finam' | 'spc' | 'mma';

interface TxGlobalAuthOptions {
    env: Env;
    appName: string;
    appVersion?: string;
    lang?: Lang;
    palette?: Palette;
    theme?: Theme;
    parseUrl?: boolean;
    preserveQueryParams?: boolean;
    logoutWithoutConfirmation?: boolean;
    disableInitAuthorization?: boolean;
    // For advanced txAuth/globalAuth provider config, see official docs
}

// --- Response types ---
interface ILoginResponse {
    token: string;
    session: Session;
    sessionContext: SessionContext;
    firebase: string;
    provider: string;
    created: number;
    renewExp: number;
    // Additional fields: exp, spinExp, spinReq, receiveTime, confirmation (deprecated)
}

interface Session {
    person: PersonData;
    accounts: unknown[];
    permissions: unknown[];
    userType: string;  // e.g. "EMPLOYEE_USER_TYPE"
    companyId: string;
    firms: unknown[];
    segments: unknown[];
}

interface PersonData {
    id: string;           // WARNING: semantics change by provider! L1=kratosId, L2=clientGUID, AD=AD GUID
                          // Use for session-level identity tracking only. See "Data Availability" table.
                          // Anti-pattern: using person.id as a stable primary key — it CHANGES when user transitions L1→L2
    name?: string;        // First name — ABSENT for Kratos L1
    lastname?: string;    // Last name — ABSENT for Kratos L1
    middlename?: string;  // Patronymic (AD-specific, may be absent)
    email?: string;       // ABSENT for Kratos L1. Do NOT use for identity comparison.
    login?: string;       // AD sAMAccountName or Kratos login — ABSENT for L1. Do NOT use for identity comparison.
    verifiedPhone?: string; // Phone — ABSENT for L1
    gender?: number;
    kratosId?: string;    // STABLE identity for Kratos users (same across L1→L2). Absent for AD.
                          // For Kratos: use this as primary key, not person.id
                          // For AD: use person.id (AD GUID) — kratosId is absent
}

interface TokenProvider {
    getFreshToken(): Promise<string>;
    getKratosId(): string | null;
    getPersonId(): string | null;
    getResponse(): ILoginResponse;
    subscribeTokenChanges(callback: (token: string | null) => void): () => void;
}

interface RequireUserIdentifiersResult {
    hasVerifiedEmail: boolean;
    hasVerifiedPhone: boolean;
}

type UserProfileData = {
    avatar?: string;
    firstName?: string;
    lastName?: string;
    gender?: 'male' | 'female' | null;
    phone?: string;
};

type RequireUnauthorizedChoiceResult = {
    ok: boolean;
    button: 'default' | 'custom';
};

// --- Inline mount options ---
interface HighLevelInlineMountOptions {
    prepareContainer: () => HTMLElement | Promise<HTMLElement>;
    removeContainer?: () => void | Promise<void>;
}

// --- Global event types ---
type SubscribeGlobalEvent =
    | 'authorization' | 'registration' | 'logout' | 'finish'
    | 'authenticationChange' | 'agreementAccepted' | 'passwordChange'
    | 'unverifiedIdentifierEntered'
    | 'visitorChange' | 'backgroundVisitorChange'
    | 'beforeMount' | 'afterMount' | 'beforeUnmount' | 'afterUnmount';

// --- Error types ---
interface ServiceError {
    code?: number;
    message?: string;
    details?: string;
}

// --- Window declaration ---
interface Window {
    TxGlobalAuth?: {
        // Lifecycle
        init(options: TxGlobalAuthOptions): Promise<typeof TxGlobalAuth>;
        dispose(): Promise<void>;
        subscribeInitialized(callback: (widget: typeof TxGlobalAuth) => void): () => void;

        // Authentication
        authenticate(options?: { initialData?: { email?: string; phone?: string }; mountInline?: HighLevelInlineMountOptions; disableSkip?: boolean }): Promise<ILoginResponse>;
        authorize(options?: { mountInline?: HighLevelInlineMountOptions }): Promise<ILoginResponse>;
        authorizeAnonymously(options?: Record<string, unknown>): Promise<ILoginResponse>;
        logout(options?: Record<string, unknown>): Promise<void>;
        loseClientAuthorization(): Promise<void>;

        // State checks (all synchronous)
        isAuthenticatedAnonymous(): boolean;
        isAuthenticatedUserAccount(): boolean;
        isAuthenticatedUserClient(): boolean;

        // JWT & tokens
        subscribeJWT(callback: (response: ILoginResponse | null, variant?: Variant) => void): () => void;
        getJwt(): ILoginResponse | null;
        getTokenProvider(variant?: Variant): TokenProvider;
        forceRenew(): Promise<ILoginResponse>;
        getReadOnlyToken(): Promise<{ token: string }>;
        getGlobalVisitor(): Promise<{ userId: string | null }>;
        updateTokenScope(payload: { scope: Record<string, unknown>; variant?: Variant }): Promise<void>;

        // Progressive verification
        requireAgreements(options?: { mountInline?: HighLevelInlineMountOptions; disableSkip?: boolean }): Promise<void>;
        requirePassword(options?: { mountInline?: HighLevelInlineMountOptions; disableSkip?: boolean }): Promise<void>;
        requirePhone(options?: { initialData?: { phone?: string }; mountInline?: HighLevelInlineMountOptions; disableSkip?: boolean }): Promise<RequireUserIdentifiersResult>;
        requireEmail(options?: { initialData?: { email?: string }; mountInline?: HighLevelInlineMountOptions; disableSkip?: boolean }): Promise<RequireUserIdentifiersResult>;
        requireUserProfileData(options?: { requirements?: Record<string, boolean>; initialData?: Partial<UserProfileData>; validator?: (key: string, value: string) => void; mountInline?: HighLevelInlineMountOptions; disableSkip?: boolean }): Promise<UserProfileData>;
        requireUnauthorizedChoice(options: { title: string; description: string; buttonTitle?: string }): Promise<RequireUnauthorizedChoiceResult>;

        // Error subscriptions
        subscribeOnLoginError(callback: (error: ServiceError) => void): () => void;
        subscribeOnTokenRenewError(callback: (error: ServiceError) => void): () => void;

        // Global events
        subscribeGlobal(eventName: SubscribeGlobalEvent, callback: (...args: unknown[]) => void, once?: boolean): () => void;

        // UI dialogs
        showUserProfileData(options?: { requirements?: Record<string, boolean>; initialData?: Partial<UserProfileData>; disableSkip?: boolean; mountInline?: HighLevelInlineMountOptions }): Promise<UserProfileData>;
        showPasswordChange(options?: { mountInline?: HighLevelInlineMountOptions }): Promise<void>;
        showSessions(options?: { mountInline?: HighLevelInlineMountOptions }): Promise<void>;
        showSettings(options?: { mountInline?: HighLevelInlineMountOptions }): Promise<void>;
        showAgreements(options?: { mountInline?: HighLevelInlineMountOptions }): Promise<void>;

        // User data
        getUserProfileData(): Promise<UserProfileData>;
        setUserProfileData(data: Partial<UserProfileData>): void;
        getVerificationInfo(): Promise<RequireUserIdentifiersResult>;
        getIdentifiers(): Promise<{ emails: string[]; phones: string[]; logins: string[] }>;

        // Navigation & cross-product
        openApp(payload: { url: string; target?: '_self' | '_blank' }): Promise<Window | undefined>;

        // UI customization
        setLang(lang: Lang): void;
        setTheme(theme: Theme): void;
        setPalette(palette: Palette): void;

        // Client creation
        createClient(options?: { type?: string; source?: number; initialData?: Record<string, string>; mountInline?: HighLevelInlineMountOptions }): Promise<ILoginResponse>;

        // Debug
        debug: {
            enableLogging(options?: Record<string, unknown>): void;
            disableLogging(options?: Record<string, unknown>): void;
            screenNavigationEnable: boolean;
        };
    };
}
```

**Note**: The `isAuthenticated*` methods and `getJwt()` are **synchronous** — do not `await` them. The `authenticate()`, `authorize()`, `logout()`, and all `require*()` / `show*()` methods return Promises.

## Vanilla JS Quick Start

Complete working example showing initialization, token subscription, and auth level detection. This pattern works for **both Kratos and AD modes** — the `subscribeJWT` callback receives the same `ILoginResponse` structure regardless of provider:

```javascript
function onloadTxGlobalAuth() {
    TxGlobalAuth.init({
        env: 'prod',
        appName: 'myApp'  // Ask developer for the correct value
    }).then(() => {
        // Subscribe to authentication state changes
        // Works for ALL modes: Kratos, AD, Anonymous
        TxGlobalAuth.subscribeJWT((response) => {
            const isAnonymous = TxGlobalAuth.isAuthenticatedAnonymous();
            const isL1 = TxGlobalAuth.isAuthenticatedUserAccount();
            const isL2 = TxGlobalAuth.isAuthenticatedUserClient();

            if (isL2) {
                console.log('Authenticated: L2 (client level)');
            } else if (isL1) {
                console.log('Authenticated: L1 (account level)');
                // Universal: works for both Kratos and AD
                const person = response.session.person;
                console.log('User:', person.name, person.lastname);
                console.log('Person ID:', TxGlobalAuth.getTokenProvider()?.getPersonId());
                // Kratos-specific (empty for AD):
                // const kratosId = TxGlobalAuth.getTokenProvider()?.getKratosId();
            } else if (isAnonymous) {
                console.log('Anonymous device session');
            } else {
                console.log('Guest — not authenticated');
            }

            document.title = isL1 ? 'Welcome back' : 'Please log in';
        });

        // Wire up login button
        // For AD mode: replace authenticate() with authorize()
        document.getElementById('login-btn')?.addEventListener('click', async () => {
            try {
                await TxGlobalAuth.authenticate();  // or authorize() for AD
                await TxGlobalAuth.requireAgreements();
            } catch (e) {
                console.log('User dismissed the dialog');
            }
        });
    }).catch((error) => {
        console.error('TxGlobalAuth init failed:', error);
    });
}
```

**Troubleshooting**: If `subscribeJWT` fires with `null` and `isAuthenticatedUserAccount()` returns `false` — the user is in **guest state** (not authenticated). This is not mode-specific. Verify: (1) `init()` completed successfully, (2) the `appName` is correct for the target environment, (3) the user has actually completed the authentication flow (`authenticate()` or `authorize()` resolved).

## Error Handling

Handle init errors separately from auth flow errors:

```javascript
// Init errors — widget failed to load or configure
try {
    await TxGlobalAuth.init({ env: 'prod', appName: 'myApp' });
} catch (error) {
    console.error('Widget initialization failed:', error);
    // Show fallback UI or error message
    return;
}

// Auth flow errors — user dismissed dialog (normal behavior)
try {
    await TxGlobalAuth.authenticate();
    await TxGlobalAuth.requireAgreements();
} catch (error) {
    // User closed the dialog — handle gracefully
    console.log('Auth flow interrupted by user');
}
```

## Diagnosing Existing Integrations

When reviewing or refactoring existing TxGlobalAuth code, the integration is often broken in non-obvious ways. The rest of this skill describes how to build correct integrations — this section helps you recognize and fix common problems in code that already exists.

**Always start with the Pre-Integration Health Check** (top of this skill) before analyzing code logic.

### Symptom: Method Names Don't Match This Skill

**Diagnosis**: The project is loading the **internal build** (`global-auth.v8.js`) instead of the official wrapper (`auth-widget/@8.js`).

**What you'll see in code**:
- `subscribeToJwt` instead of `subscribeJWT`
- `getPublicSession` instead of `getJwt`
- `getAuthenticationState` instead of `isAuthenticatedUserAccount`
- Fallback chains like `obj.getJwt || obj.getJWT || obj.getToken`

**Why this happens**: Developers found the wrong CDN URL (internal URL that looks official) and wrote code against whatever API they could discover through trial and error.

**Fix**: Change the CDN URL to the official wrapper (path must be `/auth-widget/@8.js` — ask the developer for the correct CDN host), then update all method calls to match the official API documented in this skill.

**Do NOT**: Remove fallback chains without fixing the CDN URL first — the chain may be compensating for the wrong build. Do NOT treat unfamiliar method names as "typos" or "fictional" — they are methods from the internal build. But do NOT keep using them either — they are private API without backward compatibility guarantees and may change in any release.

### Symptom: Auth State Updates Through Multiple Paths

**Diagnosis**: The integration has **competing state management** — some auth state flows through `subscribeJWT`, some through direct event handlers, and some through custom endpoints.

**What you'll see in code**:
- `subscribeJWT` callback updates one store, but a separate `initializeAuthFx.doneData` or similar effect updates another
- `subscribeGlobal('logout', ...)` clears state, duplicating what `subscribeJWT(null)` should do
- Direct calls to `/sessions/token` or other Kratos endpoints to resolve user data

**Why this happens**: The integration was built incrementally — developers added data sources as they discovered the original approach didn't provide what they needed (e.g., email not in token, wrong build loaded).

**Fix**: Consolidate to `subscribeJWT` as the single source of truth. Remove duplicate state paths **only after verifying that `subscribeJWT` actually fires correctly** (which requires the correct CDN build — check Step 0 first). If the duplicate path exists because `subscribeJWT` doesn't provide needed data (e.g., email), address the root cause (wrong build, or use `requireEmail()` to ensure the identity has an email, or backend S2S `get-identity`).

**Do NOT**: Remove a "redundant" state update path without understanding why it was added. It may be the only working path if the primary one (subscribeJWT) is broken due to a CDN issue.

### Symptom: Direct Calls to Kratos Endpoints

**Diagnosis**: The frontend bypasses TxGlobalAuth to call Kratos endpoints directly — typically to get data that isn't available through the widget API for their provider type.

**What you'll see in code**:
- `fetch('https://ga.*/sessions/token', ...)` — resolving session data directly
- `fetch('https://ga.*/sessions/whoami', ...)` — checking session validity
- Custom "introspect" calls from the frontend

**Why this happens**: The developer needed data (usually email) that isn't in the JWT for their provider type (Kratos L1 has no email — see Data Availability table). Instead of using `requireEmail()` or a backend S2S flow, they called Kratos directly.

**Fix**: See the `global-auth:backend` skill anti-pattern "Do NOT Call /sessions/token from the Frontend". Route data needs through the backend (S2S introspect → `get-identity`) or use `requireEmail()` / `requirePhone()` to ensure the identity has the needed data.

### Symptom: person.id Used as Database Primary Key

**Diagnosis**: The app stores `person.id` as the user identifier in its database and correlates sessions by matching it.

**What will break**: When the user transitions from L1 to L2, `person.id` changes from `kratosId` to a client account GUID. The app creates a duplicate user record or fails to find the existing one.

**Fix**: Use `person.kratosId` as the stable key for Kratos users. For AD users (where `kratosId` is absent), `person.id` is stable and safe to use. See "person.id Semantics" in the `global-auth:backend` skill.

### Symptom: Email Extraction Functions That Always Return Null

**Diagnosis**: The code has functions like `extractEmailFromJwt()` or `getEmailFromSession()` that parse the token for email — but the product uses Kratos L1 where email is **never present** in the token.

**What you'll see**: The function exists, has proper implementation, but always returns `null` at runtime. It may have been written in anticipation of L2 support or copied from another project using a different provider.

**Fix**: If the product will always be Kratos L1 — remove the dead code. If email is needed, use `requireEmail()` (checks Kratos identity, prompts only if missing, saves to Kratos) or obtain it via backend S2S `get-identity` flow. Do NOT leave the function "just in case L2 might be added someday" — YAGNI applies.

### Review Priority Order

When reviewing an existing TxGlobalAuth integration, work in this order:

1. **CDN URL** — verify the correct build is loaded (Step 0). If wrong, fix this first; everything else may be a symptom.
2. **API method names** — check if they match the official API. If not, see "Internal Build API vs Official Wrapper API" above.
3. **State management** — verify `subscribeJWT` is the single source of truth. Look for competing state paths.
4. **Identity handling** — check which `person` fields are used and whether they're available for this provider type.
5. **Direct Kratos calls** — search for `fetch` calls to `ga.*` domains from the frontend.
6. **Logout flow** — verify logout goes through `TxGlobalAuth.logout()`, not direct cookie/store clearing.

**Only after verifying 1-2 should you proceed to logic-level code review.** Fixing logic bugs in code that uses the wrong API is wasted effort.
