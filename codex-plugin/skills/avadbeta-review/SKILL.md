---
name: avadbeta-review
description: "Evidence-backed pre-landing code review. Inspect the current branch diff against the base branch, read the changed files, and report defects the tests do not catch: bugs, regressions, behavior mismatches, missing tests, security or privacy risks, data-loss or migration risks, and CI / test-coverage gaps. Findings must be specific and actionable; style preferences and speculation are not findings. Use when the user asks for a review, says \"review my diff\", \"code review\", or \"pre-landing review\", or invokes /avadbeta-review. Does not fix findings unless the user explicitly asks."
---

# /avadbeta-review

Evidence-backed code review of the current branch diff against the base branch.
Find defects the tests do not catch, report them as specific findings the user
can act on, and stop.

The skill runs only on the diff and the changed files. It does not praise code
before evaluating it, does not stand in for `/ship`, and does not apply fixes
unless the user explicitly asks after seeing the findings.

## When to run

Run `/avadbeta-review` when the user asks for any of:

- a pre-landing review of the current branch ("review this PR", "check my
  diff", "code review before I merge")
- a second-opinion review on a diff they have already written
- a risk sweep before a rebase, squash, or hand-off

"Read this file and tell me if it looks good" is a single-file question, not a
branch review. Answer it as a normal read, not as `/avadbeta-review`.

## Empty-state

If the working branch has no diff against the base, say so and stop. Reporting
findings against an empty diff is fabrication. "No diff" means **all** of:

- `git diff "origin/$base"` is empty after fetching the base (so committed,
  staged, and unstaged changes are all clean against the base), and
- `git ls-files --others --exclude-standard` is empty (no new untracked files
  in the working tree).

A single new file the user has not run `git add` on yet is a real change and
the review must cover it. An empty `git diff` plus a non-empty untracked list
is **not** empty-state.

If the base branch cannot be confirmed (see Preflight base-confirmation), ask
the user using `$AVAD_ROOT/shared/ask-format.md` and complete as
`NEEDS_CONTEXT` until they answer. Do not review against an assumed default
silently.

## Preflight

Before Phase 1, resolve the runtime and confirm the base branch.

### Step 1: Runtime

- `eval "$(avad-init)"` to set `AVAD_ROOT` and `AVAD_HOST`.

### Step 2: Resolve the base branch

- `avad-base-branch` prints the detected base. Capture its output as `$base`.

`avad-base-branch` always prints exactly one branch name — its detection chain
ends in a `main` fallback so the helper never fails. A clean detection and a
worst-case fallback look identical to the caller. The skill therefore must not
trust `$base` blindly; it must classify the *signal strength* before letting
findings depend on it.

### Step 3: Confirm the base branch came from a strong signal

Run the same probes `avad-base-branch` uses internally and decide which one
"won". `$base` came from a **strong signal** if any of the following is true:

1. `[ -n "$AVAD_BASE_BRANCH" ]` — the user pinned the base explicitly.
2. `gh pr view --json baseRefName --jq .baseRefName 2>/dev/null` exits 0 and
   prints a non-empty value — there is a real PR with a real base.
3. `git symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null`
   exits 0 and prints `origin/<X>` — the remote has set its default branch
   and the helper's resolution agrees with it.

Otherwise `$base` came from the helper's bottom fallback (`main`/`master`
guessed from a verified-or-defaulted lookup with no PR or remote-HEAD signal).
That is **not a confirmation**, it is a guess.

Behaviour by signal:

- **Strong signal (cases 1–3 above):** continue to Phase 1; print
  `Base: origin/$base (signal: override|pr|remote-head)` so the user still
  sees what was used.
- **Weak signal (no case 1–3 hit):** stop before Phase 1 and ask the user
  using `$AVAD_ROOT/shared/ask-format.md`. Do not produce findings until they
  pick. If the user does not pick, the run completes as `NEEDS_CONTEXT`.

Example ask for the weak-signal case:

```
ASK: What base branch should /avadbeta-review diff against?
WHY: No PR base was found, origin/HEAD is not set, and AVAD_BASE_BRANCH is unset; avad-base-branch fell back to "$base" without a real signal.
OPTIONS:
  A. Use $base (accept the fallback).
  B. dev (common team default).
  C. Other — name the branch; the run reruns with AVAD_BASE_BRANCH=<name>.
DEFAULT: none, please pick
```

If the user picks A, treat the answer as a confirmation: re-run from Phase 1
with the resolved `$base`. If the user picks B or C, set
`AVAD_BASE_BRANCH=<chosen>` and re-run preflight; the override path then makes
the signal strong.

Both helpers (`avad-init`, `avad-base-branch`) are pure bash with no runtime
dependencies. If either cannot run, see Error behavior below.

## Phase 1: Collect the diff

Reaching Phase 1 means the base passed the Preflight Step 3 check. Record, in
order:

1. Current branch: `git branch --show-current`.
2. Base branch: print `Base: origin/$base (signal: <override|pr|remote-head>)`
   so the trust source is visible alongside the name.
