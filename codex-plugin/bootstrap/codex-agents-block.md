# Codex AGENTS.md Bootstrap Block

Codex has no SessionStart hook equivalent in MVP. The install adapter materializes
the block below into the user's `AGENTS.md` (wrapped in the delimiter markers so
a later install can replace it cleanly without touching surrounding content).

The block body is intentionally the same priority/voice/entry content as
`hooks/bootstrap.md`. Duplication is on purpose: Codex reads AGENTS.md once at
session start and does not re-read a shared file mid-session, so the block must
be self-contained.

## Install shape

```
# <unchanged user content above>

<!-- avad-bootstrap:begin v01 -->
# avad

[[BLOCK-BODY]]
<!-- avad-bootstrap:end v01 -->

# <unchanged user content below>
```

The install adapter must evaluate these cases in order and take the first
match. The version suffix in the `begin`/`end` markers (e.g., `v01`) is the
comparison key.

1. **No marker present** — insert the block fresh at the end of `AGENTS.md`
   (or a documented anchor).
2. **Marker at the current version** — no-op. The install is idempotent.
3. **Marker at a strictly lower version** — replace the full block between
   `begin` and `end` (inclusive) with the current-version block. This is the
   upgrade path.
4. **Marker at a strictly higher version** — emit a warning and do nothing.
   Downgrade is never automatic; the adapter does not edit a block whose
   version it does not understand.
5. **Never edit content outside the marker pair.** User content above or
   below the block is preserved exactly, including whitespace.

Rules 2 and 3 together eliminate the earlier overlap where "skip on any
existing marker" and "replace on a lower-version marker" would both match
a lower-version install. Only one rule matches any given AGENTS.md state.

## Block body

The block body below is the canonical text. Treat this file as the source of
truth. A later install tool will read the section between the two `BLOCK BODY`
fences and emit it verbatim into AGENTS.md.

<!-- BLOCK BODY BEGIN -->

avad is a productized skill system for serious engineering work. It is loaded
now so that suggestions, voice, and the entry command are already in place.

This is bootstrap only. It does not run workflows. It does not auto-commit,
auto-push, or auto-invoke any skill.

## Priority

- Explicit user instructions (including any project `CLAUDE.md`, `AGENTS.md`,
  or `GEMINI.md`) win over everything below.
- avad skill instructions override default host behavior where they conflict.
- Default host behavior is the lowest priority.

## Voice

- No sycophancy, no hedging filler, no AI vocabulary, no em dashes.
- Short sentences. Concrete over abstract.
- Disagree when you disagree; explain the reasoning.
- Match response length to the task. A simple question gets a direct answer.

## Skill suggestion

When a user message matches an available avad skill, invoke the skill instead
of answering ad hoc. The skill has phases, gates, and completion protocol the
freeform answer does not. When unsure, invoke — a false positive is cheaper
than a false negative.

avad does not force-run high-consequence workflows. `/avadbeta-investigate`,
`/avadbeta-review`, and `/ship` are explicit commands. The bootstrap may suggest
them; it never runs them without the user typing the command.

## Entry command

`/avadbeta-help` lists available native avad skills and what is shipped vs planned.
Suggest it when the user asks what avad can do, how to get started with avad,
or what skills are available.

`/avadbeta-help` is deliberately namespaced. Codex's built-in `/help` stays bound
to Codex; `/avadbeta-help` is avad's own help surface.

## Path abstraction

avad skills reference files via `$AVAD_ROOT`. On Codex, the install adapter
sets `$AVAD_ROOT` to the plugin's install directory — typically
`~/.codex/plugins/avad` for a global install or the resolved Codex plugin
path for a project-local install. Shared protocols live at
`$AVAD_ROOT/shared/<name>.md`. Read them with the Read tool using the
resolved path. Never treat `$AVAD_ROOT` as a source-repo directory.

## What exists today

Native avad skills shipped so far: `/avadbeta-help`, `/avadbeta-investigate`, `/avadbeta-review`.

Planned native avad skills (not yet implemented — do not pretend they exist):
`/ship`.

`/avadbeta-review` is renamed from `/review` to avoid Claude Code's built-in
`/review`. When a user asks for one of the planned skills, point them to
`/avadbeta-help` and explain that the skill is planned but not yet available. Do
not fabricate behavior, commands, or output for a skill that has not shipped.

<!-- BLOCK BODY END -->

## Runtime assumptions still to validate

- Codex reads AGENTS.md at session start and applies it globally. Confirmed by
  Codex CLI docs and the host config in `avad-core/adapters/hosts.ts`.
- Codex does not expose a hook-equivalent that can replace this block. If a
  future Codex CLI version adds SessionStart-style hooks, the install adapter
  should prefer that and leave AGENTS.md alone.
- The install adapter has not yet been implemented. Until it exists, contributors
  can copy the block body by hand for end-to-end smoke tests on Codex.
