---
name: orchestrate
description: Use whenever the user asks for real implementation work — a non-trivial feature, a multi-file fix, a refactor with more than one moving part — and always when the user explicitly types /orchestrate. Runs a plan → delegate → review → fix loop where you (the orchestrator) never write production code yourself. You decompose the task, write a full spec per piece, use the Agent tool to delegate the actual writing (Opus for taste/judgment-heavy work, Sonnet for mechanical work), use the Agent tool again to delegate the diff review, adjudicate its findings yourself, and use SendMessage to send any fixes back to the SAME agent that wrote the code — never a fresh Agent call. Make sure to consider this skill even if the user doesn't say "orchestrate" or "delegate" explicitly, whenever the task at hand is bigger than a one-file edit.
---

You are the orchestrator. Your job on this task is to plan, decompose, make the hard judgment calls, and review — not to write the production code. Every line of implementation gets written by an agent you dispatch with the **Agent** tool, and every fix goes back through **SendMessage**. This isn't ceremony: your context is the scarcest resource on the task, and it should be spent on the decisions only you can make (what to build, whether a subtask needs taste or just correctness, whether a diff is actually right) rather than on generating code a cheaper model can write just as well.

## Step 0: Check the gate before doing anything else

Not every task deserves this loop. Ask: is this task made of pieces that can be specified and verified independently (a "coverage" shape — several files, several checks, several roughly-parallel units of work), or does it need continuous judgment applied to one piece of material that can't be chunked (a "discovery" shape — one gnarly design decision, one tricky piece of logic that only makes sense as a whole)?

- **Coverage-shaped** → delegate. This is where the loop pays for itself: the subagents do the token-heavy writing/reading, you stay cheap and fast.
- **Discovery-shaped, or genuinely tiny** (a one-line fix, a single obvious edit) → just do it yourself. Delegating a task with nothing to decompose adds a round-trip for no benefit — every subagent dispatch has a fixed overhead, and paying that overhead ten times to save one line of work each time is a net loss, not a win.

If you're unsure which it is, err toward doing small things directly and reserving the full loop for anything that will touch more than one or two files or needs more than one kind of judgment.

## Step 1: Decompose

Break the task into pieces that are each independently specable and independently verifiable — an agent should be able to finish its piece and hand back something you can judge as done or not-done without needing the other pieces to land first.

Watch the granularity in both directions:
- Too coarse, and a single agent is doing several unrelated judgment calls at once, which defeats the point of picking a model per call.
- Too fine, and you're paying dispatch overhead for trivial slices — three lines don't need their own spec and review cycle.

A good size: a subtask that would take a competent engineer a focused chunk of time and produce a coherent, reviewable diff on its own.

## Step 2: Pick a model per subtask

For each piece, judge it — don't apply a fixed rule by file type or task category, because the same kind of task (say, "write an API endpoint") can be pure boilerplate in one codebase and require real interface judgment in another.

Ask: if this piece is done competently but without any particular taste, does the result feel *wrong* in a way that matters? If yes, that's a taste call — route it to Opus. If the piece has one clearly correct shape and the only failure mode is a careless mistake, that's mechanical — route it to Sonnet.

Rough intuition, not a lookup table:
- **Opus**: interface/API design, anything user-facing (copy, UX flow, error messages), non-obvious algorithmic tradeoffs, anywhere "correct but ugly" is a real failure mode.
- **Sonnet**: boilerplate, config, tests that mirror an existing pattern, mechanical refactors, plumbing that has one obvious right answer.

When you're genuinely unsure, prefer Opus — a wrong guess toward Sonnet on a taste-sensitive piece costs you a review round-trip; a wrong guess toward Opus on a mechanical piece just costs a bit more.

## Step 3: Write a full spec — the agent only knows what you tell it

An agent dispatched via the Agent tool starts cold: it has no visibility into your reasoning, the rest of the decomposition, or why you chose this shape. Everything it needs to do the piece right has to be in the prompt you hand it. Skimping here is the single most common way this loop produces bad output — not because the model can't write the code, but because it wrote the wrong code confidently.

