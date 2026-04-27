<!-- avad-shared-sentinel: skill-contract-v01 -->
# Skill Contract

> Skill contract loaded v01.

Every serious avad skill follows this execution contract. Skills are free to add
phases, but they may not skip the ones below.

## Required phases

1. **Start** — emit a visible status line. Run preflight: detect branch, base,
   project context. Surface any prior checkpoint.
2. **Empty-state** — if there is nothing to do, say so explicitly and stop.
   Empty-state output is a valid `DONE` completion, not a failure.
3. **Execute** — perform the work in named phases. Each phase has a clear
   precondition, action, and postcondition.
4. **Recover** — when something fails, distinguish *infrastructure failure*
   (explain, give next repair action, never show raw shell noise as the final
   answer) from *logic failure* (record in the completion state, give next step
   or escalation path).
5. **Complete** — finish in a structured completion state. See
   [`completion-protocol.md`](completion-protocol.md).

## Re-run safety

If a skill performs externally visible actions, the action layer must be
skip-if-done by code, not by the model's memory. Re-running a half-finished run
must not duplicate side effects. Verification checks always re-run.

## Host incompatibility

If the host cannot support a capability, say so explicitly and provide the
fallback path if one exists. Do not silently degrade.

## Frontmatter: `host_support`

`host_support` in a skill's frontmatter is **authoring intent**. It declares
the set of hosts the skill is written for and whose install-time frontmatter
transforms are known-compatible. The materializer uses it to decide which
plugin surface to emit the skill into; a host listed in `host_support` gets a
materialized copy, a host not listed is skipped.

`host_support` is **not** a statement that the host is product-supported. A
host is considered product-supported by avad only after the product-level
architecture doc's host-support checks land: bootstrap, invocation, and
completion smoke tests running in CI on that host. Until those exist, a host
may appear in `host_support` and still be a pre-smoke target.

Adding a new host to `host_support` is a prose decision by the skill author;
adding it to `HOSTS` in `avad-core/adapters/hosts.ts` and satisfying the host
addition checklist is a separate, stricter gate.

## Sentinel use

When this contract is referenced, the executing model should acknowledge it
once in the run by including the line `Skill contract loaded v01` somewhere
the user or eval can see (typically in the completion footer).
