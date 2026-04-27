---
name: avadbeta-help
description: List native avad skills and show what is shipped vs planned. Use when the user asks what avad does, what skills are available, how to get started with avad, or invokes /avadbeta-help.
---

# /avadbeta-help

Lists the native avad skill surface. Answers the question "what can avad do
right now?" without guessing at unshipped behavior.

This is deliberately a namespaced skill. Both Claude Code and Codex ship a
built-in `/help` that always wins at runtime, so avad uses `/avadbeta-help` as
its own help surface.

## When to run

Run `/avadbeta-help` when the user asks any of:

- "what is avad?"
- "what can avad do?"
- "which skills do you have?"
- "how do I get started with avad?"

The bootstrap may also suggest it when the user types `/help` and the answer
they want is avad-specific.

## What to output

`/avadbeta-help` produces a short status report. Three sections, in order, nothing
extra:

### 1. Shipped native skills

| Command | What it does |
|---|---|
| `/avadbeta-help` | This skill. Lists avad skills and shipped-vs-planned status. |
| `/avadbeta-investigate` | Systematic root-cause debugging: reproduce the failure, identify the cause with evidence, fix once, verify. |
| `/avadbeta-review` | Evidence-backed pre-landing code review: inspect the diff against the base branch, read the changed files, report defects with severity and location; does not apply fixes unless the user explicitly asks. |

### 2. Planned native skills (not yet shipped)

Name the planned skills from the MVP core and mark them as not yet available.
Do not fabricate commands, flags, or output for skills that have not shipped.

| Command | Planned purpose | Status |
|---|---|---|
| `/ship` | Ship workflow: preflight, review, PR, rerun-safe. | Planned. Not shipped. |

If the user asks to run one of these, explain that the skill is planned but
not yet available, and do not stand in for it.

### 3. How avad is loaded right now

- On Claude Code, avad loads via a `SessionStart` hook that injects priority,
  voice, and the entry command before the first user message.
- On Codex, avad loads via an `AGENTS.md` block installed once per project.
- Neither path runs workflows. `/avadbeta-investigate`, `/avadbeta-review`, and `/ship`
  are explicit commands the user types. Of those, `/avadbeta-investigate` and
  `/avadbeta-review` are shipped; `/ship` is still planned.

## Voice and completion

Follow the voice rules in `$AVAD_ROOT/shared/voice.md`: no sycophancy, no AI
vocabulary, no em dashes, short sentences, concrete over abstract.

End the run with the completion block from
`$AVAD_ROOT/shared/completion-protocol.md`. `/avadbeta-help` is a read-only
listing, so the expected state is `DONE`.

`$AVAD_ROOT` is defined in the bootstrap: on Claude Code it resolves to
`${CLAUDE_PLUGIN_ROOT}`; on Codex it resolves to the avad install path named
in the installed `AGENTS.md` block. Do not substitute a source-repo path.

Example completion footer:

```
COMPLETION: DONE
SUMMARY: Listed three shipped skills (/avadbeta-help, /avadbeta-investigate, /avadbeta-review) and one planned skill (/ship).
NEXT: none
NOTES:
  - Voice contract loaded v01
  - Completion protocol loaded v01
```

## What /avadbeta-help must not do

- Invent skills that have not shipped.
- Stand in for `/avadbeta-investigate`, `/avadbeta-review`, or `/ship` when the user asks
  for one of those directly.
- Modify repo state, open PRs, or run tests. This skill is read-only.
- Re-invoke the bootstrap. Bootstrap ran at session start already.
