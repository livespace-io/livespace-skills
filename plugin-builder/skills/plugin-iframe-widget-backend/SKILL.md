---
name: plugin-iframe-widget-backend
version: "1.1"
description: Use when an iframe-widget Livespace plugin must keep third-party secrets server-side — after `plugin-iframe-widget`'s backend question was answered "Backend-enabled". The plugin ships its own Node / PHP-Symfony / Python backend that holds the secret and exposes a narrow JSON or HTML-fragment API to the iframe.
---

# Livespace iframe-widget plugin — backend-enabled variant

Use this skill when, in the `plugin-iframe-widget` flow, the backend question was answered **"Backend-enabled"** — i.e. the plugin must call a third-party service (JIRA, Linear, internal API, …) whose credentials cannot live in the iframe.

## Prerequisites

**All client-only rules in `plugin-iframe-widget` still apply** — iframe embedding, the postMessage handshake, breakpoints, the state machine, the visual palette, i18n. The only addition here is that the plugin now also ships **its own backend**, and the iframe calls that backend instead of (or in addition to) the third-party service directly.

## Is this variant right?

| Take the backend branch when… | Stay client-only when… |
|---|---|
| A third-party token (JIRA, GitHub PAT, Slack bot token, internal HR system) must stay on the server. | The plugin only consumes Livespace data — the regular credential handshake covers that. |
| You must enrich/transform with logic you don't want in the browser (proprietary scoring, business rules). | The third-party API supports CORS + public read — call it directly from the iframe. |
| You must aggregate multiple upstreams into one response and want the browser to see one call. | The "secret" is a config value the user can already see in Livespace — keep it client-side. |
| The third-party API has no CORS support, so the iframe can't call it regardless of secrets. | |

## Architecture at a glance

```
┌──────── Livespace host page ─────────┐
│  <iframe src="https://plugin.foo">    │
│   ┌─────────────────────────────────┐ │
│   │  GET /          → SSR shell      │ │ ← initial HTML rendered by backend
│   │  GET /widget.js → handshake +    │ │
│   │                   fetch /api/... │ │
│   │  GET /api/xyz?... → JSON         │ │ ← backend talks to third-party
│   └─────────────────────────────────┘ │
└──────────────────────────────────────┘
                │ postMessage
                ▼
        Livespace host JS
```

Commit this shape as a C4 **container** diagram at `docs/architecture/c4-container.mmd` before writing the backend. Show four containers — the **Livespace host page**, the **iframe SPA**, this plugin's **backend**, and the **third-party service** — and the calls between them: `postMessage` host↔iframe, HTTPS iframe→backend, HTTPS backend→third-party (and backend→Livespace API if the plugin uses it).

The iframe still does the postMessage handshake to get the record context (deal / company / person / space) + `apiBaseUrl`, but then calls **its own backend** (`/api/...`). The backend is the only thing that holds the third-party secret. Prefer routing every upstream call (Livespace and third-party) through the backend — see "Proxying upstream APIs" below.

## Pick a backend stack

Pick the one the team already runs — none is meaningfully "better" for a tiny widget, so familiarity wins.

| Stack | Server | Templating | Dynamic FE | Upstream HTTP |
|---|---|---|---|---|
| **Node** (Express / Fastify) | `node:20-alpine` | tagged template literals or EJS | vanilla JS modules (htmx / Lit if needed) | native `fetch` |
| **PHP / Symfony Minimal** | `php:8.3-fpm-alpine` + `nginx:alpine`, or `php:8.3-cli` with `symfony/runtime` | Twig | Symfony UX (Stimulus + LiveComponent) via AssetMapper — no JS build step | `symfony/http-client` |
| **Python** (FastAPI / Flask) | `python:3.12-slim` | Jinja2 | vanilla JS modules or htmx | `httpx` |

Decision shortcuts: one or two endpoints, no preference → **Node** (lowest ceremony). PHP shop, or SSR + reactivity without a JS build → **Symfony Minimal + UX**. Python shop, or an existing data/ML stack → **FastAPI**.

**Per-stack project layouts (Node / Symfony / Python) and the Symfony UX setup: [stack-layouts.md](stack-layouts.md).** Read the section for your chosen stack.

## Backend rules (language-agnostic)

