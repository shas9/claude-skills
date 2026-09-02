---
name: chunked-commit-review
description: Split a messy working tree into small, reviewed, independently-valid commits — one logical change at a time, with a human approval gate that is configured once at the start of the session. Use this whenever the user has uncommitted changes they want broken into multiple commits, or says things like "let's go through the diff", "commit these chunk by chunk", "break this into commits", "review my changes before I commit", "clean up history before the PR", or wants to work through a batch of code issues together. Trigger even when the word "chunk" never appears — any request to split, review, or gate commits belongs here. Do NOT use for a single quick commit of everything.
---

# Chunked Commit Review

Take an unstructured pile of working-tree changes and land it as a series of small commits, each one reviewed before it exists.

The point is not tidiness. It is that **every commit on the branch should be independently valid** — it compiles, its tests pass, and `git bisect` can land on it without confusion. That property is what makes the extra ceremony worth it, and it's the thing to protect when a judgment call comes up.

## The contract

Four rules that hold for the entire session. Everything else is procedure.

1. **Never commit outside the approval mode the human chose at the start of the session.** In **manual** mode that means a fresh, explicit approval in the conversation for every single commit — not "seems fine", not "they approved a similar chunk earlier". In **auto** mode a chunk may commit unattended *only* when it comes back completely clean: no findings at any severity, checks green against the baseline, isolation verified. Anything less stops for a fresh approval regardless of mode. See Phase 0.
2. **Never fix anything on your own initiative.** Report findings and stop. The human decides what gets fixed, when, and in which commit. A fix you invented is a change the human never asked to review. This holds in both modes — auto mode auto-*commits*, it never auto-*edits*.
3. **Never edit files during the review phase.** Reading, running tests, and running lint only. If your hands are on the code while you're reviewing it, the diff the human approves isn't the diff you showed them.
4. **Never push, amend, rebase, force, or hard-reset.** This workflow only ever moves forward, one commit at a time.

---

## Phase 0 — Session setup

Do this once, before touching the index.

### Orient

```bash
git status --short --branch
git log --oneline -10
git diff --stat
git diff --cached --stat
```

Read the recent log for house style even though the convention is fixed (below) — scopes, tense, and how granular past commits were are all useful signal.

**If the index is already dirty**, say so and ask what to do before anything else. Silently resetting someone's staged work is the fastest way to lose it.

**Untracked files:** check them against `.gitignore` intent first. Build output, `.env`, editor cruft, and large binaries don't get committed — list them and move on. For genuine new source files, run `git add -N <path>` so they show up in `git diff` and become hunk-stageable later.

### Ask about the branch and the approval mode — once, together

Ask both questions in a single turn and wait for the answer. Never assume either one. Never re-ask the approval question later in the session — one answer covers the whole run.

> Two setup questions before I start:
>
> 1. Current branch is `<name>`. New branch for this work, or commit here?
> 2. Approval mode — **manual** (I stop for your approval on every chunk) or **auto** (I commit any chunk that comes back with no findings, and stop for you on anything that has them)?

If they want a new branch, `git switch -c <name>`.

**Manual mode.** Every chunk gets a report and a full stop. Nothing is committed until the human says so in the conversation.

**Auto mode.** A chunk commits without stopping only when *all* of these hold:

- **Findings: none** — not a single 🔴, 🟡, or ⚪. One nit is enough to stop.
- **Checks green** — no new test failures against the baseline, lint clean.
- **Isolation ran** — the stash isolation actually executed, or was correctly skipped because there was nothing outside the index. A result labeled non-isolated stops.
- **Independence holds** — the commit stands on its own and doesn't depend on a later chunk.

Anything short of that stops and asks, exactly as in manual mode.

Be straight with the human about the trade when they pick auto: "no findings" means *I found nothing*, which is not the same as *there is nothing*. Auto mode trades away a human review gate for speed. It does not weaken the bisect guarantee — isolation and the full checks still run on every chunk — and nothing is ever pushed, so a bad commit remains recoverable with a follow-up.

If the answer lands somewhere in between ("auto, but stop on anything touching auth", "auto for the chore commits"), take it literally and scope it as stated. If the answer is ambiguous, default to manual and say that's what you're doing.

### Permission scope

**One approval covers the whole chunk.** When the human approves a chunk — or picks auto mode and a chunk qualifies — that covers every git operation needed to land it: `git add`, `git add -N`, `git apply --cached`, `git stash push` / `pop`, and the `git commit` itself. Don't come back for a second confirmation between staging and committing, and don't ask permission per command. The approval is for the change, not for each verb.

In auto mode this extends to unattended staging and committing: no per-command prompting at all for chunks that qualify.

This changes nothing about the **Never** list at the bottom. No operation there becomes available because a chunk was approved.

### Baseline the checks

Run the full test suite and the linter **now**, before any staging.

This matters more than it looks. Every chunk gets a full test run, so without a baseline the first pre-existing failure looks like it was caused by the chunk in front of you, and the session stalls arguing about a bug that was already there. Record the result and report it:

