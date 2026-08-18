---
layout: post
title: "AI Orchestration, Episode 1: Five CVEs in the Tree I Was Standing On"
author: ataberk-xyz
categories: [ai-research]
tags: [orchestration, gossipcat, npm, cve, fuzzing, supply-chain]
ledger:
  target: "re2 (node-re2) · stream-json · install-artifact-from-github — ~13.4M downloads/week combined"
  severity: "HIGH 7.5"
  vector: "CWE-407 · CWE-617 · CWE-494 · CWE-125 · CWE-835"
  advisory: "CVE-2026-71429 · 71430 · 73864 · 67550 · 68499"
  impact: "event-loop DoS, process-level crash, install-time RCE, bounded heap read"
  status: "PATCHED — re2 1.25.2, stream-json 3.5.0, install-artifact-from-github 1.7.0"
  method: "dependency-tree audit → orchestrated review + fuzzing"
summary: "I pointed my own multi-agent review system at my own dependency tree — re2, stream-json, and the installer re2 delegates its own binary fetch to. Five accepted advisories came out, one of them install-time code execution, and this is the honest account of how, in the order things were actually found, including the parts where the process nearly threw a finding away."

---

> **Episode 1 of a series on AI orchestration applied to real targets rather than demonstrations.**

### The starting point

The result first. Five findings across three npm packages in one dependency chain, all five accepted by the maintainer, published and patched. Four are Medium, denial-of-service class, in the two packages the audit set out to examine. The exception is High — an install-time code-execution bug, found not in the regex engine but in the tooling that fetches it.

