---
name: avadbeta-investigate
description: Root-cause debugging. Reproduce the failure, identify the cause with evidence, fix once, verify the original failure is gone. Use when the user reports a bug, a failing test, an error, or unexpected behavior, or invokes /avadbeta-investigate. Refuses to guess a fix before the evidence supports it and stops to escalate after three disconfirmed hypotheses.
---

# /avadbeta-investigate

Systematic root-cause debugging. Turn a reported symptom into a confirmed
root cause, apply the minimum fix, and verify the original failure is gone.

The skill runs only on evidence. It refuses to propose a fix before the
evidence supports it. It does not stand in for `/avadbeta-review` or `/ship`.

## When to run

Run `/avadbeta-investigate` when the user reports any of:

- a concrete bug ("X returns 500 instead of 200")
- a failing test or assertion with an error message
- a stack trace or panic
- unexpected behavior with a repeatable description
- a regression the user can name ("it worked yesterday, now it is broken")

"Make it faster" or "improve it" is a change request, not an investigation.
Those do not belong here.

## Empty-state

If the user invokes `/avadbeta-investigate` with no bug, no failing test, no error,
and no concrete symptom, ask for one using the format in
`$AVAD_ROOT/shared/ask-format.md`. Do not start investigating a vague
description. Empty-state without a symptom is a valid `NEEDS_CONTEXT`
completion.

Example ask:

```
ASK: What concrete failure should /avadbeta-investigate target?
WHY: /avadbeta-investigate needs a reproducible symptom or error message to pursue.
OPTIONS:
  A. A failing test. Paste the test name and error output.
  B. A runtime error. Paste the stack trace or error message.
  C. Unexpected output. Paste the command, expected output, and actual output.
DEFAULT: none, please pick
```

## Preflight

Before Phase 1, resolve the runtime:

- `eval "$(avad-init)"` to set `AVAD_ROOT` and `AVAD_HOST`.
- `avad-base-branch` to record the base branch for any later diff or
  comparison. Capture its output as `$base`.

Both helpers are pure bash with no runtime dependencies. If either cannot
run, see Error behavior below.

## Phase 1: Gather symptoms

Record, in order:

1. The exact failure artifact: error text, assertion, stack trace, or the
   user-visible symptom.
2. Reproduction inputs: command, arguments, environment, fixture, input data.
3. Scope: which file, endpoint, service, or test is failing. Which are not.
4. Recent changes: `git log --oneline $base..HEAD` for branch-local commits,
   and a wider `git log` window when the failure's introduction window is
   known.

Do not propose a fix in this phase.

## Phase 2: Reproduce or inspect failure evidence

Prefer deterministic reproduction. Run the failing command, test, or code
path and confirm the symptom.

If the failure cannot be reproduced locally (intermittent, staging-only,
production-only data), inspect the failure artifact directly: the log line,
stack trace, or test output. Record that reproduction was not possible and
continue on inspection-only evidence. The completion state downgrades to
`DONE_WITH_CONCERNS` unless later phases can fully verify the fix.

## Phase 3: Identify the likely root cause before editing

State one testable hypothesis in plain language. A hypothesis says what the
cause is, which file and code path it lives in, and what evidence would
confirm or refute it.

**Iron Law: no fixes without evidence.** Editing code before the hypothesis
is confirmed is guessing. Guessing is the failure mode this skill exists to
prevent.

Check recent changes against the suspected surface. Recurring bugs in the
same file are an architectural smell, not a coincidence.

## Phase 4: Test the hypothesis with minimal changes

Confirm or refute the hypothesis with the smallest possible instrumentation:

- Read the code on the exact path.
- Add temporary logging or assertions.
- Run a narrowed test that isolates the hypothesized cause.

Remove temporary instrumentation before finishing.

### 3-strike rule

If three hypotheses are tested and disconfirmed, stop. Do not keep guessing.
Surface the state using `$AVAD_ROOT/shared/ask-format.md`:

```
ASK: Three hypotheses tested and disconfirmed. How should /avadbeta-investigate continue?
WHY: Repeated disconfirmation signals an architectural or assumption error, not a simple bug.
OPTIONS:
  A. New hypothesis with evidence. Describe it and the evidence that supports it.
  B. Escalate to architectural review. Someone with deeper system knowledge looks next.
  C. Add logging and pause. Instrument the area, wait for the next occurrence.
DEFAULT: B
```

## Phase 5: Apply the fix once the cause is confirmed

### Blast radius guard

Before writing the diff, count the files the fix would change. If it
touches more than 5 files, stop and ask using
`$AVAD_ROOT/shared/ask-format.md`. A bug fix that spans more than 5 files
is usually either wrong-layer symptom patching or a structural change
pretending to be a bug fix. Either way, the user decides before the diff
grows.

```
ASK: This fix touches N files. Continue, narrow, or rethink?
WHY: A bug fix spanning more than 5 files is usually wrong-layer symptom patching or a structural change pretending to be a bug fix.
OPTIONS:
  A. Continue. The evidence confirms this is the correct root-cause scope.
  B. Narrow. Land the minimum fix now, track the rest as follow-ups.
  C. Stop and rethink. Return to Phase 3 with a new hypothesis.
DEFAULT: B
```