> Baseline: 212 passed, 2 failing (`auth.spec.ts`, `queue.spec.ts`) — pre-existing, present before any staging. Lint clean.

If the baseline is red, note which failures are pre-existing and carry that list forward. A chunk is "clean" if it adds no *new* failures.

### Plan the chunks

A chunk is **one logical change**. It may be part of a file, or span several files — whatever keeps a single coherent change together. Related edits stay in one commit even if they're scattered; unrelated edits get split even if they're adjacent in the same file.

Print the plan as a short numbered list, then **go straight into chunk 1**. Don't wait for approval on the plan — approval happens per-chunk, and the plan will change as you learn what's actually in the diff. Say so:

> Plan (will adjust as we go):
> 1. `fix(auth)` — null guard in session lookup
> 2. `feat(api)` — new `/health` endpoint + route registration
> 3. `chore(deps)` — eslint bump and resulting fixes
>
> Starting on chunk 1.

---

## The chunk loop

Repeat until the tree is clean.

### Step 1 — Stage the chunk

`git add -p` needs a TTY and is not usable here. Use one of these instead.

**Whole file**, when the file's entire diff is this one logical change:

```bash
git add path/to/file
```

**Specific hunks**, when a file contains changes belonging to different chunks:

```bash
git diff -- path/to/file > /tmp/chunk.patch
# edit /tmp/chunk.patch: delete the @@ hunks that belong to other chunks
git apply --cached --recount /tmp/chunk.patch
```

Keep the `diff --git`, `index`, `---`, and `+++` headers intact; only remove whole `@@` blocks. `--recount` recalculates the line counts in the hunk headers, so you don't have to keep them consistent by hand — this is what makes the approach reliable rather than fiddly.

**Untracked file, partial:** `git add -N path/to/file` first, then use the patch method above.

### Step 2 — Verify what actually got staged

```bash
git diff --cached          # this is the commit
git diff                   # this is what's left over
```

Confirm the staged set matches the intent, and that nothing leaked in. If `git apply` silently did something unexpected, this is where it surfaces — before the human wastes attention on a diff that isn't real.

### Step 3 — Review (read-only)

Read the staged diff and look for:

**Correctness** — inverted conditions, off-by-one, null/undefined dereference, unhandled error path, resource left open, wrong variable in a copy-pasted block.

**Completeness** — a rename that didn't propagate everywhere, a new branch of logic with no test, a refactor left half-migrated.

**Leftovers** — `console.log`, `print`, `debugger`, `dd()`, commented-out code, a `TODO` introduced by this diff.

**Secrets** — scan added lines specifically for keys, tokens, passwords, `.env` contents, internal hostnames, hardcoded credentials.

**Accidents** — unrelated lockfile churn, build output, `.DS_Store`, editor config, minified bundles, large binaries.

**Independence** — would this commit compile and pass *on its own*? If it depends on a later chunk, it breaks bisect. Flag it and propose merging the two chunks rather than committing something known-broken.

**Scope** — anything here that belongs in a different commit? Say so; don't quietly rearrange it.

### Step 4 — Run the checks against the staged state in isolation

The working tree still holds the other chunks, so testing it as-is tells you nothing about whether *this commit* is valid. Stash everything except the index:

```bash
git stash push --keep-index --include-untracked -m "chunk-isolation"
# run the full test suite
# run the linter
git stash pop
```

**The `git stash pop` is the most dangerous line in this skill.** Run it immediately after the checks finish — before analyzing results, before writing the report, and regardless of whether the tests passed. Then confirm with `git stash list` that the isolation entry is gone and `git status` looks as expected.

If the tree has nothing outside the index (last or only chunk), skip the stash entirely — there's nothing to isolate.

If `git stash pop` conflicts: stop and report it. The stash entry still exists; resolve the conflict manually and never `git stash drop`.

If stashing isn't viable (submodules, a hook that misbehaves), run the checks against the working tree instead and **label the result as non-isolated** in the report so the human knows what they're getting. A non-isolated result never auto-commits.

### Step 5 — Report

Use this shape:

```markdown
## Chunk N — <short title>

**Files** · `path/a.ts` (+12/−3) · `path/b.ts` (+4/−0)

**What it does**
Two or three sentences in plain language. Lead with why, not what — the
diff already says what.

**Proposed commit message**
    fix(auth): guard against missing profile before dereference

**Checks** · Tests: 214 passed · Lint: clean · Isolated: yes

**Findings**
- 🔴 `a.ts:42` — <bug and the consequence it causes>
- 🟡 `b.ts:17` — <risk or unclear intent>
- ⚪ `a.ts:8` — <nit>

**Your call:** approve · fix · re-split · skip
```

Severity: 🔴 will produce incorrect behavior · 🟡 risky or unclear · ⚪ style/nit. Write "Findings: none" when there are none — don't manufacture a finding to look thorough, and don't bury a 🔴 under three nits.

