# The Ledger Theme v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reskin the Jekyll blog (`ataberk-xyz.github.io`) from the restrained kami look into "The Ledger" — a dark-warm editorial security-journal theme with generative SVG covers, per spec `docs/superpowers/specs/2026-08-02-ledger-theme-design.md`.

**Architecture:** Token-level palette swap (two palettes via `prefers-color-scheme`, manual toggle removed), new Liquid includes for post numbering / generative covers / ledger metadata block, magazine layouts for index and post pages. All styling stays in `_sass/kami/*`; all templates stay in `_layouts` + `_includes`. No new dependencies, no webfonts, no build step beyond what GitHub Pages already does.

**Tech Stack:** Jekyll (GitHub Pages, no local install), Liquid, SCSS compiled locally for verification with dart-sass (`sass` CLI), git.

## Global Constraints

- Work in repo `~/Desktop/projects/saiyajin-blog` on branch `master`. Commit per task; **push only in Task 9**.
- Palette values verbatim from spec — dark: bg `#181512`, sunken `#100e0c`, text `#EDE6DA`, accent `#9DB8E8`, amber `#E8B44F`, hairline `#2e2a25`, muted `#8a8078`/`#b5aca1`. Light: keep existing parchment tokens, accent `#1B365D`, amber (light) `#9A6716`.
- No webfonts. Serif stays `Charter` stack, mono stays existing `--mono` stack.
- No JavaScript may remain for theme switching. `<meta name="color-scheme" content="light dark">` stays.
- Jekyll is NOT installed locally. Local verification = SCSS compile with dart-sass + grep assertions. Full rendering is verified live in Task 9 (GitHub Pages build).
- SCSS compile check (used in many steps, referred to as **COMPILE CHECK**):
  ```bash
  cd ~/Desktop/projects/saiyajin-blog && tail -n +5 assets/css/main.scss > /tmp/ledger-main.scss && sass --no-source-map --load-path=_sass /tmp/ledger-main.scss /tmp/ledger-main.css && echo COMPILE-OK
  ```
  Expected output ends with `COMPILE-OK`. (`tail -n +5` strips the 4-line Jekyll front matter; if `sass` is missing, `npm i -g sass`.)
- Liquid on GitHub Pages is v4: `{% break %}`, `at_least`, string `slice` with negative index are available; **`md5`/`sha1` filters are NOT** — never use them.

---

### Task 1: Two-palette tokens via `prefers-color-scheme`

**Files:**
- Modify: `_sass/kami/_tokens.scss`

**Interfaces:**
- Produces: CSS custom properties consumed by all later tasks — notably `--accent-warn` (new), plus existing `--bg`, `--surface`, `--ink*`, `--border*`, `--accent*`, `--code-*`, `--serif`, `--mono`, `--measure`, `--page`, `--radius`, `--rule`, and SCSS vars `$bp-tablet`, `$bp-mobile` (all keep their names).

- [ ] **Step 1: Rewrite `_tokens.scss`**

Replace the entire file content with:

