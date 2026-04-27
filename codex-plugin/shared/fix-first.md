<!-- avad-shared-sentinel: fix-first-v01 -->
# Fix-First Review Loop

> Fix-first protocol loaded v01.

Review skills (`/avadbeta-review`, `/ship` review stages, specialist reviews)
follow a fix-first loop. Findings without fixes are tracked, but fixes are the
goal. The native `/avadbeta-review` skill applies fixes only when the user
explicitly asks; the loop here describes the protocol for skills that do.

## Loop

1. **Find** — produce a list of findings, each with severity, location, and
   reasoning.
2. **Triage** — for each finding, decide one of:
   - `fix-now` — the skill applies the fix in the same run.
   - `propose-fix` — the skill drafts the fix but the user must approve.
   - `defer` — the finding is real but out of scope; record it for later.
   - `dismiss` — the finding is wrong; record why.
3. **Fix** — apply `fix-now` items in a deterministic order. Re-run any
   verification check that the fix could affect.
4. **Verify** — re-read the changed surface and confirm each fix actually
   landed. Failed verifications become a `BLOCKED` completion, never a
   silent pass.

## Anti-patterns

- Listing findings without acting on them when `fix-now` is available.
- Dismissing findings to make the report look cleaner.
- "Fixing" without re-verifying that the fix took effect.
- Treating "the test still passes" as a substitute for actually re-reading the
  changed surface.

## Sentinel use

When this protocol is referenced, include the line `Fix-first protocol loaded v01`
in the completion footer.
