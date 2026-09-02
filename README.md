# Claude Skills

Custom skills for [Claude Code](https://claude.com/claude-code).

Prepared and organized by [Shahwat Hasnaine](https://www.linkedin.com/in/shas9/) — hasnaine@venture144.com

## Skills

### [chunked-commit-review](chunked-commit-review/SKILL.md)
Splits a messy working tree into small, reviewed, independently-valid commits — one logical change at a time, with a human approval gate configured once at the start of the session. Use it when breaking a large diff into a clean series of commits before a PR, or reviewing changes chunk by chunk before committing.

### [humanizer](humanizer/SKILL.md)
Rewrites AI-sounding text so it reads naturally without changing what it says. Targets inflated claims, sales language, vague sourcing, repetitive structure, stock AI phrasing, passive voice, filler, and other chatbot artifacts, based on Wikipedia's "Signs of AI writing" guide.

### [issue-queue](issue-queue/SKILL.md)
Works through a batch of pasted issues, code-review comments, bug reports, or audit findings one at a time: triaging by severity, validating each against the current code, confirming with the user before implementing, then verifying and committing in small chunks. Use it when handed a list of problems to fix without scope creep or one giant unreviewable commit.

### [prompt-architect](prompt-architect/SKILL.md)
Turns a short command into a single, optimal, ready-to-use prompt for Claude — prompt design only, no task execution. Triggered only by messages starting with `PA:` or the `/claude-prompt-architect` command.

### [prompt-executor](prompt-executor/SKILL.md)
Finds the most recently generated prompt in the conversation (from a `prompt-architect` run) and carries it out in full, without editing or reinterpreting it. Triggered only by messages starting with `PE:` or the `/claude-prompt-execute` command.

## License

MIT — see [LICENSE](LICENSE).
