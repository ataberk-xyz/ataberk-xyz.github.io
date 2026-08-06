# AI Orchestration Series, Episode 1 — "Four bugs in my own dependency tree"

**Date:** 2026-08-05
**Status:** Approved (structure A + key-learnings emphasis)
**Deliverable:** one English post in `_posts/`, category `ai-research`, with `summary:` front matter and a `ledger:` block.

## Goal

Tell how a question asked while building gossipcat — *can I point it at my own dependencies?* —
produced four accepted advisories in two packages that ship ~12M downloads a week. The post is
**Episode 1** of the **AI Orchestration Series**: it shows the orchestrator/operator division of labour
on a real target, not a toy.

Reader: an engineer who has heard "AI finds bugs" and is sceptical. The post has to survive that
scepticism — every claim traceable to a published advisory, a code path, or a measurement.

## Structure (A — finding bulletin)

1. **Opening (~200 words).** The question, and the chain that made it answerable:
   `gossipcat → re2 (^1.25.0) → install-artifact-from-github`, verified in the repo's own
   `node_modules`. State the outcome up front: four findings, two packages, ~12.1M weekly downloads
   (re2 3.31M + stream-json 8.81M, npm, as of 2026-08-05 — re-check at publish).
2. **Four finding blocks**, same shape each, ~200–250 words:
   *what broke · how it was found · why it survived this long*.
3. **Key learnings** — the spine the user asked to emphasise. Woven as a short callout at the end of
   each finding block, then gathered in a closing section.
4. **Closing (~250 words).** What the orchestrator actually contributed: independent reproduction
   and killing false positives. Honest scoreboard: all Medium, DoS-class, no cash — CVE numbers and
   maintainer credit.

## The four findings (facts to use — all from the disclosure records)

| # | Package | ID | Class | Core fact |
|---|---|---|---|---|
| 1 | re2 | CVE-2026-68499 (GHSA-6hxr-mr5r-9836) | CWE-835, Medium 6.2 | Global branch of `WrappedRE2::Match` (`lib/match.cc:44`) advances by `match.size()`; a zero-width match never advances → infinite loop, ~550MB→2.3GB in ~3s, uncatchable. `split.cc:50-55` and `exec` already guard it. Fixed 1.25.2. |
| 2 | re2 | CVE-2026-67550 (GHSA-ff84-5f28-78qj) | CWE-125, Medium 5.7 | `lastIndex` validated against UTF-8 **byte** length (`addon.cc:229`) but walked by UTF-16 **char count** (`getUtf16PositionByCounter`, `wrapped_re2.h:264`, no bounds check). Non-ASCII subject → OOB heap read, uncatchable crash, bounded info-leak. Fixed 1.25.2. |
| 3 | re2 | CVE-2026-71430 (GHSA-8hcv-x26h-mcgp) | CWE-617, Medium 6.2 | `WrappedRE2::Replace` calls `.ToLocalChecked()` without checking the empty `MaybeLocal` V8 returns past `String::kMaxLength` → `FATAL ERROR` → SIGABRT, uncatchable. PoC: `"a".repeat(50000).replace(new RE2("a","g"), "$'")`; 30k ok / 40k crash matches N²/2 > 536M. Native `/a/g` throws a catchable `RangeError` — that contrast is what makes it a defect. |
| 4 | stream-json | CVE-2026-71429 (GHSA-528h-pc64-c93x) | CWE-407, Medium 6.2 | `pick`/`ignore`/`filter`/`replace` recompute `stack.join(separator)` per token (`filter-base.js:29,37`, called at `:194`) → O(depth²). ~360KB structural payload blocks the event loop ~12s. Measured 5k→40k depth: 160ms → 603 → 2511 → 11823ms (~4× per 2× depth). Reported 7.5, maintainer published Medium 6.2. |

## Key learnings (the emphasised spine)

