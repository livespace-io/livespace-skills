---
name: plugin-iframe-widget
version: "1.1"
description: Use when building an iframe-widget Livespace plugin — after the `plugin-builder` router selected the iframe-widget type. A pure client-side app embedded as an `<iframe>` inside a deal or company profile, receiving entity context (and optionally Livespace API credentials) via postMessage. For plugins that must keep third-party secrets server-side, see the `plugin-iframe-widget-backend` sub-skill.
---

# Livespace iframe-widget plugin — builder recipe

A Livespace **iframe widget** is a **pure client-side app** (HTML/CSS/JS) loaded as an `<iframe>` inside a Livespace profile — **Deal, Company, Person, or Space**. It is served over **HTTPS** from a **Livespace-allowlisted domain** (Livespace-hosted origins are auto-approved; external domains need manual approval — see Security caveats). No backend assumption beyond the static asset server. Communication with the host happens exclusively through `window.postMessage`. An account administrator registers the plugin (name + URL) and configures which profile types show it, where, its size, and who can see it.

## Prerequisites

Reach this skill **after the `plugin-builder` router captured the plugin's purpose and selected the iframe-widget type.** If you arrived directly, confirm the purpose is captured (`plugin-builder` Step 1) — it drives every decision below. Follow the **Interaction Rules** from the `plugin-builder` router: one question at a time, multiple-choice preferred, short headers.

## Build flow

### Question 1 — API access?

Ask whether the plugin needs to call the Livespace API at all (header `API access`):

- **Static plugin (no API access)** — renders only from the postMessage `context` payload (`deal`/`company` id + name, `user` id). Skips the credential handshake entirely. Examples: an informational badge, a link to an external system, a form posting to a non-Livespace endpoint.
- **API-backed plugin** — fetches deal / company / user / product data from Livespace. Requires the full credential handshake. See [livespace-api.md](livespace-api.md).

A static plugin is materially simpler: no credential request, no pending-request Map, no timeout handling, no `apiBaseUrl` plumbing. Only opt into the API branch when the plugin actually needs server-fetched data.

### Question 2 — secrets server-side?

Ask whether the plugin must call a **third-party service** (JIRA, Linear, internal HR system, …) whose credentials must never reach the iframe (header `Backend`):

- **Client-only** — everything runs in the iframe; hostable from any static origin. Continue here.
- **Backend-enabled** → **invoke `plugin-iframe-widget-backend`** before scaffolding. The frontend half (this skill) still fully applies; the backend skill adds the server, layout, and deployment story.

A backend is a real operational cost (deploy target, env vars, secret rotation, monitoring). Don't take it for a plugin that only needs a public API or only consumes Livespace data via the regular handshake — push for client-only first.

## Project layout

Generic client-side app. Kebab-case, English file names.

```
<project>/
├── index.html        # entry point + plugin shell
├── assets/           # optional: JS modules, CSS, JSON data, images
│   ├── plugin.js
│   ├── plugin.css
│   └── data.json
├── docs/architecture/c4-context.mmd   # host ↔ iframe ↔ (API) — required: a plugin is a separate deployable
├── README.md
└── CLAUDE.md
```

A plugin is a separate deployable, so capture its boundary as a C4 **context** diagram at `docs/architecture/c4-context.mmd` before building. Keep it to a single level of detail — three actors and the relationships between them:

- the **Livespace host page** (the profile embedding the plugin),
- **this plugin** (the iframe), and
- the **Livespace API** (only if the plugin is API-backed).

Show `postMessage` between host and iframe, and HTTPS between iframe and API. When the backend variant is chosen, add a **container** diagram (`c4-container.mmd`) — see `plugin-iframe-widget-backend`.

Single-file `index.html` (inline JS+CSS, Vue 3 CDN or vanilla) suits tiny plugins. Split into `assets/*` modules once logic, state, or data grows.

## postMessage at a glance

Three message types under the `livespace-plugin:` namespace; the plugin is the iframe, the host is `window.parent`:

1. `livespace-plugin:context` (host → iframe) — the record's context: exactly one of `deal` / `company` / `person` / `space` (whichever profile embeds the iframe) + `user.id` (always present, the only source of the logged-in user identity). **`person` carries `firstName`/`lastName`, not a single `name`.**
2. `livespace-plugin:request-api-credentials` (iframe → host) — API-backed only; one per call, correlated by a generated `requestId`.
3. `livespace-plugin:api-credentials` (host → iframe) — the reply, echoing `requestId`.

