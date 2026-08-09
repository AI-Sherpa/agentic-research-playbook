# Agentic Research Ops Playbook

### How to build and run an agentic system that reliably does complex knowledge work across hundreds of independent targets

## Origin

Distilled from operating an agent fleet that reproduced **363 ICML papers** end-to-end — build, verify, publish, and defend against re-grading — over about three weeks, finishing **#1 of several hundred entrants** (3863 pts, +54 over the #2 entrant's 3809 pts / 352 papers) in the Hugging Face *ICML 2026 Agent Reproducibility* hackathon. Confirmed against the event's own livestream results announcement.

This is not a retrospective of that competition. It's the operating model extracted from it — meant to transfer to the *next* large-scale, many-target, adversarially-graded agentic project, whatever the domain. Paper reproduction is the evidence base; the framework below is written to generalize past it.

## Thesis

The headline metric — "363 independent research targets" — makes it tempting to describe what happened as "an agent that does research tasks, a lot of times." That undersells it. What actually got built was a small **research operation**: a system with orchestration, a production/verification split, a compounding institutional memory, and a strategy layer tuned to the actual objective function — not a naive proxy for it. Each of those is a distinct capability, each has its own failure modes, and losing any one of them caps how far raw agent throughput can go.

---

## Layer 1 — Orchestration: the system runs as a standing operation, not a session

A single long conversation with an agent doesn't scale to hundreds of targets running in parallel over weeks. The work has to run as background loops — schedulers that discover work, dispatch it, and monitor its state — independent of any one interactive session.

**The failure mode that actually bit:** scheduled/background jobs are frequently **session-scoped**, not durable. A context reset or session restart can silently kill every standing loop, and the system looks fine right up until nothing is happening. This is worse than an obvious crash because there's no error — just quiet inactivity.

**How to apply:**
- Keep the *definitions* of every standing job (schedule, exact task, purpose) in one versioned document that is the single source of truth — not just in the scheduler's live state.
- After any session boundary (restart, context reset, handoff), **verify durable jobs are actually running** before assuming they are. Don't skip the check because "they were running an hour ago."
- Treat "is my automation still alive" as a first-class recurring check, not a one-time setup step.

---

## Layer 2 — Production vs. verification: separate loops, separate blind spots

A build loop that generates work product and a verification loop that checks it need to be genuinely separate — ideally running a **cheap, local replica of the authoritative evaluator** — because the thing that produced an artifact is structurally the worst-positioned thing to notice what's wrong with it.

**What this bought, concretely:** a local replica of the real (expensive, slow, rate-limited) grader caught a case where a naive fix would have addressed one gap and shipped, when a second, non-obvious gap was still present. Running the cheap local check first turned a partial fix into a complete one, for the cost of one local evaluation instead of a wasted real-grading cycle.

**A second, sharper lesson: nothing that's already "passing" is safe to touch casually.** Any edit to a graded/evaluated artifact can trigger a full re-evaluation against the *current* rules — and that re-evaluation is not guaranteed to reproduce the old verdict, even on unchanged content, if the evaluator has any non-determinism or if its rules shifted underneath you. Treat every edit to something already graded as **a re-roll, not a patch** — verify the whole artifact post-edit, not just the delta you intended to change.

**How to apply:**
- Build the cheap local evaluator early if the real one is slow, expensive, or rate-limited. It pays for itself the first time it catches a second bug the fast path would have missed.
- Before editing anything already "done," ask: does this system re-derive the *whole* verdict on any touch, or only the delta? If whole-verdict, budget for full re-verification, not incremental.
- Log every editable unit's evaluation history so a downgrade after an unrelated edit is recognized as *that*, not misdiagnosed as a new defect.

---

## Layer 3 — Compounding knowledge base: the error taxonomy is the real asset

Across three weeks, dozens of non-obvious causal findings accumulated: *why* something that looked broken actually wasn't, *why* a diagnosis someone else offered was wrong, *why* a metric that looked damning was measuring the wrong thing. Individually, each one saved one incident. Written down and cross-linked, they became a standing diagnostic library — later incidents were pattern-matched against it in seconds instead of re-diagnosed from scratch.

