---
status: accepted
date: 2026-10-13
---

# Record architecture decisions in docs/adr

Spur kept the reasons for its architecture decisions in `AGENTS.md`, in pull
request descriptions, or nowhere. We record each architecture decision as a
short markdown file in `docs/adr/`. Agents and humans find the reason with
`grep` before they change the code.

## Format

File name: `NNNN-slug.md`. Use the next free number.

```md
---
status: accepted
date: YYYY-MM-DD
---

# Short title of the decision

1-3 sentences: the context, the decision, and the reason.

References: https://github.com/ROCm/spur/pull/NNN
```

- `status` is mandatory: `proposed`, `accepted`, or `superseded by NNNN`.
- `date` is the date of the original decision. If you do not know it, omit
  the field.
- Add the sections `## Considered options` and `## Consequences` only when
  they add information.
- `References:` is optional. It contains only full URLs to public pull
  requests and issues of this repository.
- Do not add names. Git history records the authors.

## When to write an ADR

All three conditions must be true:

1. The decision is hard to reverse.
2. A reader of the code is surprised without the context.
3. The decision is the result of a real trade-off between alternatives.

## Rules

- An ADR is self-contained. Do not link to internal or private documents.
- Do not change the decision in an accepted ADR. To change a decision, write
  a new ADR and set the status of the old ADR to `superseded by NNNN`. You
  can correct typos and links.
- Do not copy the reason into other files. Other files state the rule and
  link to the ADR.
- The Sphinx site does not publish `docs/adr/` (`exclude_patterns` in
  `docs/conf.py`).
