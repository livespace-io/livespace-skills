# Backend stack layouts

Load this file once a stack is chosen (see the stack-selection table in `SKILL.md`). Read only the section for your stack. The widget is a containerised web app — package it however your hosting expects; a `Dockerfile` is the portable lowest common denominator.

## Node (Express)

```
<project>/
├── Dockerfile                  # node:20-alpine multi-stage, USER node
├── package.json                # express only — Node 20 has native fetch
├── server.js                   # SSR shell + /api/... + /healthz
├── public/
│   ├── widget.css
│   └── widget.js               # postMessage handshake + dynamic rendering
└── README.md
```

Worked example (illustrative): a "JIRA-tasks-for-this-company" widget embedded on a CRM company profile. It reads `company.id` from the postMessage context, resolves a filter value by calling the CRM API server-side, then runs a JQL search and renders the issues. That single example exercises most of this skill: entity context, server-side proxying of an upstream API, a scoped JIRA Cloud token through the `api.atlassian.com` gateway, auth-protected avatar URL rewriting, client-side pagination, and a `?label=` query-param override for dev.

## PHP / Symfony Minimal (with Symfony UX)

```
<project>/
├── Dockerfile                  # php:8.3-cli-alpine + symfony/runtime, or fpm+nginx
├── composer.json               # framework-bundle, twig, asset-mapper, stimulus-bundle, http-client
├── bin/console
├── public/
│   ├── index.php               # standard Symfony front controller
│   └── assets/                 # ← static fallback only; AssetMapper serves from src
├── assets/                     # mapped to /assets/* by AssetMapper
│   ├── widget.css
│   ├── app.js                  # registers controllers, imports widget.css
│   └── controllers/
│       └── widget_controller.js   # Stimulus controller (handshake + fetch + render)
├── config/
│   ├── bundles.php
│   ├── packages/
│   │   ├── framework.yaml             # session: enabled: false
│   │   ├── twig.yaml
│   │   ├── asset_mapper.yaml
│   │   └── stimulus.yaml
│   └── routes.yaml
├── src/
│   ├── Controller/
│   │   ├── ShellController.php        # GET / — renders Twig shell
│   │   ├── IssuesController.php       # GET /api/issues — JSON
│   │   └── HealthController.php       # GET /healthz
│   └── Service/
│       └── JiraClient.php             # symfony/http-client wrapper
├── templates/
│   └── shell.html.twig                # SSR layout, mounts <main data-controller="widget">
└── README.md
```

Notes on a "minimal" Symfony setup:

- Do **not** start from `symfony new --webapp`; that pulls security, doctrine, mailer, profiler. For a widget, you want just: `symfony/runtime`, `symfony/framework-bundle`, `symfony/yaml`, `twig/twig`, `symfony/twig-bundle`, `symfony/asset-mapper`, `symfony/stimulus-bundle`, `symfony/http-client`.
- Disable session (`framework.session: enabled: false`) — sessions in an iframe are a `SameSite=Lax` minefield and you don't need them.
- Disable CSRF protection unless you have form submission inside the iframe.
- Use AssetMapper, not Encore. It serves ESM directly, no `node_modules`, no build container.
- Provide a single `widget_controller.js` (Stimulus) instead of bare DOM scripting. The Twig shell mounts it via `data-controller="widget"`; targets/values/actions replace inline event handlers.

### Symfony UX specifics

Symfony UX is the "modern dynamic FE" answer for the PHP branch. Key points so the scaffold lands on the first try:

- **AssetMapper, not Encore/Webpack.** No `node_modules`, no build container, no `package.json`. `composer require symfony/asset-mapper symfony/stimulus-bundle` is enough. Assets live in `assets/` and are mapped to `/assets/*` at runtime; importmap is auto-managed via `bin/console importmap:require`.
- **Stimulus controllers for behaviour.** One `assets/controllers/widget_controller.js` exporting a default class extending `Controller`. Wire it via `data-controller="widget"` on the shell's `<main>` element. Use `connect()` to start the postMessage handshake, `targets` for the body/footer regions, `values` for any config injected from Twig (e.g. backend base URL — usually empty since same-origin).
- **Twig shell renders the static layout + Stimulus mount points.** Example skeleton:
  ```twig
  {# templates/shell.html.twig #}
  <!DOCTYPE html>
  <html lang="pl">
    <head>
      <meta charset="utf-8">
      <meta name="viewport" content="width=device-width,initial-scale=1">
      <title>{{ title }}</title>
      {{ importmap('app') }}
    </head>
    <body>
      <main class="widget" data-controller="widget" data-state="loading">
        <header class="widget__header">…</header>
        <section class="widget__body" data-widget-target="body">
          <ul class="skeleton" aria-hidden="true"><li></li><li></li><li></li></ul>
        </section>
      </main>
    </body>
  </html>
  ```
- **LiveComponent is overkill for read-only lists.** It's worth it when the widget has interactive forms, multi-step flows, or server-driven re-renders. For a "fetch JSON, render list" widget, vanilla Stimulus + `fetch()` is simpler and lighter than turning every state change into a server round-trip.
- **HTTP client.** `symfony/http-client` with scoped configuration in `config/packages/framework.yaml`:
  ```yaml
  framework:
    http_client:
      scoped_clients:
        jira.client:
          base_uri: '%env(JIRA_BASE_URL)%'
          auth_basic: ['%env(JIRA_EMAIL)%', '%env(JIRA_TOKEN)%']
          headers:
            Accept: application/json
  ```
  Inject `HttpClientInterface $jiraClient` (autowired by name) into the controller. Never build the auth header manually.
- **Caching primitive.** `symfony/cache` with a filesystem or APCu adapter is enough for tiny widgets. Skip Redis until you actually need cross-instance cache.
- **Routing.** Define routes in `config/routes.yaml` (not annotations) for the minimal setup — the `#[Route]` attribute pulls in `doctrine/annotations` if you're not careful. YAML keeps the dep footprint small.

## Python (FastAPI)

```
<project>/
├── Dockerfile                  # python:3.12-slim, uvicorn entrypoint
├── pyproject.toml              # fastapi, uvicorn[standard], jinja2, httpx
├── server.py                   # FastAPI app + Jinja2 templates + /api + /healthz
├── templates/
│   └── shell.html.j2
├── public/
│   ├── widget.css
│   └── widget.js
└── README.md
```

Notes:

- Flask is equally fine; FastAPI is slightly less verbose for JSON endpoints and gives request validation for free.
- Mount static via `app.mount("/", StaticFiles(directory="public"))` after the routes — or serve via a separate route to avoid colliding with `/`.