```scss
// ─────────────────────────────────────────────────────────────
// Ledger design tokens — one skeleton, two palettes.
// Light: parchment Ledger (brand bridge to CV/profile).
// Dark: warm-soot editorial journal. prefers-color-scheme picks.
// ─────────────────────────────────────────────────────────────

:root {
  // Foundation — parchment Ledger
  --bg:            #FAF7F2;  // warm parchment canvas
  --surface:       #FFFEFB;  // cards, recessed panels
  --surface-2:     #F4EFE5;  // inline code, subtle fills
  --ink:           #1A1916;  // primary text, headings
  --ink-2:         #4A4640;  // secondary text
  --ink-3:         #6B6862;  // meta, timestamps
  --ink-4:         #8A857C;  // hairline glyphs, non-text only
  --border:        #E8E1D6;  // hairline borders, dividers
  --border-strong: #D7CEBE;  // stronger rules

  --accent:        #1B365D;  // ink-blue
  --accent-2:      #2C4F7C;  // hover / lighter ink-blue
  --accent-soft:   #E6EAF1;  // pill backgrounds, washes
  --accent-warn:   #9A6716;  // amber, AA on parchment (severity, warnings)

  // Sunken surfaces (code, ledger block, covers) — dark in BOTH palettes
  --code-bg:       #17171C;
  --code-fg:       #E7E2D6;
  --code-border:   #2A2A31;
  --code-inline-fg: #5A3E2B;  // inline `code` text

  // Type
  --serif: "Charter", "Bitstream Charter", "Cambria", "Georgia", "Times New Roman", serif;
  --mono:  "SF Mono", "SFMono-Regular", ui-monospace, "JetBrains Mono", "Menlo", "Consolas", monospace;

  // Rhythm
  --measure: 54rem;   // prose line length
  --page:    64rem;   // content column (wide elements: tables, code)
  --radius:  6px;
  --rule:    1px solid var(--border);
}

// ── Dark palette — warm soot, lifted ink-blue, amber severity ──
@media (prefers-color-scheme: dark) {
  :root {
    --bg:            #181512;  // warm soot
    --surface:       #211d19;  // cards, blockquote, recessed panels
    --surface-2:     #2a2520;  // inline code, subtle fills
    --ink:           #EDE6DA;  // primary text, headings
    --ink-2:         #b5aca1;  // secondary text
    --ink-3:         #8a8078;  // meta, timestamps
    --ink-4:         #6a625a;  // hairline glyphs
    --border:        #2e2a25;  // hairline borders
    --border-strong: #46423A;  // stronger rules

    --accent:        #9DB8E8;  // lifted ink-blue
    --accent-2:      #BBD0F0;  // hover
    --accent-soft:   rgba(157, 184, 232, 0.16);
    --accent-warn:   #E8B44F;  // amber on dark

    --code-bg:       #100e0c;  // sunken, a touch darker than --bg
    --code-fg:       #E7E2D6;
    --code-border:   #2e2a25;
    --code-inline-fg:#D8A87E;  // warm tan for inline code on dark
  }
}

// SCSS breakpoints
$bp-tablet: 820px;
$bp-mobile: 540px;
```

- [ ] **Step 2: Verify no `data-theme` selector remains in tokens**

Run: `grep -c 'data-theme' _sass/kami/_tokens.scss || true`
Expected: `0`

- [ ] **Step 3: COMPILE CHECK** (command in Global Constraints)

Expected: `COMPILE-OK`

- [ ] **Step 4: Commit**

```bash
git add _sass/kami/_tokens.scss && git commit -m "theme: two-palette Ledger tokens via prefers-color-scheme"
```

---

### Task 2: Remove the manual theme toggle

**Files:**
- Delete: `_includes/theme-toggle.html`
- Modify: `_layouts/default.html` (line 8), `_includes/head.html` (lines 7–18), `_sass/kami/_base.scss` (lines 25–54 region)

**Interfaces:**
- Consumes: nothing. Produces: a site with zero `data-theme` / toggle references (later greps rely on this).

- [ ] **Step 1: Delete the include and its reference**

```bash
git rm _includes/theme-toggle.html
```
In `_layouts/default.html`, delete the line `    {% include theme-toggle.html %}`.

- [ ] **Step 2: Remove the no-FOUC script from `head.html`**

Delete this whole block (keep the `color-scheme` meta above it):

```html
  <!-- Set theme before paint to avoid a flash of the wrong mode -->
  <script>
    (function () {
      try {
        var t = localStorage.getItem('theme');
        if (t !== 'light' && t !== 'dark') {
          t = (window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches) ? 'dark' : 'light';
        }
        document.documentElement.setAttribute('data-theme', t);
      } catch (e) {}
    })();
  </script>
```

- [ ] **Step 3: Prune toggle styles from `_base.scss`**

Delete the `.theme-toggle { … }` rule block and every following rule whose selector mentions `.theme-toggle` or `html[data-theme="dark"]` (lines ~25–54), including the mobile override `.theme-toggle { top: 0.6rem; … }` inside the media query (delete the whole media query only if the toggle rule is its sole content).

- [ ] **Step 4: Verify zero references site-wide**

Run: `grep -rn 'theme-toggle\|data-theme' _sass _includes _layouts | wc -l`
Expected: `0`

- [ ] **Step 5: COMPILE CHECK**, then commit

```bash
git add -A && git commit -m "theme: remove manual toggle; prefers-color-scheme only"
```

---

### Task 3: `post-number.html` include (auto N° numbering)

