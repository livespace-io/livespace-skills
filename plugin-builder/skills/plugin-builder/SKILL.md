---
name: plugin-builder
version: "1.0"
description: Use when the user wants to build a Livespace plugin — a custom extension embedded inside a Livespace deal or company profile. Captures the plugin's purpose, picks the plugin type, and routes to the matching builder sub-skill.
---

# Plugin Builder

## Purpose

A **Livespace plugin** is a custom extension embedded inside a Livespace profile — **Deal, Company, Person, or Space**. An account administrator registers it (name + HTTPS URL on a Livespace-allowlisted domain) and configures which profile types show it, where, its size, and who can see it. The widget receives the current record's context from the host and renders something useful for the person viewing that record.

"Plugin" is the umbrella concept. **Today there is one plugin type — the iframe widget** (a client-side app loaded in an `<iframe>`, talking to the host over `window.postMessage`). More types will be added over time; this router exists so that choosing a type is an explicit step, not a baked-in assumption that "plugin" means "iframe".

This skill is the **intake + routing layer**. It captures *what the plugin should do*, helps pick the *type*, then hands off to the type-specific builder sub-skill. It contains no build instructions, layout rules, or API detail — those live in the sub-skills.

Use this skill when:
- The user asks to build a Livespace plugin / embedded plugin.
- The user wants to embed a custom panel in a Deal, Company, Person, or Space profile.
- The task mentions a "plugin on the deal/company/person/space profile" or "iframe in Livespace".

---

# Interaction Rules

Always use `AskUserQuestion` for user input. Follow these principles:

- **One question at a time.** Never batch multiple questions into one message.
- **Multiple choice preferred.** Provide 2-4 concrete options. Easier to answer than open-ended.
- **"Other" is automatic.** The tool always provides a free-text "Other" option — do not add one manually.
- **Use headers.** Short labels (max 12 chars) like "Purpose", "Type", "Why", "Scope".

These principles are referenced by the builder sub-skills (`plugin-iframe-widget`, …) by this name. They are defined here once.

---

## Step 1 — Plugin responsibility

Before any technical decisions, ask the user what the plugin should *do*. Drive this with `AskUserQuestion`, header `Purpose`. Offer 3–4 broad categories as starting points; the auto "Other" lets the user free-text the actual requirements in their own words.

Suggested categories (adjust as needed):

- **Read-only display** — render derived information about the record (KPI, score, commission, status, computed value).
- **Interactive form** — collect input from the user and submit it somewhere (Livespace API or external).
- **Shortcut / external link** — a button or link that opens another tool with the record id as context (note: the iframe `sandbox` restricts top-level navigation — see `plugin-iframe-widget` security caveats).
- **Multi-step workflow** — guided process (wizard, approval flow, configuration).

Capture the answer verbatim into the project's `README.md` or `CLAUDE.md` so later work has the original requirement to refer back to. If the user picks "Other", quote their description directly — don't paraphrase.

This answer drives every later decision: what data to fetch, which fields to consume, what the empty / error / not-applicable states look like, how to lay out content. Don't skip ahead to scaffolding before this question is resolved.

### Escalation — right-size the intake

The question above is a *lightweight* intake, right-sized for a small embedded plugin. When the plugin is non-trivial or you want a tracked spec, do a fuller discovery pass before building rather than improvising: capture the WHY → HOW → WHAT, write a short spec, and turn it into a tracked backlog item.

Keep the default light; escalate only when the plugin's complexity earns it.

---

## Step 2 — Plugin type

Once the purpose is clear, ask which **type** of plugin to build. Drive with `AskUserQuestion`, header `Type`: *"What plugin do you want to build today?"*

Available types:

- **Iframe widget** → invoke `plugin-iframe-widget`. A client-side app embedded via `<iframe>`, communicating with the host over `postMessage`. **This is the only type available today**, so unless the user has a specific reason to wait, this is where every plugin currently goes.

More types will be added to this list as they land. Even while only one type exists, surface this as an explicit choice — it sets the expectation that "plugin ≠ iframe" and makes adding the next type a one-line change here.

---

## Out of scope

This skill handles only purpose capture, discovery escalation, and type routing. Build instructions, project layout, the postMessage contract, API access, responsive design, and the backend-enabled path all live in the type-specific sub-skills.

---

## Red Flags

| Thought | Reality |
|---------|---------|
| "They said plugin, so I'll scaffold an iframe" | Capture purpose and confirm the type first — even with one type today. |
| "Skip the purpose question, the type is obvious" | Purpose drives every later build decision. Ask it first. |
| "This plugin is complex, but I'll wing the intake" | Do a fuller discovery/spec pass first for a tracked spec. |
| "I'll write the build steps here" | This is a router. Build detail lives in `plugin-iframe-widget` and its sub-skills. |
