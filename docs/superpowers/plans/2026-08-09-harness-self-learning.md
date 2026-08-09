# Harness Self-Learning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the weekly harvest job that promotes generalizable per-campaign
findings into `harness`'s and `loop-engineering`'s own reference docs — auto-applying
additive/low-risk changes, gating everything else as a review-queue badge on the
existing `projIndex` dashboard.

**Architecture:** New `self-learning/` tooling (a stable harvest prompt + JSON dedup
state + an append-only log) and a `pending-review/` card directory in the
`ai-sherpa-loop-engineering-harness` monorepo; a matching extension to `projIndex`'s
existing project-card scan (backend: one new field + two new routes; frontend: one new
badge + expand/launch/dismiss). No new services, no new apps — the review surface is
the dashboard the user already has open.

**Tech Stack:** Markdown + JSON for the harness-side tooling (no new dependencies).
Python (stdlib only — no new PyPI dependency) for the `projIndex` backend routes.
Vanilla JS/HTML for the `projIndex` frontend (matches its existing no-framework style).
Claude Code's own scheduling primitive for cadence (not hand-rolled cron+bash).

## Global Constraints

- Repos touched: `/opt/dev/ai-sherpa-loop-engineering-harness` (existing monorepo) and
  `/opt/dev/projIndex` (existing, live, currently-running localhost service).
- Pending-review card format is frozen for this plan: YAML frontmatter `id`, `title`,
  `skill`, `target`, plus a body that IS the literal prompt a launched session follows.
  Filename is `<id>.md`, `id` is `YYYY-MM-DD-<slug>`.
- `pending-review/*.md` is gitignored except `README.md` (ephemeral runtime state, not
  committed). `self-learning/harvest-log.md` and `self-learning/harvested-state.json`
  ARE committed (durable audit trail / dedup state).
- The harvest job's auto-apply and gate targets are restricted to `references/*.md`
  files inside `harness/` and `loop-engineering/` only — never `templates/*`,
  `SKILL.md`, or `scripts/*` in either skill. This scope restriction is stated
  explicitly in `HARVEST_PROMPT.md` itself, not just here.
- `projIndex` is a live service the user has open continuously. Every task touching it
  must verify with the real running server (start/restart it, curl or browser-check the
  actual behavior) and must leave it running and working when the task finishes — never
  leave it stopped or broken without saying so loudly in the task report.
- No new backend dependency: parse the card frontmatter with a small hand-written
  parser, not a new `PyYAML` import, since `projIndex` doesn't currently depend on it.
- The dismiss route deletes a file based on a user-suppliable `id` — it MUST validate
  `id` against a strict pattern (`^[A-Za-z0-9_.-]+$`) and resolve the final path with
  `os.path.realpath` confirmed to still be inside the project's `pending-review/`
  directory before deleting anything. Reuse whatever existing helper `projIndex`
  already uses to resolve a project `name` into a validated filesystem path (used by
  `handle_open_claude`) — do not invent a second, parallel resolution mechanism.
- Task 11 (registering the recurring weekly cron) is gated on explicit user
  confirmation in the same style as the previous plan's Task 11 — do not run it as
  part of an unattended batch.

---

### Task 1: `self-learning/` tooling scaffold

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/self-learning/README.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/self-learning/HARVEST_PROMPT.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/self-learning/harvested-state.json`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/self-learning/harvest-log.md`

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: `HARVEST_PROMPT.md` is what Task 10 dispatches verbatim and what Task 11's
  cron eventually runs on a schedule. `harvested-state.json` and `harvest-log.md` are
  written to by whatever runs `HARVEST_PROMPT.md`, starting from the empty/header state
  this task creates.

- [ ] **Step 1: Write self-learning/README.md**

