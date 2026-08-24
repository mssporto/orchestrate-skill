# Contributing

This repo is a single Claude Code skill (`skills/orchestrate/SKILL.md`). Changes to it change how the model behaves for every task that triggers it, so please keep edits focused and explain the failure mode you're fixing.

## Before opening a PR

- Read the current `SKILL.md` in full — most "improvements" turn out to already be covered by an existing step, and duplicating guidance makes the skill longer without making it clearer.
- If you're changing a step's instructions, say what behavior you observed that motivated the change (e.g. "backgrounding the fix-loop call orphaned the work") — this skill's steps were each added to fix a specific observed failure, and PRs are easier to review with the same context.
- Prefer tightening or clarifying existing steps over adding new ones. Every additional step is more for the model to hold in context on every invocation.

## Testing a change

There's no automated test suite yet. To validate a change:

1. Drop the edited `SKILL.md` into a local skills directory (see the README's Install section).
2. Run it against a real non-trivial task and check that the gate, decomposition, model selection, spec, delegate, review, and fix-loop steps each behave as described.
3. Note in the PR description what task you tested with and what you observed.

## Scope

Keep this skill single-purpose: planning, delegating, reviewing, and fixing through subagents. Feature requests that are really "a different skill" (e.g. domain-specific review checklists, a new tool integration) are probably better as their own skill rather than a branch added here.
