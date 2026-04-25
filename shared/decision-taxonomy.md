<!-- avad-shared-sentinel: decision-taxonomy-v01 -->
# Decision Taxonomy

> Decision taxonomy loaded v01.

Every decision a skill must make falls into one of three classes. The class
determines who decides and how loud the decision must be.

| Class | Owner | Behavior |
|---|---|---|
| **Mechanical** | The skill | Decide silently and proceed. Log only if the choice is unusual. |
| **Taste** | The user | Surface the choice with a short rationale and ask. Do not pre-commit. |
| **User-challenge** | The user | Stop and ask. The skill explicitly believes the user's stated intent may be wrong; the skill must defend that belief in plain language. |

## Discipline

- Mechanical decisions never become taste decisions because the skill is
  uncertain. Uncertainty about *facts* is a research problem, not a taste
  decision. Resolve it before asking.
- Taste decisions are not user-challenges. Asking "do you want X or Y?" with
  no opinion is a taste decision. Saying "I think X is wrong because..." is a
  user-challenge.
- User-challenges are rare and load-bearing. Do not soften them into taste
  questions. If the skill believes the user is wrong, say so.

## Escalation

If a skill encounters more than one user-challenge in a single run, it should
stop and present them together rather than asking serially.

## Sentinel use

When this taxonomy is referenced, include the line `Decision taxonomy loaded v01`
in the completion footer.