**Files:**
- Create: `_includes/post-number.html`

**Interfaces:**
- Produces: `{% include post-number.html post=<post object> %}` → emits exactly `N°07` (zero-padded, chronological: oldest post = `N°01`). Consumed by Tasks 5 and 6, always via `{% capture %}`.

- [ ] **Step 1: Create the include**

```liquid
{%- comment -%}
  Chronological issue number: oldest post = N°01.
  site.posts is newest-first, so number = total - index0.
{%- endcomment -%}
{%- assign pn_total = site.posts | size -%}
{%- assign pn_num = 0 -%}
{%- for pn_p in site.posts -%}
  {%- if pn_p.url == include.post.url -%}
    {%- assign pn_num = pn_total | minus: forloop.index0 -%}
    {%- break -%}
  {%- endif -%}
{%- endfor -%}
{%- assign pn_pad = pn_num | prepend: '00' | slice: -2, 2 -%}
N°{{ pn_pad }}
```

- [ ] **Step 2: Sanity-check the Liquid by eye**

There is no local Jekyll; confirm: whitespace-trimmed tags everywhere (`{%- -%}`) so the capture contains only `N°NN`; loop breaks on first URL match. (Real render is verified in Task 9.)

- [ ] **Step 3: Commit**

```bash
git add _includes/post-number.html && git commit -m "theme: add auto chronological post numbering include"
```

---

### Task 4: Generative cover include

**Files:**
- Create: `_includes/cover.html`
- Modify: `_sass/kami/_components.scss` (append cover styles)

**Interfaces:**
- Produces: `{% include cover.html post=<post object> %}` → inline `<svg class="cover">`, deterministic per slug, motif by first category: `ai-research`/`projects` → node graph; `vulnerability-research`/`pentest` → topographic contours; everything else (incl. `writeup`) → circuit traces. Colors via CSS vars, adapt to both palettes. Consumed by Tasks 5 and 8.

- [ ] **Step 1: Create `_includes/cover.html`**

