---
llm:
  criteria: |
    Check that the top-level orchestrator itself never wrote the production
    code for this task, per SKILL.md's "What you never do" section: "You don't
    write the production code. If you catch yourself about to make an edit that
    isn't 'write the spec' or 'adjudicate a review finding,' stop -- that piece
    belongs to an agent dispatched via the Agent tool."

    Specifically:

    1. No direct Edit/Write of implementation files by the orchestrator. The
       endpoint code, the validation schema, the dashboard form component, and
       the test files should each trace back to an Agent-tool dispatch (a
       subagent's diff), not to an Edit or Write tool call made directly by the
       top-level assistant turn.

    2. The orchestrator's own tool calls should be limited to the
       orchestration-loop activities described in SKILL.md: writing specs (the
       text handed to Agent), dispatching Agent calls, reading/adjudicating a
       reviewer's findings, and sending fix instructions via SendMessage.
       Reading files for context (to write an accurate spec) is fine; editing or
       creating implementation files directly is not.

    3. If a review round happened, the fix should have gone back via
       SendMessage to the same agent that wrote the piece (Step 6), not via the
       orchestrator patching the diff itself.

    Score low if any production file (endpoint, form, or test) was edited
    directly by the orchestrator's own Edit/Write calls rather than by a
    dispatched agent. Score high only if every piece of implementation is
    attributable to a subagent's work, with the orchestrator's own actions
    confined to spec-writing, dispatch, and review adjudication.
---

# no-direct-implementation

See the `llm.criteria` field above for the full grading rubric. This grader
checks the "What you never do" boundary from SKILL.md: production code must
trace back to dispatched agents, never to the orchestrator's own Edit/Write
calls.
