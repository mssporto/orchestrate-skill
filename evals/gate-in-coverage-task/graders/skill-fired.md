---
llm:
  criteria: |
    Inspect the transcript and score whether the orchestrate skill actually fired
    and followed its own loop for this task, not just whether the final code looks
    reasonable.

    Check specifically for:

    1. Step 0 gate reasoning is visible. The transcript (or the skill's own
       narration) explicitly weighs whether this task is "coverage-shaped"
       (independently specable, independently verifiable pieces) versus
       "discovery-shaped, or genuinely tiny," and concludes it's coverage-shaped
       because the task has several roughly-parallel units of work (endpoint,
       form, tests). A bare decision to delegate with no stated reasoning about
       shape does not satisfy this.

    2. Step 1 decomposition into multiple named pieces. The task is broken into
       separate, independently-specable pieces -- at minimum something like
       (a) the POST /invites endpoint plus its validation, (b) the dashboard
       invite form, and (c) test coverage for both. Each piece should read as
       something an agent could finish and hand back as done/not-done on its
       own, per Step 1's definition.

    3. Step 2 model-selection rationale per piece. For at least the form
       (user-facing copy/UX, a taste call per Step 2's own examples) and the
       endpoint or tests (mechanical, pattern-following), there's an explicit
       rationale for choosing Opus vs. Sonnet -- not just a model tag with no
       reasoning. The rationale should reflect Step 2's actual test: "if this
       piece is done competently but without taste, does it feel wrong in a way
       that matters?"

    4. At least one Agent-tool dispatch actually happened. The transcript shows
       at least one real Agent tool call with a full spec (per Step 3: concrete
       goal, context, constraints/non-goals, acceptance criteria) dispatched to
       do a piece of the implementation -- not the orchestrator merely
       describing that it would delegate.

    Score high only if all four are clearly present and specific to this task
    (not generic boilerplate that could apply to any task). Score low if the
    skill's gate/decomposition/model-selection reasoning is missing, vague, or
    contradicted by what actually happened (e.g., it says it delegated but no
    Agent dispatch appears).
---

# skill-fired

See the `llm.criteria` field above for the full grading rubric. This grader
checks that the orchestrate skill's Step 0 through Step 4 behaviors (gate,
decomposition, model selection, dispatch) are all concretely present in the
transcript for the gate-in-coverage-task case, grounded in SKILL.md's own step
names rather than a generic "did it do a good job" check.
