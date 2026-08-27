# Livespace Skills

A [Claude Code](https://docs.claude.com/en/docs/claude-code) **plugin marketplace** from the Livespace team, starting with a builder for Livespace platform plugins and growing over time.

## Plugins

| Plugin | What it does |
|---|---|
| [`plugin-builder`](./plugin-builder) | Build a **Livespace plugin** — an iframe widget embedded in a Deal, Company, Person, or Space profile. A router skill captures the plugin's purpose and routes to a client-only or backend-enabled builder. |

_More plugins will be added over time._

## Install

Add this marketplace, then install the plugin you want.

```bash
claude plugin marketplace add https://github.com/livespace-io/livespace-skills.git
claude plugin install plugin-builder@livespace
```

Or browse interactively inside a `claude` session:

```
/plugin
```

## Repository layout

```
livespace-skills/
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest — lists the plugins below
├── plugin-builder/               # a plugin (one directory per plugin)
│   ├── .claude-plugin/
│   │   └── plugin.json           # plugin manifest
│   ├── README.md
│   ├── CHANGELOG.md
│   └── skills/                   # one directory per skill
│       ├── plugin-builder/                 # router / intake
│       ├── plugin-iframe-widget/           # client-only builder
│       └── plugin-iframe-widget-backend/   # backend-enabled builder
├── CONTRIBUTING.md               # how to add a new public skill (incl. the "public-ification" checklist)
├── LICENSE
└── README.md
```

## Adding more skills

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the repo layout and conventions.

## License

[MIT](./LICENSE).
