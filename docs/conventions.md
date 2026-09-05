# Working conventions

Branch, commit and workflow conventions for this repository. The plan is [`docs/plan.md`](plan.md);
architecture and code rules are in [`CLAUDE.md`](../CLAUDE.md); decisions and their reasoning are in
[`docs/adr/`](adr/).

## Branches

`main` is always green: tests pass, `pre-commit run -a` is clean. Work happens on a branch and
lands via a squash merge.

| Pattern | Use |
| --- | --- |
| `phase-NN-slug` | A whole roadmap phase — `phase-03-auth-core`, `phase-09a-order-placement` |
| `feat/slug` | A single feature inside a phase — `feat/order-idempotency` |
| `fix/slug` | A bug fix |
| `chore/slug` | Tooling, dependencies, CI |
| `docs/slug` | Documentation or ADRs only |

Lowercase, kebab-case, no personal prefixes. Delete the branch after merge.

## Commits

[Conventional Commits](https://www.conventionalcommits.org/), enforced by a `pre-commit` hook
(Phase 1).

```
<type>(<scope>): <subject>

<body — why, not what>

<footer — BREAKING CHANGE:, refs>
```

**Types:** `feat` · `fix` · `refactor` · `perf` · `test` · `docs` · `chore` · `build` · `ci` ·
`style` · `revert`

**Scopes** are the area touched: `auth`, `orders`, `wallet`, `menu`, `restaurants`, `db`, `api`,
`core`, `workers`, `frontend`, `ci`, `deps`.

Rules:

- Subject in the **imperative mood**, lowercase, no trailing period, ≤ 72 characters.
  "add refresh token rotation", not "added" or "adds".
- The body explains **why**. The diff already says what.
- One logical change per commit. A migration ships in the same commit as the model change that
  requires it.
- Never amend or rebase a commit that has been pushed to a shared branch.

```
feat(auth): revoke the whole token family on refresh replay

Presenting an already-rotated refresh token is either theft or a bug.
Revoking only the presented token would leave the attacker's copy live,
so the entire family_id chain is revoked and a HIGH audit event written.

Refs: ADR-0003
```

## Pull requests

Even solo, work lands through a PR — it is where the diff gets read as a whole.

- Title follows the commit convention.
- Body: what changed, why, how it was verified, and anything deliberately left out.
- **Squash merge**, with the PR title as the final commit subject.
- CI must be green (Phase 14 onward: lint → typecheck → backend tests → frontend tests → E2E).

**The one exception is the repository seed.** Phase 0 is committed directly to `main` — a pull
request needs a base to diff against, and there isn't one. From Phase 1 onward, `main` is only ever
written by a squash merge, and branch protection enforces that once CI exists (Phase 14).

## Definition of done for a phase

A phase is finished when **all** of the following hold:

1. `just test` is green — unit, integration (real Postgres), and API tests.
2. `pre-commit run -a` is clean: ruff, mypy, import-linter, alembic check, secret scan.
3. New public services and repository functions have Google-style docstrings.
4. New endpoints have OpenAPI summaries, descriptions, and declared error `responses={...}`.
5. README updated if setup or commands changed.
6. Non-obvious decisions taken during the phase are recorded as ADRs.
7. The **Current state** line in `CLAUDE.md` is moved to the next phase.
8. The branch is merged and deleted.

## Anti-scope-creep

An idea that arrives mid-phase goes on the backlog, **not** into the branch. This is the rule that
keeps the roadmap meaningful. The phase list is §7 of [`docs/plan.md`](plan.md); the board that
tracks it lives outside this repository.
