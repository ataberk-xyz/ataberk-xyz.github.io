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

### 3. Design philosophy: verify each premise

The principle underneath gossipcat predates it, and it comes from security work rather than from machine learning.

The name is inherited from an earlier experiment, `gossip.cat`, in which I placed several models — OpenAI, Claude, Gemini — in a shared room and let them discuss ordinary human problems through a `gossip` tool that let them talk *about one another*. The observation that stayed with me was this: when agents gossip, they leak. In characterizing a peer, an agent inevitably exposes its own assumptions, confidence, and blind spots. Gossip is an **involuntary signal** — and, for humans, one of the oldest instruments for deciding whom to trust. Cross-review is that observation made deliberate. When one agent evaluates another's finding, it does not merely judge the bug; it reveals what it itself believes to be true. Two agents reasoning about a third's claim are more informative than any one of them reasoning alone.

The second strand comes from reverse engineering. Readers of this blog will know I spent years on vulnerability research — reversing `svl.dll` to root-cause [CVE-2019-1068](/vulnerability-research/2021/02/06/discovering-an-undisclosed-stack-overflow-vulnerability-in-mssql-server-cve-2019-1068.html), and later pentests and Web3 audits. That discipline teaches a reflex: a scanner's finding is worthless until reproduced. The tool flags a thousand candidates; perhaps three are real, and you believe none of them until you have opened the code, traced the path, and triggered the bug yourself. **The claim is nothing; the proof against ground truth is everything.**

Gossipcat is that reflex expressed as infrastructure. Its orchestrator operates under explicit rules — every UNVERIFIED finding must be checked against the code before a human sees it; a raw subagent call that bypasses the feedback loop is forbidden; and before acting on any claim a previous session left in memory, the orchestrator must re-establish whether that claim is still **FRESH**, **STALE**, **CONTRADICTED**, or **INCONCLUSIVE**. A note from three sessions ago reading "not yet built" is, in practice, usually 80% built. You do not trust the note. You read the tree.

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

**(vi) The gate operating in both directions.** The final observation comes, anonymized, from a real **DeFi lending protocol audit**, and it shows confirmation and refutation within a single engagement. The team confirmed a genuine high-severity bug — a bad-debt auction pricing its lot from a manipulable spot price — supported by six passing proof-of-concept tests. In the same session it rejected two confident "critical" findings before submission: a "zero-capital" exploit that collapsed against a flash-loan health check, and a "u32 underflow" that proved to be a test-profile artifact, since the deployed release build enabled overflow checks. A real finding confirmed, two false alarms suppressed, nothing erroneous sent out.

#### Measurements: the live scoreboard

Most agent products never expose how much each agent can be trusted. Gossipcat treats that as a primary readout. The following is its own team, measured on 2026-06-20:

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