The question that produced them was narrow. I was building [gossipcat](https://github.com/gossipcat-ai/gossipcat-ai), a multi-agent code-review server, and it had only ever been pointed at code I had written. I wanted to know what it would do against the code I install. The chain was verifiable in the repository's own `node_modules`: `gossipcat → re2 (^1.25.0) → install-artifact-from-github`. re2 is the package installed specifically to **avoid** regular-expression denial of service, which is what made its binding layer worth examining. stream-json came in through the maintainer rather than the tree, since Eugene Lazutkin (`uhop`) also maintains it — and the chain went one hop further, into `install-artifact-from-github` itself, the package re2 delegates its own native-binary fetch to.

One note on attribution. For the re2 findings my records do not identify which session surfaced which bug, and I will not reconstruct that after the fact. Two roles appear below: the orchestrator, which dispatched work and verified it, and me. What follows is ordered by when each advisory was actually opened, not by the chain above.

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

### A process abort in Replace

A crash-fuzzer reached this defect within approximately 150 cases, watching the real binary for exit 134. Reproducing it took substantially longer, for two reasons. The PRNG was seeded from `process.pid`, so no run replayed another. The crash log also truncated the input at 400 characters when the crash required 89KB of it; the failure was input-*size* dependent, and the size had been discarded. Minimisation was similarly misleading: emoji, named groups and the sticky flag all appeared essential, and none were.

The minimised case is a single line:

```js
"a".repeat(50000).replace(new RE2("a", "g"), "$'");   // SIGABRT
```

`WrappedRE2::Replace` calls `.ToLocalChecked()` without checking the empty `MaybeLocal` that V8 returns when a string exceeds `String::kMaxLength`. An output-amplifying template — `$'` or `` $` `` — grows output to O(input²); past the limit the result is `FATAL ERROR: v8::ToLocalChecked Empty MaybeLocal` and **SIGABRT** — a process-level abort rather than a JavaScript exception, so `try`/`catch` cannot contain it. 30k succeeds and 40k crashes, which is where N²/2 crosses 536M.

The comparison is what makes this a defect rather than a documented limit: the native engine throws a catchable `RangeError` on the same input, where re2 aborts. The orchestrator reproduced the minimised case on a clean install of the then-current published version as well as on the repository's own, confirming the abort was not local to my tree. The C++ source-audit agents were queued for this target and never dispatched, because fuzzing reached it first. CVE-2026-71430, CWE-617, Medium 6.2.

> **The lesson: fuzzing hygiene, learned the hard way.** Fixed PRNG seed, never `process.pid`. Full untruncated input logs. Minimise deterministically, because the interesting-looking parts are usually decoration.

---

### A downloaded artifact nothing verifies

The chain from the opening paragraph has a third hop I hadn't followed yet. re2 doesn't compile its native binding by default — it fetches a prebuilt `.node` file, and that fetch is delegated to a separate package, `install-artifact-from-github`, via `install-from-cache --artifact build/Release/re2.node --host-var RE2_DOWNLOAD_MIRROR ...`. Scanning gossipcat's own dependency tree is what put that install script in front of me — not a fuzzer, another read.

`bin/install-from-cache.js` downloads the artifact over the network and writes it straight to disk. There is no checksum, no signature, no SRI — a grep for `createHash|sha256|checksum|integrity|digest|signature` across the whole package returns nothing:

```js
// bin/install-from-cache.js
let assetUrl = mirrorHost || process.env[mirrorEnvVar] || 'https://github.com';
// ...
const write = async (name, data) => {
  await fsp.mkdir(path.dirname(name), {recursive: true});
  await fsp.writeFile(name, data);   // data is the raw HTTP response body
};
```

`mirrorEnvVar` is fully attacker-steerable with no host pinning: re2 passes `--host-var RE2_DOWNLOAD_MIRROR`, so anything able to set that variable — CI config, a `.npmrc` env passthrough, a compromised CI job, a poisoned shell profile — redirects the entire download to an arbitrary host. The transfer also accepts plaintext `http://`, and a redirect re-derives its protocol from the new URL rather than pinning the original scheme, so an `https://` asset that 302s to `http://` is followed silently — no warning, no downgrade guard.

The package does run a post-download check, `verify-build` or `npm test`, but only after the file is written, and for re2 that check's first line is `require()`-ing the binary it's supposed to be verifying. The native module-init code has already executed by the time any check runs. It's a smoke test, not a gate.

The proof of concept is deliberately non-malicious: stand up a plaintext HTTP "mirror" serving sentinel bytes, point `RE2_DOWNLOAD_MIRROR` at it, and show the served bytes land on disk unmodified.

```
attacker mirror (plaintext http) on 127.0.0.1:PORT
  [mirror] served 2083 attacker bytes over HTTP for /uhop/node-re2/releases/download/1.24.1/darwin-arm64-137
served   sha256: ec6fdde6...4ebc
written  sha256: ec6fdde6...4ebc
install-from-cache exit: 0
```

The install succeeds. A real `.node` file executes its native init code the moment something `require()`s it, so substituting one is code execution in the installing or running process — before any of node-re2's own code, the code this whole audit was actually about, ever runs.

CVE-2026-73864, GHSA-88q3-gch3-5396, CWE-494, High 7.5. The advisory opened at 06:07 UTC on 2026-07-07 — after the stream-json and Replace-abort findings above, before the two re2 findings below.

It wasn't theoretical, either. `install-artifact-from-github` sits under re2 in a lot of real trees, and its pre-fix versions were, as of this check, still pinned in several: `PostHog/posthog` (37.7k stars — `re2@1.22.1`/`1.24.1` in their `pnpm-lock.yaml`), `coralproject/talk` (2k stars — `re2@1.21.4`), and `google-labs-code/stitch-sdk` (1.8k stars, a Google org — `re2@1.23.2`). `democratic-csi/democratic-csi` (1.3k stars) is the sharper example: its Dockerfile sets

```
ENV RE2_DOWNLOAD_MIRROR="https://grpc-uds-binaries.s3-us-west-2.amazonaws.com/re2"
```

— the exact variable the advisory names as the attack surface, in real and entirely legitimate use as a build-speed mirror. `renovatebot/renovate` (22.3k stars) isn't a dependent, but its own Docker build carries a comment written specifically for this package (`# set npm_config_platform_arch for install-artifact-from-github`) — evidence the maintainers had already reasoned about its install behaviour, unrelated to this bug.

The fix shipped fast: 1.7.0 landed roughly 20.5 hours after the advisory opened, and it isn't a one-line patch. The consuming package now stamps a SHA-256 digest per platform slot into its own `package.json` at publish time; `install-from-cache` checks the download against that digest before writing it, and falls back to building from source on a mismatch. That's a fast, well-scoped response, not a rushed one.

One caveat worth stating plainly, because it's specific rather than a knock on the design: verification only covers the default `github.com` source. An explicitly configured mirror — the `RE2_DOWNLOAD_MIRROR` case above — remains trust-the-mirror by design, since there's no way to pre-stamp a digest for a host the consuming package doesn't control. `democratic-csi`, even fully updated to 1.7.0, isn't covered by the new check for exactly that reason. Verifying an arbitrary user-configured source isn't the same problem as verifying a fixed one, and scoping the fix that way is a reasonable call — but its coverage and its real-world usage don't fully overlap.

> **The lesson: audit what delivers the code, not just the code.** A binding layer can be memory-safe end to end and still be exposed if nothing checks the bytes that become it before they run. Any package that fetches a prebuilt artifact at install time is a supply-chain edge, whether or not anyone building on top of it thinks of it that way.

---

### A byte length validating a character index

The fuzzer did not find this defect and would not have found it. Reaching it requires `re.lastIndex` set to a specific incorrect value, and a random input generator has no reason to touch that property. Reading the byte-versus-character conversion arithmetic located it in minutes.

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

### One further lesson

**Audit the marshalling layer, not the engine.** re2 is installed *to avoid* ReDoS, and that reputation is precisely why its hand-written C++ N-API layer is not examined. The engine core held. Every defect in node-re2 itself was in the binding, where the wrapper reintroduced the class the engine exists to prevent. This generalises to any FFI or WASM layer marshalling attacker-controlled strings between runtimes.

### The division of labour

The orchestrator dispatched review, reproduced findings on clean installs, and held claims against controls. I chose the target, judged severity, minimised the crash and wrote the advisories. Two defects came from fuzzing and three from reading — the install-artifact-from-github finding surfaced during the orchestrator's own review pass over the dependency chain, not a fuzzing run, which is a fitting way for a code-review tool to find a bug. None was produced by the orchestration unaided, and this post does not claim otherwise.

The refutations are the part I would defend most. On stream-json, four candidates were examined and one was reported: prototype pollution, a deep-nesting stack overflow and unbounded string buffering were each refuted, and a measurement artifact — a synchronous fuzzing loop starving the event loop and faking a leak — was caught before it reached anyone. Four items that never arrived in a maintainer's inbox as noise, which is as much of the value as the five that shipped.

The scoreboard is correspondingly modest, with one exception. Four of the five findings are Medium, denial-of-service class, in node-re2 and stream-json's own runtime code — the write-primitive audit found no out-of-bounds write anywhere in node-re2, so the ceiling for that engine specifically is denial of service with a weak read. The exception breaks that ceiling: High severity, install-time code execution — but in the tooling that delivers the binary, not the engine, a different bug class in a different package rather than a hole in the memory-safety work the other four represent. The outcome is five CVE identifiers and credit in the advisories. gossipcat's own dependency chain moved off every vulnerable pin — re2 1.25.0 → 1.26.1 in PR #699, pulling install-artifact-from-github forward to 1.7.0 with it — which closed the loop: the tree I set out to audit was the tree that required patching.

### What the exercise produced

Five advisories, all patched upstream, in packages that were already in my tree before I looked at them. That last part is the finding that outlasts the five CVE identifiers: I had installed re2 precisely because it is the safe choice, and it was the component whose binding layer needed fixing — and the tool it silently delegates its own delivery to needed fixing even more. A dependency is not audited because its reputation is good; it is audited because someone read it, all the way down to the install script.

The method carried further than the target. Both re2 memory-safety defects came from questions that can be asked of any native binding without knowing the codebase: *which of the paths that walk this structure fails to share its siblings' guard*, and *what unit does this validator measure compared with what the consumer walks by*. The stream-json defect came from the same habit applied to cost — the quantity that was expensive was not the quantity anything bounded. The install-artifact-from-github defect came from applying it to trust instead: *what verifies the bytes that end up on disk before they run?* Nothing did — the one check that existed ran after the artifact was already loaded. That question applies to any package that fetches something at install time, language or ecosystem aside. Those questions transfer; the specific bugs do not.

---

### References

- **[CVE-2026-71429](https://www.cve.org/CVERecord?id=CVE-2026-71429)** · [GHSA-528h-pc64-c93x](https://github.com/uhop/stream-json/security/advisories/GHSA-528h-pc64-c93x) — stream-json: `pick`/`ignore`/`filter`/`replace` filters are O(depth²) on nested input — small crafted JSON blocks the event loop for seconds→minutes (DoS). CWE-407, Medium 6.2. Fixed in 3.5.0.
- **[CVE-2026-71430](https://www.cve.org/CVERecord?id=CVE-2026-71430)** · [GHSA-8hcv-x26h-mcgp](https://github.com/advisories/GHSA-8hcv-x26h-mcgp) — node-re2: `String.prototype.replace(re2, template)` aborts the Node process (uncatchable `ToLocalChecked` on an empty `MaybeLocal`) when the result exceeds V8's max string length. CWE-617, Medium 6.2. Fixed in 1.25.2.
- **[CVE-2026-73864](https://www.cve.org/CVERecord?id=CVE-2026-73864)** · [GHSA-88q3-gch3-5396](https://github.com/advisories/GHSA-88q3-gch3-5396) — install-artifact-from-github: `install-from-cache` writes and loads a network-downloaded native `.node` with no integrity check (env-controlled mirror + plaintext-HTTP redirect downgrade) → arbitrary code execution at install. CWE-494, High 7.5. Fixed in 1.7.0.
- **[CVE-2026-67550](https://www.cve.org/CVERecord?id=CVE-2026-67550)** · [GHSA-ff84-5f28-78qj](https://github.com/advisories/GHSA-ff84-5f28-78qj) — re2: out-of-bounds heap read in `exec`/`test`/`match` via attacker-influenced `lastIndex` on a non-ASCII subject → uncatchable process crash (DoS). CWE-125, Medium 5.7. Fixed in 1.25.2.
- **[CVE-2026-68499](https://www.cve.org/CVERecord?id=CVE-2026-68499)** · [GHSA-6hxr-mr5r-9836](https://github.com/advisories/GHSA-6hxr-mr5r-9836) — re2: global `String.prototype.match` with an empty-matchable pattern never advances → infinite loop with unbounded native memory growth (DoS). CWE-835, Medium 6.2. Fixed in 1.25.2.
