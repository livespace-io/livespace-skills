# Calling the Livespace API (client-direct) — full reference

Load this file when the plugin is **API-backed** (it fetches deal / company / user / product data from Livespace). It assumes the credential handshake in [postmessage-contract.md](postmessage-contract.md) is already wired.

## One-shot sessions, 15-minute TTL

**Every `apiSession` is consumed by exactly one Livespace API call.** Reusing the same session for a second call returns an auth error. Two parallel calls require two parallel credential requests with distinct `requestId`s.

In addition, each session is valid for **15 minutes** from the moment it is issued. Sessions older than 15 minutes are rejected even if they have never been used. In practice this means: don't pre-fetch credentials at mount time and cache them for a later user action — request them at the moment of the call. The one-shot + per-call request pattern below already satisfies both constraints.

The pattern that scales:

```js
async function callLivespace(path, params) {
  const c = await requestFreshCredentials();   // posts a new request, awaits matching reply
  const qs = new URLSearchParams({
    _api_auth: 'key',
    _api_key: c.apiKey,
    _api_sha: c.apiSha,
    _api_session: c.apiSession,
    ...params,
  });
  const res = await fetch(`${c.apiBaseUrl}${path}?${qs}`);
  // ... unwrap, throw on error, return JSON ...
}

// parallel calls each get their own session:
const [deal, user] = await Promise.all([
  callLivespace('/api/public/json/Deal/get', { id: dealId }),
  callLivespace('/api/public/json/Default/User_getInfo'),
]);
```

## `apiBaseUrl` is the API host

`apiBaseUrl` carries the **host (scheme + domain + optional port)** of the Livespace API the plugin should talk to. Build every URL from this value:

```
GET {apiBaseUrl}/api/public/json/<Module>/<action>?<auth + endpoint params>
```

Do **not** assume the API lives on `location.origin` and do **not** hardcode a host. The plugin must be portable — the host decides where the API lives. In dev setups where browser CORS blocks direct calls to the real Livespace, hosts may instead send `apiBaseUrl = location.origin` and front the API with a same-origin proxy. The plugin code stays identical either way.

## Endpoints worth knowing

- `GET /api/public/json/Deal/get?id=<deal-id>` — single deal. Response shape: `{ data: { deal: { id, name, value, currency, ... } }, status: true }`. Note the double nesting — unwrap `data.deal`.
- `GET /api/public/json/Contact/get?type=<company|person>&id=<uuid>` — single CRM contact. Response shape: `{ data: { company: { id, name, note, nip, regon, ... } | person: { ... } }, status: true }`. Useful fields on `company`: `name`, `note` (free-text), `nip`, `regon`, `phones`, `emails`, `address_*`, `tags`, `dataset` (custom fields keyed by UUID, with `dataset_field_name` providing human labels).
- `GET /api/public/json/Default/User_getInfo` — current user *profile*: `login`, `email`, `firstname`, `lastname`, `name`, `locale`, `structures`. **Does not return `id`** — use `user.id` from postMessage.

Most actions accept their parameters either as URL query string (skill default, standard HTTP) or as a JSON request body on a GET (matches the curl example in the Postman collection — some actions are sensitive to one or the other). If a call returns no data via one style, try the other before assuming the endpoint is broken.

**Full API reference**: <https://api-docs.livespace.io/> — Postman collection covering every public JSON endpoint (modules, actions, request/response shapes). Import it into Postman to explore beyond the two endpoints above. Treat the collection as authoritative for path names and parameters.

## Defend against shape drift

Read fields via candidate lists (`pickFirst(obj, ['value', 'amount', 'value_net'])`), not direct property access. Log full responses to the console on parse failure so the field-name candidate list can be extended without redeploying.