```markdown
# Self-learning harvest — maintainer notes

This directory is the operational tooling behind the "self-learning" capability
described in each skill's `references/self-learning.md`. It is not a skill itself —
nothing here is loaded by Claude Code's skill discovery.

## What runs, and when

`HARVEST_PROMPT.md` is the stable prompt handed to a Claude Code agent — currently a
weekly scheduled cron routine, set up via Claude Code's own scheduling primitive
(not a hand-rolled cron+bash script, matching `loop-engineering`'s own guidance to
prefer the product's built-in loop pieces). To change cadence, update the schedule
directly rather than editing this prompt.

## The pipeline

1. **Discover** — scan `/opt/dev/*` for `KNOWLEDGE_LOG.md` / `GRADER_MODEL.md` /
   `DIRECTIVES_LOG.md` (harness campaigns) and `progress.md` + `guardrails.md`
   co-located with `PROMPT.md` (loop-engineering campaigns — the co-location
   requirement avoids false positives on unrelated projects that happen to have a
   generically-named `progress.md`).
2. **Dedup** — `harvested-state.json` stores one content hash per discovered file from
   its last-processed state; a run only looks at what changed since then.
3. **Classify** — for each new finding: does it generalize past its one campaign, and
   if so, is applying it additive or does it require rewriting existing guidance.
4. **Auto-apply** — additive + generalizable findings get a second (checker) pass
   confirming no duplication and no required prose rewrite, then commit directly.
5. **Gate** — everything else becomes one `pending-review/<id>.md` card: YAML
   frontmatter (`id`, `title`, `skill`, `target`) + a body that is the literal prompt a
   Claude session needs to make the edit. `projIndex`'s existing project-card scan
   picks these up as a badge; see `pending-review/README.md` for the card format.
6. **Log** — every run appends one row to `harvest-log.md`, regardless of whether it
   found anything.

## Card lifecycle

A `pending-review/<id>.md` file's existence *is* its pending state. When a human
launches it from the dashboard, the resulting Claude session applies the change,
verifies it, commits, and deletes the card itself as its last step. A dismissed card
is just deleted without anything being applied. There is no separate status field to
keep in sync.

## Safety

This job never bypasses the maker/checker split that governs every other write in this
system: the checker pass on the auto-apply path, and the human-launch gate on
everything else. It also never touches `loop-engineering`'s or `harness`'s existing
mechanics/safeguards content directly — only appends new, clearly-scoped entries or
proposes changes for a human to apply.
```

- [ ] **Step 2: Write self-learning/HARVEST_PROMPT.md**

```markdown
# Self-Learning Harvest — Prompt

You are running a scheduled, unattended pass. Work from
`/opt/dev/ai-sherpa-loop-engineering-harness`. Read `self-learning/README.md` first if
you haven't already — it documents this pipeline in full. This prompt is the
per-run checklist; the README is the reference.

## Step 1 — Discover

Scan `/opt/dev/*` (one level of project directories) for:
- `KNOWLEDGE_LOG.md`, `GRADER_MODEL.md`, `DIRECTIVES_LOG.md` anywhere under a project
  (harness campaigns).
- `progress.md` and `guardrails.md`, but ONLY when co-located in the same directory as
  a `PROMPT.md` (loop-engineering campaigns — the co-location requirement is required,
  not optional; it's what distinguishes a real loop-engineering campaign from an
  unrelated project that happens to have a generically-named `progress.md`).

## Step 2 — Dedup

Read `self-learning/harvested-state.json` (a JSON object mapping absolute file path to
a SHA-256 hex digest of that file's content as of the last run it was seen). For each
file found in Step 1: compute its current hash. If the path isn't in the state file, or
its hash changed, that file has new material — read its full current content (these
files are small) and identify what's new. If the hash matches, skip it.

## Step 3 — Classify

For each candidate finding in the new/changed material: does it generalize past its
one campaign (would it hold for a different project using this skill)? If yes, would
applying it to the skill's own reference docs be additive — a new entry matching the
structure an existing doc already uses (see how `references/knowledge-base.md` and
`templates/KNOWLEDGE_LOG.md` structure entries as claim/why/how-to-apply, or how
`references/anti-patterns.md` structures a bulleted list) — or would it require
rewriting existing prose?

Discard findings that don't generalize. Nothing gets written for those — this is not a
log of everything seen, only of what was promoted or proposed.

## Step 4 — Auto-apply (additive + generalizable only)

For each finding classified as additive: before writing anything, re-check it
yourself as a second, skeptical pass — does an equivalent entry already exist in the
target doc (don't duplicate), is it genuinely additive (no existing sentence needs to
change), does it meet the same no-placeholder bar as everything else in this skill (no
TBD/TODO, a real claim with a real reason, not a vague restatement). Only if it
survives that check: add the entry to the correct file — a `references/*.md` file
inside `harness/` or `loop-engineering/` ONLY, never `templates/*`, `SKILL.md`, or
`scripts/*` in either skill — commit it with a message describing what was learned and
from which campaign(s), and note it in this run's harvest-log.md row (Step 6).

## Step 5 — Gate everything else

For every generalizable-but-not-additive finding, and every finding you're genuinely
unsure about: write one `pending-review/<id>.md` card, per the format in
`pending-review/README.md`. The card's body must be a complete, actionable prompt — a
future Claude session reading only that file should be able to apply the change,
verify it, commit it, and delete the card, without needing anything else from you.
Same restriction as Step 4: the proposed target must be a `references/*.md` file in
`harness/` or `loop-engineering/`, never `templates/*`, `SKILL.md`, or `scripts/*`.

## Step 6 — Log

Append one row to `self-learning/harvest-log.md`: today's date, how many campaign
files you scanned, how many findings you classified as generalizable, how many you
auto-applied, how many you gated. Append this row even if every count is zero — a
"nothing found" run is itself information (it means the pipeline ran, not that it's
silently dead — the same durability discipline `references/durability.md` teaches
about any standing job).

## Step 7 — Update dedup state

Write the current content hash for every file you read in Step 1 back into
`self-learning/harvested-state.json`, so the next run only looks at what's new after
this one.

## What NOT to do

- Do not touch anything under a campaign project's own directory except reading it.
- Do not edit `loop-engineering/scripts/loop.sh`, `loop-engineering/SKILL.md`,
  `harness/SKILL.md`, or `harness/templates/*` — only the `references/*.md` files in
  each skill are ever targets of a promotion, additive or gated.
- Do not skip the Step 4 second-pass check to save time — an unchecked auto-apply is
  exactly the failure mode this whole system exists to prevent.
```

- [ ] **Step 3: Write self-learning/harvested-state.json**

```json
{}
```

- [ ] **Step 4: Write self-learning/harvest-log.md**

```markdown
# Harvest Log

Audit trail for the self-learning harvest job — one line per run, appended by the
job itself. See `self-learning/README.md` for what each column means.

| Date | Campaigns scanned | Findings found | Auto-applied | Gated |
|---|---|---|---|---|
```

- [ ] **Step 5: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
test -f self-learning/README.md && test -f self-learning/HARVEST_PROMPT.md && echo "docs OK"
python3 -c "import json; json.load(open('self-learning/harvested-state.json'))" && echo "state.json valid OK"
grep -q "^| Date" self-learning/harvest-log.md && echo "harvest-log.md OK"
grep -iE "TBD|FIXME" self-learning/README.md self-learning/HARVEST_PROMPT.md && echo "FOUND PLACEHOLDER" || echo "no placeholders OK"
```

Expected: `docs OK`, `state.json valid OK`, `harvest-log.md OK`, `no placeholders OK`.
(`TODO` is intentionally excluded from this grep — Step 4's own numbered "Step" list
would otherwise false-positive on nothing here, but double check by eye there is no
literal `TODO`/`TBD`/`FIXME` marker.)

- [ ] **Step 6: Commit**

```bash
git add self-learning/
git commit -m "feat(self-learning): add harvest tooling scaffold"
```

---

### Task 2: `pending-review/` scaffold

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/pending-review/README.md`
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/pending-review/.gitignore`

**Interfaces:**
- Consumes: nothing from other tasks.
- Produces: the card-format contract every other task in this plan depends on —
  `HARVEST_PROMPT.md` (Task 1) writes cards in this format, `projIndex`'s new routes
  (Tasks 6-7) parse and delete them in this format, Task 9's integration test hand-crafts
  one in this format.

- [ ] **Step 1: Write pending-review/README.md**

```markdown
# Pending Review

Populated at runtime by the self-learning harvest job (see
`../self-learning/README.md`). Files here (other than this README) are not committed —
see `.gitignore` in this directory.

## Card format

One file per candidate change: `<id>.md`, where `<id>` is `YYYY-MM-DD-<slug>`.

```markdown
---
id: 2026-08-16-example-slug
title: Short human-readable summary shown on the dashboard
skill: harness
target: references/anti-patterns.md
---

The exact prompt a Claude Code session should follow to apply this change: what
file to edit, what to add, and how to verify it before committing. This body IS the
prompt — nothing else gets prepended to it when a card is launched.
```

- `skill` is `harness` or `loop-engineering` — which skill's reference docs this
  proposes changing.
- `target` is the path (relative to that skill's root) the proposed change touches.
- The card is deleted by the launched session itself once the change is applied,
  verified, and committed — not by the dashboard or the harvest job.
```

- [ ] **Step 2: Write pending-review/.gitignore**

```
*.md
!README.md
```

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
test -f pending-review/README.md && test -f pending-review/.gitignore && echo "scaffold OK"
git check-ignore pending-review/some-test-card.md && echo "gitignore works OK"
```

Expected: `scaffold OK`, `gitignore works OK`.

- [ ] **Step 4: Commit**

```bash
git add -f pending-review/README.md pending-review/.gitignore
git commit -m "feat(self-learning): add pending-review card format + scaffold"
```

(`-f` on the `add` is needed only because the directory's `.gitignore` would otherwise
also make `git add pending-review/` ambiguous about intent — the two files being added
here are explicitly named, so this is safe and does not override the ignore rule for
any `*.md` card.)

---

### Task 3: harness self-learning.md + SKILL.md update

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/references/self-learning.md`
- Modify: `/opt/dev/ai-sherpa-loop-engineering-harness/harness/SKILL.md`

**Interfaces:**
- Consumes: Task 1's `self-learning/README.md` (referenced by path).
- Produces: the agent-facing explanation a campaign-running agent sees when it
  encounters this skill; referenced from `harness/SKILL.md`'s reference index.

- [ ] **Step 1: Write harness/references/self-learning.md**

```markdown
# Self-learning — how this skill updates itself

Layer 3 (`references/knowledge-base.md`) already compounds findings within one
campaign. This is the same idea one level up: a scheduled harvest job periodically
reads every campaign's `KNOWLEDGE_LOG.md`, `GRADER_MODEL.md`, and `DIRECTIVES_LOG.md`
across this machine, and promotes findings that generalize past their one campaign
back into this skill's own reference docs — the same promotion that turned one
hackathon's incidents into the source playbook this skill is built from.

## What this means for how you write campaign artifacts

Write `KNOWLEDGE_LOG.md` entries the way `references/knowledge-base.md` already
prescribes — claim, why, how-to-apply, cross-linked — not because it helps only this
campaign, but because the harvest job classifies findings by reading exactly that
structure. A vague entry harvests poorly or not at all; a well-formed one is a
candidate for promotion into every future campaign that uses this skill.

## What happens to a finding after it's written

- If the harvest job judges a finding generalizes and applying it is purely additive
  (a new cross-linked entry, matching how an existing reference doc is already
  structured), it gets applied directly — verified by a second pass before it commits,
  the same maker/checker split every other verification in this system uses.
- If applying it would mean rewriting existing guidance, or its generalizability is
  ambiguous, it's staged instead: a small proposal file lands in this repo's
  `pending-review/` directory, and shows up as a badge on the project's card in the
  `projIndex` dashboard. A human reviews it there — launch to apply, or dismiss.
- Nothing about this changes how you use the skill day to day. Write findings the
  moment they're discovered, as `references/knowledge-base.md` already says; the
  harvest is a separate, scheduled process that reads what you've already written.

See `/opt/dev/ai-sherpa-loop-engineering-harness/self-learning/README.md` for the full
mechanism (discovery, dedup, classification, scheduling) if you're maintaining the
harvest job itself rather than just running a campaign.
```

- [ ] **Step 2: Update harness/SKILL.md's reference-files list**

Find this exact block (the last section of the file):

```
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

Replace it with:

```
## Reference files

- `references/durability.md` — Layer 1: fleet-wide durability and the re-arm check.
- `references/verification.md` — Layer 2: cheap local evaluators and the re-roll rule.
- `references/knowledge-base.md` — Layer 3: the compounding findings log.
- `references/grader-model.md` — Layer 4: modeling the actual objective function.
- `references/human-steering.md` — Layer 5: durable capture of human direction.
- `references/anti-patterns.md` — six generalized failure modes to watch for.
- `references/self-learning.md` — how this skill's own reference docs evolve from
  campaign findings across every project that uses it.
- `templates/` — starter files to scaffold into a user's project, one per layer
  (`JOBS_REGISTRY.md`, `EVAL_LOG.md`, `KNOWLEDGE_LOG.md`, `GRADER_MODEL.md`,
  `DIRECTIVES_LOG.md`).

## Self-learning

This skill's own reference docs compound the same way a campaign's knowledge log
does, just across campaigns instead of within one — see `references/self-learning.md`.
You don't need to do anything differently; write `KNOWLEDGE_LOG.md` entries well and
the rest is automatic.
```

If the file's actual current content differs from the "Find this exact block" text
above in any way (even whitespace), read the live file first and locate the real
`## Reference files` section by its heading rather than assuming the block above is
byte-exact — then make the equivalent addition (the new bullet plus the new
`## Self-learning` section) in place.

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
grep -q "references/self-learning.md" harness/SKILL.md && echo "SKILL.md reference OK"
grep -q "## Self-learning" harness/SKILL.md && echo "SKILL.md section OK"
grep -q "compounds findings within one" harness/references/self-learning.md && echo "self-learning.md OK"
grep -iE "TBD|TODO|FIXME" harness/references/self-learning.md && echo "FOUND PLACEHOLDER" || echo "no placeholders OK"
```

Expected: all four `OK` lines.

- [ ] **Step 4: Commit**

```bash
git add harness/references/self-learning.md harness/SKILL.md
git commit -m "feat(harness): document the self-learning mechanism"
```

---

### Task 4: loop-engineering self-learning.md + SKILL.md update

**Files:**
- Create: `/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/references/self-learning.md`
- Modify: `/opt/dev/ai-sherpa-loop-engineering-harness/loop-engineering/SKILL.md`

**Interfaces:**
- Consumes: Task 1's `self-learning/README.md` (referenced by path).
- Produces: the agent-facing explanation, referenced from `loop-engineering/SKILL.md`'s
  reference index.

- [ ] **Step 1: Write loop-engineering/references/self-learning.md**

```markdown
# Self-learning — how this skill updates itself

This skill's safeguards and guidance are meant to be stable — but they're also
observed, across every loop that runs, by a scheduled harvest job shared with the
sibling `ai-sherpa-loop-engineering-harness` skill. It periodically reads every
project's `progress.md` and `guardrails.md` (the per-iteration learnings and the
"signs" appended after a failure, per `SKILL.md` Step 4) and promotes findings that
generalize past one project back into `references/safeguards.md` or
`references/when-to-loop.md`.

## What this means for how you write progress.md / guardrails.md

Write a `guardrails.md` entry the way it's meant to be used elsewhere — the specific,
concrete "sign" that would have prevented the failure ("check imports exist before
adding"), not a vague restatement of what went wrong. A sign that only makes sense
inside its own project doesn't generalize and won't be promoted; a sign about how
loops fail in general is exactly the harvest's target.

## What happens to a finding after it's written

- Purely additive, clearly-generalizable findings (a new bullet matching an existing
  safeguard's structure) get applied directly, verified by a second pass before
  committing — the same maker/checker discipline this skill already mandates for every
  loop's own verification step.
- Anything that would mean rewriting existing safeguard prose, or whose
  generalizability is ambiguous, is staged as a proposal in
  `ai-sherpa-loop-engineering-harness/pending-review/` and surfaces as a badge on the
  `projIndex` dashboard for a human to launch or dismiss.
- This does not relax the human-merge-gate rule for your own loop's work — it only
  applies to this skill's own documentation evolving from what many loops, across many
  projects, have already shown to be true.

See `/opt/dev/ai-sherpa-loop-engineering-harness/self-learning/README.md` for the full
mechanism if you're maintaining the harvest job itself.
```

- [ ] **Step 2: Update loop-engineering/SKILL.md's reference-files list**

Find this exact block (the last section of the file):

```
## Reference files

- `references/when-to-loop.md` — full decision guide: optimal vs. poor-fit tasks, with
  examples, and the greenfield/brownfield distinction.
- `references/safeguards.md` — detailed safeguards: isolation tiers, budget caps,
  verification rubrics, merge gates, and the failure modes each one prevents.
- `scripts/loop.sh` — reference loop runner with sandbox detection and iteration caps.
```

Replace it with:

```
## Reference files

- `references/when-to-loop.md` — full decision guide: optimal vs. poor-fit tasks, with
  examples, and the greenfield/brownfield distinction.
- `references/safeguards.md` — detailed safeguards: isolation tiers, budget caps,
  verification rubrics, merge gates, and the failure modes each one prevents.
- `references/self-learning.md` — how this skill's own safeguards/guidance evolve from
  `progress.md` / `guardrails.md` findings across every project that uses it.
- `scripts/loop.sh` — reference loop runner with sandbox detection and iteration caps.
```

If the file's actual current content differs from the "Find this exact block" text
above in any way, read the live file first and locate the real `## Reference files`
section by its heading rather than assuming the block above is byte-exact, then make
the equivalent one-line addition in place.

- [ ] **Step 3: Verify**

```bash
cd /opt/dev/ai-sherpa-loop-engineering-harness
grep -q "references/self-learning.md" loop-engineering/SKILL.md && echo "SKILL.md reference OK"
grep -q "safeguards and guidance are meant to be stable" loop-engineering/references/self-learning.md && echo "self-learning.md OK"
grep -iE "TBD|TODO|FIXME" loop-engineering/references/self-learning.md && echo "FOUND PLACEHOLDER" || echo "no placeholders OK"
```

Expected: all three `OK` lines.

- [ ] **Step 4: Commit**

```bash
git add loop-engineering/references/self-learning.md loop-engineering/SKILL.md
git commit -m "feat(loop-engineering): document the self-learning mechanism"
```

---

### Task 5: projIndex backend — pending_reviews count

**Files:**
- Modify: `/opt/dev/projIndex/server.py` (the `_build_project()` function, reported at
  approximately lines 671-702 — confirm the real current line numbers by reading the
  file, do not assume the report's line numbers are still exact)

**Interfaces:**
- Consumes: `pending-review/*.md` files (Task 2's format) inside a project's directory.
- Produces: a `pendingReviews` integer key in the dict `_build_project()` returns,
  which Task 8's frontend code reads as `p.pendingReviews`.

- [ ] **Step 1: Read the current implementation**

Read `/opt/dev/projIndex/server.py` around `_build_project()` (search for
`def _build_project`) and confirm its exact signature, return dict shape, and the
imports already present at the top of the file (specifically whether `os` is already
imported — it almost certainly is, given the rest of the file's behavior described in
prior investigation).

- [ ] **Step 2: Add the pending_reviews count**

Inside `_build_project(name, full_path, catalog, sum_cache, archived)`, before the
`return {...}` statement, add:

```python
pending_review_dir = os.path.join(full_path, "pending-review")
pending_reviews = 0
if os.path.isdir(pending_review_dir):
    pending_reviews = len([
        f for f in os.listdir(pending_review_dir)
        if f.endswith(".md") and f != "README.md"
    ])
```

Add `"pendingReviews": pending_reviews,` as a new key in the returned dict, alongside
the existing keys (`name`, `lang`, `desc`, `cat`, `sizeKB`, `lastMod`, `isNew`,
`archived`, `hasClaudeMD`, `onGithub`, `githubUrl`) — match the existing dict's
formatting style exactly (trailing commas, quoting) rather than introducing a different
style.

- [ ] **Step 3: Verify with the real server**

```bash
cd /opt/dev/projIndex
python3 -m py_compile server.py && echo "syntax OK"
```

Then start (or restart, if already running) the server using whatever mechanism this
repo's own docs/README specify, and confirm via:

```bash
curl -s http://localhost:48721/api/projects | python3 -c "import json,sys; d=json.load(sys.stdin); print('pendingReviews' in d[0] if d else 'no projects returned')"
```

Expected: `syntax OK`, then `True` (or manually confirm the key is present on at least
one project's entry in the JSON if the one-liner's exact shape of `/api/projects`
turns out to differ from a bare list — read the actual response first if the one-liner
errors, and adapt the check rather than skipping verification).

Create a throwaway `pending-review/` directory with one dummy `.md` file inside
`/opt/dev/ai-sherpa-loop-engineering-harness/` (a project `projIndex` already scans)
before running the curl check, to confirm the count is genuinely non-zero and not just
defaulting to `0` by coincidence. Remove the dummy file afterward
(`pending-review/*.md` is gitignored, so this is disk-only cleanup, not a git concern).

- [ ] **Step 4: Commit**

```bash
cd /opt/dev/projIndex
git add server.py
git commit -m "feat(dashboard): add pendingReviews count to project cards"
```

---

### Task 6: projIndex backend — list route

**Files:**
- Modify: `/opt/dev/projIndex/server.py` (new handler function + new route dispatch
  entry in `do_GET`, reported at approximately lines 1089-1109 — confirm real current
  line numbers by reading the file)

**Interfaces:**
- Consumes: Task 5's `pending_review_dir` convention; Task 2's card format; whatever
  existing helper resolves a project `name` string to a validated filesystem path
  (used by `handle_open_claude` — find and reuse it, per Global Constraints).
- Produces: `GET /api/project/{name}/pending-reviews` → `{"items": [{id, title, skill,
  target, prompt}, ...]}`, consumed by Task 8's frontend `refreshReviewList()`.

- [ ] **Step 1: Read the current implementation**

Read `handle_open_claude` (search for `def handle_open_claude`) specifically to find
how it turns a `name` string into a validated project path — this is the exact
resolution logic/helper to reuse here, not a new one. Also read the `do_GET` method's
dispatch style (the `elif self.path == "/api/x":` pattern) to match it precisely, and
confirm whether `urllib.parse` (for `unquote`) is already imported at the top of the
file.

- [ ] **Step 2: Add a small frontmatter parser**

Add this function near other small parsing helpers in the file (or near the new route
handler if there's no clear existing "helpers" section):

```python
def _parse_pending_review_card(path):
    """Parse a pending-review markdown card: '---'-delimited frontmatter + prompt body."""
    with open(path, "r", encoding="utf-8") as f:
        content = f.read()
    parts = content.split("---\n")
    if len(parts) < 3:
        return None
    frontmatter_text = parts[1]
    body = "---\n".join(parts[2:]).strip()
    meta = {}
    for line in frontmatter_text.splitlines():
        line = line.strip()
        if not line or ":" not in line:
            continue
        key, _, value = line.partition(":")
        meta[key.strip()] = value.strip()
    return {
        "id": meta.get("id", os.path.splitext(os.path.basename(path))[0]),
        "title": meta.get("title", ""),
        "skill": meta.get("skill", ""),
        "target": meta.get("target", ""),
        "prompt": body,
    }
```

- [ ] **Step 3: Add the route handler**

```python
def handle_pending_reviews(self, name):
    project_path = self._resolve_project_path(name)  # use the REAL helper name found in Step 1
    if project_path is None:
        self.send_json_response({"error": "unknown project"}, status=404)
        return
    pending_dir = os.path.join(project_path, "pending-review")
    items = []
    if os.path.isdir(pending_dir):
        for fname in sorted(os.listdir(pending_dir)):
            if not fname.endswith(".md") or fname == "README.md":
                continue
            card = _parse_pending_review_card(os.path.join(pending_dir, fname))
            if card:
                items.append(card)
    self.send_json_response({"items": items})
```

`self._resolve_project_path(name)` is written here as the call signature to use, but
its NAME must be replaced with whatever Step 1's reading of `handle_open_claude`
revealed the real helper to be called — this codebase already has a working
name-to-path resolver with path-traversal protection built in; the requirement is to
call that exact existing method, not to write a second one. If its real failure
signature differs from "returns `None`" (e.g. it raises, or returns a falsy value some
other way), adapt the `if project_path is None:` check to match its actual behavior
rather than assuming this shape blindly.

- [ ] **Step 4: Wire the route into `do_GET`**

Add, matching the file's existing `elif self.path == ...:` dispatch style:

```python
elif self.path.startswith("/api/project/") and self.path.endswith("/pending-reviews"):
    parts = self.path.split("/")
    if len(parts) == 5:
        name = urllib.parse.unquote(parts[3])
        self.handle_pending_reviews(name)
    else:
        self.send_error(404)
```

Place it alongside the other `/api/project/...` routes if any exist, otherwise
alongside `/api/projects` for locality. If `urllib.parse` isn't already imported,
add `import urllib.parse` to the file's existing import block, following its existing
import style (grouped stdlib imports, alphabetical if that's the existing convention).

- [ ] **Step 5: Verify with the real server**

```bash
cd /opt/dev/projIndex
python3 -m py_compile server.py && echo "syntax OK"
```

Restart the server, create a real test card at
`/opt/dev/ai-sherpa-loop-engineering-harness/pending-review/2026-08-16-test-card.md`
with valid frontmatter (`id`, `title`, `skill: harness`, `target:
references/anti-patterns.md`) and a short prompt body, then:

```bash
curl -s "http://localhost:48721/api/project/ai-sherpa-loop-engineering-harness/pending-reviews" | python3 -m json.tool
```

Expected: JSON with `"items"` containing exactly one object whose `id`, `title`,
`skill`, `target`, and `prompt` match what you wrote. Remove the test card file
afterward.

- [ ] **Step 6: Commit**

```bash
cd /opt/dev/projIndex
git add server.py
git commit -m "feat(dashboard): add GET /api/project/{name}/pending-reviews"
```

---

### Task 7: projIndex backend — dismiss route

**Files:**
- Modify: `/opt/dev/projIndex/server.py` (new handler function + new route dispatch
  entry in `do_POST`)

**Interfaces:**
- Consumes: same project-path-resolution helper as Task 6.
- Produces: `POST /api/project/{name}/pending-reviews/{id}/dismiss` → `{"ok": true}`,
  called by Task 8's frontend `dismissPendingReview()`.

- [ ] **Step 1: Read the current implementation**

Read the `do_POST` method's dispatch style to match it exactly (same
`elif self.path == ...:` pattern as `do_GET`, per prior investigation).

- [ ] **Step 2: Add the route handler**

```python
import re  # add to the top-level imports if not already present

_SAFE_ID_RE = re.compile(r"^[A-Za-z0-9_.-]+$")

def handle_dismiss_pending_review(self, name, item_id):
    if not _SAFE_ID_RE.match(item_id):
        self.send_json_response({"error": "invalid id"}, status=400)
        return
    project_path = self._resolve_project_path(name)  # same real helper as Task 6
    if project_path is None:
        self.send_json_response({"error": "unknown project"}, status=404)
        return
    pending_dir = os.path.realpath(os.path.join(project_path, "pending-review"))
    target = os.path.realpath(os.path.join(pending_dir, item_id + ".md"))
    if not target.startswith(pending_dir + os.sep):
        self.send_json_response({"error": "invalid path"}, status=400)
        return
    if os.path.basename(target) == "README.md":
        self.send_json_response({"error": "cannot delete README"}, status=400)
        return
    if os.path.exists(target):
        os.remove(target)
    self.send_json_response({"ok": True})
```

This is a security-sensitive handler (it deletes a file based on user-suppliable
input) — do not simplify away the `_SAFE_ID_RE` check or the `os.path.realpath` +
`startswith` containment check. Both must survive review.

- [ ] **Step 3: Wire the route into `do_POST`**

```python
elif self.path.startswith("/api/project/") and "/pending-reviews/" in self.path and self.path.endswith("/dismiss"):
    parts = self.path.split("/")
    if len(parts) == 7:
        name = urllib.parse.unquote(parts[3])
        item_id = urllib.parse.unquote(parts[5])
        self.handle_dismiss_pending_review(name, item_id)
    else:
        self.send_error(404)
```

- [ ] **Step 4: Verify with the real server**

```bash
cd /opt/dev/projIndex
python3 -m py_compile server.py && echo "syntax OK"
```

Restart the server. Create a real test card (same as Task 6 Step 5), confirm it's
listed via the Task 6 route, then:

```bash
curl -s -X POST "http://localhost:48721/api/project/ai-sherpa-loop-engineering-harness/pending-reviews/2026-08-16-test-card/dismiss"
ls /opt/dev/ai-sherpa-loop-engineering-harness/pending-review/2026-08-16-test-card.md 2>&1
```

Expected: the POST returns `{"ok": true}` and the subsequent `ls` reports "No such file
or directory". Then explicitly test the security checks:

```bash
curl -s -X POST "http://localhost:48721/api/project/ai-sherpa-loop-engineering-harness/pending-reviews/../../../etc/passwd/dismiss"
curl -s -X POST "http://localhost:48721/api/project/ai-sherpa-loop-engineering-harness/pending-reviews/README/dismiss"
```

Expected: both return an error JSON (invalid id / cannot delete README), and
`/opt/dev/ai-sherpa-loop-engineering-harness/pending-review/README.md` still exists
afterward — verify with `ls` that it does.

- [ ] **Step 5: Commit**

```bash
cd /opt/dev/projIndex
git add server.py
git commit -m "feat(dashboard): add POST pending-reviews dismiss route"
```

---

### Task 8: projIndex frontend — badge, list, launch/dismiss

**Files:**
- Modify: `/opt/dev/projIndex/dashboard.html` (the `buildCard()` function reported at
  approximately line 1295, the doc-badge rendering reported at approximately lines
  1485-1502, the `openClaude(name, tier)` function reported at approximately lines
  2215-2228 — confirm real current line numbers/content by reading the file first)

**Interfaces:**
- Consumes: `p.pendingReviews` (Task 5), `GET .../pending-reviews` (Task 6),
  `POST .../dismiss` (Task 7), the existing `POST /api/open-claude` (already accepts
  an optional `prompt` field, confirmed in the design's prior investigation — no
  backend change needed for this).
- Produces: the visible badge + review UI Task 9 tests end-to-end in a browser.

- [ ] **Step 1: Read the current implementation**

Read `buildCard()`, the doc-badge block near it, the commits-block expand/collapse
code (reported around similar lines, used as the UI pattern to reuse), and the current
`openClaude()` function's exact signature and POST body construction.

- [ ] **Step 2: Add CSS**

Add near the existing `.svc-badge` / doc-badge CSS rules:

```css
.review-badge {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 2px 8px;
  border-radius: 10px;
  background: #fef3c7;
  color: #92400e;
  font-size: 0.75rem;
  cursor: pointer;
}
.review-badge:hover { background: #fde68a; }
.review-list { display: none; margin-top: 6px; }
.review-list.expanded { display: block; }
.review-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 4px 8px;
  font-size: 0.8rem;
  border-top: 1px solid #eee;
}
.review-item button { margin-left: 6px; font-size: 0.75rem; }
```

Match the existing stylesheet's formatting conventions (indentation, whether rules are
grouped by component) rather than appending in a different style.

- [ ] **Step 3: Add the badge-building and review-list JS**

```javascript
function escapeHtml(s) {
  return String(s).replace(/[&<>"']/g, c => ({
    '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;'
  }[c]));
}

function buildReviewBadge(p) {
  if (!p.pendingReviews) return '';
  return `<span class="review-badge" onclick="toggleReviewList(event, '${p.name}')">
    🔍 ${p.pendingReviews} to review
  </span>
  <div class="review-list" id="review-list-${p.name}"></div>`;
}

function toggleReviewList(evt, name) {
  evt.stopPropagation();
  const el = document.getElementById(`review-list-${name}`);
  if (!el) return;
  if (el.classList.contains('expanded')) {
    el.classList.remove('expanded');
    return;
  }
  el.classList.add('expanded');
  refreshReviewList(name);
}

async function refreshReviewList(name) {
  const el = document.getElementById(`review-list-${name}`);
  if (!el) return;
  el.innerHTML = '<div class="review-item">Loading…</div>';
  try {
    const res = await fetch(`/api/project/${encodeURIComponent(name)}/pending-reviews`);
    const data = await res.json();
    const items = data.items || [];
    el.dataset.items = JSON.stringify(items);
    if (!items.length) {
      el.innerHTML = '<div class="review-item">Nothing pending</div>';
      return;
    }
    el.innerHTML = items.map(item => `
      <div class="review-item">
        <span>${escapeHtml(item.title || item.id)}</span>
        <span>
          <button onclick="launchPendingReview(event, '${name}', '${item.id}')">Launch</button>
          <button onclick="dismissPendingReview(event, '${name}', '${item.id}')">Dismiss</button>
        </span>
      </div>
    `).join('');
  } catch (e) {
    el.innerHTML = '<div class="review-item">Failed to load</div>';
  }
}

async function launchPendingReview(evt, name, id) {
  evt.stopPropagation();
  const el = document.getElementById(`review-list-${name}`);
  const items = el && el.dataset.items ? JSON.parse(el.dataset.items) : [];
  const item = items.find(i => i.id === id);
  if (!item) return;
  await openClaude(name, undefined, item.prompt);
}

async function dismissPendingReview(evt, name, id) {
  evt.stopPropagation();
  await fetch(`/api/project/${encodeURIComponent(name)}/pending-reviews/${encodeURIComponent(id)}/dismiss`, { method: 'POST' });
  await refreshReviewList(name);
  fetchProjects();
}
```

`item.id` values are safe to interpolate directly into the `onclick` attribute strings
above because the backend's `_SAFE_ID_RE` (Task 7) already constrains what an `id` can
be — but `item.title` is NOT constrained that way, hence `escapeHtml()` around it.
Keep that distinction; do not add `escapeHtml()` around `item.id` interpolations
(unnecessary) and do not remove it from the `title` interpolation (necessary).

- [ ] **Step 4: Wire the badge into `buildCard()` and extend `openClaude()`**

In `buildCard()`, find where the existing doc-badge block is rendered into the card's
HTML string and add `buildReviewBadge(p)`'s output alongside it, following the exact
string-concatenation/template-literal style already used there.

Modify the existing `openClaude(name, tier)` function to accept an optional third
parameter and include it in the request body when present, without changing behavior
for existing callers that don't pass it:

```javascript
async function openClaude(name, tier, prompt) {
  const body = { name };
  if (tier !== undefined) body.tier = tier;
  if (prompt !== undefined) body.prompt = prompt;
  // ...keep the rest of the function's existing fetch/POST logic exactly as it is,
  // just build `body` this way instead of whatever literal object it currently sends.
}
```

Read the function's real current body first — if it does more than build and POST a
body object (e.g. UI state updates before/after the call), preserve all of that;
only change how the request body is constructed.

- [ ] **Step 5: Verify**

```bash
cd /opt/dev/projIndex
node --check <(sed -n '/<script>/,/<\/script>/p' dashboard.html | sed '1d;$d') 2>&1 || echo "node not available or check failed — fall back to manual review"
```

(This is a best-effort syntax check — if `node` isn't installed or the sed extraction
doesn't cleanly isolate the script, don't block on it; Task 9's real browser test is
the authoritative verification for this task.)

Restart the server, then manually load `http://localhost:48721` in a browser (or use
whatever tooling Task 9 will use) far enough to visually confirm no JS console errors
on page load, and that a project with a `pending-review/*.md` file present (create one
temporarily, as in Task 6) shows the badge. Remove the temporary card afterward — Task
9 creates its own for the full interactive test.

- [ ] **Step 6: Commit**

```bash
cd /opt/dev/projIndex
git add dashboard.html
git commit -m "feat(dashboard): render pending-review badge with launch/dismiss"
```

---

### Task 9: End-to-end browser integration test

**Files:**
- No new files. Creates and removes one temporary test card at
  `/opt/dev/ai-sherpa-loop-engineering-harness/pending-review/2026-08-16-integration-test.md`.

**Interfaces:**
- Consumes: every route and UI element from Tasks 5-8, together, for the first time.
- Produces: confidence the whole chain works before Task 10 runs the real harvest job
  against it.

- [ ] **Step 1: Create a real test card**

```markdown
---
id: 2026-08-16-integration-test
title: Integration test card — safe to dismiss
skill: harness
target: references/anti-patterns.md
---

This is a test card created by the Task 9 integration test. If you are a Claude
session that was launched by clicking "Launch" on this card in the projIndex
dashboard: do nothing except delete this file
(`pending-review/2026-08-16-integration-test.md`) and report that the launch mechanism
works end-to-end. Do not edit any other file.
```

Write this to `/opt/dev/ai-sherpa-loop-engineering-harness/pending-review/2026-08-16-integration-test.md`.

- [ ] **Step 2: Confirm the badge appears**

Restart (or confirm running) the `projIndex` server. Load `http://localhost:48721` in
a real browser using the `claude-in-chrome` browser tools available in this
environment (load them via `ToolSearch` if not already loaded — see this session's own
system-reminder on Claude in Chrome tool loading). Navigate to the
`ai-sherpa-loop-engineering-harness` project's card and take a screenshot or read the
page to confirm a "1 to review" badge is visible.

- [ ] **Step 3: Confirm the list expands correctly**

Click the badge. Confirm the review list expands and shows one item titled
"Integration test card — safe to dismiss" with Launch and Dismiss buttons.

- [ ] **Step 4: Confirm dismiss works**

Click Dismiss. Confirm the item disappears from the list and the badge either
disappears or updates to reflect zero pending items on the next `fetchProjects` poll
(you may need to wait up to 60 seconds for the poll, or trigger a manual refresh if the
page supports one — check `dashboard.html` for a manual refresh affordance first).
Confirm via `ls` that the file `pending-review/2026-08-16-integration-test.md` no
longer exists on disk.

- [ ] **Step 5: Confirm launch sends the right request (without necessarily letting it fully complete)**

Re-create the same test card. Use the browser tools' network-inspection capability
(`read_network_requests`, load via `ToolSearch` if needed) to click Launch and confirm
the outgoing `POST /api/open-claude` request body contains `{"name":
"ai-sherpa-loop-engineering-harness", "prompt": "<the card's body text>"}` — the exact
prompt content should match what Step 1 wrote as the card body. If a new Ghostty window
actually opens as a result (this is expected, harmless, real behavior — spawning a
Claude Code session is not destructive by itself), let it run; if it applies the card's
instructions (delete the test card, report success) that's a stronger positive signal
than the network-request check alone, but is not required for this task to pass —
the network-request confirmation is the primary bar. Clean up the test card manually
afterward if it's still present (either via the dashboard's Dismiss button or by
deleting the file directly).

- [ ] **Step 6: Report**

No commit for this task (nothing new to commit — it's a live-system integration test).
Write a brief report to
`/opt/dev/agentic-research-playbook/.superpowers/sdd/2026-08-09-harness-self-learning/task-9-report.md`
covering what was confirmed at each step, including any screenshots/observations from
the browser tools.

---

### Task 10: HITL manual harvest run

**Files:**
- Creates and removes a temporary fixture directory:
  `/opt/dev/.self-learning-test-fixture/` (containing `KNOWLEDGE_LOG.md`, matching the
  harness template's header, with two entries).
- Reads/writes: `self-learning/harvested-state.json`, `self-learning/harvest-log.md`,
  and (if the auto-apply path triggers) `harness/references/anti-patterns.md`, plus
  writes one `pending-review/*.md` card (if the gate path triggers).

**Interfaces:**
- Consumes: Task 1's `HARVEST_PROMPT.md`, run verbatim for the first time against real
  (synthetic-but-realistic) data; Task 9's now-verified dashboard UI, to visually
  confirm a real (not hand-crafted) gated card renders correctly.
- Produces: the first real `harvest-log.md` row and (if applicable) the first real
  auto-applied commit and/or real pending-review card.

- [ ] **Step 1: Create a synthetic test campaign fixture**

Write `/opt/dev/.self-learning-test-fixture/KNOWLEDGE_LOG.md`:

```markdown
# Knowledge Log

Compounding findings, written at discovery time. Every entry: the claim, why it's
true, and how to apply it. Cross-link related entries by name.

## silent-partial-write-looks-complete

**Claim:** A write operation that returns success after writing only part of a
multi-file update leaves the system in a state that looks complete from the outside
(no error surfaced) but is actually inconsistent, and nothing about the success
response distinguishes this from a genuine full write.

**Why:** Success/failure was only checked at the level of "did the write call return
without throwing," not "did every file the operation was supposed to touch actually
get the update" — a partial write and a full write return identically.

**How to apply:** Any operation that's supposed to update N files atomically needs a
post-write check that all N actually changed, not just that the write call didn't
error. This generalizes past this specific campaign; it's a shape of bug, not a
one-off.

**Related:** none yet

## our-vendor-api-rate-limit-is-47-per-minute

**Claim:** The specific third-party vendor API used in this campaign's data pipeline
enforces a rate limit of 47 requests per minute, not the 60 documented in their public
docs.

**Why:** Discovered empirically after repeated 429s at request 48 in a 60-second
window, confirmed by contacting vendor support who acknowledged the docs are stale for
this endpoint.

**How to apply:** Throttle this specific vendor's client to 45 req/min as a safety
margin. This is specific to this one vendor integration and does not generalize to
other campaigns.

**Related:** none yet
```

The first entry is deliberately shaped to generalize (a bug pattern that would apply to
any campaign) and additively fit `references/anti-patterns.md`'s existing bulleted
structure. The second is deliberately campaign-specific (a single vendor's actual rate
limit) and should NOT be promoted anywhere — it exists in this fixture specifically to
verify the harvest correctly discards non-generalizable findings rather than promoting
everything it sees.

- [ ] **Step 2: Dispatch the harvest**

Run `self-learning/HARVEST_PROMPT.md` verbatim (read the file and follow it exactly —
this is the same content a scheduled cron run would receive) from
`/opt/dev/ai-sherpa-loop-engineering-harness`.

- [ ] **Step 3: Verify the results**

- Confirm `self-learning/harvest-log.md` has one new row with today's date and
  plausible counts (at least 1 campaign scanned, at least 1 finding found).
- Confirm the first (generalizable) finding resulted in EITHER a new bullet in
  `harness/references/anti-patterns.md` (auto-applied) OR a new
  `pending-review/*.md` card proposing that addition (gated) — either outcome is
  acceptable and depends on the harvest's own judgment call at Step 3 of the prompt
  about whether it counted as cleanly additive; what's NOT acceptable is the finding
  being silently dropped with no trace in either place.
- Confirm the second (campaign-specific) finding produced NEITHER a doc edit NOR a
  pending-review card anywhere.
- If a pending-review card was created, load the `projIndex` dashboard (reusing Task
  9's browser-tool approach) and confirm it renders correctly as a real badge — this is
  the first real, non-hand-crafted card exercising Task 8's UI.
- Confirm `self-learning/harvested-state.json` now contains an entry for the fixture's
  `KNOWLEDGE_LOG.md` path.

- [ ] **Step 4: Clean up the fixture**

```bash
rm -rf /opt/dev/.self-learning-test-fixture
```

Remove that path's entry from `self-learning/harvested-state.json` (edit the JSON
directly). If a pending-review card was created for the fixture's finding, dismiss it
via the dashboard (or delete the file directly) rather than launching it — it was a
test, not a real proposal for the actual skill content. If an auto-apply commit
happened, evaluate whether the added anti-pattern bullet is something worth actually
keeping (it describes a real, generalizable bug shape, so keeping it is likely
correct) or reverting (`git revert` the specific commit) if judged not to add real
value on reflection — use judgment here and record which choice was made and why in
the report.

- [ ] **Step 5: Report**

Write a report to
`/opt/dev/agentic-research-playbook/.superpowers/sdd/2026-08-09-harness-self-learning/task-10-report.md`
covering the full run: what was discovered, how each finding was classified, what
happened to each, the harvest-log.md row, and the fixture cleanup outcome. No
task-level commit beyond whatever Step 3/4 already produced (the harvest's own commit,
if any, plus the fixture-cleanup housekeeping — commit the `harvested-state.json`
cleanup and any anti-patterns.md revert if one happened).

---

### Task 11: GATED — register the weekly recurring harvest

**This task must not run as part of an unattended batch.** Stop before Step 1 and ask
the user to explicitly confirm before registering a recurring, unattended automation
that will commit to this repo and write dashboard-visible files on a schedule going
forward, indefinitely, without further human involvement to start each run (though
every write it makes still passes through the same auto-apply checker / human-gate
split as every other task in this plan).

**Files:** None — this is a scheduling registration, not a file change.

**Interfaces:**
- Consumes: Task 1's `HARVEST_PROMPT.md`, Task 10's confirmation that a live run
  behaves correctly.
- Produces: a recurring Claude Code cron routine.

- [ ] **Step 1: Ask for explicit confirmation**

State plainly: "This will register a weekly recurring Claude Code cron routine that
runs `self-learning/HARVEST_PROMPT.md` unattended, going forward indefinitely. It can
commit directly to `ai-sherpa-loop-engineering-harness` (auto-apply path) and will
write files visible on your projIndex dashboard (gate path) on its own schedule, with
no further confirmation before each individual run starts — though every write still
passes through the checker/human-gate split the prompt itself enforces. Proceed?" Wait
for an explicit yes before continuing.

- [ ] **Step 2: Load the scheduling tool**

Load the `CronCreate` tool's schema (`ToolSearch` with `query: "select:CronCreate"`) —
it is a deferred tool not loaded by default.

- [ ] **Step 3: Register the routine**

Using `CronCreate`'s actual parameters (read its loaded schema — do not guess field
names), register a weekly routine whose prompt is the full content of
`self-learning/HARVEST_PROMPT.md` (read the file fresh at registration time, don't
paste a stale copy), working directory
`/opt/dev/ai-sherpa-loop-engineering-harness`, and a cadence of once weekly. Choose a
specific day/time (any reasonable default, e.g. Sunday 03:00 local time, avoiding
collision with any other known scheduled job on this machine if `CronList` shows one)
and state the chosen cadence back to the user in your final report.

- [ ] **Step 4: Verify registration**

Run `CronList` (load via `ToolSearch` if needed) and confirm the new routine appears
with the correct prompt reference, working directory, and schedule.

- [ ] **Step 5: Report**

State the registered schedule, confirm `CronList` shows it, and note that the first
scheduled run will happen at the next occurrence of the chosen day/time — it does not
run immediately upon registration (Task 10 already validated the prompt's behavior
manually, so an immediate first run is not necessary here).
