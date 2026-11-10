---
name: marketing
description: "TxGlobalAuth (GlobalAuth, GA) marketing integration: CRM leads, UTM/marketing context, guest-identified chat. Use when sending a lead or response to CRM with TxGlobalAuth.marketing.sendResponse (sourceId, new_website, WebformData), reading UTM/INT/referral marks with getContextBuilder, collecting phone/name/email with showContactsForm, or embedding a support chat that identifies the guest."
---

# Global Auth - Marketing Integration Skill

Three widget features that products use around marketing: sending leads to CRM, reading the visitor's marketing context, and the contacts form, which together give a support chat that identifies the guest. Login, tokens and logout are in `global-auth:frontend`; everything here assumes the widget is loaded and `init()` has resolved.

`TxGlobalAuth.marketing` needs the GA module: the getter throws when the widget runs without it (a mode added in 8.21.0).

| Feature | Widget ≥ |
|---|---|
| `marketing.getContextBuilder()`, `marketing.GROUPS`, `marketing.STAGES` | 8.9.1 |
| `marketing.sendResponse()` | 8.15.0 |
| `showContactsForm()` | 8.21.0 |

Older code may call `TxGlobalAuth.debug.unstable.sendMarketingLead()` or `debug.unstable.sendLead()`. Both were removed in 8.15.0; replace them with `marketing.sendResponse()`.

## Sending a Lead to CRM — `marketing.sendResponse`

GA never sends a lead by itself. Widget events such as a registration are only events; the lead is the product's own call, made at the point of the action in its funnel.

```javascript
await TxGlobalAuth.marketing.sendResponse({
    data: {
        sourceId: 777,              // the funnel action; issued by the CRM team
        new_website: 'FinamRu',     // a value of the CRM WebSite enum, not a host
        firstname: 'Ivan',
        emailaddress: 'ivan@example.com',
        telephone: '+79990000000',
        description: 'Opened the support chat',
    },
});
```

The widget sends `POST <contour>/self-service/marketing/crm-response` with the body `{ "payload": data }`; success is `204`. It works for guests and for logged-in users; "logged in" here means a GA session on the contour, not the product's JWT.

### The two required fields

- **`sourceId`** (integer) names the **action in the product's funnel**, not the application. One `appName` sends different SourceIDs for different actions, for example `777` for "opened the support chat" and `333` for "registered on the site" (illustrative numbers). The CRM team issues them. Set the SourceID in the code of that action, not once in the app's configuration.
- **`new_website`**: the key must be present, and its value is one of the CRM web form's `WebSite` values: `FinamRu`, `WhoTrades`, `Demo`, `Comon`, `WTCRM`, `EDOX`, `SHOP`, `FinamBankRu`, `LiveFinamRu`, `LMS`, `FinamInvest`, `Other`, `FinamTrade`, `UI`, `AgentAPI`, `TRADE_API_RU` (or `null`). It is **not a host or a URL**. The CRM's "Website" column shows a display label that looks like a host (`Comon` appears as `www.comon.ru`), which is where the confusion comes from. The page URL goes into `new_source_url`, and the widget fills that one.

The widget's typings mark `new_website` optional, but the server's schema requires it: a lead without it gets `400`, and so does one without `sourceId`.

The product passes the person's data and the details of the request: `firstname`, `lastname`, `emailaddress`, `telephone`, `description`, and any other field of the CRM web form.

### Fields the widget fills

The widget adds these keys itself and **overwrites** whatever the product passed under the same names, even when its own value is empty (the console warns that the keys are ignored). Leave them out:

- `new_kratosid`, `new_device_local_id`;
- `new_source_url` (`location.href`), `new_source_url_parameters` (`location.search`);
- `new_referer` (spelled that way), taken from the `utm_referrer` URL parameter;
- `new_utm_source`, `new_utm_medium`, `new_utm_campaign`, `new_utm_content`, `new_utm_term`;
- `new_int_source`, `new_int_medium`, `new_int_campaign`, `new_int_content`;
- `AgentGUID`, `AgencyBackOfficeId` (only a non-zero integer).

Click identifiers (`gclid`, `yclid` and the rest of the CLICKS group) are **not** sent with the lead. `new_external_ids` is a string or `null`: an array passes through the widget and is rejected by the server with `400`.

### What `204` means

`204` means the lead was **stored and queued**, not delivered. A separate worker delivers it to CRM later, and a delivery failure is never reported back to the caller. For an identified user the worker also fills empty `emailaddress` / `telephone` from the user's **verified** identifiers and replaces `new_utm_*` with the server-side marketing context. A guest's lead is delivered as sent.

