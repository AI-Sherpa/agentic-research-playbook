# Design: Self-learning / self-updating loop engineering harness

## Origin

`ai-sherpa-loop-engineering-harness` ships two skills — `loop-engineering` (single-loop
mechanics) and `harness` (fleet-level coordination, built around a compounding
per-campaign knowledge base: `KNOWLEDGE_LOG.md`, `GRADER_MODEL.md`, `DIRECTIVES_LOG.md`).
That knowledge base already compounds *within* one campaign. This spec applies the same
idea one level up: the skills' own reference docs should compound across every campaign
that has used them, the same promotion that turned one hackathon's incidents into
`PLAYBOOK.md` in the first place — automated, ongoing, and gated where it matters.

## Problem

Nothing currently reads the per-campaign artifacts (`KNOWLEDGE_LOG.md` etc. for
`harness`; `progress.md` / `guardrails.md` for `loop-engineering`) that pile up across
every project that scaffolds these skills' templates. Findings that would generalize
past their one campaign — the exact kind of thing `PLAYBOOK.md` is made of — just sit
in scattered project directories forever unless a human manually notices and promotes
one.

## Decision

Add a scheduled **harvest job** (a Claude Code cron routine, not hand-rolled bash —
matching `loop-engineering`'s own "prefer the product's built-in loop pieces"
guidance) that discovers campaign artifacts, classifies new findings, auto-applies the
low-risk additive ones directly, and stages everything else as a small review queue
surfaced on the user's existing `projIndex` dashboard — not a new app, since the user
already has that dashboard open continuously.

## Architecture

```
ai-sherpa-loop-engineering-harness/        (existing monorepo)
├── loop-engineering/
│   └── references/self-learning.md         NEW — explains the mechanism to the agent
├── harness/
│   └── references/self-learning.md         NEW — explains the mechanism to the agent
├── self-learning/                           NEW — operational tooling, not a skill
│   ├── HARVEST_PROMPT.md                    the stable prompt the scheduled cron runs
│   ├── harvested-state.json                 dedup tracking (see Discovery/Dedup below)
│   ├── harvest-log.md                       audit trail: one line per run
│   └── README.md                            mechanism doc for a human maintaining this
└── pending-review/                          NEW — populated at runtime by the harvest job
    └── README.md                            card format doc (committed placeholder)

projIndex/                                   existing repo, EXTENDED not rebuilt
├── server.py                                _build_project() gains pending_reviews count;
│                                             two new routes (list, dismiss)
└── dashboard.html                           buildCard() gains a badge; click-to-launch
                                              via the EXISTING /api/open-claude(prompt=...)
```

## Pipeline

**1. Discovery.** Filesystem scan across `/opt/dev/*` for the known per-campaign
filenames. To avoid false positives on generically-named files (`progress.md` /
`guardrails.md` are common names unrelated projects could also use), a `loop-engineering`
match requires co-location with that skill's own scaffold artifact (`PROMPT.md`, per
its Step 4) in the same directory — not just the filename alone. `harness` artifacts
(`KNOWLEDGE_LOG.md`, `GRADER_MODEL.md`, `DIRECTIVES_LOG.md`) are distinctive enough by
name; a light first-line header check (e.g. `# Knowledge Log`) is enough disambiguation
there.

**2. Dedup.** `self-learning/harvested-state.json` stores one content hash per
discovered source file from its last-processed state. A run only feeds a file's content
to the classification step if its hash changed since last time — and only the new
portion (diffed against the stored hash's snapshot), not the whole file, so the agent
judges genuinely new material each run rather than re-litigating old entries.

**3. Classify.** For each new finding, an agent step judges two questions: does this
generalize past its one campaign, and — if yes — is applying it *additive* (a new
cross-linked entry matching an existing reference doc's structure, the same kind of
change Task 6 made earlier in this session) or does it require *rewriting existing
guidance*.

**4. Auto-apply (additive + generalizable only).** A second pass — a fresh
maker/checker split, same discipline as everywhere else in this system — confirms the
change really is additive (no duplicate of an existing entry, no prose rewrite needed,
passes the same no-placeholder hygiene bar used throughout this repo) before it commits
directly to the target skill's reference doc. Every auto-applied change gets one line
in `self-learning/harvest-log.md`.

**5. Gate everything else.** Writes one `pending-review/<id>.md` per candidate: YAML
frontmatter (`id`, `title`, `skill`, `target` file) + a body that *is* the exact prompt
a Claude session would need to make the edit — same "body is the prompt" convention
`scaffolding-todo-board` already uses elsewhere on this machine. No separate app: these
files are picked up by `projIndex`'s **existing** 60-second project scan
(`scan_directories()` → `_build_project()`), which gains a `pending_reviews` count via
a directory glob — the same shape as the dashboard's already-designed-but-never-wired-up
`.pptx` doc-badge, now actually implemented. A new `GET
/api/project/{name}/pending-reviews` route lists full items on click; a badge renders
next to the existing service/doc badges (reusing the commits-block expand/collapse UI
already in `dashboard.html`). Clicking an item calls the **already prompt-capable**
`/api/open-claude` — no backend change needed there, the wire format already accepts
`{name, prompt}`, only the frontend caller currently omits `prompt`. The launched
session applies the change, verifies it, commits, then deletes its own
`pending-review/<id>.md` — the file's existence *is* the pending state, no separate
tracking needed. A small new `dismiss` route deletes the file without launching
anything, for proposals the user disagrees with.

Because `dashboard.html` already polls `fetchProjects` every 60 seconds, the badge
appears on its own next refresh after a harvest run — no push notification, no
browser-launch step; the user already has the dashboard open, per their own framing of
this requirement.

**Both skills covered.** The classify/apply logic is generic over "which skill + which
reference doc a finding can feed" — it is not `harness`-specific. `loop-engineering`
findings can land as new bullets in `references/safeguards.md` or
`references/when-to-loop.md` exactly the same way `harness` findings land in its five
reference docs.

## Scheduling

A weekly Claude Code cron routine (via the platform's own scheduling primitive, not a
hand-rolled `cron`+bash loop) runs `self-learning/HARVEST_PROMPT.md`. Cadence is
adjustable; weekly is the starting default, not a hard constraint.

## Auditability

`self-learning/harvest-log.md` gets one line per run: date, campaigns scanned, findings
found, auto-applied count, gated count — the same discipline `loop-engineering`'s own
`safeguards.md` §7 already mandates for a single loop, applied to this one.

## Out of scope

- No changes to `loop-engineering`'s or `harness`'s existing safeguards, mechanics, or
  templates beyond the one new `references/self-learning.md` doc each and the small
  addition to each `SKILL.md`'s reference index pointing at it.
- No changes to `projIndex`'s existing services/doc-badge/commit features — this adds
  one new badge type following the same pattern, it does not touch the others.
- No authentication/access-control changes to `projIndex` (it's already localhost-only
  by design, per `register-service`'s conventions) or to the harvest job's write access
  — it runs as the same local user as everything else in this environment.
- No UI for editing a pending-review's proposed prompt before launching it — v1 is
  launch-as-proposed or dismiss, not edit-then-launch. A future iteration could add
  inline editing if that turns out to matter in practice.
