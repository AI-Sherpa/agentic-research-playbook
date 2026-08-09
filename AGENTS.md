# agentic-research-playbook

## What this repository is

A distilled, reusable playbook for operating agentic systems across large numbers of independent, adversarially-graded knowledge-work targets — extracted from a first-place finish (363/363 papers reproduced, +54pt margin) in the Hugging Face ICML 2026 Agent Reproducibility hackathon. See `PLAYBOOK.md` for the framework; `README.md` for the short pitch.

This is primarily a **documentation repo** — there is no build, test, or deploy step. Edits here are almost always prose edits to `PLAYBOOK.md` or `README.md`.

## Working conventions

- Keep the playbook **domain-agnostic**. When adding a new lesson, generalize it past the source competition before it lands here — competition-specific jargon, orids, or tool names belong in the source project's own memory, not in this repo.
- Every generalized claim should trace back to a concrete incident (even if the incident itself isn't named in the text) — don't add generic best-practice advice that isn't grounded in something that actually happened.
- Structure new lessons the same way the existing ones are structured: the claim, why it's true, how to apply it.

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
