---
layout: post
title: "Orchestration, Episode 1: Four CVEs in the Tree I Was Standing On"
author: ataberk-xyz
categories: [ai-research]
tags: [orchestration, gossipcat, npm, cve, fuzzing, supply-chain]
ledger:
  target: "re2 (node-re2) · stream-json — ~12.1M downloads/week"
  severity: "MEDIUM 6.2"
  vector: "CWE-835 · CWE-125 · CWE-617 · CWE-407"
  advisory: "CVE-2026-68499 · 67550 · 71430 · 71429"
  impact: "event-loop DoS, uncatchable crash, bounded heap read"
  status: "PATCHED — re2 1.25.2, stream-json 3.5.0"
  method: "dependency-tree audit → orchestrated review + fuzzing"
summary: "I pointed my own multi-agent review system at my own dependency tree — re2 and stream-json, about 12.1M npm downloads a week between them. Four accepted advisories came out, and this is the honest account of how, including the parts where the process nearly threw a finding away."

---

> **Episode 1 of a series on AI orchestration aimed at real targets rather than demos.**

### The question

Most posts about AI finding vulnerabilities show you the one bug and hide the ninety-nine that were noise. So here is the scoreboard first: four findings, two packages, roughly **12.1M npm downloads a week** between them. All four accepted by the maintainer, published, patched. All four Medium, all four denial of service, and not one of them paid a cent.