```liquid
{%- comment -%}
  Generative cover art. Pure Liquid, no JS, no md5 filter (not on Pages).
  Seed: position-weighted char values of the slug. Stream: LCG
  (seed = seed*1103 + 12345 mod 100003), one digit (mod 10) per draw,
  collected into string cv_ds → array cv_d of digit chars.
{%- endcomment -%}
{%- assign cv_post = include.post -%}
{%- assign cv_charset = "abcdefghijklmnopqrstuvwxyz0123456789-" | split: "" -%}
{%- assign cv_letters = cv_post.slug | split: "" -%}
{%- assign cv_seed = 7 -%}
{%- for cv_c in cv_letters -%}
  {%- assign cv_val = 1 -%}
  {%- for cv_a in cv_charset -%}
    {%- if cv_a == cv_c -%}{%- assign cv_val = forloop.index -%}{%- break -%}{%- endif -%}
  {%- endfor -%}
  {%- assign cv_seed = cv_seed | times: 31 | plus: cv_val | modulo: 100003 -%}
{%- endfor -%}
{%- assign cv_ds = "" -%}
{%- for cv_i in (1..16) -%}
  {%- assign cv_seed = cv_seed | times: 1103 | plus: 12345 | modulo: 100003 -%}
  {%- assign cv_dig = cv_seed | modulo: 10 -%}
  {%- assign cv_ds = cv_ds | append: cv_dig -%}
{%- endfor -%}
{%- assign cv_d = cv_ds | split: "" -%}
{%- assign cv_cat = cv_post.categories | first | default: "writeup" -%}
<svg class="cover" viewBox="0 0 120 120" role="img" aria-label="Generative cover art for {{ cv_post.title | escape }}" preserveAspectRatio="xMidYMid slice">
  <rect width="120" height="120" class="cover-bg"/>
{%- if cv_cat == "ai-research" or cv_cat == "projects" -%}
  {%- comment -%} Node graph: 5 chained nodes {%- endcomment -%}
  {%- assign cv_px = 0 -%}{%- assign cv_py = 0 -%}
  {%- for cv_i in (0..4) -%}
    {%- assign cv_xi = cv_i | times: 2 -%}
    {%- assign cv_yi = cv_xi | plus: 1 -%}
    {%- assign cv_x = cv_d[cv_xi] | plus: 0 | times: 10 | plus: 12 -%}
    {%- assign cv_y = cv_d[cv_yi] | plus: 0 | times: 10 | plus: 12 -%}
    {%- unless forloop.first -%}
  <line x1="{{ cv_px }}" y1="{{ cv_py }}" x2="{{ cv_x }}" y2="{{ cv_y }}" class="cover-line"/>
    {%- endunless -%}
  <circle cx="{{ cv_x }}" cy="{{ cv_y }}" r="3" class="cover-node{% if forloop.index == 3 %} cover-node-hot{% endif %}"/>
    {%- assign cv_px = cv_x -%}{%- assign cv_py = cv_y -%}
  {%- endfor -%}
{%- elsif cv_cat == "vulnerability-research" or cv_cat == "pentest" -%}
  {%- comment -%} Topographic contours: 4 stacked curves {%- endcomment -%}
  {%- for cv_i in (0..3) -%}
    {%- assign cv_o = cv_i | times: 3 -%}
    {%- assign cv_o1 = cv_o | plus: 1 -%}
    {%- assign cv_o2 = cv_o | plus: 2 -%}
    {%- assign cv_y1 = cv_i | times: 24 | plus: 16 | plus: cv_d[cv_o] -%}
    {%- assign cv_c1 = cv_d[cv_o1] | plus: 0 | times: 8 | plus: 10 -%}
    {%- assign cv_y2 = cv_i | times: 24 | plus: 12 | plus: cv_d[cv_o2] -%}
  <path d="M8 {{ cv_y1 }} Q 45 {{ cv_c1 }}, 75 {{ cv_y2 }} T 112 {{ cv_y1 }}" class="cover-line"/>
  {%- endfor -%}
  {%- assign cv_hx = cv_d[13] | plus: 0 | times: 9 | plus: 12 -%}
  {%- assign cv_hy = cv_d[14] | plus: 0 | times: 9 | plus: 12 -%}
  <circle cx="{{ cv_hx }}" cy="{{ cv_hy }}" r="2.5" class="cover-node-hot"/>
{%- else -%}
  {%- comment -%} Circuit traces: 3 orthogonal runs, node endpoints {%- endcomment -%}
  {%- for cv_i in (0..2) -%}
    {%- assign cv_o = cv_i | times: 5 -%}
    {%- assign cv_o1 = cv_o | plus: 1 -%}
    {%- assign cv_o2 = cv_o | plus: 2 -%}
    {%- assign cv_o3 = cv_o | plus: 3 -%}
    {%- assign cv_o4 = cv_o | plus: 4 -%}
    {%- assign cv_x1 = cv_d[cv_o] | plus: 0 | times: 9 | plus: 8 -%}
    {%- assign cv_y1 = cv_d[cv_o1] | plus: 0 | times: 9 | plus: 8 -%}
    {%- assign cv_xm = cv_d[cv_o2] | plus: 0 | times: 9 | plus: 8 -%}
    {%- assign cv_y2 = cv_d[cv_o3] | plus: 0 | times: 9 | plus: 8 -%}
    {%- assign cv_x2 = cv_d[cv_o4] | plus: 0 | times: 9 | plus: 8 -%}
  <path d="M{{ cv_x1 }} {{ cv_y1 }} H{{ cv_xm }} V{{ cv_y2 }} H{{ cv_x2 }}" class="cover-line"/>
  <circle cx="{{ cv_x1 }}" cy="{{ cv_y1 }}" r="2.5" class="cover-node"/>
  <circle cx="{{ cv_x2 }}" cy="{{ cv_y2 }}" r="2.5" class="cover-node{% if forloop.index == 2 %}-hot{% endif %}"/>
  {%- endfor -%}
{%- endif -%}
</svg>
```

Note the `cv_y1` computation in the topo branch: `cv_d[cv_o]` is a digit **string**; `plus`-coercion happens because the previous filter output is a number — to be safe it is written as `| plus: cv_d[cv_o]` only where the left side is already numeric. If in doubt, normalize with `| plus: 0` first (the node/circuit branches already do).

- [ ] **Step 2: Append cover styles to `_sass/kami/_components.scss`**

