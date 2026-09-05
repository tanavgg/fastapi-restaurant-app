---
name: workflow-analyze
description: Write an analysis document for a decision, investigation, or design question. Use when asked to "analyze", "create an analysis file", "look into", "compare options for", or "write up" something before implementing it — anything where the human needs to decide before code is written.
---

# workflow-analyze

Produce **one self-contained document** that a human reads to make a decision, and that a future
Claude session reads to pick up the task cold.

## Where it goes

```
personal/analysis/{phase}/{NNN}_{PascalCaseName}.md
```

- `{phase}` — `phase-00`, `phase-01`, … matching the roadmap in `docs/plan.md` §7. Use the phase the
  work belongs to, not today's date.
- `{NNN}` — zero-padded, next free number **within that phase folder**. Check the folder; do not
  guess.
- `{PascalCaseName}` — 2–4 words naming the subject: `003_RefreshTokenRotation.md`,
  `001_SearchIndexStrategy.md`. Name the *subject*, not the activity ("`Analysis`", "`Notes`",
  "`Investigation`" are all wrong).

Create the phase directory if it does not exist.

## The rules that matter

**Analysis files are immutable.** Once written, never edited. A later correction is a *new* file
that states the corrected position in full.

**Each file stands alone.** A phase may accumulate a dozen of these. File 010 must be readable by
someone who has read none of the previous nine. This is the rule most easily broken and the one
that makes the difference:

- Do **not** write as a diff against an earlier file ("as established in 004, but now…").
- Do **not** open with a recap of what came before.
- Restate the context you need, in a paragraph, in your own words. Repetition across files is the
  correct cost.
- Reference a previous file only when the human genuinely needs to open it — at most once or twice,
  and never as a substitute for explaining something.

**Write plainly.** Short sentences. Concrete nouns. No throat-clearing, no "it is worth noting
that", no summary of what the document is about to say. If a table is clearer than prose, use a
table. Assume an expert reader in a hurry.

**Cover only what was asked.** Adjacent problems you notice go in *Open questions*, not into the
body as unrequested work.

## Structure

Adapt it — this is a shape, not a form to fill in.

```markdown
# {NNN} — {Title}

One or two sentences: what this decides or investigates, and why it came up now.

## Context
What a reader needs to know to follow the rest. Self-contained. Include the constraint or
requirement that makes this non-trivial.

## {Body — the substance}
Findings, mechanics, whatever the question actually needs. Use as many sections as the subject
has parts. Name them for their content.

## Options            (only when the ask has more than one answer)
| Option | Pros | Cons |
Then: **Recommendation:** <one option>, because <one or two sentences>.

## Open questions
Numbered, each with a recommendation in bold. Only things the human must actually decide —
not rhetorical questions.

## Next step
Two or three sentences. What happens after this is read. Concrete.

## Branch
`phase-NN-slug` or `feat/slug` — the branch the resulting work should be done on.
Omit this section entirely if the analysis leads to no code change.
```

## When the ask involves choosing

Never present one option as if it were the only one. Give the real alternatives, compare them on
what actually differs (not a generic pros/cons wash), state a recommendation, and give the short
reason. One recommendation, stated plainly — not a survey with a shrug.

## After writing

Tell the human the path, and summarise the recommendation and open questions in the reply — do not
make them open the file to learn whether you found anything.