1. **Server holds the secret, period.** No third-party token or internal API key in any file served to the browser. Read secrets from env vars at startup; refuse to start if a required one is missing (or return a clearly-marked `503 not_configured` per endpoint).
2. **One narrow endpoint per resource.** `GET /api/<thing>?<args>` returning JSON. **Shape the response** — never proxy the upstream payload byte-for-byte: that leaks fields and couples the widget to upstream shape drift.
3. **Validate at boundaries, escape at call sites — don't conflate them.** Validate inbound params to reject *malformed* input (control chars, oversize, wrong type, bad id format) with `400 invalid_<param>`. Then **always escape at the upstream call site** (JQL, SQL, shell, URL component) — don't rely on a restrictive input regex for injection safety; restrictive regexes age poorly (Polish characters, free-text company names flowing into JQL). Prefer "loose validation + tight escape" over "tight validation + no escape".
4. **Health endpoint.** `GET /healthz` → `200 { ok: true }`. The deploy target needs a readiness signal.
5. **Bind to all interfaces, not loopback.** Node `app.listen(port, "0.0.0.0")`, FastAPI `uvicorn ... --host 0.0.0.0`, Flask `app.run(host="0.0.0.0")`, Symfony behind nginx `listen 0.0.0.0:8080`. Loopback-only is a common silent orchestrator failure.
6. **Log only metadata, never secrets.** Status codes, durations, upstream codes, param names. Not the full URL (auth in query string), not response bodies (PII), not env values.
7. **Run as a non-root user.** Use the base image's user (Node's `node`) or create one (`adduser -D app` on Alpine, then `USER app`). Never run the entrypoint as root.
8. **Rewrite auth-protected upstream asset URLs through a backend proxy.** Many upstream APIs return URLs that need auth to fetch (JIRA custom avatars, Slack files, Confluence attachments). The browser can't send auth headers on `<img>`/`<link>` requests, so they render broken. Detect the URL shape in the response shaper, rewrite to your own route (e.g. `/api/avatar/<type>/<id>`), and proxy server-side with the upstream's auth attached. Cache aggressively (`Cache-Control: public, max-age=3600`). Public URLs (gravatar, CDN) pass through untouched.

## Frontend rules (in addition to all client-only rules)

1. **SSR a shell, not a blank page.** `GET /` returns the complete HTML scaffold — header, skeleton, footer — with the palette linked/inlined. The iframe must never flash empty before JS runs.
2. **Defer all data fetching to the client.** The server doesn't know the deal / company context at render time (postMessage hasn't fired). Render `loading` in HTML, then let the client controller do the handshake + `/api/...` call + DOM swap. **Support a `?label=` (or similar) query-param override** for direct dev access — skip postMessage when present; saves staring at a 12 s handshake timeout.
3. **Use the credential handshake even if you don't call the Livespace API.** Often the widget just needs `apiBaseUrl` (tenant id) or the entity id. Request credentials, take only the field(s) you need, discard the rest. Document in the README that the credentials aren't used to authenticate any call.
4. **Escape every value injected into HTML.** Issue titles, status names, user names — anything from upstream is untrusted. Use an `escape()` helper (Node) or default autoescape (Twig/Jinja2). The backend validated input shape but does **not** sanitise upstream response content.

## Communication contract (still postMessage)

Same three message types as client-only plugins — `livespace-plugin:context`, then `request-api-credentials` → `api-credentials` (request-id correlation, one-shot session, 15-minute TTL). See the **postMessage contract** section of the `plugin-iframe-widget` skill for the full shape and re-send semantics. A backend-enabled plugin that doesn't need Livespace data can still use the reply to extract `apiBaseUrl` and derive the tenant hostname; treat `apiKey`/`apiSha`/`apiSession` as if they don't exist.

## Proxying upstream APIs through your backend (preferred)

For backend-enabled plugins, route every upstream call (Livespace, third-party) through your own backend rather than client-direct:

```
Browser (iframe)
   │  request/receive one-shot creds via postMessage
   │  POST /api/resolve-X  { apiBaseUrl, apiKey, apiSha, apiSession, … entity ids }
   ▼
Your backend
   │  GET <apiBaseUrl>/api/public/json/<Module>/<action>?<auth + params>   (Livespace)
   │  GET https://api.example.com/...                                       (third party)
   ▼
Upstream APIs
```

