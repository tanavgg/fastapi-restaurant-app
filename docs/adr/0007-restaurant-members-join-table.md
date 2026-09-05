# ADR-0007 — Restaurant ownership lives in `restaurant_members`, not `restaurants.owner_id`

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

Every restaurant has exactly one owner, and one owner may have several restaurants. The obvious
model is a nullable-never `owner_id` column on `restaurants` — one FK, one join, and every
authorization check becomes `restaurant.owner_id == current_user.id`.

The pressure that breaks it is not hypothetical for this domain: a restaurant has staff. A manager
who edits the menu and accepts orders, a staff account for the till, an owner who wants to hand over
day-to-day operation without transferring the business. The moment any of those appears, an
`owner_id` column has to become a join table, and **every authorization call site in the codebase
has to change with it** — which is precisely the class of change most likely to be done in a hurry
and to leave one endpoint checking the old thing.

## Decision

**Restaurant-level access lives in a `restaurant_members` join table from day one, and every
restaurant-scoped authorization check goes through it. `restaurants` has no `owner_id`.**

```
restaurant_members
  restaurant_id · user_id · role ∈ (OWNER, MANAGER, STAFF)
  UNIQUE (restaurant_id, user_id)
  UNIQUE (restaurant_id) WHERE role = 'OWNER'    -- partial: exactly one owner
```

The partial unique index is what preserves the "exactly one owner" guarantee that `owner_id` gave
for free — it is invariant 13, and it is enforced by the database rather than by convention.

This is a **separate axis from platform roles.** `user_roles` says what someone is on the platform
(`CUSTOMER`, `RESTAURANT_OWNER`, `ADMIN`, `SUPER_ADMIN`); `restaurant_members` says what they may do
to *this* restaurant. Holding `RESTAURANT_OWNER` grants nothing on its own — it is membership that
grants access. Combined with the string-permission rule
(`require_permission("menu:edit")`, never `require_role`), the authorization question is always the
same shape: *does this user hold this permission, and are they a member of this restaurant?*

## Consequences

- Every restaurant-scoped query carries a join or an existence check. Encapsulated once in an
  ownership helper in the service layer, not repeated in routers.
- Adding `MANAGER`/`STAFF` behaviour later is a permission-matrix change, not a schema migration and
  not a sweep of call sites. That is the entire point.
- Ownership transfer is an `UPDATE` of one row, and it is auditable, rather than a mutation of the
  restaurant record.
- **This is the decision most at risk of being "simplified" later by someone who sees a join table
  with one row per restaurant and assumes it is over-engineering.** It is not: it is one table now
  in exchange for not rewriting authorization later. If it is ever removed, that must supersede this
  ADR with a stated reason.
- IDOR tests are mandatory on every restaurant-scoped endpoint — the join table makes the check
  explicit, which also makes forgetting it explicit.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **`restaurants.owner_id`** | Simplest today, but staff/manager support turns it into this table anyway *and* requires touching every authorization site at the moment the app is busiest. |
| **`owner_id` now, join table when staff arrives** | The migration is easy; the call-site sweep is not, and it is the part that leaks authorization bugs. |
| **Platform role `RESTAURANT_OWNER` as the check** | Would let any owner act on any restaurant. This is the IDOR bug the whole design is trying to make structurally impossible. |
| **Permissions per user per restaurant (full ACL)** | Correct and general, but three roles cover every case here; an ACL is a lot of machinery for no current requirement. Additive later. |

## References

- `docs/plan.md` §1 (Actors), §9.2, §12 (invariant 13)
- ADR-0003 (self-built authentication)
