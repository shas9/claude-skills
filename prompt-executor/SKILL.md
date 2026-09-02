---
name: prompt-executor
description: >
  ONLY invoke this skill when the user's message literally begins with the
  trigger "PE:" (e.g. "PE: go") or is exactly the command
  "/claude-prompt-execute". Do NOT invoke based on general requests to
  "run this" or "do it" that don't use the literal trigger. If the literal
  trigger is not present at the start of the user's message, this skill
  must not activate.
---

# Prompt Executor

You are the executor half of a two-stage workflow. Your only job is to find
the most recently generated prompt in this conversation and carry it out
in full. You do not design or edit the prompt — that already happened in a
separate `prompt-architect` invocation.

---

## 1. Locate the prompt

Search backward through this conversation for the most recent fenced code
block whose first line is `GENERATED PROMPT`. Use that block, in full,
exactly as written. Do not summarize it, do not paraphrase it, do not
"clean it up."

If no such block exists anywhere earlier in the conversation, stop and tell
the user plainly that no generated prompt was found to execute — do not
guess at what they might have meant instead.

## 2. Read the prompt's own instructions

The generated prompt will typically contain:
- A `Task type:` line — adopt the implied working mode (e.g. treat a
  `coding` task type as a real implementation task, a `research-analysis`
  type as an investigation task).
- `Context:` / `Scope:` sections — treat scope as your primary working
  boundary; don't wander outside it without good reason.
- `Requirements & constraints:` — follow these as hard constraints, not
  suggestions.
- `Task-specific instructions:` — follow the specific process it lays out
  (e.g. root-cause-then-fix for a bugfix, plan-then-implement for a
  feature, structured critique for a review).
- `Output format:` — match this exactly in how you present your result.

## 3. Execute

- If the prompt references specific files or paths, read/inspect them
  before proceeding.
- If the prompt implies a persona (developer, reviewer, analyst), adopt it
  for the duration of this task.
- If the prompt calls for tests-first or verification steps, follow that
  sequence rather than skipping to the end state.
- Do NOT ask for confirmation before starting — the confirmation step
  already happened implicitly when the prompt was generated and approved
  by the user asking you to execute it now.
- Do NOT restate or summarize the plan before doing it. Go straight to
  execution.
- Do NOT deviate from the prompt's stated scope or requirements. If
  something in the prompt turns out to be infeasible or clearly wrong once
  you're in the work (e.g. a referenced file doesn't exist), say so plainly
  rather than silently improvising a large departure from scope.

## 4. Close out

When finished, output a closing note of 3 lines or fewer:
- What was done
- Which files/areas changed (or what was produced, for non-code tasks)
- Any immediate follow-up required or open question

Do not pad this with additional commentary, caveats, or a restatement of
the whole task.

---

## Note on tool-use safety

This skill inherits whatever tool permissions are normally in effect on
this surface (file edits, bash, etc.) — nothing about being invoked via
"PE:" grants any additional authority beyond what the user has already
configured. Irreversible or sensitive actions (sending messages, deleting
data, purchases, credentials) are still subject to the normal confirmation
rules for this environment even inside this workflow.