Outbound uses target origin `'*'`; Livespace validates the iframe's origin against its domain allowlist on its side, so the plugin needs no outbound origin check — but **must validate `event.data.type`** on every inbound message before acting. **Full envelope, TypeScript shapes, the profile→fields table, validation, and correlation rules: [postmessage-contract.md](postmessage-contract.md).** Never invent new message types without host coordination.

## State machine

Distinguish at minimum these four states; never show a blank iframe (it looks broken):

1. `loading` — fetches in flight. Show a skeleton, not a spinner.
2. `ready` (or a domain equivalent like `eligible`) — main render.
3. `not-applicable` — context arrived but nothing to show (e.g. user not authorized). Render a polite placeholder.
4. `error` — fetch / parse / API failure. Show a message **and** a retry CTA that re-runs the failed step (not a full iframe reload).

Styling for each state — skeleton, empty, error, motion — is in [ui-design.md](ui-design.md).

## UI & i18n

Make it feel like a polished component, not a debug page: a persistent header label, one hero value, a supporting metadata row, three width breakpoints, and the shared visual palette. User-facing copy is **Polish first, English fallback**. **Full design reference — breakpoints, palette, anatomy, state styling, number formatting: [ui-design.md](ui-design.md).**

## Security caveats

### Platform constraints (Livespace-enforced)

- **HTTPS only.** Plain-HTTP plugin URLs are rejected at registration.
- **Domain allowlist.** A plugin URL is accepted only if its domain is on Livespace's allowlist — the primary security boundary. Livespace-hosted origins are auto-approved; **external domains need manual approval** (contact Livespace support with the domain, a short app description, and the account name; ~a few business days). "Host it anywhere" is wrong — confirm the domain is allowlisted before promising a deploy target.
- **Iframe `sandbox`.** Livespace loads every plugin in a sandboxed iframe with a **fixed** capability set — `allow-scripts`, `allow-same-origin`, `allow-forms`, `allow-popups`, `allow-downloads` — identical for every plugin and not configurable per plugin. So scripts, same-origin storage, form submission, popups (e.g. a "shortcut" plugin opening an external tool in a new tab), and downloads all work; anything outside that set — notably top-level navigation of the host — does not. Design within these capabilities. Full details: <https://plugin.docs.livespace.io/security/>.
- **Inbound validation.** Always validate `event.data.type` before acting on a message. Livespace already drops messages from non-allowlisted origins on its side.

### Data exposure (always disclose in README)

- Credentials arrive in the iframe and are reachable from JS / DevTools.
- Any data the plugin fetches (e.g. a static config JSON) is reachable from the network panel.
- Production-grade plugins that need to keep data confidential must move sensitive logic behind a backend that returns only rendered results. This skill scaffolds client-only plugins; if the plugin needs server-side secrets, use `plugin-iframe-widget-backend` and flag the gap in the project README.

## Naming & language

- Project & file names, code identifiers: English, kebab-case.
- User-facing copy: Polish first, English fallback.
- Message types: keep the `livespace-plugin:` prefix and dash-case.

## Verification

Exercise the plugin from the bundled host harness — [examples/host-harness.html](examples/host-harness.html) — which embeds the plugin, sends `context`, and answers credential requests (no real Livespace needed). Serve both over http (e.g. `npx serve .`) and point the harness at your plugin URL. Confirm:

1. Sending `livespace-plugin:context` transitions the plugin out of `loading` into the correct end state.
2. (API-backed) Two parallel API calls succeed simultaneously — proves the one-shot credential handshake works under concurrency.
3. Missing context fields or a bad id triggers `not-applicable` or `error`, with the retry CTA working.
4. PL/EN switch produces the right strings (toggle browser locale).
5. RWD is visually correct at each breakpoint width — use the harness's 200 / 320 / 480 px presets.

## Reference files

Loaded on demand — read the one relevant to the step you're on:

| File | Read it when |
|---|---|
| [postmessage-contract.md](postmessage-contract.md) | Implementing or debugging the host↔iframe handshake (envelope, message shapes, correlation). |
| [livespace-api.md](livespace-api.md) | The plugin is API-backed — calling Livespace, one-shot sessions, endpoint reference. |
| [ui-design.md](ui-design.md) | Laying out or styling the plugin — breakpoints, palette, anatomy, state styling, i18n. |
| [examples/host-harness.html](examples/host-harness.html) | Verifying locally — a runnable host that drives the postMessage contract. |
