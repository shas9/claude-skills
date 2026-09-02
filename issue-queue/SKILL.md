---
name: issue-queue
description: Work through a batch of pasted issues, code-review comments, bug reports, or audit findings one at a time — triaging by severity, validating each against the current code, confirming with the user before implementing, verifying, and committing in small chunks. Use this whenever the user pastes multiple issues, review comments, a QA list, or a findings dump and wants them addressed, fixed, or worked through — even if they don't say "one at a time" or name a process. Also use when the user says things like "address these review comments", "here are the bugs, fix them", "let's go through these issues", or asks to plan and fix a group of problems together.
---

# Issue Queue

A disciplined loop for working through a batch of issues without scope creep, silent
assumptions, or giant unreviewable commits. The user stays in control of *what* gets
fixed and *when* it lands; you own triage, validation, implementation, and verification.

The core rhythm: **triage once → then one issue at a time, gated by the user.**

## Phase 0 — Intake and triage

Run this once, before touching any code.

1. Parse the pasted content into discrete, atomic issues. Split anything that bundles
   two unrelated problems; merge exact duplicates. If an item is too vague to act on,
   keep it but mark it `needs-clarification`.
2. Locate each one in the codebase. Read the actual code — do not trust the issue text's
   description of what the code does. This is what makes the validity check in Phase 1
   meaningful.
3. Assign severity:
   - **Critical** — data loss, security hole, crash, broken build, prod breakage
   - **Major** — wrong behavior, significant bug, missing error handling
   - **Minor** — edge cases, small correctness nits, missing tests
   - **Trivial** — naming, formatting, comments, style
4. Note dependencies. If issue B's fix touches code that issue A rewrites, order A first
   and say so. Dependency order overrides severity order when they conflict — call this
   out when it happens.
5. Detect the verification commands (see Phase 5) and record them.
6. Write `ISSUES.md` at the repo root.
7. Show the user the full ranked list as a compact table (ID, severity, one-line summary,
   validity) plus any ordering exceptions. Then go to Phase 1.

Keep the table short — this is an overview, not the detail. Detail comes one issue at a
time.

### ISSUES.md format

This file is the durable state. If context is lost, reading it must be enough to resume.
Update it at every state transition — not in a batch at the end.

```markdown
# Issue Queue

Verify: `<test cmd>` | `<lint cmd>` | `<build cmd>`
Branch: <branch> | Started: <date>

| ID | Severity | Summary | Location | Validity | Status |
|----|----------|---------|----------|----------|--------|
| 1  | Critical | Token refresh races on concurrent requests | src/auth/session.ts:88 | Valid | Done — a1b2c3d |
| 2  | Major    | Missing null guard on user lookup | src/api/users.ts:41 | Valid | In progress |
| 3  | Major    | Retry loop has no backoff | src/net/client.ts:120 | Stale — fixed in 9f8e7d6 | Skipped |
| 4  | Minor    | Inconsistent error shape | src/api/errors.ts | Valid | Pending |

## Deferred / declined
- #3 — user confirmed already fixed upstream, no action.

## Discovered during work
- Config loader silently swallows parse errors (found while fixing #2). Not yet raised.
```

Status values: `Pending`, `In progress`, `Awaiting review`, `Done — <sha>`, `Skipped`,
`Deferred`.

## Phase 1 — Present one issue

Present exactly one issue, the highest-ranked `Pending` one. Never present two, and never
start work while presenting.

Use this structure:

```
### Issue #N — <one-line summary>  [Severity]

**Location:** path/to/file.ext:LINE (plus other affected paths)
**Validity:** <verdict + evidence>
**What's wrong:** <the actual defect, in terms of the code as it exists now>
**Why it matters:** <concrete consequence — what breaks, for whom, when>
**Proposed fix:** <the approach, not the diff>
**Blast radius:** <files touched, callers affected, migration/API/schema impact>
**Size:** <Small — inline fix | Large — needs a plan>

Proceed?
```

### The validity check

Issues arrive stale, second-hand, or based on a misreading. Checking validity before the
user spends attention on the issue is the single highest-value part of this loop. Verdict
is one of:

- **Valid** — reproduces against current code; cite the line that proves it.
- **Partially valid** — the concern is real but the described cause or scope is wrong.
  State what's actually true and what the fix should really cover.
- **Stale** — already fixed or refactored away. Cite the commit or current code and
  recommend skipping.
- **Invalid** — rests on a wrong assumption about the code, framework, or intent.
  Explain the assumption and why it doesn't hold.
- **Out of scope** — real, but belongs to a different system, a dependency, or a
  deliberate design decision.

Present non-valid issues too — do not silently drop them. The user may still want the
change, and they may want to know the reviewer was wrong. Lead with the verdict and
recommend an action; keep these brief.

## Phase 2 — Wait for the response

Stop after presenting. Then:

- **Confirmed** → go to Phase 3.
- **Confirmed with extra guidance** → treat the guidance as overriding your proposed fix.
  If it conflicts with something you found in Phase 0, say so once, then follow their
  call.