Why it beats client-direct: **no CORS dance** (server-to-server has no Origin header to allowlist); **one narrow, shaped contract surface** (upstream drift doesn't reach the client); **server-side validate + escape in one place**. The credentials travel one extra HTTPS hop but aren't stored — the backend uses them immediately and discards them, so net security ≈ client-direct.

Node sketch (validate all fields, reject 400 on missing; shape the response — never proxy byte-for-byte):

```js
app.post("/api/resolve-label", async (req, res) => {
  const { apiBaseUrl, apiKey, apiSha, apiSession, companyId } = req.body || {};
  const base = String(apiBaseUrl).replace(/\/$/, "");
  const auth = { _api_auth: "key", _api_key: apiKey, _api_sha: apiSha, _api_session: apiSession };
  const qs = new URLSearchParams({ ...auth, type: "company", id: companyId });
  const r = await fetch(`${base}/api/public/json/Contact/get?${qs}`, { headers: { Accept: "application/json" } });
  const body = await r.json();
  res.json({ label: body?.data?.company?.note ?? null });
});
```

Some Livespace actions ignore query params and want a JSON body on GET — expose a `LIVESPACE_PARAM_STYLE` env flag (`"qs"` | `"body"`) so you can switch styles without code changes.

**Pagination:** for lists under ~100 items, prefer single-fetch + client-side pagination over threading upstream page tokens through your backend. Fetch up to a max (e.g. `maxResults=50`), shape `{ total, items: [...] }`, cache client-side, slice per page on click. When upstream has more than your max, surface `50+` in the footer. Add real upstream pagination only when you actually need it.

## Deployment

The widget is a containerised web app — deploy wherever you host containers (PaaS, k8s, cloud container service, plain VM). Three non-negotiables: it must be reachable over **HTTPS** at a stable origin (an `https://` host embedding an `http://` iframe is blocked as mixed content); its public origin's domain must be on the Livespace **domain allowlist** (external domains need manual approval — Livespace-hosted are auto-approved); and it must expose **`GET /healthz`**.

Inject configuration as **environment variables** (twelve-factor). Keep **secrets and per-environment values** (tokens, instance ids, debug flags) out of the image and out of version control — provide them at deploy time from the platform's secret store, locally from a git-ignored `.env`. **Fixed non-secret values** (upstream base URL, page size, field ids) can be baked in as defaults with an env override. Three rules:

- Prefer one mechanism per variable. If you use both shell/templating interpolation **and** a container env-file, know their precedence (in Docker Compose a literal under `environment:` overrides `env_file:`, so an unset interpolated var can silently blank a value).
- An unset required secret should fail **loudly and early**: `503 not_configured` from the affected endpoint, plus a startup warning for any "X set but its companion Y missing" combination (e.g. scoped token present but gateway/cloud id missing).
- Defaults for optional knobs belong in **app code**, not only the deploy template.

## Security caveats (also disclose in README)

- The third-party token lives in the container's environment. Not reachable from the browser, but anyone who can read the container's env can see it. Use a least-privilege service account with the **minimum** scopes, rotate on staff changes, and reference a secret manager if the platform offers one.
- The `/api/...` endpoints have **no auth** by default — anyone who can reach the origin can call them. If that's not OK: (a) VPN / SSO proxy, (b) a shared secret header the iframe injects post-handshake, or (c) validate `Origin`/`Referer` against the expected host (weak, but better than nothing).
- The backend is a real attack surface client-only plugins don't have. Run a basic security review (input validation, rate-limit obvious abuse, dependency scanning) before exposing it externally.

## Verification (in addition to client-only checks)

1. `GET /healthz` returns 200 on a fresh container.
2. `GET /api/<thing>` with a malformed parameter returns 400 with a stable error code — not a stack trace, not a 500.
3. With env vars unset, the endpoint returns `503 not_configured` (or equivalent) rather than crashing the process.
4. The SSR shell renders the full `loading` layout with JavaScript disabled in DevTools (i.e. before the client controller executes).
5. End-to-end: from the `plugin-iframe-widget` host harness ([examples/host-harness.html](../plugin-iframe-widget/examples/host-harness.html)), the widget transitions loading → ready and the rendered list matches what `curl http://localhost:8080/api/<thing>?...` returns.

## Reference files

Loaded on demand:

| File | Read it when |
|---|---|
| [stack-layouts.md](stack-layouts.md) | After picking a stack — per-stack project layout (Node / Symfony / Python) + Symfony UX setup. |
| [jira-cloud.md](jira-cloud.md) | The upstream third-party service is JIRA Cloud — token-shape/URL, scope, and avatar gotchas. |