The question that started it was small. I was building [gossipcat](https://github.com/gossipcat-ai/gossipcat-ai), a multi-agent code-review server, and it had only ever been pointed at code I wrote. So I pointed it at the code I *install*. The chain sat in the repo's own `node_modules`: `gossipcat → re2 (^1.25.0) → install-artifact-from-github`. re2 is the package people install specifically to **avoid** regex denial of service, which is what made it worth a look — nobody audits the safe one. The thread then followed the maintainer rather than the tree, since Eugene Lazutkin (`uhop`) also maintains `stream-json`.

One note first. For the re2 work my records do not say which session surfaced what, so I will not pretend otherwise. There are two roles in this story: the orchestrator that dispatched work and checked it, and me.

---

### The loop that never advances

A fuzzer had been running under ASAN for twenty-one minutes and I nearly killed it for being slow. It had wedged on `A*((((((((a)?)?))*)*)?)*)*` with the global flag, sitting at 2.5GB. No sanitizer report, no crash, just a process that would not finish — precisely what a badly written test looks like.

Sampling the stuck process saved it. The stack went straight to `WrappedRE2::Match` → `RE2::Match` → `DFA::Search`, in a tight loop. The reason is four lines of `match.cc`:

```cpp
while (re2->regexp.Match(str, byteIndex, str.size, anchor, &match, 1)) {
    groups.push_back(match);
    byteIndex = match.data() - str.data + match.size();  // += 0 for a zero-width match
}
```

The cursor advances by the length of the match it just found. A zero-width match has no length, so the cursor does not move, so the same match is found again. Forever, while `groups` keeps growing a `std::vector<StringPiece>`.

The pathological pattern turned out to be irrelevant. Any global RE2 that can match the empty string does it:

```js
const RE2 = require('re2');
'x'.match(new RE2('a*', 'g'));   // never returns
```

On the stock binary that climbs from 736MB to 3.7GB in about four seconds, and only an external `SIGKILL` stops it. It is a synchronous native call, so it owns the thread — `try`/`catch`, `AbortController` and JS timers all fail, because the timers never get a turn to fire. V8 handles the same pattern in 0ms.

I did not trust that memory figure, and I was right not to; "memory grew" is the easiest claim for a busy machine to fake. So the bug ran side by side with a control, `a+`, which cannot match empty. The bug climbed to 3860MB. The control sat flat at 44MB.

What makes it almost embarrassing is that node-re2 already knew. Three paths walk a subject with a cursor; `split.cc` handles the zero-width case by advancing one code point, and `exec` advances through `lastIndex`. Only the global `Match` path forgot. CVE-2026-68499, CWE-835, Medium 6.2, fixed in 1.25.2.

> **The lesson: enumerate the siblings.** When a structure is walked in several places, diff their advance and termination logic. The path that does not share its siblings' guard is the bug, and zero-length elements are the classic trigger.

---

### The bug no fuzzer was going to find

The next one the fuzzer missed, and would have missed forever. It needs `re.lastIndex` set to a specific wrong number, and a random input generator has no reason to touch that property. Reading the byte-versus-character conversion math found it in minutes.

`lastIndex` is user-settable to any positive integer. `prepareArgument` stores the subject's **UTF-8 byte** count as the length, then `setIndex` validates the user's **UTF-16** index against that byte count. For anything non-ASCII the byte count is larger than the character count, so an index between the two sails through. `getUtf16PositionByCounter` then walks that many characters through the buffer with no bounds check at all.

```js
const RE2 = require('re2');
const re = new RE2('a', 'y');   // 'g' also works
re.lastIndex = 3;               // 3 <= byteLen(4) passes; there are only 2 real chars
re.exec('éé');                  // ASAN: heap-buffer-overflow READ
```

On the shipped binary a large non-ASCII subject marches the counter well past the buffer into unmapped memory and takes the process down with an uncatchable SIGSEGV. The same `lastIndex` on an ASCII subject returns normally, which pins the cause to the unit mismatch rather than the size. There is a bounded information leak too, but my over-reads came back as zeros, so I reported the crash as the real impact.

It survived because ASCII is safe: byte length equals character count, the guard is accidentally correct, and every ordinary test passes. CVE-2026-67550, CWE-125, Medium 5.7, also fixed in 1.25.2.

> **The lesson: ask what unit a length is in.** A validator measuring one quantity guarding a consumer that walks by another is a bug class, not a typo. Bytes against characters, UTF-8 against UTF-16, count against size.

---

### The crash I could not reproduce

A crash-fuzzer hit this one inside about 150 cases, watching the real binary for exit 134. Finding it was the easy part. I then spent considerably longer unable to make it happen again.

Two reasons, both mine. The PRNG was seeded from `process.pid`, so no run reproduced another. And the crash log truncated the input at 400 characters when the crash needed 89KB of it — the failure was input-*size* dependent and I had thrown the size away. Minimisation went the same way: emoji, named groups and the sticky flag all looked essential, and all turned out to be decoration.

What was left is one line.

```js
"a".repeat(50000).replace(new RE2("a", "g"), "$'");   // SIGABRT
```

`WrappedRE2::Replace` calls `.ToLocalChecked()` without checking the empty `MaybeLocal` V8 returns when a string exceeds `String::kMaxLength`. An output-amplifying template — `$'` or `` $` `` — grows output to O(input²), and past the limit you get `FATAL ERROR: v8::ToLocalChecked Empty MaybeLocal` and **SIGABRT, uncatchable**. 30k works, 40k crashes, exactly where N²/2 crosses 536M.

It counts as a defect rather than a documented limit because of the comparison: native `/a/g` on the same input throws a catchable `RangeError`. re2 aborts where the thing it replaces politely throws. CVE-2026-71430, CWE-617, Medium 6.2 — found by the fuzzer while the C++ audit agents were still queued.

> **The lesson: fuzzing hygiene, learned the hard way.** Fixed PRNG seed, never `process.pid`. Full untruncated input logs. Minimise deterministically, because the interesting-looking parts are usually decoration.

---

### Quadratic in the dimension nobody bounds

A 360KB file froze an event loop for twelve seconds. It contained nothing clever: `{"meta":` repeated a few thousand times, a `1`, and the closing braces. It never even matched the filter it was fed to.

`stream-json`'s path filters — `pick`, `ignore`, `filter`, `replace` — recompute `stack.join(separator)` on every checkable token. `stack.length` is the nesting depth, so a document of depth D costs O(D²). This is the documented flagship API, `pick({filter: 'data'})`, not some corner of the library.

| Nesting depth | Time |
|---|---|
| 5,000 | 160 ms |
| 10,000 | 603 ms |
| 20,000 | 2,511 ms |
| 40,000 | 11,823 ms |

Roughly 4× per doubling — the shape you want to see before believing your own theory. A megabyte or two reaches minutes. The fix is cheap: cache the joined path, update it on push and pop.

It survived because the cost is priced in *structure* and every guard anyone writes is priced in *bytes*. Whatever payload limit protects your JSON endpoint, 360KB passes it. Nothing caps depth.

I filed it at 7.5 High. The maintainer published at Medium 6.2, and he was probably right — reachability depends on an attacker-controlled document in a way my score glossed over. CVE-2026-71429, CWE-407.

> **The lesson: check which dimension your cost model is in.** If the expensive quantity is not the quantity your validators bound, the validators are decorative.

---

### Two more, from the shape of the work

**Audit the marshalling layer, not the engine.** re2 is the library people install *to avoid* ReDoS, which is exactly why its hand-written C++ N-API layer goes unaudited — re2 is the safe one, so who checks it. The engine core held up fine. Every bug was in the binding, and the wrapper reintroduced the very class the engine exists to prevent. That generalises to any FFI or WASM layer marshalling attacker-controlled strings between runtimes.

**Disclosure has a human-only step.** Private reporting was enabled on both repositories, but `POST /repos/{owner}/{repo}/security-advisories` is maintainer-only and 403s anyone else. An orchestrator can write the whole advisory; a human still pastes it into the form.

### What the orchestration actually bought

Not the bugs. Two less glamorous things that I think matter more.

The first is independent reproduction. Every finding was re-run against a clean install rather than the tree it was discovered in — the index bugs on the prebuilt binary users actually get *and* on an ASAN build from source, the stream-json scaling on a clean `stream-json@3.4.0`. A finding that only reproduces where it was found is not a finding.

The second is refutation, which never appears on anyone's scoreboard. On stream-json, four candidates went in and one came out. Prototype pollution in the assembler: refuted, since bracket `__proto__` assignment re-parents the local object only, with no global gadget. Deep-nesting stack overflow: refuted, parser and assembler are both iterative. Unbounded string buffering: refuted, there is a documented `packStrings:false` escape hatch.

The same pass caught me nearly disclosing a measurement artifact: a synchronous fuzzing loop starves the event loop, keeps stream callbacks alive, and produces an OOM that looks exactly like a leak. Drain between iterations and the test sits flat at 4MB. Four things, none of which reached a maintainer's inbox as noise.

The scoreboard stays modest. All four Medium, all four denial of service, none remote code execution — the write-primitive audit found no out-of-bounds write anywhere in node-re2, so the ceiling for this class is DoS plus a weak read. Both packages are solo-maintainer open source with no bounty program, so the payout was zero: four CVE numbers and credit in the advisories. gossipcat's own pin went 1.25.0 → 1.26.1 in PR #699, which closed the loop neatly — the tree I set out to audit was the tree that needed patching.

Orchestration did not conjure bugs out of nothing. It made auditing a dependency tree I would otherwise have trusted by default cheap enough to actually do, and — the part I did not expect — it made *disproving* things cheap too.

Episode 2 stays on the same dependency tree and goes one link further down it.
