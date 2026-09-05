# ADR-0009 — `VARCHAR` + `CHECK` for evolving statuses; native Postgres enums only for fixed sets

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

Postgres has a native `ENUM` type. It is compact, self-documenting in the catalogue, and gives
type safety at the database level. The alternative — a `VARCHAR` column with a `CHECK` constraint
listing the permitted values — looks strictly worse on paper.

The difference is what happens when the set changes. Order statuses, job types, refund reasons and
audit actions will change: this project already knows it will add `OUT_FOR_DELIVERY` handling,
new job types in Phase 11, and new audit actions in every phase.

Altering a native enum in Postgres is awkward in specific ways:

- `ALTER TYPE ... ADD VALUE` **cannot run inside a transaction block** in older versions, which
  fights Alembic's transactional migrations. (Postgres 12+ relaxed this, but not for a value used
  in the same transaction.)
- **Removing or renaming a value has no direct support at all.** The procedure is: create a new
  type, alter every column that uses it, drop the old type — across every table, in one migration,
  with a table rewrite.
- Alembic's autogenerate **does not detect enum value changes**. The migration is silently empty and
  the mismatch surfaces at runtime as a constraint violation.
- Types are database-global, so two tables sharing an enum are coupled to each other's evolution.

A `CHECK` constraint has none of these problems: changing it is `DROP CONSTRAINT` + `ADD CONSTRAINT`,
transactional, autogenerate-visible, and per-table.

## Decision

**`VARCHAR` + a named `CHECK` constraint for any value set that will evolve. Native Postgres enums
only for sets that are genuinely fixed by the domain.**

| Storage | Used for |
| --- | --- |
| `VARCHAR` + `CHECK` | `payment_status` · `fulfillment_status` · `refund_status` · `restaurants.status` · `users.status` · `uploads.status` / `purpose` · `jobs.type` / `status` · `ledger_transactions.kind` · `ledger_accounts.account_type` · `audit_log.action` / `actor_type` · `order_adjustments.type` · `restaurant_members.role` · `user_roles.role` |
| Native enum | `food_type` (`VEG`/`NON_VEG`/`EGG`/`VEGAN`) · `day_of_week` (0–6) |

Supporting rules:

- The **Python side is the source of truth**: a `StrEnum` in `core/`, with the `CHECK` generated
  from it so the two cannot drift.
- `MetaData(naming_convention=...)` gives the constraint a stable name (`ck_orders_payment_status`),
  so an Alembic migration that alters it is readable rather than a random identifier.
- The test asserting "no pending autogenerate diff" (Phase 2) is what catches a `StrEnum` changed
  without its migration — and it works here precisely because autogenerate *can* see `CHECK`
  changes.

The two native enums stay native because they are closed sets defined by the world, not by the
product: a day of the week is not going to gain a value, and the veg/non-veg/egg/vegan
classification is the domain's own vocabulary.

## Consequences

- Slightly larger storage per row and no catalogue-level type. Irrelevant at this scale, and both
  columns are indexed as text anyway.
- Adding an order status is a one-line `StrEnum` change plus a generated migration that swaps one
  constraint — transactional, reviewable, reversible.
- No `ALTER TYPE` outside a transaction, and no "the migration was empty and production broke".
- The `CHECK` list appears in the migration rather than in a shared type, so two tables with
  similar-looking values are independent. That is intended.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **Native enums everywhere** | `ALTER TYPE` friction, no autogenerate detection, no supported removal or rename, and database-global coupling — all on exactly the columns most likely to change. |
| **Lookup tables with FKs** | Fully dynamic and referentially sound, but turns every status read into a join and every status constant into a row that seeding must guarantee. Right for user-editable vocabularies (`cuisines` *is* one); wrong for values the code branches on. |
| **Plain `VARCHAR`, validation in the application only** | The database is the last line of defence, and this is the layer that catches a bad value written by a migration, a script, or `psql`. |
| **Native enums for statuses, `VARCHAR` for the rest** | The inverse of the right split — statuses are the ones that change. |

## References

- `docs/plan.md` §5 (Database naming), §9
- ADR-0008 (three order status columns)
