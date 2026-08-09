# AI Sherpa Loop Engineering Harness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a new monorepo, `ai-sherpa-loop-engineering-harness`, containing two
independently-triggerable Claude Code skills — the existing `loop-engineering` (moved in
with history preserved) and a new `harness` skill that operationalizes `PLAYBOOK.md`'s
five layers as the fleet/campaign coordination layer above it.

**Architecture:** Plain-file Claude Code skill package, no build/test tooling. Two skill
subdirectories (`loop-engineering/`, `harness/`), each with its own `SKILL.md`
frontmatter and its own `~/.claude/skills/` symlink. `harness/` pairs one reference doc
with one pre-filled template per PLAYBOOK.md layer, mirroring `loop-engineering`'s own
`references/safeguards.md` + `scripts/loop.sh` pairing.

**Tech Stack:** Markdown only (no code, no scripts, no dependencies) in `harness/`; the
moved `loop-engineering/scripts/loop.sh` (bash) is unchanged content, revalidated with
`bash -n` / `shellcheck` after the move. Git (`git subtree`) for the history-preserving
move. GitHub CLI (`gh`) for repo creation.

## Global Constraints

- Parent repo path: `/opt/dev/ai-sherpa-loop-engineering-harness`, default branch `main`.
- GitHub target: `AI-Sherpa/ai-sherpa-loop-engineering-harness`, visibility `PRIVATE`
  (matches the existing `AI-Sherpa/loop-engineering`'s visibility).
- `loop-engineering/`'s content (SKILL.md body, references, `loop.sh`) does not change
  except the one stale path reference in its `CLAUDE.md` fixed during the move — its
  mechanics, safeguards, and trigger phrases stay exactly as they are.
- `harness/` contains **no bash scripts** — every layer is a discipline plus a file
  format (reference doc + template), not procedural logic.
- Every template in `harness/templates/` ships pre-filled with its schema and one
  worked example — never a blank page.
- Both skills get independent `~/.claude/skills/` symlinks
  (`loop-engineering`, `ai-sherpa-loop-engineering-harness`) so each triggers on its own
  phrasing.
- Parent repo `AGENTS.md` carries the cascaded Must-Dos v3 block verbatim (same text as
  `/opt/dev/loop-engineering/AGENTS.md` and this repo's own `AGENTS.md`); `CLAUDE.md` is
  a single-line `@AGENTS.md` reference, matching this repo's own convention.
- Task 11 (retiring the old `AI-Sherpa/loop-engineering` GitHub repo and deleting
  `/opt/dev/loop-engineering`) is gated on **explicit user confirmation immediately
  before execution** — do not run it as part of an unattended batch, even under
  subagent-driven or inline batch execution.

---

### Task 1: Move `loop-engineering` into the monorepo with history preserved

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/.gitignore`
- Create (via `git subtree`, not hand-written): `/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/**`
- Modify: `/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/CLAUDE.md`

**Interfaces:**
- Consumes: `/opt/dev/loop-engineering` (existing local git repo, branch `main`, untouched
  by this task).
- Produces: `/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/` — the path
  every later task and every subsequent `~/.claude/skills/loop-engineering` symlink
  points at.

- [ ] **Step 1: Initialize the parent repo with a minimal first commit**

```bash
mkdir -p /opt/dev/ai-sherpa-loop-engineering-harness
cd /opt/dev/ai-sherpa-loop-engineering-harness
git init -b main
printf '.DS_Store\n.codegraph/\n' > .gitignore
git add .gitignore
git commit -m "chore: initialize ai-sherpa-loop-engineering-harness monorepo"
```

`git subtree add` requires an existing commit on the target branch to merge into, hence
the standalone `.gitignore` commit first.

- [ ] **Step 2: Subtree-import loop-engineering with full history**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
git subtree add --prefix=loop-engineering /opt/dev/loop-engineering main
```

- [ ] **Step 3: Verify the import — history and files both present**

```bash
git -C /opt/dev/ai-sherpa-loop-engineering-harness log --oneline -- loop-engineering
ls /opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/SKILL.md \
   /opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/scripts/loop.sh
```

Expected: the `log` output includes the original commits (`fix(agents): correct
CodeGraph...`, `chore: migrate to AGENTS.md-first layout...`, `chore: onboard repo
standards...`) plus a new `Add 'loop-engineering/' from commit ...` merge commit; both
`ls` paths resolve with no error.

- [ ] **Step 4: Fix the stale symlink-path reference in the moved CLAUDE.md**

Find this exact text in `loop-engineering/CLAUDE.md`:

```
The repo is installed live by symlink: `~/.claude/skills/loop-engineering -> /opt/dev/loop-engineering`.
Editing files here changes the installed skill immediately. It is not a git repository.
```

Replace it with:

```
The skill is installed live by symlink: `~/.claude/skills/loop-engineering -> /opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering`.
Editing files here changes the installed skill immediately. This directory is part of the `ai-sherpa-loop-engineering-harness` monorepo — there is no nested `.git` here; commits happen at the parent repo root.
```

- [ ] **Step 5: Revalidate the moved script**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
bash -n loop-engineering/scripts/loop.sh
shellcheck -S warning loop-engineering/scripts/loop.sh
```

Expected: both exit 0 with no output (or only pre-existing shellcheck notes, if any —
this script did not change, so any output here should already have existed before the
move).

- [ ] **Step 6: Commit the path fix**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
git add loop-engineering/CLAUDE.md
git commit -m "fix(loop-engineering): update symlink path for new monorepo location"
```

---

### Task 2: Parent repo README, CLAUDE.md, AGENTS.md

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/README.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/CLAUDE.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/AGENTS.md`

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: repo-root documentation referenced by Task 10's `gh repo create` description
  and by anyone cloning the repo.

- [ ] **Step 1: Write README.md**

```markdown
# AI Sherpa Loop Engineering Harness

Two Claude Code skills for running autonomous agent work safely, at any scale:

- **[`loop-engineering/`](loop-engineering/)** — how to run one autonomous loop
  without destroying your machine: isolation, maker/checker verification, budget
  caps, a human merge gate.
- **[`harness/`](harness/)** — how to run *many* of those loops as a standing
  research operation across hundreds of independent targets, distilled from a
  first-place finish (363/363 papers reproduced, +54pt margin) in the Hugging Face
  ICML 2026 Agent Reproducibility hackathon. See
  [agentic-research-playbook](https://github.com/AI-Sherpa/agentic-research-playbook)
  for the narrative this skill operationalizes.

## Install

```bash
ln -s "$(pwd)/loop-engineering" ~/.claude/skills/loop-engineering
ln -s "$(pwd)/harness" ~/.claude/skills/ai-sherpa-loop-engineering-harness
```

Start a new Claude Code session — both skills appear in `/skills`.

## Which one triggers when

`loop-engineering` answers "how do I run **one** loop safely" — it's the mechanics
layer: isolation, a separate verifier, budget caps, a human merge gate. `harness`
answers "how do I run **many** independent targets as a standing operation" — it
assumes a single-loop execution mechanism already exists (`loop-engineering`'s
`loop.sh`, the `/ralph-loop` plugin, or a human) and adds fleet-wide durability, a
compounding knowledge base, a model of the actual grading/evaluation function, and
durable capture of human course-corrections. Use `loop-engineering` alone for a single
bounded task; reach for `harness` once you're coordinating multiple independent
targets over more than one session.
```

- [ ] **Step 2: Write CLAUDE.md**

```markdown
@AGENTS.md
```

- [ ] **Step 3: Write AGENTS.md**

```markdown
# ai-sherpa-loop-engineering-harness

## What this repo is

A monorepo of two independently-triggerable Claude Code skills:

- `loop-engineering/` — safe mechanics for running **one** autonomous Claude Code loop
  (isolation, maker/checker verification, budget caps, human merge gate). Moved here
  from its own repo (`AI-Sherpa/loop-engineering`), history preserved via `git subtree`.
- `harness/` — the fleet/campaign layer above it: coordinating **many** independent
  loops as a standing research operation, distilled from
  [agentic-research-playbook](https://github.com/AI-Sherpa/agentic-research-playbook)'s
  `PLAYBOOK.md`.

There is no build, test, or deploy step — both are Markdown-plus-templates skill
packages. Each is symlinked independently into `~/.claude/skills/` (`loop-engineering`
and `ai-sherpa-loop-engineering-harness`) so each triggers on its own phrasing; editing
either subdirectory changes the installed skill immediately.

## Working conventions

- Keep the two skills' responsibilities separate. `loop-engineering` owns single-loop
  mechanics (isolation, safeguards, `loop.sh`); `harness` owns fleet coordination
  (durability, knowledge base, grader model, human steering). If new content could go
  in either, it almost always belongs in `harness` unless it changes how one loop runs
  safely.
- `harness/SKILL.md` references `loop-engineering` by name rather than restating its
  content — don't let the two drift into duplicating each other.
- Templates in `harness/templates/` ship pre-filled with their schema and one worked
  example, not a blank page — new templates should follow that pattern.
- Validate `loop-engineering/scripts/loop.sh` the same way its own history always has:
  `bash -n loop-engineering/scripts/loop.sh` and
  `shellcheck -S warning loop-engineering/scripts/loop.sh`. `harness/` has no scripts
  to validate this way — its content is checked by grepping for required frontmatter
  and section structure (see each `harness/` task's verify step).

<!-- projindex-cascaded-must-dos v3 -->

## Must-Dos — cascaded standards for every /opt/dev/ repo

These are hard requirements inherited by every project created via projIndex.
Treat them as non-negotiable, not suggestions.

### 1. Secrets live in the macOS Keychain behind Touch ID — never in plaintext

Any API key, token, or secret this project needs MUST be retrieved from the macOS
Keychain via Touch ID (Secure Enclave) — NEVER from a plaintext `.env`, a shell rc,
or a plaintext `security add-generic-password` item.

- Store and retrieve secrets with the Touch-ID-gated pattern from the
  `elevenlabs-touchid-keychain` skill (or `bitwarden-touchid-macos` for
  Bitwarden-backed secrets).
- Never commit secrets. Any `.env*` file holding real values must be gitignored.
- Don't use `SecItemAdd` biometric ACLs — they fail with
  `errSecMissingEntitlement -34018` under an ad-hoc code signature.
- If a secret is ever committed, pushed, or found exposed on disk, ROTATE it
  immediately (once, across every repo sharing it) — rotation is the real fix;
  gitignore entries and history rewrites do not un-leak an exposed key.

### 2. Confirm destructive ops — ask before anything irreversible

Ask the user before `rm -rf`, force-push, history rewrites, schema drops,
dependency removal, bulk overwrites, or anything that affects shared state or
may destroy the only copy of something.

- State the exact scope (paths, count of affected items) when asking.
- Never emit a delete without a paired backup/preservation mechanism when one
  exists (e.g. `--delete` always with `--backup-dir`).
- Never silently auto-apply bulk or regenerative writes — surface the plan
  first and let the user pick the mode.

### 3. Verify before you commit — the repo's gate must pass, and logic changes carry tests

Run the repo's declared verification suite (type-check, lint, format, tests —
whatever the repo's CLAUDE.md/AGENTS.md or CI defines) and require it to pass
before every commit or push that touches logic.

- New or changed logic lands with tests in the same change; "I'll add tests
  later" and "it's a one-liner" are not justifications. If a change is truly
  test-neutral, say so in the commit message.
- Never weaken or disable a failing check (linter, hook, test) to get green —
  fix the code.

### 4. If a change makes anything else stale, update it in the same change

A change is not done while any duplicated copy, derived artifact, or doc that
describes the changed thing is stale.

- Declared duplicates (version in two files, config + its `.example`, copies-not-
  symlinks, parallel generators): apply the edit to EVERY copy in the same commit.
- The repo's instruction file (CLAUDE.md/AGENTS.md) and its designated
  source-of-truth docs: correct any port, status, convention, or architecture
  line your change invalidates, in the same commit.
- Stale instructions poison every future agent session; wrong is worse than missing.

### 5. Record durable learnings in persistent memory when something significant changes

When you learn something a future session would waste time rediscovering — a
non-obvious constraint, the root cause behind a fix, a workflow gotcha, or an
architectural decision that isn't derivable from the code — capture it in this
project's persistent memory (`~/.claude/projects/<project-slug>/memory/`) in the
same session, not just at session end.

- One durable fact per memory file, with a one-line pointer in `MEMORY.md`; update
  the existing file rather than creating a duplicate, and delete memories that turn
  out to be wrong.
- Don't record what the repo already captures — code structure, git history, or
  this instruction file. Record only what was non-obvious.
- A stale or missing memory costs every future agent the same rediscovery; treat
  memory hygiene like doc hygiene (see #4).

<!-- projindex-cascaded-must-dos:end v3 -->
```

- [ ] **Step 4: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
test "$(head -1 CLAUDE.md)" = "@AGENTS.md" && echo "CLAUDE.md OK"
grep -q "Must-Dos" AGENTS.md && echo "AGENTS.md OK"
grep -q "loop-engineering" README.md && grep -q "harness" README.md && echo "README.md OK"
```

Expected: all three `OK` lines print.

- [ ] **Step 5: Commit**

```bash
git add README.md CLAUDE.md AGENTS.md
git commit -m "docs: add parent-repo README, CLAUDE.md, AGENTS.md"
```

---

### Task 3: `harness/SKILL.md`

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/SKILL.md`

**Interfaces:**
- Consumes: nothing from other tasks (it references paths Tasks 4–9 will create, by
  relative path, matching `loop-engineering/SKILL.md`'s own forward-reference style).
- Produces: the skill's trigger frontmatter (`name: ai-sherpa-loop-engineering-harness`)
  and the reference-file index every later task's file must appear in.

- [ ] **Step 1: Write harness/SKILL.md**

```markdown
---
name: ai-sherpa-loop-engineering-harness
description: >
  Coordinate a fleet of agent loops across many independent, adversarially-graded
  targets as a standing research/knowledge-work operation — not a single loop, but
  the operation around dozens or hundreds of them running over days or weeks. Use
  this skill when the user wants to run a large-scale agent campaign, reproduce or
  process many independent targets (papers, tickets, repos, documents), operate
  against an evaluator/grader whose real scoring mechanism isn't fully visible,
  run AFK for days rather than hours, or asks "how do I scale this past one loop?"
  or "how do I run this like a research operation?". Distinct from loop-engineering,
  which handles the mechanics of one safe loop — this skill assumes a single-loop
  execution mechanism already exists (loop-engineering's loop.sh, /ralph-loop, or a
  human) and adds the coordination layer above it: fleet durability, a compounding
  knowledge base, grader-aware strategy, and durable human-steering capture.
---

# AI Sherpa Loop Engineering Harness

This skill coordinates a **fleet of agent loops** running against many independent,
adversarially-graded targets over days or weeks — the operational layer above any
single loop. It was distilled from operating an agent fleet that reproduced 363
independent research targets end-to-end (build, verify, publish, defend against
re-grading) and finished #1 of several hundred entrants. The five layers below are
what separated a system that scaled from one that didn't.

**This skill does not run loops.** It assumes something already does — the
`loop-engineering` skill's `loop.sh`, the `/ralph-loop` plugin, or a human executing
tasks by hand. If no single-loop execution mechanism is in place yet, invoke
`loop-engineering` first to scaffold one; come back here once individual loops are
running safely and the question becomes "how do I run **many** of these as one
coherent operation?"

## Step 0 — Confirm there's a fleet, not a single task

This skill earns its keep at meaningful scale: multiple independent targets running
over more than a single session, especially against an evaluator whose actual
scoring mechanism isn't fully visible up front. A single bounded task with a known
success check belongs in `loop-engineering`, not here. If in doubt, ask: will this
still be running, with state to track, after a session boundary? If no, this skill
is overkill — use `loop-engineering` alone.

## Step 1 — Orchestration: treat the fleet as a standing operation

Background loops and schedulers are frequently session-scoped, not durable — a
context reset or session restart can silently kill every standing job while the
system looks fine right up until nothing is happening. Read
`references/durability.md` for the full failure mode. Scaffold
`templates/JOBS_REGISTRY.md` into the project as the single source of truth for
every standing job's schedule, task, and purpose, and re-verify it against
actually-running jobs after every session boundary — never assume a job that was
running an hour ago still is.

## Step 2 — Production vs. verification: a cheap local replica, and the re-roll rule

The agent that produces work is structurally the worst-positioned to notice what's
wrong with it — `loop-engineering`'s maker/checker safeguard already establishes a
separate verifier. At fleet scale, add two more disciplines: build a cheap local
replica of the real evaluator when it's slow, expensive, or rate-limited, and treat
every edit to already-graded work as a re-roll (full re-verification), not a patch
(delta-only), unless you've confirmed the evaluator only re-checks deltas. Read
`references/verification.md`. Scaffold `templates/EVAL_LOG.md` per evaluated
artifact so a score change after an unrelated edit is recognized as *that*, not
misdiagnosed as a new defect.

## Step 3 — Compounding knowledge base: write findings the moment they're found

Non-obvious causal findings — why something that looked broken wasn't, why a
diagnosis someone else offered was wrong, why a metric that looked damning was
measuring the wrong thing — pay for themselves once and then evaporate unless
written down immediately with the reasoning attached. Read
`references/knowledge-base.md` for the claim/why/how-to-apply structure and the
cross-linking discipline. Scaffold `templates/KNOWLEDGE_LOG.md` and add an entry
the moment a finding is discovered, not at session end.

## Step 4 — Grader-aware strategy: model the actual objective function

A surprising fraction of the largest gains in a scaled campaign come not from
better underlying work but from a better model of how the evaluator actually
computes a score — which is often meaningfully different from what the visible
rubric implies. Read `references/grader-model.md` for the four concrete mechanics
that generalize: the visible target list vs. the actually-scored list, the
per-unit ceiling as a formula rather than a fixed number, additive evidence being
free vs. touching scored evidence being risky, and the bottleneck flipping from
production to shared evaluation throughput near a deadline. Scaffold
`templates/GRADER_MODEL.md` as a living worksheet, and revisit it whenever the
visible rubric and the observed scoring behavior seem to disagree.

## Step 5 — Human steering, captured durably

The biggest strategic pivots in a scaled campaign typically arrive as short,
plain-language directives from the human operator at specific moments — and they
only stay durable if captured as structured state with the reasoning attached, in
the same turn they're given. Read `references/human-steering.md`. Scaffold
`templates/DIRECTIVES_LOG.md` and log every course-correction there, marked
standing or one-off, so later decisions can be judged against intent rather than
followed blindly or ignored.

## Anti-patterns

Read `references/anti-patterns.md` before a campaign scales past a handful of
targets — six recurring failure modes that cost real points/time when they went
unnoticed, none of them specific to any one domain.

## Starter checklist for a new campaign

1. Find the authoritative objective function before scaling effort — not the
   friendliest-looking rubric summary.
2. Confirm a single-loop execution mechanism exists (`loop-engineering` or
   equivalent) before adding the fleet layer on top of it.
3. Stand up all five layers before scaling volume: durability re-arm checks, a
   separate verifier (ideally backed by a cheap local evaluator replica), a
   structured knowledge log, a grader-model worksheet, and a directives log.
4. Log durable findings the moment they're discovered — claim, why, how-to-apply,
   cross-linked to related entries.
5. Treat every edit to already-evaluated work as a re-roll, not a patch, unless
   the evaluator is confirmed to only re-check deltas.
6. Separate diagnostic tracking of externalities (competitors, environment) from
   decision-making — watch continuously, act only on genuine anomalies.
7. Identify the true bottleneck as any deadline nears — usually shared
   evaluation throughput, not production rate — and re-point priority at
   whichever is binding.

## Reference files

- `references/durability.md` — Layer 1: fleet-wide durability and the re-arm check.
- `references/verification.md` — Layer 2: cheap local evaluators and the re-roll rule.
- `references/knowledge-base.md` — Layer 3: the compounding findings log.
- `references/grader-model.md` — Layer 4: modeling the actual objective function.
- `references/human-steering.md` — Layer 5: durable capture of human direction.
- `references/anti-patterns.md` — six generalized failure modes to watch for.
- `templates/` — starter files to scaffold into a user's project, one per layer
  (`JOBS_REGISTRY.md`, `EVAL_LOG.md`, `KNOWLEDGE_LOG.md`, `GRADER_MODEL.md`,
  `DIRECTIVES_LOG.md`).
```

- [ ] **Step 2: Verify frontmatter and reference index**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
test "$(grep -c '^---$' harness/SKILL.md)" = "2" && echo "frontmatter OK"
grep -q "^name: ai-sherpa-loop-engineering-harness$" harness/SKILL.md && echo "name OK"
for f in durability verification knowledge-base grader-model human-steering anti-patterns; do
  grep -q "references/$f.md" harness/SKILL.md || echo "MISSING reference: $f"
done
for f in JOBS_REGISTRY EVAL_LOG KNOWLEDGE_LOG GRADER_MODEL DIRECTIVES_LOG; do
  grep -q "templates/$f.md" harness/SKILL.md || echo "MISSING template: $f"
done
```

Expected: `frontmatter OK`, `name OK`, and no `MISSING` lines.

- [ ] **Step 3: Commit**

```bash
git add harness/SKILL.md
git commit -m "feat(harness): add SKILL.md entry point"
```

---

### Task 4: Layer 1 — durability.md + JOBS_REGISTRY.md

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/references/durability.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/templates/JOBS_REGISTRY.md`

**Interfaces:**
- Consumes: referenced by `harness/SKILL.md` Step 1 (Task 3).
- Produces: `references/durability.md`, `templates/JOBS_REGISTRY.md` — the latter is
  cross-referenced by `references/knowledge-base.md`'s `rules-can-change-mid-project`
  discussion is not required here, but is cross-referenced by
  `references/anti-patterns.md` (Task 9).

- [ ] **Step 1: Write harness/references/durability.md**

```markdown
# Durability — Layer 1

Standing loops don't scale as a single long conversation — running many independent
targets over days or weeks requires background loops: schedulers that discover
work, dispatch it, and monitor state independently of any one interactive session.

## The failure mode

Scheduled/background jobs are frequently **session-scoped, not durable**. A context
reset or session restart can silently kill every standing loop, and the system
looks fine right up until nothing is happening. This is worse than an obvious crash
because there's no error signal — just quiet inactivity that looks identical, from
the outside, to automation that's working perfectly and has nothing to report.

## How to apply

- Keep the *definitions* of every standing job (schedule, exact task, purpose) in
  one versioned document — `templates/JOBS_REGISTRY.md` — that is the single source
  of truth, not just the scheduler's live state.
- After any session boundary (restart, context reset, handoff), **verify durable
  jobs are actually running** before assuming they are. Don't skip the check
  because "they were running an hour ago."
- Treat "is my automation still alive" as a first-class recurring check, not a
  one-time setup step — schedule the check itself if the tooling allows it.
- Build in a way to tell "nothing to report" apart from "nothing is running" — a
  status heartbeat, a last-checked timestamp, anything that makes silence
  diagnosable instead of ambiguous.

## Re-arm checklist (run after every session boundary)

- [ ] List every job `JOBS_REGISTRY.md` says should be running.
- [ ] For each, confirm it's actually running now (not "was running before the
      reset") — check the scheduler's live state, not just the registry.
- [ ] For any job that isn't running, re-arm it or explicitly mark it paused in
      the registry with a reason — never leave the registry silently wrong.
- [ ] Note the re-arm check itself (timestamp, what was found) so a future
      session can tell this check actually happened.

This mirrors `loop-engineering`'s `references/safeguards.md` §7 (Auditability), but
at fleet scope: that safeguard covers one loop's audit trail; this one covers
whether the whole fleet of loops is even still standing.
```

- [ ] **Step 2: Write harness/templates/JOBS_REGISTRY.md**

```markdown
# Jobs Registry

Single source of truth for every standing job in this campaign. Update this file
whenever a job is added, paused, or removed — the scheduler's live state is not the
source of truth, this file is. Re-verify every row against actual running state
after every session boundary (see `references/durability.md`'s re-arm
checklist).

| Job | Schedule | Task | Purpose | Last verified alive | Status |
|---|---|---|---|---|---|
| example-nightly-triage | Daily 02:00 UTC | Run `/loop` over `prd.json`, pick highest-priority incomplete task | Keep backlog burning down without a human present overnight | 2026-08-09T02:05Z | active |

Add one row per standing job. When you re-verify a job after a session boundary,
update its "Last verified alive" timestamp — don't leave a stale one in place,
since a stale timestamp is what lets a dead job look alive.
```

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
grep -q "Re-arm checklist" harness/references/durability.md && echo "durability.md OK"
grep -q "Last verified alive" harness/templates/JOBS_REGISTRY.md && echo "JOBS_REGISTRY.md OK"
grep -iE "TBD|TODO|FIXME" harness/references/durability.md harness/templates/JOBS_REGISTRY.md && echo "FOUND PLACEHOLDER (fix before commit)" || echo "no placeholders OK"
```

Expected: `durability.md OK`, `JOBS_REGISTRY.md OK`, `no placeholders OK`.

- [ ] **Step 4: Commit**

```bash
git add harness/references/durability.md harness/templates/JOBS_REGISTRY.md
git commit -m "feat(harness): add Layer 1 durability reference and template"
```

---

### Task 5: Layer 2 — verification.md + EVAL_LOG.md

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/references/verification.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/templates/EVAL_LOG.md`

**Interfaces:**
- Consumes: referenced by `harness/SKILL.md` Step 2 (Task 3); cross-referenced by
  `references/grader-model.md` (Task 7) and `references/anti-patterns.md` (Task 9).
- Produces: `references/verification.md`, `templates/EVAL_LOG.md`.

- [ ] **Step 1: Write harness/references/verification.md**

```markdown
# Verification — Layer 2

`loop-engineering`'s maker/checker safeguard already establishes the top rule: the
agent that produces work must never be the agent that certifies it as done. At
fleet scale, apply two more disciplines on top of that separation.

## Build a cheap local replica of the real evaluator, early

A build loop that generates work product and a verification loop that checks it
need to be genuinely separate — and when the real evaluator is slow, expensive, or
rate-limited, the checker should be a **cheap local replica of it**, not a wait on
the real thing every iteration. A local replica pays for itself the first time it
catches a second, non-obvious gap that a naive fix would have missed — turning a
partial fix into a complete one for the cost of one local evaluation instead of a
wasted real grading cycle.

## Treat every edit to graded work as a re-roll, not a patch

Nothing that's already "passing" is safe to touch casually. Any edit to a
graded/evaluated artifact can trigger a full re-evaluation against the *current*
rules — and that re-evaluation is not guaranteed to reproduce the old verdict,
even on unchanged content, if the evaluator has any non-determinism or its rules
shifted underneath you.

## How to apply

- Before editing anything already "done," ask: does this system re-derive the
  *whole* verdict on any touch, or only the delta? If whole-verdict, budget for
  full re-verification, not incremental.
- Log every evaluated unit's history in `templates/EVAL_LOG.md` — timestamp,
  verdict, and (if known) which ruleset/evaluator version produced it — so a
  downgrade after an unrelated edit is recognized as *that*, not misdiagnosed as a
  new defect.
- Rules can change mid-project, and old verdicts don't automatically know that.
  Periodically re-validate anything that looks like a stable, banked, "done" state
  against the *current* rules, especially anything graded before a known or
  suspected rule change — don't assume it's still valid because it was valid once.
- Passing a self-consistency check is not the same as matching what the evaluator
  currently rewards; treat "structurally looks right" as necessary, not
  sufficient.
```

- [ ] **Step 2: Write harness/templates/EVAL_LOG.md**

```markdown
# Evaluation Log

History of evaluation outcomes per artifact, so a score change after an unrelated
edit is recognized as a re-roll effect, not misdiagnosed as a new defect (see
`references/verification.md`).

## Format

Each entry: artifact, timestamp, verdict, evaluator/ruleset version (if known), and
what changed since the last entry for that artifact.

## Log

### example-artifact-001
- 2026-08-01T10:00Z — verdict: PASS (score 8/10) — ruleset: v1 — initial submission
- 2026-08-05T14:30Z — verdict: PASS (score 6/10) — ruleset: v2 — no content edit
  made; score drop traced to a ruleset change, not a regression (confirmed via
  `references/knowledge-base.md` entry `rules-can-change-mid-project`)

Add one section per artifact the first time it's evaluated. Append a new bullet
every time it's re-evaluated — including re-evaluations triggered by edits to
*other* artifacts, if the evaluator's whole-verdict behavior makes that possible.
```

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
grep -q "re-roll, not a patch" harness/references/verification.md && echo "verification.md OK"
grep -q "example-artifact-001" harness/templates/EVAL_LOG.md && echo "EVAL_LOG.md OK"
grep -iE "TBD|TODO|FIXME" harness/references/verification.md harness/templates/EVAL_LOG.md && echo "FOUND PLACEHOLDER (fix before commit)" || echo "no placeholders OK"
```

Expected: `verification.md OK`, `EVAL_LOG.md OK`, `no placeholders OK`.

- [ ] **Step 4: Commit**

```bash
git add harness/references/verification.md harness/templates/EVAL_LOG.md
git commit -m "feat(harness): add Layer 2 verification reference and template"
```

---

### Task 6: Layer 3 — knowledge-base.md + KNOWLEDGE_LOG.md

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/references/knowledge-base.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/templates/KNOWLEDGE_LOG.md`

**Interfaces:**
- Consumes: referenced by `harness/SKILL.md` Step 3 (Task 3); its
  `rules-can-change-mid-project` entry is cross-referenced by `templates/EVAL_LOG.md`
  (Task 5, already written — no back-edit needed, the reference is by name only) and by
  `references/anti-patterns.md` (Task 9).
- Produces: `references/knowledge-base.md`, `templates/KNOWLEDGE_LOG.md`.

- [ ] **Step 1: Write harness/references/knowledge-base.md**

```markdown
# Knowledge base — Layer 3

Across a scaled campaign, dozens of non-obvious causal findings accumulate: why
something that looked broken actually wasn't, why a diagnosis someone else offered
was wrong, why a metric that looked damning was measuring the wrong thing.
Individually, each one saves one incident. Written down and cross-linked, they
become a standing diagnostic library — later incidents get pattern-matched against
it in seconds instead of re-diagnosed from scratch.

## How to apply

- Write findings down **the moment they're discovered**, not at session end — the
  "why" gets fuzzy fast, and a fuzzy why produces a memory that's just a fact with
  no diagnostic power.
- Structure each entry in `templates/KNOWLEDGE_LOG.md` as: the claim, **why** it's
  true (the causal mechanism or incident that revealed it), and **how to apply**
  it (when this should change future behavior). The "why" is what lets a future
  reader judge edge cases the original writer never saw.
- Cross-link related entries. The value is disproportionately in the network, not
  any single node — a later finding often only makes sense as a correction or
  refinement of an earlier one.
- Two findings worth watching for specifically, because they generalize
  unusually well across domains:
  - **The fidelity trap** — when reproducing or deriving something and being
    graded on faithfulness, working the target's own native object at its own
    scale reads as genuine; substituting a reduced-scale stand-in reads as a toy
    demo, even when the mechanism is identical and the numbers agree. If a result
    is misjudged as a stand-in, point at the exact native-scale object and show
    the measured quantity matches the target's own stated quantity to high
    precision — don't argue the mechanism is "morally" the same.
  - **Rules changing mid-project** — an evaluator's rules can shift partway
    through, and every artifact graded before the shift keeps displaying a score
    computed under the *old* rules until something forces a re-evaluation (see
    `references/verification.md`).
```

- [ ] **Step 2: Write harness/templates/KNOWLEDGE_LOG.md**

```markdown
# Knowledge Log

Compounding findings, written at discovery time. Every entry: the claim, why it's
true, and how to apply it. Cross-link related entries by name.

## fidelity-trap-native-scale

**Claim:** When graded on faithfulness, working the target's own native object at
its own scale reads as a genuine reproduction; a reduced-scale stand-in reads as a
toy demo, even with excellent numerical agreement.

**Why:** The grader (or a human reviewer) is judging fidelity partly on the object
itself, not only the measured outcome — a stand-in signals "approximation"
regardless of how close the numbers land.

**How to apply:** If a result gets misjudged as a stand-in, point at the exact
native-scale object being reproduced and show the measured quantity matches the
target's own stated quantity to high precision. Don't argue the mechanism is
"morally" the same — show the native-scale artifact.

**Related:** `rules-can-change-mid-project`

## rules-can-change-mid-project

**Claim:** An evaluator's rules can shift partway through a campaign, and every
artifact graded before the shift keeps displaying a score computed under the
*old* rules until something forces a re-evaluation.

**Why:** Most evaluators don't proactively re-score everything on a rule change —
re-scoring is usually triggered by a touch (an edit, a resubmission), so untouched
"done" artifacts silently go stale.

**How to apply:** Periodically re-validate "done" state against current rules,
especially anything graded before a known or suspected rule change. See
`references/verification.md`'s re-roll rule and `templates/EVAL_LOG.md`.

**Related:** `fidelity-trap-native-scale`

---

Add a new entry per finding, the moment it's discovered. Reuse this
claim/why/how-to-apply/related structure exactly — it's what makes an entry
useful to a reader who wasn't there when it was found.
```

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
grep -q "fidelity-trap-native-scale" harness/references/knowledge-base.md && echo "knowledge-base.md OK"
grep -q "## rules-can-change-mid-project" harness/templates/KNOWLEDGE_LOG.md && echo "KNOWLEDGE_LOG.md OK"
grep -iE "TBD|TODO|FIXME" harness/references/knowledge-base.md harness/templates/KNOWLEDGE_LOG.md && echo "FOUND PLACEHOLDER (fix before commit)" || echo "no placeholders OK"
```

Expected: `knowledge-base.md OK`, `KNOWLEDGE_LOG.md OK`, `no placeholders OK`.

- [ ] **Step 4: Commit**

```bash
git add harness/references/knowledge-base.md harness/templates/KNOWLEDGE_LOG.md
git commit -m "feat(harness): add Layer 3 knowledge-base reference and template"
```

---

### Task 7: Layer 4 — grader-model.md + GRADER_MODEL.md

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/references/grader-model.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/templates/GRADER_MODEL.md`

**Interfaces:**
- Consumes: referenced by `harness/SKILL.md` Step 4 (Task 3); cross-references
  `references/verification.md` (Task 5, already written).
- Produces: `references/grader-model.md`, `templates/GRADER_MODEL.md`.

- [ ] **Step 1: Write harness/references/grader-model.md**

```markdown
# Grader-aware strategy — Layer 4

A surprising fraction of the largest point/outcome swings in a scaled campaign
come not from better underlying work but from a better model of **how the
evaluator actually computes a score** — which can be meaningfully different from
what the visible rubric implies.

## Four mechanics that generalize

- **The visible target list and the actually-scored list are not always the same
  thing.** A system can score against an authoritative internal list,
  positionally, while the visibly-declared work items are a separate, looser
  artifact. Sizing effort to the visible list can systematically leave points on
  the table, or waste effort on items that couldn't score at all.
- **The true ceiling per unit of work is a formula, not a fixed number.**
  Knowing the formula lets you identify units where even the previous best
  performer under-shot the real ceiling — turning "match the leader" into "beat
  the leader for free."
- **Additive evidence is free; touching existing scored evidence is risky.**
  Once you understand how the evaluator combines evidence (e.g. concatenation +
  positional mapping), the safe move is almost always to *add* a new unit of
  evidence alongside what already scored, never to edit what's already banked
  (see `references/verification.md`'s re-roll rule).
- **Near a deadline, the bottleneck flips.** Early on, the constraint is
  *producing* work. Near the end, the constraint becomes *shared evaluation
  throughput* — everyone rushes at once, the evaluator backs up, and only work
  already evaluated by the time the clock runs out counts. Shift from
  optimizing volume produced to optimizing volume *already confirmed* by the
  evaluator, well before the crunch.

## How to apply

- Before scaling effort, find the *authoritative* scoring source, not the
  friendliest-looking rubric summary. A gap between the two is where points are
  being left on the table or wasted.
- Model the scoring function as a formula with a real ceiling per unit, and
  actively look for units where current best-in-class work under-shoots that
  ceiling.
- Once you understand how the evaluator combines evidence, prefer additive
  changes to already-scored work over edits to it.
- Identify, explicitly, what the *bottleneck resource* is as any deadline
  approaches (production capacity vs. shared evaluation throughput), and
  re-point priority at whichever one is actually binding.
- Separate diagnostic tracking of externalities (competitors, environment) from
  decision inputs — track continuously, escalate only on genuine anomaly, not
  routine fluctuation. The controllable levers are always on your own side.
- Keep `templates/GRADER_MODEL.md` current as a living worksheet — it's the
  artifact that turns "we think the grader rewards X" into a checked, revisited
  model instead of a one-time guess.
```

- [ ] **Step 2: Write harness/templates/GRADER_MODEL.md**

```markdown
# Grader Model

Living worksheet for the actual objective function — revisit whenever the
visible rubric and observed scoring behavior seem to disagree (see
`references/grader-model.md`).

## Authoritative scoring source

Where the real score comes from (not the friendliest rubric summary). Example:
`Positional match against the evaluator's internal target list; the
publicly-listed task list is a separate, looser artifact and is not what's
scored directly.`

_Fill in for this campaign:_

## Ceiling formula per unit

The real maximum score per unit of work, as a formula, not a guess. Example:
`score(unit) = base_match(0-6) + bonus_native_scale(0-3) + bonus_early_submission(0-1)`

_Fill in for this campaign:_

## Known gaps between visible and actual

Anything discovered where the visible rubric and the real scoring mechanism
diverge. Example: `Visible task list has 40 items; the authoritative internal
list has 47 — sizing effort to the visible 40 leaves 7 unscored.`

_Fill in for this campaign:_

## Additive-safe zones

Where new evidence can be added without risking already-scored work (per the
re-roll rule in `references/verification.md`). Example: `Appending a new
proof unit to an already-submitted artifact is additive; editing an existing
proof unit within it triggers full re-evaluation.`

_Fill in for this campaign:_

## Current bottleneck

Production capacity or shared evaluation throughput — and the date this was
last checked. Example: `2026-08-09: still production-bound, 40% of targets
unstarted.`

_Fill in for this campaign, and update as the deadline approaches._
```

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
grep -q "bottleneck flips" harness/references/grader-model.md && echo "grader-model.md OK"
grep -q "Current bottleneck" harness/templates/GRADER_MODEL.md && echo "GRADER_MODEL.md OK"
grep -iE "TBD:|TODO|FIXME" harness/references/grader-model.md harness/templates/GRADER_MODEL.md && echo "FOUND PLACEHOLDER (fix before commit)" || echo "no placeholders OK"
```

Expected: `grader-model.md OK`, `GRADER_MODEL.md OK`, `no placeholders OK`. (The
template's "_Fill in for this campaign:_" prompts are intentional — they're worksheet
prompts for the user's own campaign, not unfinished plan content; the grep above
checks for bare `TBD`/`TODO`/`FIXME` markers, none of which appear.)

- [ ] **Step 4: Commit**

```bash
git add harness/references/grader-model.md harness/templates/GRADER_MODEL.md
git commit -m "feat(harness): add Layer 4 grader-model reference and template"
```

---

### Task 8: Layer 5 — human-steering.md + DIRECTIVES_LOG.md

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/references/human-steering.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/templates/DIRECTIVES_LOG.md`

**Interfaces:**
- Consumes: referenced by `harness/SKILL.md` Step 5 (Task 3).
- Produces: `references/human-steering.md`, `templates/DIRECTIVES_LOG.md`.

- [ ] **Step 1: Write harness/references/human-steering.md**

```markdown
# Human steering, captured durably — Layer 5

The biggest strategic pivots in a scaled campaign typically aren't discovered
autonomously — they arrive as short, plain-language directives from the human
operator at specific moments ("focus on what we can control," "front-load
scoring before the deadline crunch overwhelms the shared evaluator"). What makes
a directive durable rather than a one-off nudge that evaporates on the next
context reset is capturing it **as structured state, with the reasoning
attached**, and treating it as a standing rule the system keeps re-reading
rather than a comment made once and forgotten.

## How to apply

- When a human gives a course-correction, capture it as durable state — in
  `templates/DIRECTIVES_LOG.md` — in the same turn, not just "I'll remember
  this," which doesn't survive a session boundary.
- Record the *reasoning* behind the directive, not just the instruction, so a
  future decision that seems to conflict with it can be judged on intent rather
  than followed blindly or ignored.
- Distinguish standing directives (keep applying until countermanded) from
  one-time asks — conflating them either makes the system ignore real guidance
  too soon or over-apply a one-off forever. Mark each entry explicitly as one or
  the other.
```

- [ ] **Step 2: Write harness/templates/DIRECTIVES_LOG.md**

```markdown
# Directives Log

Every human course-correction, captured the same turn it's given, with the
reasoning attached (see `references/human-steering.md`).

## Format

Each entry: date, the directive verbatim, the reasoning given (or inferred,
marked as such), standing or one-off, and current status.

## Log

### 2026-08-09 — "Focus on what we can control; track competitors for anomaly detection only, don't react to routine movement"

- **Reasoning:** Competitor standings fluctuate normally; reacting to normal
  fluctuation wastes attention that should go to controllable levers (own
  production speed, own verification quality).
- **Type:** Standing — applies until explicitly countermanded.
- **Status:** Active.

---

Add one entry per directive, in the same turn it's given. Mark type (standing
vs. one-off) explicitly — conflating them either makes the system drop real
guidance too soon or over-apply a one-off forever.
```

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
grep -q "same turn" harness/references/human-steering.md && echo "human-steering.md OK"
grep -q "Type:\*\* Standing" harness/templates/DIRECTIVES_LOG.md && echo "DIRECTIVES_LOG.md OK"
grep -iE "TBD|TODO|FIXME" harness/references/human-steering.md harness/templates/DIRECTIVES_LOG.md && echo "FOUND PLACEHOLDER (fix before commit)" || echo "no placeholders OK"
```

Expected: `human-steering.md OK`, `DIRECTIVES_LOG.md OK`, `no placeholders OK`.

- [ ] **Step 4: Commit**

```bash
git add harness/references/human-steering.md harness/templates/DIRECTIVES_LOG.md
git commit -m "feat(harness): add Layer 5 human-steering reference and template"
```

---

### Task 9: anti-patterns.md

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/references/anti-patterns.md`

**Interfaces:**
- Consumes: referenced by `harness/SKILL.md`'s "Anti-patterns" section (Task 3);
  cross-references `references/verification.md` (Task 5) and `references/durability.md`
  (Task 4), both already written.
- Produces: `references/anti-patterns.md` — the last file `harness/SKILL.md`'s
  reference index points at, completing the full set.

- [ ] **Step 1: Write harness/references/anti-patterns.md**

```markdown
# Anti-patterns observed

Six recurring failure modes from operating a scaled agent campaign. None is
specific to any one domain — watch for all of them as the fleet grows past a
handful of targets.

- **Trusting a stated cause without independently verifying it.** An external
  system's (or even a collaborator's) explanation for *why* something failed is
  a hypothesis, not a fact — verify against primary evidence before acting on
  it.
- **Treating "structurally looks right" as sufficient.** Passing a
  self-consistency check is not the same as matching what the evaluator
  currently rewards; the two can diverge, especially after a rules change
  (`verification.md`).
- **Assuming a past feasibility call stays valid.** "We tried this and it's not
  tractable" ages badly — a peer succeeding at the same target later is a
  strong signal to re-open the question, not defer to the old conclusion.
- **Fitting or reporting the wrong-shaped quantity.** A result can look like it
  demonstrates unbounded growth or an anomaly when the *right* (often
  intensive, normalized) quantity would show a stable constant — always check
  what's actually being measured before drawing a conclusion from its trend.
- **Letting "no news" read as "all fine."** Standing automation that stops
  working silently is indistinguishable, from the outside, from automation
  that's working perfectly and has nothing to report (`durability.md`). Build
  in a way to tell the difference.
- **Narrow-scope negative results.** A result that comes back "inconclusive
  because the range/scope tested is too narrow" is very often literally
  telling the truth — the fix is usually to widen the test, not to look for a
  different bug.
```

- [ ] **Step 2: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
test "$(grep -c '^- \*\*' harness/references/anti-patterns.md)" = "6" && echo "6 anti-patterns OK"
grep -iE "TBD|TODO|FIXME" harness/references/anti-patterns.md && echo "FOUND PLACEHOLDER (fix before commit)" || echo "no placeholders OK"
```

Expected: `6 anti-patterns OK`, `no placeholders OK`.

- [ ] **Step 3: Commit**

```bash
git add harness/references/anti-patterns.md
git commit -m "feat(harness): add anti-patterns reference"
```

---

### Task 10: Create GitHub repo, push, register both skill symlinks, verify

**Files:**
- No new files. Modifies: GitHub (`AI-Sherpa/ai-sherpa-loop-engineering-harness`,
  created), `~/.claude/skills/loop-engineering` (symlink target repointed),
  `~/.claude/skills/ai-sherpa-loop-engineering-harness` (new symlink).

**Interfaces:**
- Consumes: every file created in Tasks 1–9 (this task's push and validation cover the
  complete repo).
- Produces: a live, private GitHub repo and two working skill symlinks — the actual
  deliverable becomes usable after this task.

- [ ] **Step 1: Create the GitHub repo and push**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
gh repo create AI-Sherpa/ai-sherpa-loop-engineering-harness --private \
  --description "Fleet-scale agent-loop coordination (harness/) built on top of safe single-loop mechanics (loop-engineering/)" \
  --source=. --remote=origin
git push -u origin main
```

- [ ] **Step 2: Verify the push**

```bash
gh repo view AI-Sherpa/ai-sherpa-loop-engineering-harness --json visibility,defaultBranchRef
git -C /opt/dev/ai-sherpa-loop-engineering-harness log --oneline origin/main -5
```

Expected: `visibility` is `PRIVATE`; the `log` output matches local `main`'s last 5
commits (no divergence).

- [ ] **Step 3: Repoint the loop-engineering symlink and add the harness symlink**

```bash
rm ~/.claude/skills/loop-engineering
ln -s /opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering ~/.claude/skills/loop-engineering
ln -s /opt/dev/ai-sherpa-loop-engineering-harness/harness ~/.claude/skills/ai-sherpa-loop-engineering-harness
```

- [ ] **Step 4: Verify both symlinks resolve to the new monorepo**

```bash
ls -la ~/.claude/skills/loop-engineering ~/.claude/skills/ai-sherpa-loop-engineering-harness
readlink ~/.claude/skills/loop-engineering
readlink ~/.claude/skills/ai-sherpa-loop-engineering-harness
```

Expected: both `readlink` outputs point inside
`/opt/dev/ai-sherpa-loop-engineering-harness/`.

- [ ] **Step 5: Manual verification (report to user, not automatable in this session)**

Ask the user to start a **new** Claude Code session (skill discovery happens at
session startup, not `/resume`) and run `/skills`, confirming both `loop-engineering`
and `ai-sherpa-loop-engineering-harness` appear in the list. This step cannot be
completed by the implementing agent inside the current session — surface it as a
follow-up for the user rather than claiming it done.

---

### Task 11: Retire the old `loop-engineering` repo — GATED, requires explicit confirmation

**This task must not run as part of an unattended batch.** Stop before Step 1 and ask
the user to explicitly confirm both the local delete and the GitHub archive, restating
the exact paths/repo affected, per this project's destructive-ops rule. Only proceed
once they say yes in this session, after Task 10's verification (Step 5) has been
confirmed working.

**Files:**
- Deletes: `/opt/dev/loop-engineering` (entire directory, its own `.git` history — now
  fully duplicated inside `/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/`
  via Task 1's `git subtree add`).
- Modifies: `AI-Sherpa/loop-engineering` on GitHub (archived, not deleted — GitHub
  archival is reversible; local deletion is not, which is why this task is gated).

**Interfaces:**
- Consumes: Task 10's confirmation that the new symlinks work.
- Produces: nothing further downstream — this is the cleanup step.

- [ ] **Step 1: Ask for explicit confirmation**

State plainly: "This will archive `AI-Sherpa/loop-engineering` on GitHub (reversible —
can be unarchived) and permanently delete the local directory
`/opt/dev/loop-engineering` (irreversible — its full history is preserved inside
`/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/`, but this exact copy
and its `.git` will be gone). Proceed?" Wait for an explicit yes before continuing.

- [ ] **Step 2: Archive the old GitHub repo with a pointer to the new location**

```bash
gh repo edit AI-Sherpa/loop-engineering \
  --description "Moved to AI-Sherpa/ai-sherpa-loop-engineering-harness (loop-engineering/ subdirectory) — see that repo for current content."
gh repo archive AI-Sherpa/loop-engineering --yes
```

- [ ] **Step 3: Remove the old local directory**

```bash
rm -rf /opt/dev/loop-engineering
```

- [ ] **Step 4: Final verification**

```bash
test ! -d /opt/dev/loop-engineering && echo "old local dir removed OK"
gh repo view AI-Sherpa/loop-engineering --json isArchived
ls -la ~/.claude/skills/loop-engineering
readlink ~/.claude/skills/loop-engineering
```

Expected: `old local dir removed OK`; `isArchived` is `true`; the symlink still
resolves cleanly into `/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering`
(unaffected by the old directory's removal, since it was already repointed in Task
10).
