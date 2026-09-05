# ADR-0001 — Record architecture decisions

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

This project makes a number of decisions that are cheap to make once and expensive to
re-litigate: async SQLAlchemy, a self-built auth stack, a double-entry ledger instead of a
balance column, and so on. [`docs/plan.md`](../plan.md) records **what** was decided; it is
deliberately terse and does not argue.

Six months from now the *reasoning* is the part that will be missing, not the code. Without
it, a decision that was made carefully is indistinguishable from a decision that was made by
accident, and the temptation is to "fix" it.

## Decision

Architecture decisions are recorded as Architecture Decision Records in `docs/adr/`, in the
Nygard/MADR style.

**File naming:** `NNNN-kebab-slug.md`, four-digit zero-padded, e.g.
`0004-double-entry-ledger.md`. The number is a stable citation ("superseded by ADR-0019") and
makes the directory sort correctly. Date-prefixed filenames were considered and rejected: they
sort chronologically but make cross-references ugly.

**Numbers are never reused and never renumbered.**

**Structure** — every ADR carries:

| Section | Content |
| --- | --- |
| Title | `# ADR-NNNN — <decision, stated as an outcome>` |
| Status | `Proposed` / `Accepted` / `Superseded by ADR-NNNN` / `Deprecated` |
| Date | ISO-8601, the date the status last changed |
| Context | The forces at play. What made this a decision rather than a default. |
| Decision | What we will do, in the active voice. |
| Consequences | What this makes easy, what it makes hard, and what it obliges us to do. |
| Alternatives considered | Each rejected option and the specific reason it lost. |
| References | Links to the analysis docs, source rules, or external material. |

**ADRs are never deleted or edited into a different decision.** A changed mind is recorded as a
*new* ADR, and the old one's status becomes `Superseded by ADR-NNNN` with a link. The record of
the change is the most valuable part of the log.

**When to write one:** any choice a competent reader might reasonably question, or any choice
that was made *against* a plausible alternative. Not for things with one obvious answer.

`docs/adr/README.md` holds the index table and must be updated in the same commit as a new ADR.

## Consequences

- Decisions arrive in the repository with their reasoning attached, so the plan can stay terse
  without the "why" being lost.
- A small, recurring cost: writing an ADR takes ~20 minutes and updating the index is a step
  that is easy to forget. The index is small enough to be checked by eye during review.
- ADR-0001 itself is somewhat ceremonial, but it documents the template, which is the point.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| Reasoning in commit messages | Not discoverable. Nobody reads six months of `git log` to find why the ledger has no balance column. |
| Reasoning inline in `docs/plan.md` | The plan is a specification, read while working. Interleaving pages of rejected alternatives makes it unusable for its actual job. |
| A single `DECISIONS.md` | Grows into an unreviewable wall, loses stable citation numbers, and invites in-place editing of past decisions. |
| Date-prefixed filenames | Sort chronologically, but `ADR-2026-09-05-async-sqlalchemy` is a poor citation. |

## References

- Michael Nygard, *Documenting Architecture Decisions* (2011)
- [`docs/plan.md`](../plan.md) — the *what*; an ADR is the *why*