- **Declined / skipped** → record it under *Deferred / declined* in `ISSUES.md` **with
  the stated reason**, mark status `Skipped`, and present the next issue. The reason
  matters: it stops the same issue resurfacing and explains the gap to whoever reads the
  file later.
- **Deferred to later** → status `Deferred`, move to the end of the queue rather than out
  of it.

## Phase 3 — Plan, if the issue is large

An issue is **large** if the fix touches more than one file, or changes more than ~50
lines, or alters a public API, schema, or config contract.

Small issues: skip straight to Phase 4.

Large issues: write a short plan first — ordered steps, files touched per step, the
verification approach, and anything genuinely uncertain framed as a question rather than
an assumption. **Wait for plan approval before writing code.** A rejected plan costs one
message; a rejected implementation costs a rewrite.

## Phase 4 — Implement

Fix the issue that was approved. Nothing else.

The scope fence:

- No fixing other queued issues along the way, even if a one-line change would do it.
  They each get their own gate.
- No opportunistic refactors, renames, reformatting, dependency bumps, or "while I'm in
  here" cleanups.
- No touching files outside the blast radius you presented. If the fix turns out to need
  them, that's a scope change — stop and say so.

**When you discover a new issue mid-fix, stop and ask.** Describe what you found, whether
the current fix depends on it, and offer the options: fold it in now, queue it as a new
issue, or ignore it. Log it under *Discovered during work* either way. Do not silently
expand the fix, and do not silently ignore something that makes the current fix wrong.

If partway through the fix reveals the approach was wrong, stop and re-present rather
than improvising a different design.

## Phase 5 — Verify before handing over

Never hand work to the user unverified. Run, in order: tests → lint/typecheck → build.

Detect the commands in Phase 0 from the repo itself — `package.json` scripts, `Makefile`,
`pyproject.toml`, `Cargo.toml`, `go.mod`, CI config, contributor docs. Record them in
`ISSUES.md` so later issues reuse them. If nothing is detectable, ask the user once and
record the answer.

Prefer scoping tests to the affected area when the full suite is slow, but run the full
suite before the final commit of a session.

If verification fails: fix it and re-run. If it fails for reasons unrelated to your change
(pre-existing breakage, flaky test, missing env), say so explicitly rather than fixing it
under this issue's scope — that's a new issue.

## Phase 6 — Hand over for review

Summarize. Do not paste full diffs.

```
### Issue #N implemented

**Changed:**
- path/to/file.ext — <what changed, one line>
- path/to/other.ext — <what changed, one line>

**Approach:** <one or two sentences, only if it differs from what was proposed>
**Verification:** tests ✅ 142 passed | lint ✅ | build ✅
**Notes:** <anything the reviewer should look at closely, or "none">

Ready for your review.
```

Then stop. The user may ask to see specific diffs — show those on request. Address review
feedback, re-verify, and re-present. Loop until they approve.

## Phase 7 — Commit after approval

Commit only once the user has approved the implementation. Never commit unreviewed work.

- One commit per logical change. If the fix has genuinely separable parts — e.g. a
  behavior fix and its test, or a refactor that enables the fix — split them into
  sequential commits rather than one blob.
- Stage deliberately. Never `git add -A` or `git add .`; add the specific paths you
  changed so unrelated working-tree noise doesn't ride along.
- Conventional Commits format: `type(scope): subject`, with `fix`, `feat`, `refactor`,
  `test`, `docs`, `chore`, `perf`. Imperative mood, no trailing period. Body only when
  the *why* isn't obvious from the subject.
- Reference the issue if the source had an ID (`Refs #412`, `Closes #412`).
- **Never push, open a PR, force-push, rebase, amend a pushed commit, or reset without
  asking.** Committing locally is the granted permission; publishing is not.
- If the current branch is `main` or `master`, flag it before the first commit and ask
  whether to branch.

Example:

```
fix(auth): guard token refresh against concurrent requests

Two in-flight requests could both trigger a refresh, invalidating
the first token before its response was consumed.

Refs #412
```

## Phase 8 — Next

After the commit lands, update `ISSUES.md` (status `Done — <sha>`) and immediately present
the next issue per Phase 1. No pause, no "shall I continue?" — the confirmation gate in
Phase 2 already gives the user their stopping point.

When the queue is empty:

1. Run the full verification suite once more.
2. Give a closing summary: issues fixed, skipped (with reasons), and anything logged under
   *Discovered during work* that was never raised — this list is easy to lose, so surface
   it explicitly.
3. Delete `ISSUES.md` and commit the deletion (`chore: remove issue queue tracking file`).
   Only delete once every issue is `Done`, `Skipped`, or `Deferred`. If anything is still
   `Pending` or `In progress`, keep the file and say why.

## Always

- One issue in flight. Ever.
- Read the code before believing the issue.
- Ask rather than assume — an unasked question becomes a wasted implementation.
- Update `ISSUES.md` as you go, not retroactively.
- Report failures and dead ends plainly. A verification failure you mention is a minor
  setback; one you paper over is a bug the user ships.
