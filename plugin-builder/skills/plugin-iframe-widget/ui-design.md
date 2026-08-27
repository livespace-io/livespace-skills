# UI, responsive design & i18n — full reference

Load this file when laying out the plugin or styling its states. Every plugin should feel like a polished, modern component — not a debug page.

> Future: when the Livespace design system is available as a shared dependency, the palette and component tokens below should be sourced from it rather than hardcoded here.

## Responsive design

Plugins render inside iframes whose outer dimensions are set by the host. The **height** is a fixed admin choice (S/M/L preset, see below); the **width adapts** to the available space in the profile layout and narrows when a side panel opens. Treat width and height as independent — never assume a particular aspect ratio. Use `width: 100%` on the root container (not a fixed pixel width), relative units (`%`, `rem`) for padding and type, and test from **260 px** upward.

### Width breakpoints

Support at minimum three width breakpoints:

| Breakpoint | Width range | Notes |
|---|---|---|
| micro   | `< 260px`     | tight spacing, small type |
| default | `260–399px`   | target first |
| wide    | `≥ 400px`     | more breathing room |

Declare every numeric style as a CSS custom property at `:root` and override the set in `@media (min-width: 260px)` and `@media (min-width: 400px)`. Component selectors consume the variables — **never hardcode pixel sizes in component rules**.

### Heights

Heights are independent of width breakpoints. The admin picks one of three height presets — **Small 240 px / Medium 380 px / Large 540 px** (an exact custom height is on the Livespace roadmap). Design against all three. The plugin must remain functional at any height — short content should center or pad gracefully, long content should scroll inside the iframe (the host controls the iframe's `scrolling` attribute).

Plugin fills `100vh` of the iframe; the host controls outer dimensions via `iframe width=… height=…`.

## Visual palette

| Token | Value | Used for |
|---|---|---|
| Ink | `#1a1a2e` | Primary text, hero numbers |
| Accent | `#3b82f6` | Interactive elements, accents |
| Muted | `#6b7280` | Subtitles, supporting text |
| Mute-mute | `#9ca3af` | Tiny uppercase labels, icons |
| Card bg | `#ffffff` | Plugin surface |
| Page bg | `#f0f4f8` | Behind the card |
| Soft hairline | `#f3f4f6` | Dividers, skeleton bg |

## Plugin anatomy

Even a minimal plugin benefits from a clear three-part structure:

```
┌─────────────────────────────────────┐
│  HEADER LABEL              [action] │  ← small uppercase, optional action
│                                     │
│  Hero / primary content             │  ← the one thing this plugin shows
│                                     │
│  Supporting metadata · breakdown    │  ← context, deal name, computed inputs
└─────────────────────────────────────┘
```

### Header (always present)

- A short uppercase label (e.g. "Twoja prowizja", "Status deala", "Powiązane projekty") in `Mute-mute` color, `letter-spacing: 0.06–0.08em`, `font-weight: 600`.
- Optional small action on the right: refresh icon, kebab menu, external-link arrow. Subtle — `Mute-mute` color, hover state in `Accent`.
- The header is **visible in every state** — `loading`, `ready`, `not-applicable`, `error`. The plugin should be identifiable even when there's nothing to render below.

### Hero content

- The single most important value the plugin exists to show: a currency amount, a count, a status badge, a stage name, a CTA.
- Type scale set via breakpoint CSS variables — large at `≥ 260px`, comfortable at `≥ 400px`.
- `Ink` for the number/text, `Accent` only on small accent tokens (currency suffix, separators, deltas).

### Supporting row

- One-line metadata under the hero: deal name, breakdown of how the hero was computed, related entities, timestamp.
- `Muted` color, smaller type, allowed to wrap. Use CSS-driven spacing (flex `column-gap`) rather than relying on template whitespace — template compilers strip leading/trailing whitespace inside elements.

## State styling

The four states are defined as standing behavior in `SKILL.md`. Style them as follows:

### Loading / placeholder state

Show a **skeleton that matches the final layout** — not a spinner, not a blank plugin, not text saying "Loading…".

- Skeleton bars at the same vertical position and roughly the same width as the real header / hero / supporting row that will replace them.
- Use the `Soft hairline` token as base; animate a subtle shimmer via a linear-gradient background-position keyframe (1.2–1.6 s).
- Hold the skeleton for at least ~150 ms even if data arrives faster — instant flashes feel like jank.
- The placeholder must convey "something is coming here", not "the plugin is broken".

### Empty / not-applicable state

- Keep the header visible.
- Replace the hero+supporting area with a centered, muted message explaining why there's nothing to show (in user language, not technical).
- No icon-soup or illustration for tiny plugins; a single calm sentence is enough.

### Error state

- Keep the header visible. Use a warm tint (`#b45309` ink, optional `#fef3c7` surface) so it's visually distinct from `not-applicable`.
- Short message + a retry call-to-action styled as a text-link (`Accent` color, underlined). The retry button must re-run the failed step, not full-reload the iframe.

### Motion & polish

- Transition state changes (`loading` → `ready`) with a 120–200 ms opacity / translateY fade. Avoid layout jumps.
- Hover states on every interactive element. Focus rings via `:focus-visible` for keyboard users.
- Respect `prefers-reduced-motion`: gate the shimmer and state transitions behind the media query so motion-sensitive users get a static rendering.

## i18n & number formatting

Default to Polish, offer English:

```js
const LANG = (navigator.language || 'pl').toLowerCase().startsWith('en') ? 'en' : 'pl';
```

Use a small `const I18N = { pl, en }` map plus a `t(key)` function — vue-i18n / i18next are overkill for tiny plugins.

Format numbers via `Intl.NumberFormat(LANG === 'pl' ? 'pl-PL' : 'en-US', …)`. When laying out lines with mixed bound and literal text, use CSS-driven spacing (flex `column-gap`) — template compilers strip leading/trailing whitespace inside elements.