```scss
// ── Generative covers ─────────────────────────────────────────
.cover {
  display: block;
  width: 100%;
  height: auto;
  border: 1px solid var(--code-border);
  border-radius: var(--radius);
}
.cover-bg       { fill: var(--code-bg); }
.cover-line     { stroke: var(--accent); stroke-width: 1.5; fill: none; opacity: .85; }
.cover-node     { fill: var(--accent); }
.cover-node-hot { fill: var(--accent-warn); }
```

- [ ] **Step 3: COMPILE CHECK**

Expected: `COMPILE-OK`

- [ ] **Step 4: Commit**

```bash
git add _includes/cover.html _sass/kami/_components.scss && git commit -m "theme: generative SVG cover include (LCG-seeded, motif per category)"
```

---

### Task 5: Magazine homepage

**Files:**
- Modify: `_layouts/index.html` (full rewrite), `_config.yml` (`paginate: 4` → `paginate: 7`)
- Modify: `_sass/kami/_layout.scss` (append home styles)

**Interfaces:**
- Consumes: `post-number.html` (Task 3), `cover.html` (Task 4), existing `pagination.html`.
- Produces: classes `.ledger-home`, `.ledger-featured`, `.lf-*`, `.ledger-grid`, `.ledger-card`, `.lc-*`, `.no-line`, `.sev-warn`, `.rule-strong` (Task 8 reuses `.ledger-grid`/`.ledger-card`/`.no-line`).

- [ ] **Step 1: Rewrite `_layouts/index.html`**

```liquid
---
layout: default
---

<div class="ledger-home">
  {{ content }}

  {% for post in paginator.posts %}
    {% capture ponum %}{% include post-number.html post=post %}{% endcapture %}
    {% assign pcat = post.categories | first | default: "writeup" %}
    {% if forloop.first and paginator.page == 1 %}
  <article class="ledger-featured">
    <div class="lf-text">
      <p class="no-line">{{ ponum }} · {{ pcat | upcase }} · {{ post.date | date: "%Y-%m-%d" }}</p>
      <h2 class="lf-title"><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h2>
      {% if post.description %}<p class="lf-dek">{{ post.description }}</p>{% endif %}
      {% assign lf_words = post.content | strip_html | number_of_words %}
      <p class="lf-meta">{{ lf_words | divided_by: 200 | at_least: 1 }} MIN READ{% if post.ledger.severity %} — SEVERITY <span class="sev-warn">{{ post.ledger.severity }}</span>{% endif %}</p>
    </div>
    <a class="lf-cover" href="{{ site.baseurl }}{{ post.url }}" aria-hidden="true" tabindex="-1">{% include cover.html post=post %}</a>
  </article>
  <hr class="rule-strong"/>
  <div class="ledger-grid">
    {% else %}
      {% if forloop.first %}<div class="ledger-grid">{% endif %}
    <article class="ledger-card">
      <a class="lc-cover" href="{{ site.baseurl }}{{ post.url }}" aria-hidden="true" tabindex="-1">{% include cover.html post=post %}</a>
      <p class="no-line">{{ ponum }} · {{ pcat | upcase }}</p>
      <h3 class="lc-title"><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h3>
    </article>
    {% endif %}
    {% if forloop.last %}</div>{% endif %}
  {% endfor %}

  {% include pagination.html %}
</div>
```

- [ ] **Step 2: Bump pagination in `_config.yml`**

Change `paginate: 4` to `paginate: 7` (featured + 6 grid cards on page 1).

- [ ] **Step 3: Append home styles to `_sass/kami/_layout.scss`**

