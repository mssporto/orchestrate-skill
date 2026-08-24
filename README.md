# orchestrate

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Claude Code skill that turns the model into a pure orchestrator: it plans, decomposes, makes model-selection and review judgment calls — but never writes production code itself. All actual implementation is delegated to agents dispatched with the **Agent** tool, review is delegated to a fresh agent, and fixes go back to the *same* agent via **SendMessage** instead of a new dispatch.

## Why

Delegation only pays off when it's structured. Left unguided, an orchestrating model tends to either do everything itself (no cost/speed benefit) or delegate sloppily (vague specs, no review discipline, fixes sent to a fresh agent that has to re-derive context from scratch). This skill encodes a specific loop to avoid both failure modes.

## What it does

1. **Gate** — decides whether a task is even worth delegating. Small or single-piece tasks get done directly; only "coverage-shaped" work (independently specable, independently verifiable pieces) goes through the full loop.
2. **Decompose** — breaks the task into pieces sized so each is independently specable and verifiable, without over-fragmenting into pieces too small to be worth a dispatch.
3. **Model selection** — judges each piece individually: taste/judgment-heavy work (interface design, UX copy, non-obvious tradeoffs) goes to Opus; mechanical work (boilerplate, config, pattern-following tests) goes to Sonnet.
4. **Spec writing** — each dispatched agent starts cold with no visibility into the orchestrator's reasoning, so the spec has to be fully self-contained: goal, context, constraints/non-goals, and acceptance criteria.
5. **Delegate** — dispatches via the Agent tool, in parallel where pieces are independent.
6. **Review** — delegates the diff review to a *fresh* agent held to the same acceptance criteria, then the orchestrator itself adjudicates the findings (real problem vs. nitpick vs. spec misunderstanding).
7. **Fix loop** — sends fixes back to the *same* agent that wrote the piece via SendMessage (not a new Agent call), since a resumed agent keeps its prior context and a fresh one starts over. Capped at 2–3 rounds before escalating.
8. **Escalate** — if the fix loop doesn't converge, the orchestrator takes the piece over directly, bumps the model tier, or surfaces the issue to the user rather than repeating the same round indefinitely.

A key implementation detail: from the review step onward, every Agent/SendMessage call is dispatched in the foreground and its result is waited on before proceeding. Review and fix rounds are sequential dependencies — an orchestrator that backgrounds them has no way to be "woken up" later, and the work gets orphaned. (This was caught by testing an early draft of the skill and traced to exactly that failure mode.)

## Requirements

- Claude Code (or any harness exposing the **Agent** and **SendMessage** tools — spawn-a-subagent and resume-a-subagent-with-context, respectively).
- No other dependencies. The skill is a single `SKILL.md`.

## Install

As a plugin — add this repo as a marketplace and install it:

```
/plugin marketplace add mssporto/orchestrate-skill
/plugin install orchestrate
```

Or manually — drop `skills/orchestrate/SKILL.md` into a skill directory Claude Code will discover, e.g.:

```
~/.claude/skills/orchestrate/SKILL.md
```

(or a project-local `.claude/skills/orchestrate/SKILL.md` to scope it to one repo).

## Use

Invoke explicitly with `/orchestrate`, or just describe non-trivial implementation work (a feature, a multi-file fix, a refactor with more than one moving part) — the skill's description is written to trigger on that shape of task even without naming it directly.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT — see [LICENSE](LICENSE).
