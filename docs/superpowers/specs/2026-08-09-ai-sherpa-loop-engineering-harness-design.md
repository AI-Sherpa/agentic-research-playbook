# Design: `ai-sherpa-loop-engineering-harness`

## Origin

`PLAYBOOK.md` in this repo distills five operational layers from a first-place finish
(363/363 papers reproduced, +54pt margin) in the Hugging Face ICML 2026 Agent
Reproducibility hackathon: standing-operation durability, a production/verification split
with a cheap local evaluator, a compounding knowledge base, grader-aware strategy, and
durable human-steering capture. This spec turns those five layers into a Claude Code skill
an agent can actually load and apply, rather than leaving them as prose only a human reads.

## Problem

Two loop-related tools already exist and are not what this skill should duplicate:

1. **`ralph-loop`** (official Anthropic marketplace plugin) — the raw mechanism: a Stop
   hook that feeds the same prompt back until a completion promise or iteration cap. No
   isolation, no separate verifier, effectively no safety net.
2. **`loop-engineering`** (`/opt/dev/loop-engineering`, AI-Sherpa's own skill, GitHub
   `AI-Sherpa/loop-engineering`) — the safety layer around that mechanism: task-suitability
   gating, mandatory isolation, separate maker/checker verification, budget + no-progress +
   green-CI halts, human merge gate, a reference `loop.sh` runner. This is solid and
   answers "how do I run **one loop** safely."

Neither answers what `PLAYBOOK.md` is actually about: how do you run **hundreds of
independent targets as a standing operation over weeks**, against an adversarial evaluator,
without the fleet-level failure modes that bit during the hackathon (silently-dead
background jobs, re-grading surprises on "done" work, lost diagnostic findings, and —
the highest-leverage lesson — misreading what the grader actually rewards)? That is a
different altitude than single-loop mechanics, and building it as a from-scratch
standalone skill would duplicate and drift from `loop-engineering`'s content over time.

## Decision

Build `ai-sherpa-loop-engineering-harness` as a **fleet/campaign layer that sits above
`loop-engineering`**, not a replacement or a duplicate. Package both as two independently
triggerable skills co-located in one parent repo, so a single PR can keep them from
drifting apart while each still activates on its own trigger phrases.

## Architecture

```
ai-sherpa-loop-engineering-harness/        (parent repo; GitHub AI-Sherpa/ai-sherpa-loop-engineering-harness)
├── loop-engineering/                       moved from /opt/dev/loop-engineering, history preserved via git subtree
│   ├── SKILL.md            (name: loop-engineering — unchanged)
│   ├── references/{when-to-loop,safeguards}.md
│   └── scripts/loop.sh
├── harness/                                 NEW — the fleet/campaign layer
│   ├── SKILL.md            (name: ai-sherpa-loop-engineering-harness)
│   ├── references/
│   │   ├── durability.md          Layer 1 — standing jobs must survive session resets
│   │   ├── verification.md        Layer 2 — cheap local evaluator + "re-roll not a patch"
│   │   ├── knowledge-base.md      Layer 3 — claim/why/how-to-apply, cross-linked findings
│   │   ├── grader-model.md        Layer 4 — authoritative scoring fn, additive-safe edits,
│   │   │                                     bottleneck-flip near deadline
│   │   ├── human-steering.md      Layer 5 — durable directive capture
│   │   └── anti-patterns.md       the 6 anti-patterns from PLAYBOOK.md
│   └── templates/                  copied into a user's project, mirroring how
│       │                           loop-engineering copies loop.sh
│       ├── JOBS_REGISTRY.md
│       ├── EVAL_LOG.md
│       ├── KNOWLEDGE_LOG.md
│       ├── GRADER_MODEL.md
│       └── DIRECTIVES_LOG.md
├── README.md
├── CLAUDE.md                                @AGENTS.md (matches loop-engineering's pattern)
└── AGENTS.md                                monorepo layout note + cascaded Must-Dos
```

Both skill subdirectories get their own `~/.claude/skills/` symlink
(`loop-engineering` and `ai-sherpa-loop-engineering-harness`), so both remain
independently discoverable and independently triggerable — moving `loop-engineering`
into this parent repo does not change its skill identity or its trigger phrases.

## Division of labor

`loop-engineering` answers "how do I run **one loop** safely" — isolation, one
maker/checker pair, budget caps on a single unattended run. `harness` answers "how do I
run **many targets** as a standing operation" — it assumes something (loop-engineering's
`loop.sh`, `/ralph-loop`, or a human) is already executing individual loops, and adds the
coordination layer: fleet-wide durability checks, a knowledge base that compounds across
targets instead of resetting each session, a model of the *actual* scoring function (not
just per-task pass/fail), and durable capture of human course-corrections.

