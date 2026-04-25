<!-- avad-shared-sentinel: completion-protocol-v01 -->
# Completion Protocol

> Completion protocol loaded v01.

Every serious skill ends with one of four completion states. The state is
mandatory and must be the last structured block in the run.

| State | Meaning | When to use |
|---|---|---|
| `DONE` | Work finished, all checks pass, no caveats. | Happy path. |
| `DONE_WITH_CONCERNS` | Work finished but the user should know about a risk, soft failure, or open question. | Skipped optional step, partial coverage, advisory warnings. |
| `BLOCKED` | The skill could not finish and the next step requires the user. | Missing credentials, conflicting state, host incompatibility. |
| `NEEDS_CONTEXT` | The skill could not proceed because it lacks information only the user has. | Ambiguous intent, missing config, taste decision. |

## Format

Skills must end with a fenced block like:

```
COMPLETION: DONE_WITH_CONCERNS
SUMMARY: <one short sentence>
NEXT: <one concrete next action, or "none">
NOTES:
  - <optional bullet>
```

The keys must appear in this order. Values must be plain text; no markdown
inside the block.

## Recovery from prior runs

If a checkpoint exists from a prior run, the skill surfaces it before
deciding the new completion state. A `BLOCKED` checkpoint that the user has
since unblocked is allowed to resume into a new `DONE` run.

## Sentinel use

When this protocol is referenced, include the line `Completion protocol loaded v01`
in the completion footer.
