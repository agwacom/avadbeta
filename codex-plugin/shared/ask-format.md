<!-- avad-shared-sentinel: ask-format-v01 -->
# Ask Format

> Ask format loaded v01.

When a skill has to ask the user, the question follows this shape. The shape
exists so the user can answer with a single short message and so future
skills can replay the exchange consistently.

## Required structure

```
ASK: <one-sentence question, ends with "?">
WHY: <one-sentence reason this is a taste or user-challenge decision>
OPTIONS:
  A. <short option, with one-line tradeoff>
  B. <short option, with one-line tradeoff>
  C. <only if a third meaningfully different option exists>
DEFAULT: <one of A/B/C, or "none — please pick">
```

`OPTIONS` must list at most three. If the real space is wider than three, the
skill should ask a narrowing question first instead of dumping a survey.

## Anti-patterns

- Asking without a default ("what should I do?").
- Hiding the real choice behind a yes/no question that pretends to be neutral.
- Listing five options because the skill has not yet decided which two matter.
- Asking serial questions when one combined ask would let the user answer at
  once.

## Sentinel use

When this format is referenced, include the line `Ask format loaded v01` in the
completion footer.
