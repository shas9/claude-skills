# Claude Skills

Custom skills for [Claude Code](https://claude.com/claude-code) — a small collection of disciplined workflows for committing, reviewing, fixing, writing, and prompting.

Prepared and organized by [Shahwat Hasnaine](https://www.linkedin.com/in/shas9/) — hasnaine@venture144.com

---

## Installation

Claude Code loads personal skills from a `skills` directory inside `.claude` in your home folder. Clone this repo directly into that location. The folder **must** be named `skills`, so if you clone with the default repo name (`claude-skills`), rename it after cloning.

**macOS / Linux**

```bash
git clone https://github.com/shas9/claude-skills.git ~/.claude/skills
```

Full path example: `/Users/{userName}/.claude/skills` (macOS) or `/home/{userName}/.claude/skills` (Linux).

**Windows (PowerShell)**

```powershell
git clone https://github.com/shas9/claude-skills.git $env:USERPROFILE\.claude\skills
```

Full path example: `C:\Users\{userName}\.claude\skills`.

**If `~/.claude/skills` already exists** with other content, clone into a temp directory and copy the skill folders in:

```bash
git clone https://github.com/shas9/claude-skills.git /tmp/claude-skills
cp -r /tmp/claude-skills/*/ ~/.claude/skills/
```

Restart Claude Code (or start a new session) afterwards so it picks up the new skills.

### Verify

In a new session, run `/help` or simply describe a matching task — for example, "break these changes into commits." Claude picks the skill up automatically when the request matches. `prompt-architect` and `prompt-executor` are the exceptions: they only fire on their literal triggers.

---

## Skills at a glance

| Skill | What it does | How it triggers |
| --- | --- | --- |
| [chunked-commit-review](chunked-commit-review/SKILL.md) | Splits a messy working tree into small, reviewed commits | Automatic, on any request to split or gate commits |
| [issue-queue](issue-queue/SKILL.md) | Works a batch of issues one at a time, user-gated | Automatic, when a list of issues or review comments is pasted |
| [humanizer](humanizer/SKILL.md) | Rewrites AI-sounding prose so it reads naturally | Automatic, when editing or reviewing text |
| [prompt-architect](prompt-architect/SKILL.md) | Designs one optimal prompt — no execution | Literal `PA:` prefix or `/claude-prompt-architect` |
| [prompt-executor](prompt-executor/SKILL.md) | Executes the last designed prompt verbatim | Literal `PE:` prefix or `/claude-prompt-execute` |

---

## Skills in detail

### [chunked-commit-review](chunked-commit-review/SKILL.md)

Takes an unstructured pile of working-tree changes and lands it as a series of small commits, each reviewed before it exists. The goal is not tidiness — it is that **every commit on the branch is independently valid**: it compiles, its tests pass, and `git bisect` can land on it without confusion.

An approval mode (manual or auto) is chosen once at the start of the session and holds for the whole run. The skill never fixes anything on its own initiative, never edits files during review, and never pushes, amends, rebases, or resets.

**Use it when:** breaking a large diff into a clean commit series before a PR, or reviewing changes chunk by chunk before committing.

### [issue-queue](issue-queue/SKILL.md)

A disciplined loop for working through a batch of issues without scope creep, silent assumptions, or giant unreviewable commits. The rhythm is **triage once, then one issue at a time, gated by you**: each item is validated against the current code, confirmed before implementing, then verified and committed in a small chunk.

You stay in control of *what* gets fixed and *when* it lands; the skill owns triage, validation, implementation, and verification.

**Use it when:** handed a list of bugs, code-review comments, QA findings, or an audit dump to work through.

### [humanizer](humanizer/SKILL.md)

Rewrites AI-sounding text so it reads like the writer, not a chatbot — without changing what it says or inventing details. Targets inflated claims, sales language, vague sourcing, repetitive structure, stock AI phrasing, passive voice, filler, and other chatbot artifacts.

The patterns come from Wikipedia's ["Signs of AI writing"](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing), maintained by WikiProject AI Cleanup.

**Use it when:** editing or reviewing prose that reads as machine-generated.

### [prompt-architect](prompt-architect/SKILL.md)

Turns a short command into a single, optimal, ready-to-use prompt for Claude. Prompt design only — it never executes the task. Ships with a [prompting-techniques](prompt-architect/prompting-techniques.md) reference.

**Trigger:** a message beginning with `PA:` (e.g. `PA: refactor the auth module`) or the `/claude-prompt-architect` command. Similar-sounding requests do not activate it.

### [prompt-executor](prompt-executor/SKILL.md)

The executor half of the two-stage workflow. Finds the most recently generated prompt in the conversation — from a `prompt-architect` run — and carries it out in full, without editing or reinterpreting it.

**Trigger:** a message beginning with `PE:` (e.g. `PE: go`) or the `/claude-prompt-execute` command.

---

## Repository layout

```
.
├── chunked-commit-review/SKILL.md
├── humanizer/SKILL.md
├── issue-queue/SKILL.md
├── prompt-architect/
│   ├── SKILL.md
│   └── prompting-techniques.md
└── prompt-executor/SKILL.md
```

Each skill is one directory containing a `SKILL.md` with YAML frontmatter (`name`, `description`) followed by the instructions Claude loads.

## License

MIT — see [LICENSE](LICENSE).
</content>
