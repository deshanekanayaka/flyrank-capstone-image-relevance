# Domain docs

This file tells the engineering skills how to use the domain documentation of this
repository during an exploration of the codebase.

## Read these before you explore

- `CONTEXT.md` at the repository root. It is the glossary of the domain terms of this
  project.
- `docs/adr/`. Read each ADR that touches the area of your next piece of work.

If one of these files does not exist, continue without a word. Do not report the absence,
and do not propose to create the file in advance. The `/domain-modeling` skill creates
these files late, at the time that a term or a decision is resolved. The skills
`/grill-with-docs` and `/improve-codebase-architecture` reach it.

## File structure

This repository is single-context. It holds one `CONTEXT.md` and one `docs/adr/` at the
root.

```
/
├── CONTEXT.md
├── docs/adr/
│   ├── 0001-some-decision.md
│   └── 0002-another-decision.md
├── app/
└── eval/
```

## Use the vocabulary of the glossary

Your output names domain concepts in issue titles, refactor proposals, hypotheses, and
test names. Use the term as `CONTEXT.md` defines it. Do not drift to a synonym that the
glossary names under `_Avoid_`.

If the glossary does not hold the concept that you need, that is a signal. Either you
invent language that the project does not use, which you must reconsider, or there is a
real gap. If there is a real gap, note it for `/domain-modeling`.

## Report a conflict with an ADR

If your output contradicts an existing ADR, state the conflict. Do not override the ADR
without a word. Write the conflict in this form:

> This contradicts ADR-0007 (event-sourced orders). It is still worth a second look,
> because ...
