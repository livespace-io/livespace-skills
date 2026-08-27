# postMessage contract — full reference

Three message types, all under the `livespace-plugin:` namespace. The plugin is the iframe; the host is `window.parent`. Load this file when implementing or debugging the handshake.

> This mirrors the official public plugin docs — <https://plugin.docs.livespace.io/context-messages/>. Treat those docs as authoritative if they ever diverge from this summary.

## Envelope

Every message is an object (or its JSON string form). Inbound handlers must accept both:

```js
function onMessage(event) {
  let data;
  try {
    data = typeof event.data === 'string' ? JSON.parse(event.data) : event.data;
  } catch (e) { return; }
  if (!data || !data.type) return;
  // dispatch on data.type ...
}
```

Livespace validates the **origin** of every message it receives from the plugin against its domain allowlist and silently ignores messages from non-allowlisted origins — so the plugin must be served from an allowlisted domain, but does not need its own outbound origin check. On **inbound** messages the plugin **must validate `event.data.type`** before acting (the handler above does this); it is not required to check `event.origin`.

Outbound from the iframe:

```js
window.parent.postMessage({ type: 'livespace-plugin:...', payload: { ... } }, '*');
```

Target origin `'*'` is used because the iframe doesn't know the host's origin a priori. Hosts may receive these on `window.addEventListener('message', ...)` and reply via `event.source.postMessage(...)`.

## Type 1 — `livespace-plugin:context` (host → iframe)

The payload shape depends on **which profile the iframe is embedded in**. Four profile types are supported; the host populates the matching entity key. Plugins should switch on which key is present.

```ts
{
  type: 'livespace-plugin:context',
  payload: {
    // Deal-profile embedding:
    deal?:    { id: string /* UUID */, name: string },
    // Company-profile embedding (CRM company):
    company?: { id: string /* UUID */, name: string },
    // Person-profile embedding (CRM contact person):
    person?:  { id: string /* UUID */, firstName: string, lastName: string },
    // Space-profile embedding:
    space?:   { id: string /* UUID */, name: string },
    user:     { id: string /* UUID */ }
  }
}
```

| Profile | Payload key | Fields |
|---|---|---|
| Deal | `deal` | `id` (UUID), `name` |
| Company | `company` | `id` (UUID), `name` |
| Person | `person` | `id` (UUID), `firstName`, `lastName` |
| Space | `space` | `id` (UUID), `name` |
| All | `user` | `id` (UUID) — always present |

- All `id` fields are Livespace UUIDs.
- Exactly one of `deal` / `company` / `person` / `space` is present, determined by the profile the user is viewing. A plugin scoped to one profile reads just that key; a plugin supporting several should branch on presence.
- **`person` carries `firstName` and `lastName` — there is no single `name` field on a person.** Don't assume `.name` when handling the person profile.
- `user.id` is **always** present and is the **only** source of the logged-in user identity. The Livespace `Default/User_getInfo` endpoint does not return a user id field, so postMessage is the only viable source.
- The host may re-send `context` (e.g. when the user navigates to a different record without a full page reload). Plugins should treat each `context` message as a fresh state and re-resolve. If the new id matches the current one, the plugin can skip the reload as an optimization.
- Validation: require the `id` of whichever entity key(s) your plugin scopes to; require `payload.user.id` if eligibility depends on the user. Plugins that support multiple profiles should error out clearly when no entity key is present.

## Type 2 — `livespace-plugin:request-api-credentials` (iframe → host)

Only for API-backed plugins.

```ts
{
  type: 'livespace-plugin:request-api-credentials',
  payload: { requestId: string /* UUID generated per request */ }
}
```

- Generate `requestId` with `crypto.randomUUID()` (small fallback for old browsers).
- Track pending requests in a `Map<requestId, { resolve, reject, timer }>`.
- Set a per-request timeout (10–15 s) that rejects the Promise; otherwise a dropped host leaves the plugin hung.

## Type 3 — `livespace-plugin:api-credentials` (host → iframe)

Reply to type 2. Only for API-backed plugins.

```ts
{
  type: 'livespace-plugin:api-credentials',
  payload: {
    requestId: string,      // echoes the requestId from the request
    apiBaseUrl: string,     // host of the Livespace API, e.g. https://acme.livespace.io
    apiKey: string,
    apiSha: string,
    apiSession: string      // one-shot — consumed by one API call
  }
}
```

Correlation rules:

- Look up `requestId` in the pending Map. If absent → **drop the reply** and log a warning. Never accept unsolicited credentials.
- On match: delete the entry, clear its timeout, resolve the Promise with `{ apiBaseUrl, apiKey, apiSha, apiSession }`.
- Strip a trailing slash from `apiBaseUrl` before using it.

## Naming rule

Keep the `livespace-plugin:` prefix and dash-case; never invent new message types without host coordination.