`harness/SKILL.md` names `loop-engineering` explicitly in its Step 0 / setup guidance and
directs the agent to it for scaffolding the mechanics of any individual loop; `harness`
never restates isolation/safeguards/six-parts content that `loop-engineering` already
owns.

## Content mapping

Each `PLAYBOOK.md` layer becomes one reference doc plus one scaffolded template, the same
pairing pattern `loop-engineering` already uses (`safeguards.md` + `loop.sh`):

| Layer | Reference doc | Template | Core discipline it encodes |
|---|---|---|---|
| 1. Orchestration | `durability.md` | `JOBS_REGISTRY.md` | Every standing job's schedule/task/purpose lives in one versioned doc; verify jobs are actually running after every session boundary, don't assume |
| 2. Production/verification | `verification.md` | `EVAL_LOG.md` | Build a cheap local replica of a slow/expensive real evaluator early; treat any edit to already-graded work as a re-roll (full re-verify), not a patch (delta-only) |
| 3. Knowledge base | `knowledge-base.md` | `KNOWLEDGE_LOG.md` | Write findings at discovery time, not session end; each entry is claim + why + how-to-apply, cross-linked to related entries |
| 4. Grader-aware strategy | `grader-model.md` | `GRADER_MODEL.md` | Find the authoritative scoring source, not the friendliest rubric summary; model it as a formula with a real ceiling; prefer additive evidence over touching already-scored work; identify whether the true bottleneck is production or shared evaluation throughput |
| 5. Human steering | `human-steering.md` | `DIRECTIVES_LOG.md` | Capture course-corrections as durable state with reasoning in the same turn; distinguish standing directives from one-off asks |

`anti-patterns.md` carries the six generalized anti-patterns from `PLAYBOOK.md`'s
"Anti-patterns observed" section with no template counterpart — they're read-only
cautionary content, not a file format to scaffold.

Templates ship pre-filled with their schema and one worked example (not a blank page),
matching the "prose + copyable templates" scope decision — no bash scripts are included
in `harness/`, since none of the five layers involve procedural logic comparable to
`loop.sh`'s isolation/timeout/no-progress checks; each is a discipline plus a file format.

## `harness/SKILL.md` outline

Following `loop-engineering`'s existing shape (frontmatter trigger description, then
numbered steps, then a reference-files index):

- **Frontmatter `description`**: trigger phrases covering "run a fleet of agent loops",
  "campaign", "many independent targets", "adversarial grader", "research operation",
  "hackathon", "reproduce N things", "backlog of hundreds of tasks", "scale agent loops
  across a portfolio" — distinct from `loop-engineering`'s single-loop trigger phrases so
  the two don't compete for activation on the same prompt.
- **Step 0**: confirm at least one individual-loop execution mechanism is in place
  (point to `loop-engineering` if not); this skill coordinates loops, it doesn't run them.
- **Steps 1–5**: one per PLAYBOOK.md layer, each pointing at its reference doc and
  instructing the agent to scaffold the matching template into the user's project on
  first use.
- **Starter checklist**: the 7-item "Starter checklist for the next scaled-agent
  campaign" from `PLAYBOOK.md`, adapted as the skill's own pre-flight checklist.

## Migration plan for `loop-engineering`

1. `git subtree add --prefix=loop-engineering <local-path-to-/opt/dev/loop-engineering> main`
   into the new parent repo, preserving full history.
2. Create `AI-Sherpa/ai-sherpa-loop-engineering-harness` on GitHub, push the parent repo.
3. Repoint the `~/.claude/skills/loop-engineering` symlink at the new path; verify the
   skill still loads (`/skills`) and its trigger phrases still work.
4. Confirm explicitly with the user before the irreversible steps: archive
   `AI-Sherpa/loop-engineering` on GitHub with a README note pointing at the new location,
   and remove the old local `/opt/dev/loop-engineering` directory.

## Out of scope

- No changes to `ralph-loop` or `loop-engineering`'s existing content beyond the physical
  move (paths in `CLAUDE.md`, symlink target) — its mechanics, safeguards, and `loop.sh`
  stay exactly as they are.
- No new bash/automation scripts for the harness layer (see Content mapping).
- No changes to this repo's (`agentic-research-playbook`) own `PLAYBOOK.md` — it remains
  the source narrative; the harness skill operationalizes it but doesn't replace it here.
