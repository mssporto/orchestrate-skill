# orchestrate eval suite

Eval cases for the `orchestrate` skill, written for `claude plugin eval`.

## Status: unverified against a live run

`claude plugin eval` is in **early access** and was **disabled in the account
used to author this suite**. That means:

- These cases were written from `claude plugin eval --help` output alone. They
  were **not** scaffold-verified with `claude plugin eval init --bare <name>`,
  and were **not** run end-to-end.
- Fields confirmed directly from the `--help` text (quoted here so it's clear
  what's ground truth versus inference):
  - The two valid on-disk case layouts are `case.yaml` **or** a directory per
    case containing `prompt.md` + `graders/*.md`. This suite uses the latter,
    since it's the layout `init --bare` scaffolds and is lower-risk.
  - `runs` — per-case run count, "default: case.runs ?? 3", overridable with
    `--runs <n>`.
  - A per-case `model` setting, overridable with `--model <model>`.
  - `tags`, filterable with `--tag <tag...>` (repeatable).
  - `scaffold_script` — "author-supplied bash," off by default, run with
    `--scaffold`.
  - A `tool_used: Skill` grader with a `with-only` marker, used under
    `--ablation with-without` as a "plugin-fired indicator" rather than part
    of the score.
  - An `llm` grader type (implied by `--judge-model <model>` overriding "the
    LLM-grader model," default haiku), which is why every grader in this suite
    uses `llm: { criteria: ... }` frontmatter rather than a different grader
    type.
- **Not confirmed and not used here beyond what's listed above:** exact
  YAML/frontmatter key names for `max_turns`, `timeout_seconds`,
  `allowed_tools`, `append_system_prompt`, `env`; the complete list of grader
  types (regex, tool_order, file_exists, baseline, etc.) or their field names
  beyond `llm.criteria`; and the internal syntax of `graders/*.md` beyond "it's
  markdown, and the confirmed shape is `llm: { criteria: ... }`." The grader
  files in this suite pair YAML frontmatter (`llm: { criteria: <text> }`) with
  a short human-readable markdown body restating the same rubric — that
  frontmatter shape is not independently verified.
- **`scaffold_script`'s exact invocation contract is not confirmed either** —
  specifically: which directory it runs in relative to the case/prompt, when
  it runs relative to the agent's turn (before the prompt is handed to the
  agent, presumably, but not verified), whether it's cleaned up after the run,
  and whether it needs a shebang or executable bit versus being run as a
  literal `bash -c` body. Both cases here supply a best-effort
  `scaffold_script` in `prompt.md` frontmatter that creates the fixture
  file(s) the prompt and graders reference (this repo has no real API layer,
  dashboard, or password-reset code to act on, so a scaffold is required for
  the cases to be runnable at all) using plain relative paths from what's
  assumed to be the case's own working directory. Re-check this against
  `claude plugin eval init --bare <name>` output before trusting it in CI, and
  adjust the working-directory assumption if the real contract differs.
- **The `tool_used: Skill` grader's `with-only` marker is documented for that
  specific grader type only.** Whether an `llm` grader can carry the same
  `with-only` / ablation-arm semantics is not confirmed by the `--help` text,
  so none of the `llm` graders in this suite assert anything about ablation
  behavior in their judge-facing `criteria` — that would be telling the judge
  model something unverified about its own scoring context. If ablation
  reporting for these `llm` graders turns out to matter, confirm the real
  field name/semantics first rather than guessing it into a grader prompt.

**Before relying on this suite in CI**, once `claude plugin eval` is enabled
for this account, run:

```
claude plugin eval init --bare sanity-check
```

and diff the scaffolded `prompt.md` + `graders/*.md` against the two cases
here. Adjust frontmatter keys/grader syntax to match whatever the real
scaffold produces, since everything above the "Not confirmed" line is the only
part that's certain.

## Cases

- `gate-in-coverage-task/` — a multi-piece request (a `POST /invites`
  endpoint, a matching dashboard form, and tests for both) that should make
  the orchestrate skill's Step 0 gate decide the task is coverage-shaped and
  run the full plan/delegate/review/fix loop. Mirrors the README's "gates in"
  example.
- `gate-out-tiny-fix/` — a one-file, one-string wording fix that should make
  Step 0 decide the task is discovery-shaped/tiny and get done directly, with
  zero Agent-tool dispatches. Mirrors the README's "gates out" example.

## Running the suite

Once `claude plugin eval` is enabled:

```
claude plugin eval . --case 'gate-*'
```

Useful variants while iterating:

```
# Just the gate-in case, with more runs than the checked-in default of 1
claude plugin eval . --case gate-in-coverage-task --runs 3

# Both cases, tagged
claude plugin eval . --tag orchestrate
```

Cases are checked in with `runs: 1` to keep eval cost low during authoring.
Bump `--runs` (or the per-case `runs` field) to 3+ before trusting results in
CI — a single run of an LLM-graded, agentic case is noisy.
