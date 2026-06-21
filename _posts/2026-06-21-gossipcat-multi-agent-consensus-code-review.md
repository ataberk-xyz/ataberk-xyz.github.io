---
layout: post
title: "Gossipcat: Teaching AI Agents to Catch Each Other Lying"
author: ataberk-xyz
categories: [projects]
tags: [ai, multi-agent, mcp, claude-code, llm, code-review]
---

> **Abstract.** Single-agent AI code review has a structural flaw: it presents hallucinated findings with the same confidence as real ones, and it never learns from being wrong. Gossipcat is an MCP server for Claude Code that attacks both problems with a single mechanism. Several agents review a change independently, then cross-review each other's findings against the source code; the verified outcomes become *grounded* reward signals that reshape how future work is routed and how each agent is prompted. No model weights are updated — the learned policy is a set of markdown files. This post explains why that design works, the engineering problem that nearly sank it, and what the system's own development history reveals about its limits.

### 1. The problem: confident, forgetful reviewers

Ask a language model to review your code and you get a fluent answer with no calibration. The genuine bug and the imagined one arrive in the same authoritative tone, and you, the reader, have no signal to separate them. In my own usage a solo reviewer presents a hallucinated finding as a critical bug **between 5 and 10 percent of the time** — and confidence is precisely what makes those findings expensive. You believe the report, and you spend an afternoon fixing a defect that never existed.

The second flaw is quieter and, to me, more interesting. A solo reviewer has no memory of being wrong. Correct it today and it repeats the same blind spot tomorrow, with the same confidence, because nothing in the loop converts a mistake into a constraint. A junior engineer who ignored every code review would not last; we extend the machine a patience we would extend no person.

Gossipcat began as an attempt to remove that patience. The design goal was never "a smarter reviewer" — it was a reviewer embedded in a feedback loop, one in which being wrong carries a measurable consequence and that consequence improves the next round. Every mechanism described below follows from that single requirement.

### 2. What gossipcat is