```scss
// ── Ledger home ───────────────────────────────────────────────
.no-line {
  font-family: var(--mono);
  font-size: 0.72rem;
  letter-spacing: 0.14em;
  color: var(--accent);
  margin: 0 0 0.4rem;
}
.sev-warn { color: var(--accent-warn); }

.ledger-featured {
  display: grid;
  grid-template-columns: 1.5fr 1fr;
  gap: 2rem;
  align-items: start;
  margin: 1.5rem 0 2rem;

  .lf-title {
    font-size: 2.1rem;
    line-height: 1.12;
    margin: 0.2rem 0 0;
    a { color: var(--ink); text-decoration: none; }
    a:hover { color: var(--accent); }
  }
  .lf-dek {
    font-style: italic;
    color: var(--ink-2);
    font-size: 1.05rem;
    margin: 0.6rem 0 0;
  }
  .lf-meta {
    font-family: var(--mono);
    font-size: 0.7rem;
    letter-spacing: 0.1em;
    color: var(--ink-3);
    margin-top: 0.9rem;
  }
}
.rule-strong {
  border: 0;
  border-top: 1px solid var(--border-strong);
  margin: 0 0 1.8rem;
}
.ledger-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.8rem 2rem;
  margin-bottom: 2rem;

  .lc-title {
    font-size: 1.1rem;
    line-height: 1.25;
    margin: 0.2rem 0 0;
    a { color: var(--ink); text-decoration: none; }
    a:hover { color: var(--accent); }
  }
  .lc-cover { display: block; margin-bottom: 0.6rem; }
}
@media (max-width: $bp-tablet) {
  .ledger-featured { grid-template-columns: 1fr; }
}
@media (max-width: $bp-mobile) {
  .ledger-grid { grid-template-columns: 1fr; }
  .ledger-featured .lf-title { font-size: 1.6rem; }
}
```

- [ ] **Step 4: COMPILE CHECK + structural grep**

Run: `grep -c 'ledger-grid' _layouts/index.html`
Expected: `2` (open + conditional open). COMPILE CHECK expected: `COMPILE-OK`.

- [ ] **Step 5: Commit**

```bash
git add _layouts/index.html _config.yml _sass/kami/_layout.scss && git commit -m "theme: magazine homepage — featured post + cover grid"
```

---

### Task 6: Post page — N° line, dek, ledger block, drop cap, pull quotes

**Files:**
- Create: `_includes/ledger.html`
- Modify: `_layouts/post.html` (header section), `_sass/kami/_post.scss` (append styles)

**Interfaces:**
- Consumes: `post-number.html` (Task 3).
- Produces: `{% include ledger.html post=page %}` — renders `<dl class="ledger-block">` ONLY if `page.ledger` exists. Front-matter contract (all fields optional):
  ```yaml
  ledger:
    target: node-re2
    vector: CWE-835 loop
    impact: event-loop DoS
    severity: MODERATE 6.2
    advisory: CVE-2026-68499
    status: PATCHED
  ```

- [ ] **Step 1: Create `_includes/ledger.html`**

```liquid
{%- assign L = include.post.ledger -%}
{%- if L -%}
<dl class="ledger-block">
  {% if L.target %}<div class="lrow"><dt>TARGET</dt><span class="dots"></span><dd>{{ L.target }}</dd></div>{% endif %}
  {% if L.severity %}<div class="lrow"><dt>SEVERITY</dt><span class="dots"></span><dd class="sev">{{ L.severity }}</dd></div>{% endif %}
  {% if L.vector %}<div class="lrow"><dt>VECTOR</dt><span class="dots"></span><dd>{{ L.vector }}</dd></div>{% endif %}
  {% if L.advisory %}<div class="lrow"><dt>ADVISORY</dt><span class="dots"></span><dd>{{ L.advisory }}</dd></div>{% endif %}
  {% if L.impact %}<div class="lrow"><dt>IMPACT</dt><span class="dots"></span><dd>{{ L.impact }}</dd></div>{% endif %}
  {% if L.status %}<div class="lrow"><dt>STATUS</dt><span class="dots"></span><dd>{{ L.status }}</dd></div>{% endif %}
</dl>
{%- endif -%}
```

- [ ] **Step 2: Rewrite the header of `_layouts/post.html`**

Replace the current `<header class="post-header">…</header>` block with:

```liquid
  <header class="post-header">
    {% capture ponum %}{% include post-number.html post=page %}{% endcapture %}
    {% assign pcat = page.categories | first | default: "writeup" %}
    {% assign p_words = content | strip_html | number_of_words %}
    <p class="no-line">{{ ponum }} · {{ pcat | upcase }} · {{ page.date | date: "%Y-%m-%d" }} · {{ p_words | divided_by: 200 | at_least: 1 }} MIN</p>
    <h1 class="post-title">{{ page.title }}</h1>
    {% if page.description %}<p class="post-dek">{{ page.description }}</p>{% endif %}
    {% assign author = site.data.authors[page.author] %}
    {% if author %}
      <p class="post-byline">Written by <a href="{{ author.web }}" target="_blank" rel="noopener">{{ author.name }}</a></p>
    {% endif %}
    {% include ledger.html post=page %}
  </header>
```

