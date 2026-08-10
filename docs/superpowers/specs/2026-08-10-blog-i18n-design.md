# Blog TR/EN Language Option — Design

**Date:** 2026-08-10
**Status:** Approved (brainstormed with Ataberk; scope decisions below are his)

## Goal

LinkedIn brings strong traffic from Türkiye. Add a per-post Turkish/English
language option to the blog and publish a Turkish version of the latest post
("AI Orchestration, Episode 1: Four CVEs in the Tree I Was Standing On").
Old posts stay English-only; future posts get Turkish versions when written.

## Decisions (settled during brainstorming)

1. **Scope: per-post only.** Site chrome (masthead, nav, categories, About,
   ledger-block labels like TARGET/VECTOR) stays English. A language link
   appears only on posts that have a translation.
2. **Listings show one card: the English post.** Homepage, category index,
   pagination, and N° numbering are untouched. The Turkish version is reached
   via the on-post language link or by direct URL (the link Ataberk shares on
   LinkedIn).
3. **Translation style: terms EN, prose TR.** Technical terms
   (heap-buffer-overflow, event loop, zero-width match…), code blocks, tables,
   CVE/CWE identifiers, and ledger values stay in English; narrative prose,
   lesson callouts, title, and summary are fluent Turkish.
4. **No auto-redirect by browser language.** Discovery is handled by hreflang
   (search) and by sharing the TR URL directly. Visitors are never bounced.

## Architecture

### Turkish posts live in a `tr` collection

`_config.yml`:

```yaml
collections:
  tr:
    output: true
    permalink: /tr/:name:output_ext   # fallback only; real URL set per file
```

Why a collection, not `_posts` with a `lang` flag: collection documents are
not in `site.posts`, so every existing surface — homepage, categories,
`jekyll-paginate` (which cannot filter), related posts, jekyll-feed RSS,
N° numbering — keeps working with zero changes. This is what makes decision 2
free. Collections are core Jekyll and build on the legacy GitHub Pages
pipeline (Jekyll 3.x).

### File and URL convention

- TR file: `_tr/<same filename as the EN post>.md`
  e.g. `_tr/2026-08-07-four-cves-in-my-own-dependency-tree.md`
- Identical filenames are the pairing key — no ref ids to maintain.
- Each TR file sets an explicit `permalink:` mirroring the EN URL under `/tr/`:
  `/tr/ai-research/2026/08/07/four-cves-in-my-own-dependency-tree.html`
  (Explicit because Jekyll 3.x collection permalink templates don't support
  `:categories/:year/:month/:day`; one front-matter line buys exact URLs.)

### Front-matter contract for a TR document

```yaml
layout: post
lang: tr
title: "<Turkish title>"
permalink: /tr/<mirror of EN path>.html
author: ataberk-xyz
categories: [<same as EN>]
summary: "<Turkish summary>"
ledger: <same keys/values as EN — values stay English>
```

Adding a Turkish version of any future post = drop one file in `_tr/` with
the same filename, translated title/summary/body, and the permalink line.
Nothing else changes anywhere.

### Pairing lookup — `_includes/translation-link.html`

One include, used by both the post layout and head.html. Logic:

- On an EN post (`page.lang` unset): scan `site.tr` for a doc whose filename
  matches this page's filename → that's the TR URL.
- On a TR doc (`page.lang == "tr"`): scan `site.posts` the same way → EN URL.
- Filename = last segment of `path`. If no match, the include renders nothing
  (old posts get zero UI).

### Toggle UI

On posts with a translation, the meta line above the title carries plain text
links: `EN · TR`, active language emphasized (not a link), styled with the
existing meta/dotted-leader vocabulary. **No pills or chips** (standing theme
decision). Absent a translation, nothing renders.

### Post layout on TR docs

TR docs reuse `layout: post`, but two of its includes assume the page is in
`site.posts`: `post-number.html` (N° from the posts index) and
`related_posts.html`. On a TR doc the N° must come from the **EN
counterpart's** index (same issue number, both languages), and related posts
may render EN cards (acceptable — chrome is EN) or be skipped. The
implementation plan must verify both includes behave on a non-post document
and guard them where they don't.

### `<html lang>` attribute

`_layouts/default.html` line 2 is hard-coded `lang="en-us"`. Change to:

    lang="{{ page.lang | default: 'en-us' }}"

TR docs set `lang: tr`, so the page is announced correctly to search engines
and screen readers.

### SEO — hreflang

In `head.html` (via the same pairing lookup), when a counterpart exists,
both versions emit:

```html
<link rel="alternate" hreflang="en" href="<EN absolute URL>" />
<link rel="alternate" hreflang="tr" href="<TR absolute URL>" />
<link rel="alternate" hreflang="x-default" href="<EN absolute URL>" />
```

URLs are absolute via `site.url` (already correctly set — that was the root
cause of the old "everything broken" episode; do not regress it).

## Content: the translation itself

Translate `2026-08-07-four-cves-in-my-own-dependency-tree.md` per decision 3.
Keep: all code blocks, the timing table, blockquote "lesson" structure (lesson
text in Turkish), ledger block values, links. Ataberk reviews and edits the
full Turkish draft **before** it is pushed — the posts are in his voice and
the translation must be too.

## Out of scope

- Full-site i18n, translated chrome/categories/About
- TR RSS feed (jekyll-feed covers `site.posts` only — accepted)
- Language auto-detection or redirects
- Translating any older post (can be done later; the mechanism supports it)

## Testing / verification

- Jekyll is not installed locally; the established habit is: push to
  `master`, force a Pages build if needed
  (`gh api -X POST repos/ataberk-xyz/ataberk-xyz.github.io/pages/builds`),
  then verify live with the browse tool.
- Checks: TR URL renders with `lang="tr"` and Turkish content; EN post shows
  `EN · TR` toggle, TR page links back; hreflang triple present on both;
  homepage/categories/pagination show exactly one card for the post; an old
  post (no translation) shows no toggle and no hreflang; RSS unchanged.
- The translation draft is committed only after Ataberk's review.