### Calling conventions

- **One argument.** `sendResponse(payload)` reads only `payload.data`. A second argument such as `{ headers: … }` is silently ignored: request headers cannot be set through the widget.
- **Rejections are not `TxGlobalAuthError`.** The promise rejects with GA's raw failure object; log it as is.

### Testing without creating CRM records: `X-Router-CRM: none`

With the header `X-Router-CRM: none` (any case), GA validates the lead and stores it with the status `no_delivery`: it is neither enriched nor delivered to CRM. Use it to check an integration on production without polluting CRM. It works on `/self-service/marketing/crm-response` and on the server-to-server `POST /api/marketing/crm-response` (body `{ "kratos_id": "…", "payload": { … } }`, an S2S token with the `crm-response` permission, granted by the GA team on request).

From the browser this mode is **not available**: the widget cannot send headers, and the contour's CORS does not allow `x-router-crm`. Test from your backend with S2S.

## Marketing Context — `marketing.getContextBuilder()`

The widget keeps the visitor's marketing marks from landing URLs. Read them when the product needs them for its own purposes; for a lead, `sendResponse` adds them by itself.

```javascript
const { GROUPS } = TxGlobalAuth.marketing;
const context = await TxGlobalAuth.marketing.getContextBuilder()
    .withGroups(GROUPS.UTM, GROUPS.INT)
    .flat()        // one level of keys; without .flat() the result is grouped
    .fetch();
```

| Group | URL parameters |
|---|---|
| `UTM` | `utm_source`, `utm_medium`, `utm_campaign`, `utm_content`, `utm_term` |
| `INT` | `int_source`, `int_medium`, `int_campaign`, `int_content` |
| `REFERRAL` | `agent`, `AgencyBackOfficeId` (case variants accepted; only non-zero integers kept) |
| `CLICKS` | `gclid`, `fbclid`, `msclkid`, `ttclid`, `li_fat_id`, `rdt_cid`, `yclid`, `wbraid`, `gbraid` |

How the marks are kept:

- In cookies (`utmdata`, `intdata`, `referals`, `clickdata`) on the base domain (the last two host labels), for 30 days, `SameSite=Lax`.
- They are saved when GA initializes, so `init()` must run on landing pages, not only on pages with a login button.
- A group is written only when the URL carries at least one of its parameters. Merging is **per key**, so marks of two campaigns can mix. On read, the current URL's parameters win over the cookies.
- `sendResponse` adds UTM, INT and REFERRAL to the lead; CLICKS are never sent.

The builder:

- `flat()` means `flat(true)`; the default is grouped output.
- **`fetch()` resets the builder.** Build a new one for every read; a second `fetch()` on the same builder throws `No groups specified!`.
- `fetch()` without `withGroups()` throws `No groups specified!`; `withGroups()` with no arguments throws `groups are required`; calling `withGroups()` or `flat()` again replaces the earlier value.
- `STAGES` has `CURRENT`, `ACQUISITION` and `LEAD`, and `.stage()` stores one, but only `CURRENT` is implemented: any other stage logs a warning and returns the current data.
- **TypeScript:** the widget types `GROUPS` as a string union (`'utm' | 'int' | 'referal' | 'clicks'`), not an object. `GROUPS.UTM` works at runtime but fails type-checking; pass the string values or cast.

## Contacts Form — `showContactsForm`

Collects a phone, a name and an email from the visitor. It neither logs in nor verifies anything: the result is what the person typed.

```javascript
const contacts = await TxGlobalAuth.showContactsForm({
    requirements: { phone: true, name: true, email: false },   // key present = shown, true = required
    showAgreements: true,
    title: 'How can we reach you?',
});
// → { phone: '+79990000000', name: 'Ivan', agreementsAccepted: true }
```

| Option | What it does |
|---|---|
| `requirements` | A field is shown when its key has a boolean value; `true` makes it required. Missing or empty `requirements` shows all three fields, none required. |
| `initialData` | Pre-fills the fields. |
| `validator(key, value)` | Throw to mark a field invalid; the thrown value's `toString()` is shown, so `throw new Error('x')` displays `Error: x` — throw a plain string. Runs on submit for **every shown field, empty ones included**, and for the name also on blur. |
| `title`, `subtitle` | Texts of the form. |
| `showAgreements` | Shows the project's agreements. Ticking a checkbox **signs that agreement at once**. |
| `disableSkip`, `mountInline` | As in the other widget methods. |

The result:

- Contains the shown, non-empty fields: `phone` normalized to `+digits`, `email` lower-cased, `name` trimmed. The field order on screen is fixed: phone, name, email.
- `agreementsAccepted: true` appears only when `showAgreements` was set, at least one agreement was a checkbox, and all of them were accepted. Some projects show agreements as text without checkboxes; then the key never appears.
- Built-in format checks: a phone is `+` and at least six digits; an email has the usual `x@y.z` shape.

**Rejection carries no reason.** Through the official wrapper, closing the form, an interruption and unknown option keys all reject with the same `TxGlobalAuthError` ("Failed to require contacts form …"). There is no `error.reason` to branch on: handle every rejection the same way.

## Chat with Guest Identification

A support chat that knows who it talks to, without forcing a full sign-up first:

1. The user clicks "Open chat".
2. Already logged in (`TxGlobalAuth.isAuthenticatedUserAccount()`) → create the chat right away. No form, no lead.
3. Otherwise (a guest, or only an anonymous device token): `showContactsForm()` → `authenticate({ initialData })` with the contacts from the form → send the lead with the chat's SourceID → create the chat.

| Step | Who does it |
|---|---|
| Login state, the form, the login itself | GlobalAuth |
| The lead to CRM, with the product's SourceID | the product |
| Creating the chat | the product (a factory called only after login) |
| Passing a refreshed token to the chat | the product, from `subscribeJWT` |

```javascript
async function openChat() {
    if (TxGlobalAuth.isAuthenticatedUserAccount()) {
        return createChat();                 // already logged in: no form, no lead
    }
    const contacts = await TxGlobalAuth.showContactsForm({
        requirements: { phone: true, name: true },
        showAgreements: true,
    });
    await TxGlobalAuth.authenticate({
        initialData: {
            phone: contacts.phone,
            email: contacts.email,
            agreementsAccepted: contacts.agreementsAccepted,
        },
    });
    await TxGlobalAuth.marketing.sendResponse({
        data: {
            sourceId: 777,                   // "opened the support chat", issued by the CRM team
            new_website: 'FinamRu',
            firstname: contacts.name,
            telephone: contacts.phone,
            emailaddress: contacts.email,
        },
    });
    return createChat({ clientName: contacts.name });
}
```

Rules for this flow:

- **An anonymous token is not a login.** Check `isAuthenticatedUserAccount()`, never the mere presence of a token.
- **`authenticate({ initialData })` sends the code at once.** The login window opens on the code step for the phone and email from the form (see `global-auth:frontend` → "Kratos Mode").
- **The chat is created only after login**, through a factory, never under an anonymous token.
- **The chat gets refreshed tokens from `subscribeJWT`**: pass only the new token, not the whole chat configuration again. On `null` (logout, account switch) close the chat.
- **Send the lead only when the login happened in this call.** A logged-in user opening the chat again sends nothing.
- **Take the name for the chat and the lead from the form.** A Kratos L1 token has no name.
- **Sites without an SPA reload the page at login.** Save the intent (for example, "open the chat") in `sessionStorage` before `authenticate()`, and finish the scenario after the page loads. While the login runs, keep a `beforeunload` guard: if the page is already leaving, do not send the lead in this load, or it is sent twice or cut off.
- Closing the login window on the code step asks "Are you sure you want to cancel registration?"; `authenticate()` rejects after the user confirms.

On dev, the contour's CORS was seen to allow only `http://localhost:8080` and `http://localhost:3000` among localhost origins. For test phone numbers and email aliases, see `global-auth:frontend` → "Test Registrations".

## Review Checklist

When reviewing code that sends leads or embeds a chat, check each point:

1. `new_website` holds a host or URL instead of a `WebSite` value.
2. One SourceID for every action, or a SourceID read from app configuration, instead of one per funnel action.
3. A lead without `sourceId` or `new_website` (the server answers `400`).
4. `sendResponse(payload, { headers })`: the second argument is ignored.
5. `new_external_ids` sent as an array (the server answers `400`).
6. The product sets keys the widget fills (`new_source_url`, `new_utm_*`, `new_kratosid` …): they are overwritten.
7. `204` treated as "delivered to CRM".
8. The chat created before login or under an anonymous token.
9. Token refreshes not passed to the chat, or the whole configuration passed again.
10. "Logged in" checked by the presence of a token instead of `isAuthenticatedUserAccount()`.
11. A lead sent on every chat opening by a logged-in user.
12. A site without an SPA that does not save the intent before login, so the chat never opens after the reload.
13. `catch (e) { if (e.reason === …) }` around `showContactsForm`: there is no such field.
14. The chat's client name read from a Kratos L1 token, where it is absent.
15. Calls to `debug.unstable.sendMarketingLead()` / `sendLead()`, removed in 8.15.0.