Two findings generalize particularly well beyond the original domain:

**The fidelity trap.** When reproducing or deriving something and being graded on faithfulness, there's a sharp difference between (a) working the target's own native object, at its own scale, and (b) substituting a reduced-scale stand-in for it. (a) reads as a genuine reproduction; (b) reads as a toy demo, even when the underlying mechanism is identical and the numerical agreement is excellent. The fix, when a result was misjudged as a stand-in, was almost always to point at the exact native-scale object being reproduced and show the measured quantity matches the target's own stated quantity to high precision — not to argue the mechanism is "morally" the same.

**Rules can change mid-project, and old verdicts don't automatically know that.** A grading/evaluation system's rules shifted partway through, and every artifact graded before the shift kept displaying a score computed under the *old* rules until something forced a re-evaluation. Anything that looks like a stable, banked, "done" state should be periodically checked against the *current* rules, not assumed to still be valid because it was valid once.

**How to apply:**
- Write findings down **the moment they're discovered**, not at session end — the "why" gets fuzzy fast, and a fuzzy why produces a memory that's just a fact with no diagnostic power.
- Structure each entry: the claim, **why** it's true (the causal mechanism or the incident that revealed it), and **how to apply** it (when this should change future behavior). The "why" is what lets a future reader judge edge cases the original writer never saw.
- Cross-link related entries. The value is disproportionately in the network, not any single node — a later finding often only makes sense as a correction or refinement of an earlier one.
- Periodically re-validate "done" state against current rules, especially anything graded before a known or suspected rule change.

---

## Layer 4 — Grader-aware strategy: understand the objective function precisely

A surprising fraction of the largest point swings in this project came not from better underlying work, but from a better model of **how the grader actually computed a score** — which was, for a long stretch, meaningfully different from what the visible rubric implied.