Apply the smallest diff that addresses the root cause.

Strongly prefer a regression test that fails without the fix and passes
with it. Write one when the failing surface is locally testable.

Some investigations are not locally testable: config drift, environment
misconfiguration, log-only symptoms, infra flakes, host-specific interactions.
Do not invent a regression test just to produce one. In those cases:

- State why a regression test is not practical.
- Record what manual or staging check would catch a recurrence.
- Use `DONE_WITH_CONCERNS` rather than `DONE`.

Re-run the affected test or command after the fix.

## Phase 6: Verify the original failure is gone

Re-run the exact reproduction from Phase 2. The original symptom must no
longer appear. If the symptom remains, the fix is wrong. Return to Phase 3
with a new hypothesis.

If reproduction was inspection-only in Phase 2, note which verification is
still outstanding and complete with `DONE_WITH_CONCERNS`.

## Decisions

Classify every decision per `$AVAD_ROOT/shared/decision-taxonomy.md`:

- **Mechanical** (which file to read first, which logging helper to use):
  decide silently and proceed.
- **Taste** (scope of the fix, whether to widen to a near-miss, naming):
  surface via ask-format with a short rationale. Do not pre-commit.
- **User-challenge** (the user's proposed fix patches a symptom and misses
  the root cause; the user's stated cause is contradicted by the evidence):
  stop and defend the disagreement in plain language. Do not soften into a
  yes/no question.

## Anti-rationalization

Stop and return to Phase 3 if any of these appear:

- "Quick fix for now, proper fix later" with no captured follow-up.
- A proposed fix before the data flow has been traced.
- Each fix reveals a new problem in an adjacent layer. Usually wrong layer,
  not wrong code.
- Trying a change to see what happens with no stated prediction.
- "The test passes now" substituted for re-reading the changed surface.

## Error behavior

If a required helper is missing or not executable (`avad-init`,
`avad-base-branch`, or a project tool the investigation uses), report
`BLOCKED` with:

- **What happened:** the exact failing step, e.g. `avad-init exited 127`.
- **Why:** the concrete cause, e.g. `helper not on PATH` or `file not executable`.
- **Next step:** the single command or action the user runs to recover.

Never echo raw shell output as the final user-facing error. The completion
block stays structured. Raw logs belong above the completion block, not
inside it.

## Voice

Follow the voice rules at `$AVAD_ROOT/shared/voice.md`: no sycophancy, no AI
vocabulary, no em dashes, concrete over abstract.

## Completion

End with a completion block per `$AVAD_ROOT/shared/completion-protocol.md`.

Valid states for `/avadbeta-investigate`:

| State | Use when |
|---|---|
| `DONE` | Root cause confirmed, fix applied, regression test passes, original failure verified gone. |
| `DONE_WITH_CONCERNS` | Fix applied but cannot be fully verified locally: intermittent bug, staging-only repro, regression test not practical. |
| `BLOCKED` | Helper or tool failure, or root cause unclear after the three-strike escalation and the user has not yet decided. |
| `NEEDS_CONTEXT` | Empty-state ask, taste decision, or user-challenge awaiting the user's reply. |

The `NOTES:` block carries a structured DEBUG REPORT:

- `Symptom:` one line describing the original failure.
- `Root cause:` one or two lines, with `file:line` when available.
- `Fix:` one line describing the change, with `file:line`.
- `Evidence:` how the hypothesis was confirmed.
- `Regression test:` test path, or `not practical: <reason>`.

Include sentinel acknowledgement lines only for the shared protocols the
run actually used:

- `Voice contract loaded v01`: the skill's output follows voice.md.
- `Completion protocol loaded v01`: the skill ends with this block.
- `Decision taxonomy loaded v01`: include only when a taste or
  user-challenge decision was classified during the run.
- `Ask format loaded v01`: include only when the run produced an `ASK:`
  block (empty-state, 3-strike escalation, taste, user-challenge).

Example completion (root cause confirmed, local fix, no asks, no taste calls):

```
COMPLETION: DONE
SUMMARY: Fixed null-pointer in cache warmup; original 500 no longer repros.
NEXT: none
NOTES:
  - Symptom: GET /api/widgets returned 500 on the first request after boot.
  - Root cause: src/cache/warmup.ts:42 called getItem() before init resolved.
  - Fix: awaited init before first getItem call (src/cache/warmup.ts:42).
  - Evidence: assertion on init state reproduced the null branch; fix removes it.
  - Regression test: test/cache/warmup.test.ts "warmup awaits init".
  - Voice contract loaded v01
  - Completion protocol loaded v01
```

`$AVAD_ROOT` is defined in the bootstrap: on Claude Code it resolves to
`${CLAUDE_PLUGIN_ROOT}`; on Codex it resolves to the avad install path named
in the installed `AGENTS.md` block. Do not substitute a source-repo path.

## What /avadbeta-investigate must not do

- Propose a fix before the evidence supports it.
- Keep guessing after three disconfirmed hypotheses. Escalate instead.
- Invent a regression test just to produce one when the failure is not
  locally testable.
- Echo raw shell output as the final completion.
- Stand in for `/avadbeta-review` or `/ship`.
