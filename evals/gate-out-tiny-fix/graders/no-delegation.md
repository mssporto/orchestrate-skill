---
llm:
  criteria: |
    Inspect the transcript and check that the orchestrate skill's Step 0 gate
    correctly gated this task OUT of the delegation loop, per SKILL.md:
    "Discovery-shaped, or genuinely tiny (a one-line fix, a single obvious
    edit) -> just do it yourself. Delegating a task with nothing to decompose
    adds a round-trip for no benefit."

    Check specifically for:

    1. Explicit gate reasoning that this is tiny/discovery-shaped. The
       transcript should show reasoning (either narrated by the skill or
       otherwise evident) that this is a one-file, one-string, single-
       judgment-call change with nothing to decompose into independently
       specable/verifiable pieces -- i.e. it does not meet the
       "coverage-shaped" bar from Step 0.

    2. The edit was made directly, not delegated. The string change (updating
       the expired-token error message) should show up as a direct Edit tool
       call made by the top-level assistant, not as an Agent-tool dispatch to a
       subagent.

    3. No Agent-tool subagent calls at all. The transcript should contain zero
       Agent tool invocations for this task. Any Agent dispatch -- even a
       single one "just to be safe" -- is a gate failure, since Step 0 and the
       README's own "gates out" example both treat this exact shape of task (a
       single wording fix in one file) as something that should never enter
       the loop.

    4. No SendMessage / review loop. Since nothing was delegated, there should
       be no reviewer dispatch and no SendMessage fix-round -- those only apply
       to pieces that went through Steps 4-6.

    Score high only if the edit was completed with zero Agent-tool calls and
    the reasoning for skipping delegation is evident and matches Step 0's
    stated criteria. Score low if any Agent dispatch occurred, or if the
    transcript shows no reasoning at all about why this task didn't warrant the
    loop.
---

# no-delegation

See the `llm.criteria` field above for the full grading rubric. This grader
checks that Step 0's gate-out path was taken for a genuinely tiny, single-file,
single-judgment-call fix, with zero Agent-tool dispatches.
