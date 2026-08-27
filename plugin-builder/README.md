# plugin-builder

Build a **Livespace plugin** — a custom extension embedded inside a Livespace profile (**Deal, Company, Person, or Space**). An account administrator registers it (name + HTTPS URL on a Livespace-allowlisted domain) and configures which profile types show it, where, its size, and who can see it.

This plugin is a small **router + two builder sub-skills**:

| Skill | Role |
|---|---|
| `plugin-builder` | Intake + routing. Captures *what the plugin should do*, picks the plugin type, hands off. Contains no build detail. |
| `plugin-iframe-widget` | Builds a **client-only** iframe widget — pure HTML/CSS/JS talking to the host over `window.postMessage`, optionally calling the Livespace API via a one-shot credential handshake. |
| `plugin-iframe-widget-backend` | Adds a **backend** (Node / PHP-Symfony / Python) for plugins that must keep third-party secrets server-side. |

## How it flows

```
plugin-builder  ──(purpose captured, type = iframe widget)──▶  plugin-iframe-widget
                                                                     │
                                        (needs server-side secrets?) │
                                                                     ▼
                                                     plugin-iframe-widget-backend
```

## Platform docs

This plugin documents the Livespace plugin platform against the official public docs:

- Context / postMessage: <https://plugin.docs.livespace.io/context-messages/>
- Security model: <https://plugin.docs.livespace.io/security/>
- Full API reference (Postman collection): <https://api-docs.livespace.io/>

## Usage

Once installed, just ask Claude to build a Livespace plugin (e.g. *"build a Livespace plugin that shows this deal's commission"*). The `plugin-builder` skill triggers, captures the purpose, and routes to the right builder.