3. Fetch the base quietly so comparisons are against the latest remote state:
   `git fetch origin "$base" --quiet`. If `origin/$base` does not exist on the
   remote after the fetch, that is a `BLOCKED` condition (see Error behavior).
4. Diff summary: `git diff "origin/$base" --stat`. The plain `git diff`
   form (no `...HEAD`) covers committed history *and* the working-tree changes
   to tracked files, so a user who runs `/avadbeta-review` before committing still
   gets reviewed.
5. Changed file list — the union of two sources, because `git diff` does not
   list untracked files:
   - tracked, modified: `git diff "origin/$base" --name-only`
   - untracked, new: `git ls-files --others --exclude-standard`

   Treat each untracked file as a whole-file addition: read the file in Phase
   2, mark it as new in the diff context. Without this step a user who creates
   a new file and runs `/avadbeta-review` before `git add` would see an empty diff
   and the review would skip the file entirely.
6. Commit summary for context only: `git log --oneline "origin/$base"..HEAD`.

Commit messages are context, not evidence. Do not generate findings from a
commit message alone — every finding must be backed by a line in the diff or a
line in the changed file.

## Phase 2: Read the changed files

Read the actual diff and the surrounding code, not just the patch hunks, for
every changed file. A finding that depends on code outside the hunk requires
reading that code before the finding is written.

Walk risky surfaces first. In order:

1. **Auth, permissions, session, and secrets** — added or changed auth paths,
   permission checks, token handling, logging of credentials, new env vars.
2. **Data writes and migrations** — schema changes, backfills, column renames,
   destructive queries, writes without transactions, concurrent-write races.
3. **External boundaries** — HTTP handlers, RPC, queue consumers, webhooks,
   CLI entrypoints. New inputs the caller can control.
4. **Security and privacy** — injection surfaces (SQL, shell, HTML, template),
   SSRF, deserialization, user data logged or exposed.
5. **Deploy, CI, and release surfaces** — workflow files, Dockerfiles, release
   scripts, version bumps, feature flags.
6. **Remaining behavior changes** — logic changes, regressions, dead code the
   diff leaves unreachable, tests removed or weakened.
7. **Test coverage** — which changed behaviors have tests, which do not, which
   tests were added but assert nothing load-bearing.

If the diff is too large to read fully in one pass (more than ~2000 changed
lines or more than ~40 changed files), say so and complete as
`DONE_WITH_CONCERNS` after reviewing the highest-risk surfaces. Do not pretend
to have read what was not read.

## Phase 3: Produce findings

A finding is specific and actionable. Each finding has:

- **Severity:** `Critical`, `High`, `Medium`, or `Low`.
- **Location:** `path:line`, or the narrowest location the diff supports.
- **What is wrong:** one sentence naming the defect.
- **Why it matters:** one or two sentences tying the defect to a concrete
  failure mode (user impact, data loss, security, regression, broken test).
- **What should change:** one or two sentences describing the correct fix,
  not a vague "consider refactoring".

Severity rubric:

| Severity | Use when |
|---|---|
| `Critical` | Exploitable security defect, data loss, or broken production path visible in the diff. |
| `High` | Likely bug, regression, or missing gate on a risky surface. |
| `Medium` | Real defect with limited blast radius, or a missing test for risky new behavior. |
| `Low` | Narrow correctness issue or a concrete test gap on low-risk code. |

Non-findings — do not report:

- Pure style preferences, naming, or layout opinions.
- Speculative issues with no line in the diff to point at.
- "Consider" suggestions that do not describe a concrete defect.
- Broad rewrites or architectural pivots unrelated to the diff.
- Anything already addressed inside the diff being reviewed.

## Phase 4: Test and residual-risk reporting

Every run reports both `Test Gaps` and `Residual Risk`, in that order, after
findings. They are part of the contract on every run, not a no-findings
consolation. Either may be `none` when genuinely empty, but the heading does
not get omitted.

- **Test gaps:** changed behaviors with no test, new error paths with no test,
  regression tests that would have caught the defect being fixed.
- **Residual risk:** risky surfaces the diff touched that cannot be fully
  verified from the diff alone (integration with an external service,
  migration behavior under load, environment-specific paths).

If there are no findings, this section carries the run. "No issues found" is
only honest when test gaps and residual risks are reported alongside it.

## Output format

Start with findings first. Findings block, then test gaps, then summary. Then
the completion block.

If findings exist:

```
Findings
- [Severity] path:line — title
  Why it matters:
  What should change:

Open Questions  (only when a finding depends on a fact the diff cannot confirm)
- ...

Test Gaps
- ...

Residual Risk
- ...

Summary
- ...
```

If no findings:

```
Findings
No issues found.

Test Gaps
- ...

Residual Risk
- ...

Summary
- ...
```

Both `Test Gaps` and `Residual Risk` are required in every run, including
runs with findings — Phase 4 reporting is part of the contract, not a
no-findings consolation. Each may be `none` if genuinely empty; do not omit
the heading. They cover different things: `Test Gaps` is missing tests on
changed behaviour, `Residual Risk` is risky surfaces the diff touched that
cannot be fully verified from the diff alone (external services, migration
behaviour under load, environment-specific paths).

