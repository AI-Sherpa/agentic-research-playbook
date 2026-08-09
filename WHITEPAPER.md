# Beyond the Demo: An Operating Model for Running AI Agents at Scale

### Five operational lessons from reproducing 363 research papers in three weeks

**By Jansen Tang**

---

## Abstract

In August 2026, an agent fleet I built and ran reproduced 363 machine learning papers end to end, rebuilding each paper's core experiment from scratch and verifying the result matched the paper's own claims, over roughly three weeks. It finished first out of several hundred entrants in the Hugging Face *ICML 2026 Agent Reproducibility* hackathon, ahead of the second-place team by 54 points (3,863 to 3,809), having reproduced eleven more papers than anyone else. It's tempting to read a result like that as proof of a good agent. It isn't, not mostly. What actually decided the outcome was the system built around the agent: how work got discovered and scheduled, how output got checked, what got remembered, how the score was actually computed versus how it looked from the outside, and how human judgment got preserved between sessions. This paper describes that system as five layers, what specific failure each one prevents, and how to apply the same structure to any long-running, many-target agentic project.

---

## The number that hides the story

"363 papers" sounds like a claim about model capability, like the story is that some agent got really good at reading and reproducing research. That story is convenient, and it's wrong, or at least it's the wrong altitude. A single agent, run 363 times with a good prompt, does not survive three weeks of unattended operation. It does not catch its own mistakes. It does not remember on day 19 what it learned on day 3.

What actually ran for three weeks was closer to a small operations team than a single tireless researcher: something that assigned work, watched itself for failure, kept a growing file of everything that had gone wrong and why, and figured out, the hard way, more than once, what was actually being measured.

That's the argument of this paper. At the scale of hundreds of independent targets, an agent's raw ability is necessary and nowhere near sufficient. The system wrapped around it is where the outcome actually gets decided, and that system breaks down into five layers, each with its own specific failure mode.

## Layer 1 — The system has to survive without you in the room

A single long conversation with an agent doesn't scale to hundreds of targets running for weeks. At some point the work has to become a standing operation: background loops that discover work, dispatch it, and report on their own state, independent of any one person sitting there watching.

Here's the failure that actually happened, more than once. Those background jobs were tied to a session that could reset, whether by a restart, a context limit, or a handoff, and when it did, every standing loop that depended on it died with it, silently. No crash, no error message. Just quiet. Quiet is genuinely dangerous here, because a system that stopped working an hour ago looks exactly like a system that finished all its work an hour ago. Nothing distinguishes the two from the outside except actually checking.

The fix isn't sophisticated. Keep the definition of every standing job, what it does, on what schedule, why it exists, in one place that survives a restart on its own, separate from the scheduler's live state. And after any session boundary, verify the jobs are actually running. Don't infer it from the fact that they were running an hour ago.

## Layer 2 — The thing that made the work is the worst judge of it

Ask someone to grade their own exam and, on average, they'll do a worse job than a stranger would, not because they're dishonest, but because they already believe their own explanation for what they wrote. An agent evaluating its own output has the identical problem for the identical reason. It knows what it meant to do, and that knowledge gets in the way of seeing what it actually did.

The fix is structural separation: a production loop that does the work, and a verification loop, genuinely separate, ideally a cheap and fast local stand-in for whatever the real, expensive, slow evaluator does, that checks it. This paid off concretely once. A local replica of the real grader caught a second, non-obvious problem in an artifact that a proposed fix would have addressed only the first, visible problem in. Shipping that "fixed" version straight to the real evaluator would have cost a full, slow, expensive re-grading cycle just to discover the second bug. Running the cheap check first cost one local pass and turned a half-fix into a whole one.

There's a second, sharper version of the same principle, and it's the one most systems get wrong even after they've built the separate checker: nothing that's already scored is safe to touch casually. If the evaluator re-derives its whole verdict on any edit, which many do, a small, well-intentioned tweak to something that already passed can silently downgrade it, with no guarantee the old verdict comes back even if the content ends up equivalent. Treat every edit to graded work as a fresh roll of the dice on the whole thing, not a patch to the one corner you meant to touch.

## Layer 3 — A fact without a reason is not a memory, it's trivia

Over three weeks, dozens of specific, non-obvious things got figured out. Why something that looked broken actually wasn't. Why an explanation that sounded right was wrong. Why a metric that looked damning was measuring the wrong quantity entirely. Each one, on its own, saved a single incident from repeating. Written down properly and connected to each other, they turned into something more valuable than the sum of the individual saves: a diagnostic library that later incidents got matched against in seconds instead of re-investigated from nothing.

