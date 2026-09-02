# Model-Specific Prompting Technique Reference

Consulted by prompt-architect at Step 3. This is a working reference, not a
rulebook — use judgment about which parts actually apply to the task at hand.
If you're ever uncertain whether something here is current, note that in
passing and don't over-commit to a stale detail.

---

## General principles (apply across all models)

- **Be explicit and direct.** State what you want clearly rather than
  hinting. Claude models respond better to direct instruction than to
  implication.
- **XML tags for structure.** When a prompt has distinct parts (context,
  task, constraints, examples, output format), wrapping them in tags like
  `<context>`, `<task>`, `<constraints>`, `<examples>`, `<output_format>`
  helps the model parse structure reliably, especially in longer prompts.
- **Examples over abstract description.** A concrete input/output example
  usually removes more ambiguity than another paragraph of explanation.
  Use 1–3 well-chosen examples rather than many mediocre ones.
- **Ask for reasoning when the task is non-trivial.** For multi-step or
  judgment-heavy tasks, explicitly inviting step-by-step reasoning before
  the final answer improves quality — but don't ask for it on tasks that
  are genuinely simple, where it just adds verbosity.
- **State the negative space.** Telling the model what NOT to do (avoid
  X, don't include Y) is often as useful as saying what to do, especially
  to prevent common failure modes (over-explaining, padding, hedging).
- **Specify output format precisely.** Length, structure, whether code
  should be in diffs vs full files, whether to use headers — say it
  explicitly rather than leaving it implicit.

---

## Per-model calibration

### Higher-capability / "flagship" tier (e.g. Opus-class models)
- Can handle longer, more nuanced prompts with multiple interacting
  constraints without losing track of them.
- Benefits most from being given genuine ambiguity to resolve ("use your
  judgment on X, given constraint Y") rather than everything pre-decided —
  over-specifying can waste its strength on trivial sub-decisions.
- Good fit for tasks needing extended thinking / multi-step planning before
  execution — explicitly inviting a brief plan before acting pays off.
- Efficiency note in the generated prompt can be lighter — trust it to
  self-regulate verbosity reasonably well, just state the target format.

### Mid tier (e.g. Sonnet-class models)
- The default workhorse profile: structured prompts with clear sections
  work well; benefits from explicit structure (XML tags, headers) more
  than flagship tier does, since it reduces re-deriving intent.
- Handles agentic / multi-tool tasks well when given explicit scope
  boundaries — state clearly what's in/out of scope to avoid overreach.
  Given this class is also commonly used for the most heavily agentic
  coding tasks, be explicit about verification steps (e.g. "run the tests
  before declaring this done") where relevant.
- A concise efficiency note in the generated prompt is worth including
  plainly (e.g. "be concise, avoid restating unchanged code").

### Lighter / faster tier (e.g. Haiku-class models)
- Favor shorter, more concrete, single-purpose prompts. Break a
  multi-part task into a more explicit sequence rather than one dense
  paragraph of interacting requirements.
- Provide examples more liberally here — concrete demonstration compensates
  for less capacity to infer from abstract description.
- Keep the requested output format simple and explicit; avoid asking for
  heavy multi-step self-critique in the same turn.
- Efficiency note should be explicit and specific (e.g. "answer in under
  200 words, no preamble") rather than a vague "be concise."

### Extended-thinking-capable invocations
- When the executing model will have extended thinking available (and the
  task is genuinely complex — multi-step reasoning, non-obvious tradeoffs,
  debugging with an unclear root cause), it's worth explicitly noting in
  the generated prompt that the model should think through the problem
  before producing a final answer, rather than assuming it will by default.
- Don't invite this for simple, mechanical tasks — it adds latency without
  improving a well-understood, low-ambiguity task.

---

## Task-type notes that interact with model choice

- **Coding / agentic tasks**: regardless of tier, be explicit about the
  boundary of the change (files/modules in scope) and what "done" means
  (tests passing, a specific behavior verified) — this matters more for
  agentic execution than for a single conversational answer.
- **Long-context tasks** (large codebase, long document): note where in
  the context the most relevant material is, if known, rather than
  assuming the model will locate it equally easily as smaller context.
- **Creative/writing tasks**: examples of desired tone/voice help every
  tier, but are especially valuable for lighter models.

---

## Reminder

This file may drift out of date as models change. If something here seems
off or a new model tier isn't covered, use general principles above as the
fallback rather than guessing at specifics.