The `Open Questions` block is optional. Omit the heading entirely when no
finding depends on information the reviewer cannot confirm from the diff.
Empty section headings are noise *for that block*; the required Phase 4
sections above are not optional and stay present with `none` when empty.

## Decisions

Classify every decision per `$AVAD_ROOT/shared/decision-taxonomy.md`:

- **Mechanical** (which file to read first, which grep to run, how to order
  findings): decide silently and proceed.
- **Taste** (whether a borderline Medium should be Low, whether a style-ish
  finding crosses into correctness): surface only when the user's preference
  genuinely controls review scope. Do not pre-commit.
- **User-challenge** (the user asks for a rubber stamp, asks to ignore a
  concrete risk the diff creates, or asks to re-label a Critical as Low):
  stop and defend the disagreement in plain language. Do not soften into a
  yes/no question.

## Anti-rationalization

Stop and reconsider if any of these appear:

- Skipping the diff and writing findings from the PR description or commit
  messages.
- "The code looks clean" or "this looks good overall" before any file has
  been read.
- Reporting a style preference dressed as a correctness finding.
- A finding whose evidence is a hunch rather than a line in the diff or in
  the changed file.
- Expanding into applying fixes when the user only asked for review.
- Marking `DONE` without having read the changed files.
- Praising the review itself ("comprehensive", "thorough") instead of
  reporting defects.

## Error behavior

If a required helper is missing or not executable (`avad-init`,
`avad-base-branch`, or `git` itself), report `BLOCKED` with:

- **What happened:** the exact failing step, e.g. `avad-base-branch exited 127`.
- **Why:** the concrete cause, e.g. `helper not on PATH` or `not a git repo`.
- **Next step:** the single command or action the user runs to recover.

If the base branch resolves but the diff cannot be read (for example,
`origin/$base` does not exist on the remote), report `BLOCKED` with the same
structure. Never echo raw shell output as the final user-facing outcome.

If the diff is readable but partially inaccessible (binary blobs, submodule
pointers, vendored generated files the reviewer intentionally skipped), report
what was skipped and why, then complete as `DONE_WITH_CONCERNS`.

## Voice

Follow the voice rules at `$AVAD_ROOT/shared/voice.md`: no sycophancy, no AI
vocabulary, no em dashes, concrete over abstract. The review is a report, not
a pitch.

## Completion

End with a completion block per `$AVAD_ROOT/shared/completion-protocol.md`.

Valid states for `/avadbeta-review`:

| State | Use when |
|---|---|
| `DONE` | Diff and changed files read, findings (or a clean result plus residual risk / test gaps) reported. |
| `DONE_WITH_CONCERNS` | Review completed but important context was unavailable: diff too large to read fully, tests missing on a risky surface, inaccessible files, or verification that depends on a service the reviewer could not reach. |
| `NEEDS_CONTEXT` | Base branch ambiguous, diff empty in a way that may not be intended, or a taste decision is waiting on the user's reply. |
| `BLOCKED` | Required helper or tool failure, or the diff cannot be produced (missing base on remote, not a git repo) and no safe fallback exists. |

The `NOTES:` block summarizes the run:

- `Base:` the base branch used and the signal it came from (e.g.,
  `origin/dev (signal: pr)`).
- `Files reviewed:` count of changed files the reviewer actually read.
- `Findings:` count by severity (e.g., `0 Critical, 1 High, 2 Medium, 0 Low`).
- `Test gaps:` short count or `none`.
- `Residual risk:` short count or `none`.

Include sentinel acknowledgement lines only for the shared protocols the run
actually used:

- `Voice contract loaded v01`: the skill's output follows voice.md.
- `Completion protocol loaded v01`: the skill ends with this block.
- `Decision taxonomy loaded v01`: include only when a taste or user-challenge
  decision was classified during the run.
- `Ask format loaded v01`: include only when the run produced an `ASK:`
  block (empty-state, ambiguous base, taste, user-challenge).

Example completion (clean diff, base confirmed via PR signal, no taste calls):

```
COMPLETION: DONE
SUMMARY: Reviewed 7 changed files against origin/dev; no findings; two test gaps on new error paths.
NEXT: none
NOTES:
  - Base: origin/dev (signal: pr)
  - Files reviewed: 7
  - Findings: 0 Critical, 0 High, 0 Medium, 0 Low
  - Test gaps: 2
  - Residual risk: none
  - Voice contract loaded v01
  - Completion protocol loaded v01
```

`$AVAD_ROOT` is defined in the bootstrap: on Claude Code it resolves to
`${CLAUDE_PLUGIN_ROOT}`; on Codex it resolves to the avad install path named
in the installed `AGENTS.md` block. Do not substitute a source-repo path.

## What /avadbeta-review must not do

- Generate findings from commit messages or the PR description without a line
  in the diff to back them.
- Praise the code, praise the branch, or praise the review before reporting
  defects.
- Report style preferences as correctness findings.
- Apply fixes, rewrite code, open PRs, push branches, or run `/ship`. The
  skill reports; the user decides when and whether to fix.
- Mark `DONE` without having read the changed files.
- Invent tests or test names to fill a "Test Gaps" section.