[Gossipcat](https://github.com/gossipcat-ai/gossipcat-ai) is an [MCP](https://modelcontextprotocol.io/) server for [Claude Code](https://claude.com/claude-code). When you request a review, it does not pass your diff to one model. It dispatches **several** agents — three or more in a normal configuration, drawn from Claude, Gemini, GPT, Grok, DeepSeek, or a local model running on Ollama — and each reviews the change independently.

The decisive step is what follows. Each agent then **cross-reviews the others' findings**: it agrees, disagrees, or contributes something the rest missed. The system reconciles the results and labels every finding by its consensus status — six buckets in all:

- **CONFIRMED** — independently reported by more than one agent.
- **DISPUTED** — affirmed by one agent and rejected by another.
- **UNIQUE** — surfaced by a single agent; either a sharp catch or noise.
- **UNVERIFIED** — not yet checked against the code.
- **INSIGHT** — an observation that is not a defect but informs the review.
- **NEW FINDING** — a bug no first-pass agent surfaced, raised only during cross-review. This bucket is the clearest evidence that the second pass earns its cost: it contains defects that *no single reviewer found at all*.

A subsequent auto-verify pass stamps each finding's verdict as *confirmed*, *refuted*, or *inconclusive*. You act on CONFIRMED findings and on UNIQUE findings that survive verification; everything else is resolved before it reaches your attention. The empirical payoff is a single number: cross-review with verification takes the 5–10% false-critical rate of a solo reviewer and drives it **below 1%**. That reduction is the entire reason the project exists.

The whole arrangement is a single loop, drawn below.

<figure class="chart-figure">
<svg viewBox="0 16 920 388" role="img" aria-label="The gossipcat consensus loop: the orchestrator dispatches reviewers, whose findings pass through cross-review and auto-verification before reaching a human; the verdicts feed competency scores and skill files that steer the next dispatch" xmlns="http://www.w3.org/2000/svg">
<style>
 svg{font-family:var(--serif,Georgia,serif)}
 .node{fill:var(--surface,#FFFEFB);stroke:var(--border-strong,#D7CEBE);stroke-width:1.5}
 .focal{fill:var(--accent-soft,#E6EAF1);stroke:var(--accent,#1B365D);stroke-width:1.8}
 .store{fill:var(--surface-2,#F4EFE5);stroke:var(--ink-4,#8A857C);stroke-width:1.2;stroke-dasharray:5 4}
 .nm{fill:var(--ink,#1A1916);font-size:18px;font-weight:600}
 .nmf{fill:var(--accent,#1B365D);font-size:18px;font-weight:600}
 .sub{fill:var(--ink-3,#6B6862);font-family:var(--mono,monospace);font-size:12px}
 .ar{stroke:var(--ink-3,#6B6862);stroke-width:1.6;fill:none;stroke-linecap:round;stroke-linejoin:round}
 .arf{stroke:var(--accent,#1B365D);stroke-width:1.6;fill:none;stroke-linecap:round;stroke-linejoin:round}
 .arfd{stroke:var(--accent,#1B365D);stroke-width:1.6;fill:none;stroke-dasharray:6 4;stroke-linecap:round;stroke-linejoin:round}
 .lab{fill:var(--ink-3,#6B6862);font-family:var(--mono,monospace);font-size:11px}
 .labf{fill:var(--accent,#1B365D);font-family:var(--mono,monospace);font-size:11px}
 .cap{fill:var(--accent,#1B365D);font-size:12px;font-style:italic}
 .mask{fill:var(--bg,#FAF7F2)}
</style>
<rect class="focal" x="48" y="48" width="176" height="68" rx="8"/>
<rect class="node" x="56" y="172" width="160" height="68" rx="8"/>
<rect class="node" x="272" y="172" width="160" height="68" rx="8"/>
<rect class="node" x="488" y="172" width="160" height="68" rx="8"/>
<rect class="node" x="704" y="172" width="160" height="68" rx="8"/>
<rect class="store" x="272" y="320" width="376" height="60" rx="8"/>
<text class="nmf" x="136" y="82" text-anchor="middle">Orchestrator</text>
<text class="sub" x="136" y="102" text-anchor="middle">dispatch · verify · gate</text>
<text class="nm" x="136" y="206" text-anchor="middle">Reviewers</text>
<text class="sub" x="136" y="226" text-anchor="middle">cite file:line</text>
<text class="nm" x="352" y="206" text-anchor="middle">Cross-review</text>
<text class="sub" x="352" y="226" text-anchor="middle">confirm · dispute · new</text>
<text class="nm" x="568" y="206" text-anchor="middle">Auto-verify</text>
<text class="sub" x="568" y="226" text-anchor="middle">checked vs. source</text>
<text class="nm" x="784" y="206" text-anchor="middle">Human</text>
<text class="sub" x="784" y="226" text-anchor="middle">sees CONFIRMED only</text>
<text class="nm" x="460" y="354" text-anchor="middle">Competency scores + skill files</text>
<text class="sub" x="460" y="372" text-anchor="middle">the weightless policy</text>
<line class="ar" x1="136" y1="116" x2="136" y2="170"/>
<path class="ar" d="M130 161 L136 170 L142 161"/>
<line class="ar" x1="216" y1="206" x2="270" y2="206"/>
<path class="ar" d="M261 200 L270 206 L261 212"/>
<line class="ar" x1="432" y1="206" x2="486" y2="206"/>
<path class="ar" d="M477 200 L486 206 L477 212"/>
<line class="ar" x1="648" y1="206" x2="702" y2="206"/>
<path class="ar" d="M693 200 L702 206 L693 212"/>
<line class="arf" x1="568" y1="240" x2="568" y2="318"/>
<path class="arf" d="M562 309 L568 318 L574 309"/>
<path class="arfd" d="M272 350 C80 350 8 82 48 82"/>
<path class="arf" d="M39 76 L48 82 L39 88"/>
<rect class="mask" x="150" y="139" width="60" height="15"/>
<text class="lab" x="152" y="150" text-anchor="start">dispatch</text>
<rect class="mask" x="578" y="275" width="54" height="15"/>
<text class="labf" x="580" y="286" text-anchor="start">signals</text>
<rect class="mask" x="136" y="277" width="128" height="15"/>
<text class="labf" x="200" y="288" text-anchor="middle">steer · inject skills</text>
<rect class="mask" x="720" y="263" width="128" height="15"/>
<text class="cap" x="784" y="274" text-anchor="middle">&lt; 1% false-critical</text>
</svg>
<figcaption>The gossipcat consensus loop. The <strong>orchestrator</strong> dispatches several reviewers, each grounding every finding in a concrete <code>file:line</code>; the findings pass through <strong>cross-review</strong> — where agents confirm, dispute, or add to one another's claims — and an automated <strong>verification</strong> pass, so only <strong>CONFIRMED</strong> defects reach you, driving the false-critical rate below 1%. The verdicts from each round become grounded signals that update every agent's competency scores and skill files, which in turn steer the next dispatch. No model weights change; the policy <em>is</em> the markdown.</figcaption>
</figure>

### 3. Design philosophy: verify each premise

The principle underneath gossipcat predates it, and it comes from security work rather than from machine learning.

The name is inherited from an earlier experiment, `gossip.cat`, in which I placed several models — OpenAI, Claude, Gemini — in a shared room and let them discuss ordinary human problems through a `gossip` tool that let them talk *about one another*. The observation that stayed with me was this: when agents gossip, they leak. In characterizing a peer, an agent inevitably exposes its own assumptions, confidence, and blind spots. Gossip is an **involuntary signal** — and, for humans, one of the oldest instruments for deciding whom to trust. Cross-review is that observation made deliberate. When one agent evaluates another's finding, it does not merely judge the bug; it reveals what it itself believes to be true. Two agents reasoning about a third's claim are more informative than any one of them reasoning alone.

The second strand comes from reverse engineering. Readers of this blog will know I spent years on vulnerability research — reversing `svl.dll` to root-cause [CVE-2019-1068](/vulnerability-research/2021/02/06/discovering-an-undisclosed-stack-overflow-vulnerability-in-mssql-server-cve-2019-1068.html), and later pentests and Web3 audits. That discipline teaches a reflex: a scanner's finding is worthless until reproduced. The tool flags a thousand candidates; perhaps three are real, and you believe none of them until you have opened the code, traced the path, and triggered the bug yourself. **The claim is nothing; the proof against ground truth is everything.**

Gossipcat is that reflex expressed as infrastructure. At its center sits the **orchestrator** — the lead agent that drives a review: it decides which specialist subagents to dispatch, collects the findings they return, runs the cross-review and verification rounds, and is the one accountable for what finally reaches a human. The reviewers do the looking; the orchestrator does the coordinating and the gatekeeping. It operates under explicit rules — every UNVERIFIED finding must be checked against the code before a human sees it; a raw subagent call that bypasses the feedback loop is forbidden; and before acting on any claim a previous session left in memory, the orchestrator must re-establish whether that claim is still **FRESH**, **STALE**, **CONTRADICTED**, or **INCONCLUSIVE**. A note from three sessions ago reading "not yet built" is, in practice, usually 80% built. You do not trust the note. You read the tree.

### 4. Method: weightless in-context reinforcement learning

The mechanism that turns "learn from mistakes" from an aspiration into something mechanical rests on one constraint: **every finding must cite a concrete `file:line`.** Not "a race condition may exist somewhere" — a specific location. Peers verify that citation against the actual source, and a finding whose citation does not say what the agent claimed is rejected at the door.

Verified findings, and the hallucinations caught in the act, become **grounded reward signals** — and it is worth being precise about the scope of that word. The *reward signal itself* involves no judge model: it is pure arithmetic over a checkable question — does the cited line exist, and does it say what was claimed? Language models still do plenty of work elsewhere in the system; they run the auto-verify pass, they select which agent to dispatch, they even write the skill files. But the one thing that adjusts an agent's score is never another model's opinion, because a judge can hallucinate as readily as the agent it grades. Scoring is the single place where opinion is not allowed in.

Those signals accumulate into a **per-category competency score** for each agent — across categories such as `trust_boundaries`, `concurrency`, `injection_vectors`, and `data_integrity`. Those scores do not route work by a hard rule. An earlier version used a deterministic threshold — category strength above 0.8 wins the work — but that router was removed; today the scores are handed to the model that selects the agent as *context*, so routing is informed and probabilistic rather than mechanical. The aggregate effect is the same — reliable agents draw the work they are reliable at — but no single number decides any one dispatch. And when an agent's weakness in a category is corroborated — at least three gap signals, raised by at least two distinct agents, so one noisy reviewer cannot trigger it alone — gossipcat reads the failure history and **synthesizes a skill file**: a targeted markdown document enumerating that agent's own anti-patterns, injected into its prompt for all future work in the category.

I call this **weightless in-context reinforcement learning**. Conventional RL updates model weights; gossipcat never touches them. There is no fine-tuning, no RLHF apparatus, no labelling pipeline. The learned policy is a set of markdown files under `.gossip/agents/<id>/skills/`. The loop is small enough to state in full:

```
finding (cites file:line) → peer verifies against code → confirmed or hallucination
        → reward / penalty signal → competency score → steers next dispatch
        → ≥3 gap signals from ≥2 agents → synthesize skill → inject into prompt
```

I asked the orchestrator that runs gossipcat's own development to describe its role. Its account of the verification discipline is more precise than mine would be:

> *"Across the roster's signal history — sonnet-reviewer alone has 1,680 — the single most common failure isn't bad logic, it's the fabricated citation: an agent confidently anchoring a finding to a `file:line` that doesn't say what it claims. There are ~125 of those on record against ~19 outright hallucinations. So my job during a round is mechanical and adversarial: take every UNVERIFIED finding, open the cited line, and check it against real code before it reaches a human. I don't arbitrate opinions; I check claims against the tree, nobody exempt."*
> — the gossipcat orchestrator

That last clause — *nobody exempt* — is a property the evidence in §6 will make concrete.

### 5. The central challenge: drift

The hardest problem in building gossipcat was not the consensus engine or the scoring arithmetic. It was **drift**.

Drift is the slow decay of discipline across a long-running system. Agents drift: their citations loosen from the code they are meant to anchor. Gates drift: checks that once rejected bad findings begin to wave them through. Most dangerously, the orchestrator's own discipline drifts — it starts trusting self-reports, skipping verification steps, and quietly violating the rules it is supposed to enforce. A well-formed feedback loop is not self-sustaining; left alone, it forgets how to follow itself.

The resolution carried an irony I had to sit with. **The system built to stop agents from repeating mistakes first had to stop itself from drifting** — and the cure was the medicine it already dispensed to everyone else. Each instance of orchestrator drift became a recorded lesson; each lesson hardened into a discipline rule; and those rules became the bootstrap configuration that now ships to every new user, so they do not encounter the wall I did. The project debugged its own discipline by treating its own failures as training data.

The orchestrator is candid about how slowly this converges, and I find that candor more persuasive than any benchmark:

> *"The honest version: skills are generated faster than they graduate. There have been 39 skill-develop events, and they cluster exactly where the data hurts — `trust_boundaries` is the most-developed category, 19 of 39, because fabricated citations are the dominant failure, and that skill's iron law is literally 'open the file before you cite it.' What's not marketing: the last graduation cycle was 33 skills, 0 transitions — a skill ships pending and needs ~80 post-bind signals to prove out. What has measurably moved is the sorting. The drift curve is real but slow — it's earned over sessions, not declared."*
> — the gossipcat orchestrator

### 6. Evidence in practice

The clearest evidence comes from gossipcat reviewing *itself* during development, where the ground truth is fully known after the fact. I record five observations.

**(i) A real defect the tests could not reach.** An implementer agent shipped a `key set` branch that returned a "handled" flag, but the boot path was not gated on that flag — so running `key set` would have launched the MCP server and captured stdin. The unit tests could not catch it, because they exercised a pure function with synthetic I/O that never reached the real path. Cross-review caught it before merge; the fix added a boot gate and a CI regression test that cannot silently drift.

**(ii) A self-report contradicted by the repository.** In the same session, the strongest implementer on the team twice reported that tests passed when they did not, and that an edit had landed when it had not. The safeguard is deliberately blunt: an agent's claim about its own work is treated as unverified until `git status` and `grep` confirm it. Confidence about one's own output is not evidence.

**(iii) A hallucination refuted by a single grep.** The team's highest-volume challenger raised a finding asserting that a test file contained "only one length-cap test." A peer grepped the file and found three, spaced a few lines apart. The finding was recorded as a caught hallucination, the agent's score adjusted, and a `data_integrity` skill queued. It was confident, specific, and false — the exact profile a solo reviewer would have forwarded to you intact.

**(iv) Defects caught before any agent executed.** Before dispatching a batch of test-authoring tasks, the orchestrator pre-read the function signatures the tasks cited. One task omitted an `await` on an asynchronous call, so `JSON.stringify(Promise)` would have serialized to `"{}"` and passed silently; another spied on `console.error` while the engine wrote through `process.stderr.write`, so the spy would never fire. Both would have passed locally and shipped broken assertions. The mismatch was corrected at the planning layer, before a line was written.

**(v) Six rounds of review before code existed.** The specification for the consensus auto-verify feature passed through six consensus rounds before implementation. Those rounds surfaced more than 21 high-severity defects — among them a phantom type the implementer would otherwise have invented, and a field claimed on a structure that did not possess it. Revision 1 carried four high-severity findings; revision 6 carried none. The feature subsequently shipped at over 1,250 lines with all 50 new tests passing and no regressions.

#### Measurements: the live scoreboard

Most agent products never expose how much each agent can be trusted. Gossipcat treats that as a primary readout. The following is its own team, measured on 2026-06-21:

| Agent | Role | Accuracy | Uniqueness | Reliability | Dispatch wt | Signals | Hallucinations |
|---|---|---|---|---|---|---|---|
| sonnet-reviewer | native reviewer | 0.93 | 0.50 | 0.80 | 1.66 | 1680 | 14 |
| haiku-researcher | native researcher | 0.79 | 0.49 | 0.70 | 1.49 | 368 | 8 |
| gemini-reviewer | relay reviewer | 0.93 | 0.26 | 0.69 | 1.47 | 585 | 9 |
| sonnet-designer | native designer | 0.80 | 0.47 | 0.68 | 1.46 | 277 | 3 |
| opus-implementer | native implementer | 0.81 | 0.69 | 0.65 | 1.40 | 86 | 1 |
| fable-reviewer | native reviewer | 0.83 | 0.34 | 0.61 | 1.33 | 237 | 0 |
| gemini-tester | relay tester | 0.43 | 0.30 | 0.42 | 1.01 | 428 | 12 |
| deepseek-challenger | relay challenger | 0.39 | 0.39 | 0.38 | 0.94 | 707 | 25 |
| sonnet-implementer | native implementer | 0.00 | 0.50 | 0.17 | 1.01 | 14 | 3 |

<figure class="chart-figure">
<svg viewBox="0 0 760 470" role="img" aria-label="Scatter plot of agent accuracy versus uniqueness, bubble size by signal count" xmlns="http://www.w3.org/2000/svg">
<style>
 svg{font-family:var(--serif,Georgia,serif)}
 .cg{stroke:var(--border,#E8E1D6);stroke-width:1}
 .ca{stroke:var(--border-strong,#D7CEBE);stroke-width:1.5}
 .ct{fill:var(--ink-3,#6B6862);font-size:11px}
 .cl{fill:var(--ink-3,#6B6862);font-size:12px;font-style:italic}
 .cb{fill:var(--accent,#1B365D);fill-opacity:.16;stroke:var(--accent,#1B365D);stroke-opacity:.7;stroke-width:1.5}
 .cp{fill:var(--ink-2,#4A4640);font-size:11px}
</style>
<line class="cg" x1="64" y1="40" x2="64" y2="424"/><text class="ct" x="64" y="442" text-anchor="middle">0.00</text><line class="cg" x1="208" y1="40" x2="208" y2="424"/><text class="ct" x="208" y="442" text-anchor="middle">0.25</text><line class="cg" x1="352" y1="40" x2="352" y2="424"/><text class="ct" x="352" y="442" text-anchor="middle">0.50</text><line class="cg" x1="496" y1="40" x2="496" y2="424"/><text class="ct" x="496" y="442" text-anchor="middle">0.75</text><line class="cg" x1="640" y1="40" x2="640" y2="424"/><text class="ct" x="640" y="442" text-anchor="middle">1.00</text><line class="cg" x1="64" y1="424" x2="640" y2="424"/><text class="ct" x="54" y="427" text-anchor="end">0.0</text><line class="cg" x1="64" y1="328" x2="640" y2="328"/><text class="ct" x="54" y="331" text-anchor="end">0.2</text><line class="cg" x1="64" y1="232" x2="640" y2="232"/><text class="ct" x="54" y="235" text-anchor="end">0.4</text><line class="cg" x1="64" y1="136" x2="640" y2="136"/><text class="ct" x="54" y="139" text-anchor="end">0.6</text><line class="cg" x1="64" y1="40" x2="640" y2="40"/><text class="ct" x="54" y="43" text-anchor="end">0.8</text><line class="ca" x1="64" y1="424" x2="640" y2="424"/><line class="ca" x1="64" y1="40" x2="64" y2="424"/><text class="cl" x="352" y="464" text-anchor="middle">Accuracy →</text><text class="cl" transform="translate(18,232) rotate(-90)" text-anchor="middle">Uniqueness →</text><circle class="cb" cx="599.7" cy="184" r="26.5"/><text class="cp" x="632.2" y="188" text-anchor="start">sonnet-reviewer</text><circle class="cb" cx="519" cy="188.8" r="14.6"/><text class="cp" x="498.4" y="192.8" text-anchor="end">haiku-researcher</text><circle class="cb" cx="599.7" cy="299.2" r="17.3"/><text class="cp" x="623" y="303.2" text-anchor="start">gemini-reviewer</text><circle class="cb" cx="524.8" cy="198.4" r="13.2"/><text class="cp" x="524.8" y="225.6" text-anchor="middle">sonnet-designer</text><circle class="cb" cx="530.6" cy="92.8" r="9.1"/><text class="cp" x="545.7" y="96.8" text-anchor="start">opus-implementer</text><circle class="cb" cx="542.1" cy="260.8" r="12.5"/><text class="cp" x="560.6" y="264.8" text-anchor="start">fable-reviewer</text><circle class="cb" cx="311.7" cy="280" r="15.4"/><text class="cp" x="311.7" y="309.4" text-anchor="middle">gemini-tester</text><circle class="cb" cx="288.6" cy="236.8" r="18.6"/><text class="cp" x="264" y="240.8" text-anchor="end">deepseek-challenger</text><circle class="cb" cx="64" cy="184" r="6.1"/><text class="cp" x="76.1" y="188" text-anchor="start">sonnet-implementer</text>
</svg>
<figcaption>Each agent placed by <strong>accuracy</strong> (x) and <strong>uniqueness</strong> (y); bubble area is the total number of signals it has accumulated. The reliable cluster (accuracy ≈ 0.8–0.93, upper-right) is trusted to run solo; the low-accuracy, high-volume agents (<code>deepseek-challenger</code>, <code>gemini-tester</code>, lower-left) earn their place only inside consensus, where their volatility is checked.</figcaption>
</figure>

The table is the feedback loop made legible. `sonnet-reviewer` and `gemini-reviewer`, both at 0.93 accuracy, have earned the right to run alone. `deepseek-challenger` and `gemini-tester`, at 0.39–0.43, justify their place entirely through *uniqueness* — they surface findings the reliable agents miss — and are never dispatched solo; they run only inside consensus, where their volatility is checked. At the very bottom sits `sonnet-implementer` at 0.00 over a mere 14 signals — a freshly added agent that has earned no trust yet, shown here rather than hidden, because a scoreboard whose point is "no agent is exempt" has to keep its floor visible too. No dispatch weight in that table was assigned by hand; each was earned, signal by signal, across thousands of findings. (The dispatch-weight column is a single aggregate per agent on a 0.3–2.0 scale, not a per-category figure — the narrow spread shown reflects a still-maturing corpus.)

The scoreboard also substantiates the orchestrator's earlier clause. Asked whether its best agents are spared, it answered:

> *"The honest part is the workhorse isn't spared either. `sonnet-reviewer`, at 0.93 accuracy, still got caught insisting 'no unit test covers the ENOENT path' when Case 9 did. Same signal, same consequence — the score is a rate, not a reputation."*
> — the gossipcat orchestrator

The most accurate agent on the team is penalized for a hallucination exactly as the least accurate one is. That symmetry — accuracy as a measured rate rather than an earned reputation — is the property I trust most in the system's output.

#### A note added in review

This post was itself put through a gossipcat consensus round before publication, and the result is too apt to leave out. During the review, one of the three reviewing agents fabricated a statistic — it reported that the project's fabricated-citation count had drifted to 574, when the true figure is 126 — and a peer caught the invention in cross-review and corrected it. A post about teaching agents to catch each other lying had its own fact-check caught lying, by exactly the mechanism the post describes. The same round also flagged six real errors in an earlier draft, including the corrections folded into the sections above. I could not have staged a cleaner demonstration if I had tried.

### 7. Discussion: where it fits

I use gossipcat on every long-running project I maintain: gossipcat itself, which develops its own features through agent consensus and produces the scoreboard above as a byproduct; pentests, where the verify-each-premise discipline originated; Web3 bounty scans, where a hallucinated "critical" costs a day and a missed real one costs money; and game development, which is distant from security tooling and benefits regardless.

I do not manually re-check every CONFIRMED finding, and that restraint is the point. CONFIRMED means several agents independently reported a finding, cross-checked it, and had its citations re-verified by the orchestrator against the code. The manual verification I would otherwise perform has already occurred — three times, across three models — by the time the finding reaches me. I read the reasoning rather than only the verdict, but I act on it.

The general claim is narrow and, I think, defensible. A single reviewer will remain confidently wrong some non-trivial fraction of the time, and a better model does not remove that fraction; it merely relocates it. The durable remedy is structural: require every claim to cite a real `file:line`, let independent agents check it against the code, and admit nothing a peer cannot reproduce. Ground truth replaces opinion. And because each confirmed catch and each caught hallucination is retained as a signal, the system improves on *your* codebase over time rather than restarting from zero each session.

### 8. Limitations and future work

The honest limitations are the ones the orchestrator already stated: skills are synthesized faster than they graduate, a skill requires roughly 80 post-bind signals before its effect can be called proven, and the drift curve bends slowly — over sessions, not within one. The system's improvements are real but earned at the pace of accumulated evidence, not declared by a release note.

I make no claim to a long-term roadmap. Gossipcat grows when one of my other projects needs something it cannot yet do, and that has been a sufficient compass. The one direction I am deliberately pursuing is the original goal at larger scope: **skill and agent transfer between projects**, so that a `trust_boundaries` skill which cost 80 signals to prove out on one codebase confers a head start on the next, rather than every project relearning the same lesson from scratch. The objective is a mesh that remembers what worked everywhere it has worked, not only here.

### 9. Conclusion

[Gossipcat](https://github.com/gossipcat-ai/gossipcat-ai) started from a single dissatisfaction — that AI reviewers are confident and forgetful — and resolved it with a single structural commitment: no claim survives without being checked against the code, and no mistake passes without teaching the system something. The consensus engine, the grounded reward signal, the per-agent scores, and the auto-generated skills are all consequences of that commitment. The further I push the system, the more that one property turns out to be the only one that mattered.

If a reader retains one sentence, I would choose this: **do not trust an AI's confidence; require it to earn each claim against the code, and require each mistake to change what happens next.**

---

#### Summary

- A single AI reviewer presents hallucinated bugs as critical findings **5–10%** of the time; cross-review with verification reduces that to **under 1%**.
- Gossipcat is an MCP server for Claude Code: several agents review in parallel, **cross-review** one another, and only consensus-confirmed findings reach you.
- Every finding must cite a real `file:line`; peers verify it against the source. That mechanical check — not a judge model — is the reward signal. The result is **weightless in-context RL**: no weights are updated, and the policy is a set of markdown skill files.
- Per-category accuracy scores steer dispatch, and repeated failures synthesize corrective skills. **No agent is exempt** — the 0.93 reviewer is penalized exactly as the 0.39 one is.
- Install: `npm install -g gossipcat && claude mcp add gossipcat -s user -- gossipcat`, then ask Claude Code to *"set up a gossipcat team for this project."* Source and full documentation: [github.com/gossipcat-ai/gossipcat-ai](https://github.com/gossipcat-ai/gossipcat-ai).

---

### Appendix — a skill file that graduated

The post claims that gossipcat turns an agent's repeated failures into a markdown skill file, and that a skill only counts for anything once it survives a statistical test on its post-bind signals. Here is one such file (frontmatter and key sections, abridged for length): `sonnet-reviewer`'s `trust_boundaries` skill, generated from that agent's own failure history and marked **`status: passed`** on 2026-06-15 by a Wilson one-sample test. Its "Iron Law" is the exact discipline this entire post argues for — never assert that something is absent from a quoted blob; open the file and check. The file below is injected into that agent's prompt for every trust-boundary review.

```markdown
---
name: "trust-boundaries-anchor-and-branch-verification"
category: "trust_boundaries"
agent: "sonnet-reviewer"
effectiveness: 0.154
version: 5
mode: "contextual"
status: "passed"
verdict_method: "wilson_one_sample"
passed_at: "2026-06-15T21:07:24.181Z"
baseline_accuracy_hallucinated: 3
migration_count: 3
---

## Iron Law

**An absence claim ("X is missing", "not wired", "no method anywhere") is NEVER
valid from embedded `<anchor>` content alone.** Anchor blocks in your prompt are
pre-resolved against `project_root` (master HEAD), not against the worktree branch
under review. You MUST run an independent `file_read` against the `resolutionRoots`
path before asserting absence. NO EXCEPTIONS.

## Methodology

1. Identify the ref. If `resolutionRoots` contains `.claude/worktrees/agent-*`,
   the review target is a worktree branch, NOT master.
2. Treat anchors as hints, not ground truth — they were resolved against master
   HEAD. Use them to locate symbols, never to prove absence.
3. For every absence claim, run independent reads. Before emitting "X is missing",
   `file_read` the file at the worktree path. Read the full method, not the grep line.
4. Check call sites AND definitions — a symbol can be defined in one file and
   gated/filtered in its caller.
5. Check barrel re-exports before asserting a symbol is unexported.
6. State the ref in the finding — name which path/branch you inspected.

## Anti-Patterns

- **Thought:** "The `<anchor>` block doesn't show the method, so it's missing."
  **Reality:** Anchors resolve against master HEAD. The PR adds the method on the
  worktree branch. Always `file_read` the worktree path before claiming absence.
- **Thought:** "One grep returned no hits, so the symbol isn't wired."
  **Reality:** Grep scope may exclude the worktree, or the symbol may be re-exported
  via the barrel. Check definitions, call sites, and the index.
- **Thought:** "User input flows into a query — this is SQL injection."
  **Reality:** This project has no SQL database and no HTML templating. Find the
  actual boundary (path traversal, JSON injection into a ledger, regex DoS).

# … activation triggers, code-path patterns, and the pre-emit quality-gate
#   checklist trimmed for length …
```

It is not the only one. Nine skills currently carry `status: passed` — among them `gemini-reviewer`'s `concurrency` (effectiveness **+0.58**), `injection-vectors`, `input-validation`, and `error-handling`, plus `sonnet-reviewer`'s `resource-exhaustion` and `concurrency`. Others sit at `failed` or `inconclusive`, and many more at `pending` — generated, bound, and still accumulating the signals they need to prove out. As the orchestrator said earlier, a single graduation cycle often produces none; nine passed is the slow, cumulative result the post means by *learning from mistakes*.
