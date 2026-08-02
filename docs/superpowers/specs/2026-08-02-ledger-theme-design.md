# The Ledger — Hacker's Playground theme v2

**Date:** 2026-08-02
**Status:** Approved
**Scope:** Theme layer only (SCSS tokens/components, layouts, includes). No content rewrites, no search/comments/RSS work.

## Goal

Replace the current restrained kami reskin with a bold editorial ("security journal") identity: a dark, warm, magazine-grade theme with generative cover art — rekt.news energy, but built on Ataberk's existing palette family instead of rekt's cold black/red.

## 1. Identity & palette

One design, two palettes, selected by `prefers-color-scheme`. The manual theme toggle is **removed** (delete `_includes/theme-toggle.html`, the `localStorage` persistence + no-FOUC script in `head.html`, and `html[data-theme]` override blocks).

| Token | Dark (default feel) | Light (parchment Ledger) |
|---|---|---|
| Page background | `#181512` (warm soot) | `#FAF7F2` (existing parchment) |
| Sunken surface (code, ledger block, covers) | `#100e0c` | existing deep-ink surface |
| Body text | `#EDE6DA` | existing near-black ink |
| Accent | `#9DB8E8` (lifted ink-blue) | `#1B365D` (ink-blue) |
| Secondary accent (severity, warnings) | `#E8B44F` (amber) | darker amber, AA-checked |
| Hairlines/borders | `#2e2a25` | existing warm hairline |
| Muted text | `#8a8078` / `#b5aca1` | existing muted ink |

Both palettes live as token sets in `_sass/kami/_tokens.scss` (`:root` + `@media (prefers-color-scheme: dark)`). Same skeleton everywhere — light mode is the brand bridge to the CV/GitHub profile.

## 2. Typography

- **Charter serif stays** but scales up: larger masthead/headlines, italic dek (standfirst) under titles, drop cap on first paragraph of posts, styled pull quotes (`blockquote`).
- **Mono voice** (`ui-monospace` system stack): issue-number lines (`N°10 · VULN RESEARCH · …`), ledger block, labels, figure captions.
- No webfonts added; theme stays self-contained.

## 3. Homepage (`_layouts/index.html`)

Magazine cover layout:
- Latest post featured large: N° line, headline, dek, generative cover, read time/severity meta.
- Remaining posts in a two-column grid with small covers + N° line + title.
- Existing pagination kept.

## 4. Post page (`_layouts/post.html`) — "Künyeli Ledger"

Order: N° line → headline → italic dek → **mono ledger block** → body.

Ledger block fields come from an optional `ledger:` front-matter map:

```yaml
ledger:
  target: node-re2
  vector: CWE-835 loop
  impact: event-loop DoS
  severity: MODERATE 6.2
  advisory: CVE-2026-68499
  status: PATCHED
```

- All fields optional; rendered as `KEY ...... value` grid (2 columns desktop, 1 mobile).
- **If `page.ledger` is absent, the block does not render at all** (e.g. Hacker's Manifesto stays bare).
- Dek comes from existing `description`/`excerpt` front matter; absent → no dek line.

## 5. Generative covers

- A Jekyll include (`_includes/cover.html`) renders an **inline SVG** at render time. No build step, no JS requirement (small inline JS fallback allowed only if pure Liquid proves insufficient).
- Deterministic seed: hash derived from `page.slug` via Liquid (e.g. sum of character codes → modular arithmetic for coordinates/variation). Same slug → same art, forever.
- Motif selected by category: `writeups` → circuit traces, `projects` → node graph, `research`/vuln posts → topographic contours. Unknown category → circuit traces (default).
- Colors use theme tokens (`currentColor`/CSS variables) so covers adapt to both palettes.

## 6. Numbering & existing content

- Posts are numbered N°01–N°10+ **automatically from chronological index** (Liquid: `site.posts` reverse index). No front-matter needed; new posts auto-increment.
- Optionally add `ledger:` blocks to the security posts (CVE writeups, MSSQL n-day) later — separate content task, not part of this theme work.

## 7. Removals

- Theme toggle include + script, `html[data-theme]` token overrides.
- Any component styles that exist only for the old restrained look and have no Ledger equivalent (audit during implementation; prune, don't hoard).

## 8. Verification

- Compile SCSS with dart-sass locally (Jekyll not installed locally).
- Serve via `python3 -m http.server` and check both color schemes with the browse tool (emulate `prefers-color-scheme`), desktop + mobile widths.
- Push to `master`; GitHub Pages builds themeless sites natively. Force build if needed: `gh api -X POST repos/ataberk-xyz/ataberk-xyz.github.io/pages/builds`.
- Contrast: body text and accents must pass WCAG AA on both palettes (amber-on-parchment is the known risk — darken for light mode).

## Out of scope (YAGNI)

Search, comments, RSS restyling, About-page content changes, per-post custom cover images, reading-progress bars.