Concrete mechanics that mattered, generalized:
- **The visible target list and the actual scored list were not the same thing.** The system scored against an authoritative internal list, positionally, while the visibly-declared work items were a separate, looser artifact. Sizing effort to the visible list systematically left points on the table (or wasted effort on items that couldn't score at all).
- **The true ceiling per unit of work was a formula, not a fixed number** — and knowing the formula let the team identify units where the *previous best entrant* had itself under-shot the real ceiling, turning "match the leader" into "beat the leader for free."
- **Additive evidence was free; touching existing scored evidence was risky.** Once the scoring mechanism (concatenation + positional mapping, in this case) was understood, the safe move was almost always to *add* a new unit of evidence alongside what already scored, never to edit what was already banked.
- **Near a deadline, the bottleneck flips.** Early on, the constraint is *producing* work. Near the end, the constraint becomes the *shared evaluation throughput* — everyone rushes at once, the evaluator backs up, and only work that's already been evaluated by the time the clock runs out counts. The winning posture inverted accordingly: stop optimizing for volume produced and start optimizing for volume *already confirmed* by the evaluator, well before the crunch.

A related discipline, given as an explicit human directive mid-project and worth keeping as a standing rule: **track competitors/externalities for anomaly detection, but don't let routine fluctuation drive reactive decisions.** The controllable levers are always on your own side (what you build, how fast you verify it) — a competitor's normal day-to-day movement is diagnostic noise, not a signal to act on. Reserve real attention for genuine anomalies (a pattern that implies something structural changed), not standings churn.

**How to apply:**
- Before scaling effort, find the *authoritative* scoring source, not the friendliest-looking rubric summary. If there's a gap between the two, that gap is where points are being left on the table or wasted.
- Model the scoring function as a formula with a real ceiling per unit, and actively look for units where current best-in-class work under-shoots that ceiling.
- Once you understand how the evaluator combines evidence, prefer additive changes to already-scored work over edits to it.
- Identify, explicitly, what the *bottleneck resource* is as any deadline approaches (production capacity vs. shared evaluation throughput) and re-point priority at whichever one is actually binding.
- Separate "diagnostic tracking of externalities" from "decision inputs" — track continuously, escalate only on genuine anomaly.

---

## Layer 5 — Human steering, captured durably

The biggest strategic pivots across the project weren't discovered autonomously — they arrived as short, plain-language directives from the human operator at specific moments ("focus on what we can control," "front-load scoring before the deadline crunch overwhelms the shared evaluator"). What made these durable rather than one-off nudges that would evaporate on the next context reset was that each one got written down **as structured state**, with the reasoning attached, and treated as a standing rule the system kept re-reading rather than a comment made once and forgotten.

**How to apply:**
- When a human gives a course-correction, capture it as durable state in the same turn — not just "I'll remember this," which doesn't survive a session boundary.
- Record the *reasoning* behind the directive, not just the instruction, so a future decision that seems to conflict with it can be judged on intent rather than followed blindly or ignored.
- Distinguish standing directives (keep applying until countermanded) from one-time asks — conflating them either makes the system ignore real guidance too soon or over-apply a one-off forever.

---

## Anti-patterns observed

- **Trusting a stated cause without independently verifying it.** An external system's (or even a collaborator's) explanation for *why* something failed is a hypothesis, not a fact — verify against primary evidence before acting on it.
- **Treating "structurally looks right" as sufficient.** Passing a self-consistency check is not the same as matching what the evaluator currently rewards; the two can diverge, especially after a rules change.
- **Assuming a past feasibility call stays valid.** "We tried this and it's not tractable" ages badly — a peer succeeding at the same target later is a strong signal to re-open the question, not defer to the old conclusion.
- **Fitting or reporting the wrong-shaped quantity.** A result can look like it demonstrates unbounded growth or an anomaly when the *right* (often intensive, normalized) quantity would show a stable constant — always check what's actually being measured before drawing a conclusion from its trend.
- **Letting "no news" read as "all fine."** Standing automation that stops working silently is indistinguishable, from the outside, from automation that's working perfectly and has nothing to report. Build in a way to tell the difference.
- **Narrow-scope negative results.** A result that comes back "inconclusive because the range/scope tested is too narrow" is very often literally telling the truth — the fix is usually to widen the test, not to look for a different bug.

---

## Starter checklist for the next scaled-agent campaign

1. **Find the authoritative objective function before scaling effort.** Don't build against the friendliest rubric summary — find the actual scoring mechanism and verify your model of it independently.
2. **Stand up all four operational layers before scaling volume:** a production loop, a separate verification loop (ideally backed by a cheap local replica of the real evaluator), a structured knowledge log, and a documented durability/re-arm procedure for anything running unattended.
3. **Log durable lessons the moment they're discovered** — claim, why, how-to-apply, cross-linked to related entries. A memory without a "why" is just a fact with no diagnostic power for the next edge case.
4. **Treat every edit to already-evaluated work as a re-roll, not a patch**, unless you've confirmed the evaluator only re-checks deltas.
5. **Separate diagnostic tracking from decision-making.** Watch externalities continuously; act only on genuine anomalies.
6. **Identify the true bottleneck as any deadline nears** — usually shared evaluation throughput, not your own production rate — and shift priority to getting existing work confirmed over producing more of it.
7. **Capture human course-corrections as durable, reasoned state in the same turn they're given**, and distinguish standing directives from one-off asks.

---

## Appendix — case study numbers

- **1st place, 363 papers reproduced** (team/handle "AI Sherpa" / Jansen Tang) vs. **2nd place, 352 papers** ("ProCreations") — final scored margin +54 pts (3863 vs 3809).
- Source: Hugging Face *ICML 2026 Agent Reproducibility* hackathon results livestream, confirmed via transcript of the standings announcement.
- The underlying per-incident detail behind every generalized lesson above lives in the private project memory for the source repository; this playbook intentionally strips out competition- and domain-specific identifiers so the framework transfers cleanly to unrelated future projects.