- **Enumerate the siblings.** When a structure is walked in N places, diff the advance/termination
  logic across all of them; the path that doesn't share its siblings' guard is the bug (finding 1).
- **Ask what unit a length is in.** Validators and consumers that measure different quantities —
  bytes vs chars, UTF-8 vs UTF-16 — are a bug class, not a typo (finding 2).
- **Fuzzing hygiene, learned the hard way** (finding 3): seed the PRNG from a fixed value, never
  `process.pid`, or crashes don't replay; log the full untruncated input — this crash was
  input-*size* dependent (89KB) and a 400-char log made replay fail; minimise deterministically
  before believing yourself — emoji, named groups and the sticky flag were all red herrings.
- **Audit the marshalling layer, not the engine.** re2 is the library people install *to avoid*
  ReDoS, so its hand-written C++ N-API layer is chronically unaudited. The wrapper reintroduced the
  very class the engine prevents.
- **The orchestrator's real job is refutation.** On stream-json it killed three candidate findings —
  prototype pollution (local re-parent only, no global gadget), deep-nesting stack overflow (parser
  and assembler are iterative), unbounded string buffering (documented `packStrings:false` escape
  hatch) — and shipped only the one that reproduced.
- **Coordinated disclosure has a human-only step.** Private reporting was enabled on both repos, but
  `POST /repos/{owner}/{repo}/security-advisories` is maintainer-only (403 for external reporters):
  the agent can prepare the report, a human must paste it into the web form.

## Voice and integrity rules

- First person, Ataberk's voice; **"Ataberk" only, never the surname** (site-wide rule).
- Short declarative sentences; no marketing adjectives; no "revolutionary AI" framing.
- **No agent attribution for the re2 findings.** The record explicitly states the source session and
  agent are unrecorded and must not be fabricated. Roles are described at the level of
  *orchestrator* and *operator*, never "agent X found Y".
- Honest scoreboard: all four are Medium, DoS-class, solo-maintainer OSS, **no bounty money**.
  The win is CVE numbers and credit — say so plainly.
- Numbers that must be re-verified at publish time: weekly downloads, and whether
  install-artifact-from-github's GHSA-88q3-gch3-5396 has a CVE assigned by then.

## Out of scope

- **install-artifact-from-github (GHSA-88q3, High 7.5).** It is the strongest finding of the batch
  but the user scoped this post to re2 + stream-json; it gets its own episode in the series.
- **GHSA-j4r3-hg7j-8chg / CVE-2026-71498** (a fourth re2 advisory, published 2026-07-21). Not in the
  user's record — ownership unconfirmed, so it is not mentioned at all.
- Turkish translation. The blog has no `lang` mechanism yet; a Turkish edition of this post is a
  separate piece of work (per-post `lang:` + a language link), to be built after this post ships.

## Front matter

```yaml
layout: post
title: "Four bugs in my own dependency tree"   # working title
author: ataberk-xyz
categories: [ai-research]
tags: [orchestration, gossipcat, npm, cve, fuzzing, supply-chain]
# title carries the series marker, e.g. "Orchestration, Episode 1: …"
summary: "<one sentence: the question, the chain, the four CVEs>"
ledger:
  target: "re2 (node-re2) · stream-json — ~12.1M downloads/week"
  severity: "MEDIUM 6.2"
  vector: "CWE-835 · CWE-125 · CWE-617 · CWE-407"
  advisory: "CVE-2026-68499 · 67550 · 71430 · 71429"
  impact: "event-loop DoS, uncatchable crash, bounded heap read"
  status: "PATCHED — re2 1.25.2, stream-json 3.5.0"
  method: "dependency-tree audit → orchestrated review + fuzzing"
```

## Verification before publish

1. Re-run the npm weekly-download numbers.
2. Confirm each CVE/GHSA link resolves and the severity strings match the published advisories.
3. Check the `ledger:` block renders (it is the second post to use it; the include is live).
4. Read the draft once against this spec's integrity rules — especially the no-attribution rule.