(The old `{% include post-meta.html post=page %}` line is removed from this layout — the N° line replaces it. `post-meta.html` itself stays; the category layout still uses it until Task 8.)

- [ ] **Step 3: Append post styles to `_sass/kami/_post.scss`**

```scss
// ── Ledger post header ────────────────────────────────────────
.post-dek {
  font-style: italic;
  color: var(--ink-2);
  font-size: 1.15rem;
  line-height: 1.5;
  margin: 0.6rem 0 0;
}

// ── Künye (ledger) block ──────────────────────────────────────
.ledger-block {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.15rem 1.6rem;
  background: var(--code-bg);
  border: 1px solid var(--code-border);
  border-radius: var(--radius);
  padding: 0.9rem 1.1rem;
  font-family: var(--mono);
  font-size: 0.74rem;
  color: var(--code-fg);
  margin: 1.4rem 0 0;

  .lrow { display: flex; align-items: baseline; gap: 0.5rem; }
  dt    { color: var(--accent); letter-spacing: 0.08em; }
  dd    { margin: 0; }
  .dots { flex: 1; border-bottom: 1px dotted var(--code-border); transform: translateY(-3px); }
  // amber on the always-dark code surface — same hex in both palettes
  .sev  { color: #E8B44F; }
}
// dark mode: --accent is already the lifted blue; block inherits correctly.

// ── Editorial body: drop cap + pull quotes ────────────────────
.post-body > p:first-of-type::first-letter {
  float: left;
  font-size: 3.1em;
  line-height: 0.85;
  padding: 0.04em 0.09em 0 0;
  font-weight: 700;
  color: var(--accent);
}
.post-body blockquote {
  border-left: 3px solid var(--accent);
  font-style: italic;
  font-size: 1.08em;
}
@media (max-width: $bp-mobile) {
  .ledger-block { grid-template-columns: 1fr; }
}
```

- [ ] **Step 4: COMPILE CHECK + greps**

Run: `grep -c 'ledger.html' _layouts/post.html` → Expected `1`.
Run: `grep -c 'post-meta' _layouts/post.html` → Expected `0`.
COMPILE CHECK → `COMPILE-OK`.

- [ ] **Step 5: Commit**

```bash
git add _includes/ledger.html _layouts/post.html _sass/kami/_post.scss && git commit -m "theme: Ledger post header — no-line, dek, kunye block, drop cap"
```

---

### Task 7: Masthead & typographic scale

**Files:**
- Modify: `_includes/masthead.html`, `_sass/kami/_layout.scss` (masthead rules)

**Interfaces:**
- Consumes: existing `nav-links.html`. Produces: `.brand-issue` class replacing `.brand-tagline`.

- [ ] **Step 1: Rewrite `_includes/masthead.html`**

```liquid
<header class="masthead">
  <div class="masthead-inner">
    <a class="brand" href="{{ site.baseurl }}/">{{ site.title }}</a>
    <p class="brand-issue">SECURITY RESEARCH · EXPLOITS · TOOLING — {{ site.tagline }}</p>
    {% include nav-links.html %}
  </div>
</header>
```

- [ ] **Step 2: Update masthead styles in `_sass/kami/_layout.scss`**

Find the existing `.masthead` block. Set/replace so the result includes:

```scss
.masthead {
  border-bottom: 3px double var(--accent);
}
.brand {
  font-family: var(--serif);
  font-weight: 700;
  font-size: 1.5rem;
  letter-spacing: 0.02em;
}
.brand-issue {
  font-family: var(--mono);
  font-size: 0.62rem;
  letter-spacing: 0.22em;
  color: var(--ink-3);
  margin: 0.15rem 0 0;
}
```

Delete any `.brand-tagline` rule (grep: `grep -n 'brand-tagline' _sass/kami/_layout.scss` → fix all hits). Keep existing masthead spacing/inner-layout rules that don't conflict.

- [ ] **Step 3: COMPILE CHECK + grep**

Run: `grep -rn 'brand-tagline' _sass _includes _layouts | wc -l` → Expected `0`.
COMPILE CHECK → `COMPILE-OK`.

- [ ] **Step 4: Commit**

