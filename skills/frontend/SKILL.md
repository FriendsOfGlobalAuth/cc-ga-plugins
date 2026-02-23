---
name: frontend
description: "TxGlobalAuth (GlobalAuth, GA) JavaScript widget integration. Use when working with TxGlobalAuth, auth-widget, Kratos authentication, JWT token management, or embedding authentication UI in frontend projects."
---

# Global Auth - Frontend Integration Skill

This skill provides expert knowledge of the TxGlobalAuth authentication widget for frontend integration.

**Important**: All TxGlobalAuth methods are available only after successful `init()`. Always structure code so that all API calls happen inside the `.then()` callback of `init()` or after `await init()`.

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

### Key Integration Concepts

**1. `subscribeJWT` is the single source of truth**: All auth state management must go through the `subscribeJWT` callback — it is the **only** place where you read and update the current user, token, and session. The callback fires immediately on subscription (with the current token or `null`), and then on every change: login, logout, token refresh, session changes from other tabs. **Handle both cases**: when `response` has a token (user is authenticated) and when it is `null` (user logged out or session expired). Button handlers for login/logout should only call `authenticate()`/`authorize()`/`logout()` — they must **not** update app state directly. The state update happens reactively when `subscribeJWT` fires in response.

> **Anti-pattern**: Do not split auth state management between `subscribeJWT` (for login) and `subscribeGlobal('logout')` (for logout). This leads to stale UI when logout happens externally (other tab, console, session expiry). `subscribeJWT` already covers logout — the callback fires with `null`.

**2. Progressive requirement chains**: For actions that require a specific auth level, build a chain of `authenticate()` → `require*()` calls in the action handler. Every method in the chain is idempotent — if the requirement is already met, it resolves instantly. For example, a "Post comment" button handler should call `authenticate()` → `requireAgreements()` → `requireEmail()` → then submit the comment with the JWT token. This way, guests get the full onboarding flow, while authenticated users skip straight to submission.

## Script Loading

### CDN Sources (v8 - current)
```html
<!-- Yandex CDN -->
<script src="https://libs-cdn.finam.ru/auth-widget/@8.js"></script>

<!-- Amazon CDN -->
<script src="https://libs-cdn.lime.co/auth-widget/@8.js"></script>
```

**CDN selection**: Use Yandex CDN (`finam.ru`) for CIS countries-targeted products. Use Amazon CDN (`lime.co`) for international products. When unsure, ask the developer.

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

// Get user identifiers (masked for privacy) — Kratos-only
const { emails, phones, logins } = await TxGlobalAuth.getIdentifiers();
```

> **Note for integrators**: `getUserProfileData`, `getVerificationInfo`, and `getIdentifiers` are under active refactoring consideration. Their behavior may be non-obvious. When using these methods, confirm with the developer that the expected behavior matches their understanding. **`getIdentifiers()` currently works only for Kratos providers** — for AD sessions, retrieve user data from `subscribeJWT` callback's `response.session.person` or from `getTokenProvider().getResponse().session.person` instead.

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
    id: string;           // User GUID — semantics vary by provider (see backend skill)
    name?: string;        // First name (may be absent for Kratos L1)
    lastname?: string;    // Last name (may be absent for Kratos L1)
    middlename?: string;  // Patronymic (AD-specific, may be absent)
    email?: string;       // Email
    login?: string;       // AD sAMAccountName or Kratos login (may be absent for Kratos L1)
    verifiedPhone?: string; // Phone (may be absent)
    gender?: number;
    kratosId?: string;    // Present for Kratos providers, absent for AD
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
