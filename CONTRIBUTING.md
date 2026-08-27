# Contributing

A **public** Claude Code plugin marketplace. Anything merged here is world-readable — keep it self-contained and free of secrets.

## Adding a new plugin

1. Create a top-level directory for the plugin, e.g. `my-plugin/`.
2. Add `my-plugin/.claude-plugin/plugin.json`:
   ```json
   {
     "name": "my-plugin",
     "description": "One-line description shown in the marketplace.",
     "version": "1.0.0",
     "author": { "name": "Livespace" }
   }
   ```
3. Put each skill under `my-plugin/skills/<skill-name>/SKILL.md` (plus any reference files the skill loads on demand).
4. Register the plugin in `.claude-plugin/marketplace.json` by adding an entry to the `plugins` array (`name`, `source`, `description`).
5. Add a `README.md` and `CHANGELOG.md` to the plugin directory.

## Keep contributions self-contained

Anything merged here is public, so before committing:

- [ ] **No secrets.** No tokens, keys, passwords, private hostnames, or real tenant/customer names. Use env-var placeholders (`%env(...)%`) and obviously-fake sample data (`acme.example`, `localhost:3000`).
- [ ] **Self-contained links.** Every `[link](...)` and every "see the X skill" resolves within this repo — don't point at a resource a reader can't open.
- [ ] **Platform claims match the public docs.** Verify anything documenting the Livespace platform against <https://plugin.docs.livespace.io/> and <https://api-docs.livespace.io/>, and link them as authoritative.

A quick grep before committing catches most accidental secrets:

```bash
grep -rniE "token=|secret|password|api[_-]?key" .
```

## Versioning

Bump the plugin's `version` in its `plugin.json` and add a `CHANGELOG.md` entry for every user-visible change.
