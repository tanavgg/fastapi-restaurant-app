---
name: workflow-change
description: Follow this whenever changing any git-tracked file — code, migrations, config, or documentation. Applies to every implementation task, whether or not the human names it. Covers what to write, what to hand back, and the two notebook files that record it.
---

# workflow-change

The standing workflow for changing git-tracked files. It applies **implicitly** — any task that
edits tracked files runs this way, without being asked.

Trivial one-liners (a typo, a version bump) do not need the notebook files. Anything that is a
feature, a fix, a migration, or a decision does.

## Inputs the human may pass

An analysis file, a list of changes, answers to open questions, or nothing at all. Use whatever is
given; when nothing is given, work from `docs/plan.md` and `CLAUDE.md`.

## Doing the work

- Follow `CLAUDE.md` — layering, naming, money, time, migrations, security. It is not advisory.
- Stay inside the current phase. New ideas go to the backlog, not the branch.
- Record non-obvious decisions as an ADR in `docs/adr/`, and update `docs/adr/README.md` in the same
  commit.
- **Never reference `personal/` from a git-tracked file.** Durable content belongs in
  `docs/plan.md`, `docs/adr/`, or `CLAUDE.md`.

## What NOT to do

- **Do not commit or push.** Hand back the command instead.
- **Do not run the high-level verification yourself** — the full test suite, `pre-commit run -a`,
  migrations against a real database, the dev server. Hand those back as commands for the human to
  run. Running a targeted check while working (a single test, a type check on one file, a syntax
  check) is fine and often sensible; the sweep at the end is theirs.

## What to hand back

Two files, plus a short summary in the reply.

### 1. `personal/change/{phase}/{NNN}_{PascalCaseName}.md` — immutable

Everything needed to finish and verify this change. Written once, never edited.

```markdown
# {NNN} — {Title}

One or two sentences on what changed.

## Files
Grouped by what they do, one line each. New files marked as new.

## Commands to run
Numbered, in order, each with one line saying what it does and what a good result looks like.
Migrations, test suites, lint sweeps, codegen, service restarts.

## Commit
    git checkout -b <branch>          # if not already on it
    git add <paths>
    git commit -m "<type>(<scope>): <imperative subject>" -m "<body: why>"

Follow docs/conventions.md. Never `git add -A` when untracked files may be present.

## Notes
Anything specific to this change the human needs: a config value to set, a service to start,
a manual verification step, a known follow-up. Omit if there is none.
```

### 2. `personal/decision/{phase}/{NNN}_{PascalCaseName}.md` — mutable

The high-level *why*, kept current. This is what a future session reads to understand the shape of a
feature without reading the diff, and what the human re-reads in three weeks. **Edit it freely,
including after the commit**, as the picture changes.

```markdown
# {Title}

**Status:** in progress | done | superseded
**Phase:** phase-NN · **Branch:** <branch>

## What this is
A paragraph. What was built and what problem it solves.

## Decisions
Each decision, what was chosen, and the one-line reason. Alternatives that were seriously
considered and dropped, with why. Link an ADR where one exists.

## Open threads
Anything unfinished, uncertain, or deliberately deferred.
```

Scope: one file per feature, or one per phase when the phase is small. **Reuse and update the
existing file when continuing the same feature** — do not create `002_SameThingAgain.md`.

## Naming

- `{phase}` — `phase-00`, `phase-01`, … from `docs/plan.md` §7.
- `{NNN}` — next free number within that phase folder; check the folder, do not guess.
- `{PascalCaseName}` — names the feature: `002_RefreshTokenRotation.md`. Change and decision files
  for the same feature should share the name.

Create directories as needed.

## In the reply

State what changed in a few lines, give the paths to both files, and surface anything the human must
decide or watch out for. Do not make them open a file to find out whether something needs their
attention.