**In manual mode: stop here.** Don't commit, don't edit, don't start the next chunk, don't ask a follow-up question that invites a "yeah go ahead". End the turn.

**In auto mode:** if the report says *Findings: none*, the checks are green against the baseline, and the isolation ran, replace the **Your call** line with `**Auto-committing** — no findings.` and go straight to Step 7, then on into the next chunk in the same turn. Report first, commit second, always — the human should be able to scroll back and see the reasoning that preceded every commit. If there is any finding at any severity, any new check failure, or a skipped or non-isolated check, keep the **Your call** line and stop exactly as in manual mode.

Auto mode is not a reason to under-report. Softening a finding so the chunk can flow through is the one failure mode that makes this entire workflow worthless. If you noticed it, it is a finding, and a finding stops the chunk.

### Step 6 — Act on the reply

**"approve" / "lgtm" / "commit it"** → Step 7.

**"fix X"** → Make only the requested fix. Then re-stage the affected hunks (Step 1), re-verify (Step 2), re-run the checks (Step 4), and **produce a fresh report** (Step 5). A fix is a new diff and gets its own approval — never roll a fix into an already-approved chunk. **A fixed chunk always returns to a full stop, even in auto mode**: the human is already engaged with this one, so show them the corrected diff and get an explicit approval before it lands.

**"re-split"** → `git restore --staged <paths>` to unstage (this keeps the working-tree changes), then re-plan and start again at Step 1.

**"skip"** → Unstage, leave the changes in the tree, note it for the final recap, move to the next chunk.

**Anything ambiguous** → Ask. Approval is the one thing worth a clarifying question.

**"switch to manual" / "stop auto-committing"** → Honor it immediately and for the rest of the session. Going the other way, from manual to auto, is also fine if the human asks for it.

### Step 7 — Commit

```bash
git commit -m "fix(auth): guard against missing profile before dereference"
```

For a body, use repeated `-m` flags or a heredoc. Use the message that was in the report — if the human edited it, use theirs verbatim.

**If a pre-commit hook fails:** never reach for `--no-verify`. Report the hook output and stop. This is a full stop in auto mode too.

**If a hook auto-formats files:** the commit may have gone through with reformatted content, or failed partway. Re-check `git status` and `git diff --cached`, re-stage if needed, and produce a fresh report before trying again. A formatter that rewrites the code after approval means the human approved something that no longer exists — in auto mode, stop and hand this one back.

### Step 8 — Next chunk

Confirm the commit landed (`git log --oneline -1`), then return to Step 1.

---

## Termination

When `git status` is clean — or holds only intentionally skipped changes — stop and give a recap:

```bash
git log --oneline <starting-ref>..HEAD
```

Include: the mode the session ran in and which commits were auto-committed versus explicitly approved, the commits created, anything skipped and why, any findings that were reported but deliberately not acted on, and an explicit reminder that **nothing has been pushed**.

---

## Commit message format — Conventional Commits

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:** `feat` `fix` `docs` `style` `refactor` `perf` `test` `build` `ci` `chore` `revert`

- **subject** — imperative mood ("add", never "added" or "adds"), lowercase after the colon, no trailing period, 72 chars max
- **scope** — optional; the module or area touched
- **body** — only when the *why* isn't obvious from the subject; wrap at 72
- **breaking** — `feat(api)!:` plus a `BREAKING CHANGE:` footer explaining what broke and how to migrate

**Examples**

Input: added a null check before reading `user.profile`
Output: `fix(user): guard against missing profile before dereference`

Input: pulled retry logic out of the http client into its own module
Output: `refactor(http): extract retry logic into separate module`

Input: bumped eslint and fixed the warnings it started emitting
Output: `chore(deps): upgrade eslint and resolve new warnings`

Input: new endpoint, and the old `/status` route now 301s to it
Output:
```
feat(api)!: replace /status with /health

BREAKING CHANGE: /status now redirects to /health. Clients pinning
/status should update; the redirect will be removed in v3.
```

---

## Never

- `git push` in any form
- `git commit --amend` or `--no-verify`
- `git rebase`, `git reset --hard`, `git clean`
- `git checkout .` or `git restore .` without a path — these destroy uncommitted work
- `git stash drop` or `git stash clear`
- Editing files during Steps 2–5
- Committing in manual mode without a fresh, explicit approval
- Committing in auto mode with any finding open, any new check failure, or an unverified isolation
- Skipping the report, the checks, or the isolation because the session is in auto mode
- Merging two chunks into one commit because they *look* related — say they're related in the report and let the human decide

## Recovery

**Stash pop conflicted** — the entry is still in `git stash list`. Resolve manually, report, don't drop it.

**Staged the wrong thing** — `git restore --staged <path>`. Safe; leaves the working tree untouched.

**Committed something that shouldn't have landed** — do not amend or reset. Report it and ask. The fix is almost always a follow-up commit, and that decision belongs to the human. If it landed via auto mode, say so plainly and offer to switch the session to manual.