```bash
git add _includes/masthead.html _sass/kami/_layout.scss && git commit -m "theme: Ledger masthead — double rule + mono issue line"
```

---

### Task 8: Category layout on the grid system

**Files:**
- Modify: `_layouts/category.html`

**Interfaces:**
- Consumes: `.ledger-grid`/`.ledger-card`/`.no-line` (Task 5), `cover.html` (Task 4), `post-number.html` (Task 3).

- [ ] **Step 1: Rewrite the post-listing loop in `_layouts/category.html`**

Keep the layout's existing front matter and heading; replace its post loop with:

```liquid
<div class="ledger-grid">
  {% for post in include_posts %}
    {% capture ponum %}{% include post-number.html post=post %}{% endcapture %}
    <article class="ledger-card">
      <a class="lc-cover" href="{{ site.baseurl }}{{ post.url }}" aria-hidden="true" tabindex="-1">{% include cover.html post=post %}</a>
      <p class="no-line">{{ ponum }} · {{ post.date | date: "%Y-%m-%d" }}</p>
      <h3 class="lc-title"><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h3>
    </article>
  {% endfor %}
</div>
```

**Note:** first read the current `_layouts/category.html` to find the actual loop variable (it may be `site.categories[...]`, `page.posts`, or similar) and substitute it for `include_posts` — keep the existing data source, change only the markup.

- [ ] **Step 2: Grep check**

Run: `grep -c 'ledger-card' _layouts/category.html` → Expected `1`.

- [ ] **Step 3: Commit**

```bash
git add _layouts/category.html && git commit -m "theme: category pages on the Ledger grid"
```

---

### Task 9: Deploy & live verification (both palettes)

**Files:**
- None new — push + fix cycle.

- [ ] **Step 1: Push**

```bash
cd ~/Desktop/projects/saiyajin-blog && git push origin master
gh api -X POST repos/ataberk-xyz/ataberk-xyz.github.io/pages/builds
```

- [ ] **Step 2: Wait for the build, confirm it succeeded**

```bash
sleep 90 && gh api repos/ataberk-xyz/ataberk-xyz.github.io/pages/builds/latest --jq '.status,.error.message'
```
Expected: `built` and `null`. If `errored`, the message names the failing Liquid — fix, commit, re-push.

- [ ] **Step 3: Browse-tool verification matrix**

Using the browse skill against `https://ataberk-xyz.github.io/`, check each cell:

| Page | Check |
|---|---|
| Home | featured post + N° line + generative cover renders; grid below; pagination works |
| Home (page 2 via pagination) | all-cards grid, no featured |
| gossipcat post | N° line, dek (if `description` present), drop cap, blockquote style, NO ledger block (no `ledger:` front matter yet) |
| Hacker's Manifesto post | renders bare-but-clean: N° line + title, no dek, no ledger block |
| A category page | grid of cards with covers |

Run the matrix twice: emulate `prefers-color-scheme: dark` and `light`. Also check Home at ~390px width (mobile: single column, featured stacks).

- [ ] **Step 4: Contrast spot-check**

In dark mode verify body text `#EDE6DA` on `#181512` and amber `#E8B44F` usages; in light mode verify `#9A6716` amber on parchment. All must pass WCAG AA for their text size (the browse tool screenshot + a contrast calculator, or manual ratio math: AA normal text ≥ 4.5:1).

- [ ] **Step 5: Fix-forward loop**

Any visual defect found: fix locally, COMPILE CHECK, commit with `theme-fix:` prefix, push, re-verify. Repeat until the matrix is clean.

- [ ] **Step 6: Final commit of any stragglers + done**

```bash
git status --short   # expect clean
```

---

## Self-review notes (already applied)

- Spec §5 said "JS fallback allowed if pure Liquid proves insufficient" — pure Liquid suffices (LCG above); no JS shipped.
- Spec §7 removals: toggle include/script/styles all covered in Task 2; `.brand-tagline` prune in Task 7.
- `md5` filter is NOT available in Jekyll on GitHub Pages — the LCG seed replaces the spec's "hash" wording; deterministic property preserved.
- Type consistency: `ponum` capture pattern identical in Tasks 5/6/8; `.no-line`, `.sev-warn`, `.ledger-grid` defined once in Task 5 and only consumed later.
```