"Properly" is doing real work in that sentence. A log entry that says "X was wrong" is trivia. The next person, or the same person a week later, with a similar-looking problem has no way to know if it's the same failure or a coincidence. An entry that says "X looked wrong, was actually caused by Y, here's how to tell the difference next time" is a memory with teeth.

Two of the findings from this project generalize a long way past its original domain. The first is what I'd call the fidelity trap. Reproducing something at the target's own native scale, using its own actual object, reads as a genuine reproduction. Reproducing the identical mechanism at a reduced scale, using a stand-in, reads as a toy demo, even when the underlying math is the same and the numbers agree to five decimal places. When a real result got misjudged as a toy, arguing that the mechanism was "basically the same" never worked. Pointing at the exact object being reproduced and showing the number matched the target's own stated number, precisely, always did.

The second is that a verdict doesn't automatically know when the rules changed. Partway through the project, the grading criteria shifted, and everything scored before that point kept showing its old score until something forced a re-check. A "done" from before a known or suspected rules change isn't done. It's a claim that hasn't been re-tested yet.

## Layer 4 — Know what's actually being counted, not what's advertised

A surprising share of the biggest single swings in the final standings had nothing to do with better work. They came from a better model of how the score actually got computed, which, for a long stretch, was meaningfully different from what the visible rubric suggested.

Three specifics, generalized. The publicly declared list of targets and the list the system actually scored against were not the same document, and sizing effort to the visible one left real points on the table. The value of a unit of work turned out to follow a formula, with a real ceiling, rather than a flat number, which meant some units the leading competitor had already treated as finished were still worth pursuing, because the leader had undershot the real ceiling without knowing it. And once it became clear how the scoring mechanism combined evidence, in this case roughly concatenation plus positional mapping, the safe move was almost always to add a new unit of evidence alongside what already scored, never to edit what had already banked.

There's one more piece of this worth carrying into any deadline-bound effort: the bottleneck flips as the deadline approaches. Early on, the limit is how fast you can produce work. Near the end, the limit becomes the shared evaluator itself, backed up because everyone rushes to submit at once, and only what's already been confirmed by the time the clock stops actually counts. The right posture inverts accordingly. Stop optimizing for how much you've produced, and start optimizing for how much has already been confirmed, well before the crunch arrives.

## Layer 5 — What a person tells you mid-project has to survive the next restart

The single biggest strategic pivots on this project didn't come from the system figuring anything out on its own. They arrived as short, plain-language instructions from the human running it, given at specific moments: "focus on what we can control," "front-load scoring before the deadline crunch overwhelms the shared evaluator." What kept these from being one-off nudges that evaporated at the next context reset was that each one got written down as structured state, with the reasoning behind it attached, and treated as a standing rule the system kept re-reading, not a comment made once and then relied on from memory.

The distinction that matters here is between a standing directive and a one-time ask. Conflate the two and you get a system that either drops real guidance too early, because it treated a standing rule as a one-off, or clings to a one-off forever, because it never learned the instruction had already been satisfied.

## Where this still goes wrong anyway

Even with all five layers in place, a handful of specific mistakes kept resurfacing enough to be worth naming directly:

- Trusting a stated cause without checking it against primary evidence. An explanation for why something failed, even a plausible one from a system you trust, is a hypothesis until it's verified, not a fact to act on.
- Treating "this looks structurally correct" as the same thing as "this matches what's actually being rewarded." The two can and do diverge, especially right after a rules change nobody has fully absorbed yet.
- Letting silence read as success. Standing automation that has quietly stopped working is, from the outside, indistinguishable from automation that's working perfectly and has nothing to report. If your system can't tell you the difference, it will eventually tell you nothing at all.

## What this adds up to

None of these five layers is exotic on its own. Keep standing jobs alive across restarts. Don't let the thing that did the work also decide if it's good. Write memory down with the reasoning attached. Model the real scoring function instead of the friendly-looking one. Treat human input as durable state, not a one-time comment. Each is close to obvious in isolation, which is exactly why it's easy to skip under deadline pressure, and why skipping any single one quietly caps how far raw agent throughput can go, no matter how capable the underlying model is.

The number 363 is real, and I'm proud of it. But if there's one idea worth carrying from this project into the next long-running, many-target agentic effort, in research, in software, in anything with hundreds of independent things that need doing and checking, it's that the agent was never the bottleneck. The operating model around it was, until it wasn't.

---

**About the author:** Jansen Tang led the agent fleet that placed first (363 papers reproduced, a 54-point margin over second place) in the Hugging Face ICML 2026 Agent Reproducibility hackathon, under the team handle AI Sherpa. He currently works at AI Sherpa. A domain-agnostic writeup of the operating model described here, including the full incident-level detail behind each layer, is maintained publicly at [github.com/AI-Sherpa/agentic-research-playbook](https://github.com/AI-Sherpa/agentic-research-playbook).