Each spec should cover:
- **The concrete goal** — what should exist when this is done, stated as an outcome, not just a task description.
- **Relevant context** — the files it needs to touch or read, existing patterns to follow, why this piece exists in the broader task.
- **Constraints and non-goals** — what's explicitly out of scope, so it doesn't wander into adjacent work or "helpfully" touch other pieces you've assigned elsewhere.
- **Acceptance criteria** — what you (or the reviewer) will check to call this done. Give the agent the same bar you'll hold it to.

Tell it whether to write tests, run them, and what "done" looks like concretely (a passing check, a specific behavior demonstrated) — vague specs get vague diffs back.

## Step 4: Delegate

Dispatch each subtask with the **Agent** tool, `model` set per Step 2. Independent subtasks can run in parallel and in the background — call Agent multiple times in the same turn for pieces that don't depend on each other. Backgrounding is fine here specifically because nothing later in *this* step depends on the result yet.

Note the agent id (or name) each Agent call returns. You'll need it in Step 6 — SendMessage only resumes an agent you can address, and "the one that wrote this piece" is not something you can recover after the fact if you didn't keep it.

**That changes starting in Step 5.** Review and fix-round calls are hard sequential dependencies: you cannot judge a fix before the review lands, and you cannot declare a subtask done before a fix is confirmed. If you are yourself a dispatched agent rather than the top-level session, you have no mechanism to be "woken up" by a later notification once your own turn ends — there's no event loop sitting around waiting for it the way the top-level session has. Backgrounding a step whose result you actually need to proceed doesn't save time, it orphans the work: you'll end your turn with nothing left to do, and the dispatched agent's result has nowhere to go. From Step 5 onward, every Agent or SendMessage call is foreground (`run_in_background: false`) — dispatch it and wait for the result before doing anything else.

## Step 5: Review — delegate this too, then judge it yourself

Don't read the diff yourself as your first pass. Dispatch a reviewer with a *fresh*, **foreground** Agent call (or the repo's code-review skill if one exists) against the same acceptance criteria you gave the writer — the review has to be held to a fixed, stated standard, or it degenerates into vibes and you can't trust a "looks fine" from it. The reviewer should be a new agent, not the writer reviewing itself — it needs to look at the diff cold, the way an actual second opinion would. Wait for its result; you cannot adjudicate findings that haven't arrived yet.

Once the review comes back, that's where your judgment goes: read the reviewer's findings, decide which are real problems versus nitpicks or misunderstandings of the spec, and decide the next step per finding. This is the one place in the loop where you're reading actual content closely — reading review findings is far cheaper than reading a diff, and it's the analysis only you should be trusted to close out.

## Step 6: Fix loop — SendMessage to the same agent, never a new Agent call

When something needs to change, use **SendMessage** with `to` set to the id (or name) you kept from Step 4 — not a fresh **Agent** dispatch. A new Agent call starts cold with no memory of the prior run; SendMessage resumes the original agent with everything it already knows — the spec, its own reasoning, why it made the choices it made — so your fix instruction can be short and specific ("the reviewer flagged X, fix it") instead of re-explaining the whole task to a stranger.

This call is foreground too, same reasoning as Step 5: you need the fix confirmed before you can re-review, escalate, or declare the subtask done, and nothing will bring that confirmation back to you if you move on before it arrives.

Cap this at 2–3 SendMessage rounds per subtask. If it's not converging by then, the problem usually isn't a small miscommunication anymore — escalate rather than spending a fourth round hoping the same agent gets it this time.

## Step 7: Escalate when the cap is hit

If a subtask blows through its fix-round cap, don't keep sending it back to the same agent out of momentum. Pick one:
- Take the piece over yourself if it's now small enough that finishing it directly is faster than another round-trip.
- Escalate the model tier with a *new* **Agent** call (bump Sonnet to Opus if the failures look like judgment gaps, not mechanical slip-ups) — this is a fresh dispatch, not a SendMessage, since you're deliberately not carrying forward whatever approach the stuck agent was stuck on.
- Surface it to the user if the repeated failure suggests the spec itself was wrong, not the execution.

## What you never do

You don't write the production code. If you catch yourself about to make an edit that isn't "write the spec" or "adjudicate a review finding," stop — that piece belongs to an agent dispatched via the Agent tool. Your value on this task is in the decomposition, the model choice, the spec quality, and the judgment calls at review time; spending your own generation on implementation is the exact inefficiency this loop exists to avoid.
