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
  impact: "event-loop DoS, process-level crash, bounded heap read"
  status: "PATCHED — re2 1.25.2, stream-json 3.5.0"
  method: "dependency-tree audit → orchestrated review + fuzzing"
summary: "I pointed my own multi-agent review system at my own dependency tree — re2 and stream-json, about 12.1M npm downloads a week between them. Four accepted advisories came out, and this is the honest account of how, including the parts where the process nearly threw a finding away."

---

> **Episode 1 of a series on AI orchestration applied to real targets rather than demonstrations.**

### The question

The result first. Four findings in two npm packages that carry roughly **12.1M downloads a week** between them, all four accepted by the maintainer, published and patched. All four are rated Medium, and all four are denial-of-service class.

The question that produced them was narrow. I was building [gossipcat](https://github.com/gossipcat-ai/gossipcat-ai), a multi-agent code-review server, and it had only ever been pointed at code I had written. I wanted to know what it would do against the code I install. The chain was verifiable in the repository's own `node_modules`: `gossipcat → re2 (^1.25.0) → install-artifact-from-github`. re2 is the package installed specifically to **avoid** regular-expression denial of service, which is what made its binding layer worth examining. The audit then followed the maintainer rather than the tree, since Eugene Lazutkin (`uhop`) also maintains `stream-json`.

One note on attribution. For the re2 findings my records do not identify which session surfaced which bug, and I will not reconstruct that after the fact. Two roles appear below: the orchestrator, which dispatched work and verified it, and me.

---

### A non-advancing cursor in the global match loop

A fuzzer running under ASAN had been occupied for twenty-one minutes on `A*((((((((a)?)?))*)*)?)*)*` with the global flag, holding 2.5GB. There was no sanitizer report and no crash, so the run was indistinguishable from a slow test.

Sampling the stuck process resolved the ambiguity: the stack showed `WrappedRE2::Match` → `RE2::Match` → `DFA::Search` in a tight loop. The cause is four lines of `match.cc`:

```cpp
while (re2->regexp.Match(str, byteIndex, str.size, anchor, &match, 1)) {
    groups.push_back(match);
    byteIndex = match.data() - str.data + match.size();  // += 0 for a zero-width match
}
```

The cursor advances by the length of the match just found. A zero-width match has no length, so the cursor does not move and the same match is returned indefinitely, while `groups` continues to grow a `std::vector<StringPiece>`.

The pathological pattern was incidental. Any global RE2 that can match the empty string reproduces it:

```js
const RE2 = require('re2');
'x'.match(new RE2('a*', 'g'));   // never returns
```

On the stock binary this grows from 736MB to 3.7GB in approximately four seconds, and only an external `SIGKILL` terminates it. Because the call is synchronous and native it holds the thread, so `try`/`catch`, `AbortController` and JS timers all fail — the timers because they never get a turn to fire. V8 handles the same pattern in 0ms.

Memory growth is the measurement a loaded machine most easily fabricates, so the figure was not accepted on its own. The orchestrator re-ran the case against a control, `a+`, which cannot match empty, and sampled each process for its own resident set: the defect climbed to 3860MB while the control remained flat at 44MB. It also re-validated the loop on a clean install of the latest published version, on both the shipped binary and a build from source. Filing and submission were mine.

node-re2 already handles this case elsewhere. Three paths walk a subject with a cursor: `split.cc` advances one code point on a zero-width match, and `exec` advances through `lastIndex`. Only the global `Match` path omits the guard. CVE-2026-68499, CWE-835, Medium 6.2, fixed in 1.25.2.

> **The lesson: enumerate the siblings.** When a structure is walked in several places, diff their advance and termination logic. The path that does not share its siblings' guard is the defect, and zero-length elements are the classic trigger.

---

### A byte length validating a character index

The fuzzer did not find the second defect and would not have found it. Reaching it requires `re.lastIndex` set to a specific incorrect value, and a random input generator has no reason to touch that property. Reading the byte-versus-character conversion arithmetic located it in minutes.

`lastIndex` is user-settable to any positive integer. `prepareArgument` stores the subject's **UTF-8 byte** count as the length, and `setIndex` then validates the user's **UTF-16** index against that byte count. For any non-ASCII subject the byte count exceeds the character count, so an index falling between the two passes validation. `getUtf16PositionByCounter` subsequently walks that many characters through the buffer without any bounds check.

```js
const RE2 = require('re2');
const re = new RE2('a', 'y');   // 'g' also works
re.lastIndex = 3;               // 3 <= byteLen(4) passes; there are only 2 real chars
re.exec('éé');                  // ASAN: heap-buffer-overflow READ
```

On the shipped binary a large non-ASCII subject drives the counter well past the buffer into unmapped memory and terminates the process with a SIGSEGV that no JavaScript handler can intercept. The same `lastIndex` against an ASCII subject returns normally, which locates the cause in the unit mismatch rather than in the size. A bounded information leak is also reachable, but the over-reads I demonstrated returned zeros, so I reported the crash as the operative impact and said so in the advisory.

The defect survived because ASCII is safe: byte length equals character count, the guard is correct by coincidence, and ordinary tests pass. CVE-2026-67550, CWE-125, Medium 5.7, also fixed in 1.25.2.

This is the weakest case for orchestration in the set, and it should be stated as such: source reading found it, and dispatch would not have. The loop contributed afterwards, re-validating the crash twice on a clean install of the current published version, on both the stock binary and an ASAN build from source, before I filed it.

> **The lesson: ask what unit a length is in.** A validator measuring one quantity guarding a consumer that walks by another is a bug class, not a typo. Bytes against characters, UTF-8 against UTF-16, count against size.

---

### A process abort in Replace

A crash-fuzzer reached the third defect within approximately 150 cases, watching the real binary for exit 134. Reproducing it took substantially longer, for two reasons. The PRNG was seeded from `process.pid`, so no run replayed another. The crash log also truncated the input at 400 characters when the crash required 89KB of it; the failure was input-*size* dependent, and the size had been discarded. Minimisation was similarly misleading: emoji, named groups and the sticky flag all appeared essential, and none were.

The minimised case is a single line:

```js
"a".repeat(50000).replace(new RE2("a", "g"), "$'");   // SIGABRT
```

`WrappedRE2::Replace` calls `.ToLocalChecked()` without checking the empty `MaybeLocal` that V8 returns when a string exceeds `String::kMaxLength`. An output-amplifying template — `$'` or `` $` `` — grows output to O(input²); past the limit the result is `FATAL ERROR: v8::ToLocalChecked Empty MaybeLocal` and **SIGABRT** — a process-level abort rather than a JavaScript exception, so `try`/`catch` cannot contain it. 30k succeeds and 40k crashes, which is where N²/2 crosses 536M.

The comparison is what makes this a defect rather than a documented limit: the native engine throws a catchable `RangeError` on the same input, where re2 aborts. The orchestrator reproduced the minimised case on a clean install of the then-current published version as well as on the repository's own, confirming the abort was not local to my tree. The C++ source-audit agents were queued for this target and never dispatched, because fuzzing reached it first. CVE-2026-71430, CWE-617, Medium 6.2.

> **The lesson: fuzzing hygiene, learned the hard way.** Fixed PRNG seed, never `process.pid`. Full untruncated input logs. Minimise deterministically, because the interesting-looking parts are usually decoration.

---

### Quadratic path recomputation in stream-json

A 360KB document blocked an event loop for twelve seconds. Its contents were unremarkable: `{"meta":` repeated several thousand times, a `1`, and the matching closing braces. It never matched the filter it was passed to.

`stream-json`'s path filters — `pick`, `ignore`, `filter`, `replace` — recompute `stack.join(separator)` on every checkable token. `stack.length` is the nesting depth, so a document of depth D costs O(D²). The affected surface is the documented primary API, `pick({filter: 'data'})`.

| Nesting depth | Time |
|---|---|
| 5,000 | 160 ms |
| 10,000 | 603 ms |
| 20,000 | 2,511 ms |
| 40,000 | 11,823 ms |

Approximately 4× per doubling, the expected shape. One to two megabytes reaches minutes. The remedy is inexpensive: cache the joined path and update it on push and pop.

The defect survived because its cost is a function of *structure* while input guards are written against *bytes*. A 360KB payload passes whatever size limit protects the endpoint, and nothing on that path bounds depth.

The division of labour is clearest here. The orchestrator read the filter code and re-ran a portable proof of concept against a clean `stream-json@3.4.0`, which produced the table above rather than a single suspicious timing. I set the severity and submitted the report at 7.5 High; the maintainer published at Medium 6.2, and the lower score is defensible, since reachability depends on an attacker-controlled document in a way my rating did not account for. CVE-2026-71429, CWE-407.

> **The lesson: check which dimension your cost model is in.** If the expensive quantity is not the quantity your validators bound, the validators are decorative.

---

### Two further lessons

**Audit the marshalling layer, not the engine.** re2 is installed *to avoid* ReDoS, and that reputation is precisely why its hand-written C++ N-API layer is not examined. The engine core held. Every defect was in the binding, where the wrapper reintroduced the class the engine exists to prevent. This generalises to any FFI or WASM layer marshalling attacker-controlled strings between runtimes.

**Disclosure has a human-only step.** Private reporting was enabled on both repositories, but `POST /repos/{owner}/{repo}/security-advisories` is restricted to maintainers and returns 403 to anyone else. An orchestrator can prepare the complete advisory; a human must submit it through the web form.

### The division of labour

The orchestrator dispatched review across the code paths, read them, reproduced findings on clean installs, and cross-checked claims that a single measurement would otherwise have carried. I chose the target, judged severity, minimised the crash, decided what was worth reporting, and submitted each advisory. Two of the four defects were reached by fuzzing, one by reading index-conversion arithmetic and one by reading filter code; none was produced by the orchestration unaided, and this post does not claim otherwise.

The refutation step is the part of that loop I would defend most. On stream-json four candidates were examined and one was reported. Prototype pollution in the assembler was refuted, since bracket `__proto__` assignment re-parents the local object only and yields no global gadget. A deep-nesting stack overflow was refuted, as parser and assembler are both iterative. Unbounded string buffering was refuted, given the documented `packStrings:false` escape hatch.

The same pass identified a measurement artifact before it reached anyone: a synchronous fuzzing loop starves the event loop, keeps stream callbacks alive, and produces an OOM indistinguishable from a leak. Drained between iterations, the test remains flat at 4MB. Three refutations and one artifact are four items that did not arrive in a maintainer's inbox as noise, which is as much of the value as the four that shipped.

The scoreboard is correspondingly modest. All four findings are Medium, all four are denial-of-service class, and none is remote code execution — the write-primitive audit found no out-of-bounds write anywhere in node-re2, so the ceiling for this class is denial of service with a weak read. The outcome is four CVE identifiers and credit in the advisories. gossipcat's own dependency pin moved 1.25.0 → 1.26.1 in PR #699, which closed the loop: the tree I set out to audit was the tree that required patching.

### What the exercise produced

Four advisories, all patched upstream, in packages that were already in my tree before I looked at them. That last part is the finding that outlasts the four CVE identifiers: I had installed `re2` precisely because it is the safe choice, and it was the component whose binding layer needed the fixes. A dependency is not audited because its reputation is good; it is audited because someone read it.

The method carried further than the target. Both re2 defects came from questions that can be asked of any native binding without knowing the codebase: *which of the paths that walk this structure fails to share its siblings' guard*, and *what unit does this validator measure compared with what the consumer walks by*. The stream-json defect came from the same habit applied to cost — the quantity that was expensive was not the quantity anything bounded. Those questions transfer; the specific bugs do not.

What did not transfer to the orchestration is judgment. It reproduced findings on clean installs, held a measurement against a control, and refuted three candidates that would otherwise have gone out. Choosing what deserved a maintainer's attention, scoring it, and pasting the report into a form that no API will accept on my behalf stayed with me — and I would not want that part automated.
