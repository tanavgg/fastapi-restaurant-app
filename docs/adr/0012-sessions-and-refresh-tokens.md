# ADR-0012 — A `sessions` parent row per login, with `refresh_tokens` as its children

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

The refresh token rotates on every use, with a constant family identifier across the chain, and
replay of a rotated token revokes the whole family. The original schema expressed this as a single
`sessions` table holding **one row per issued refresh token**, with a `family_id` column tying the
chain together.

That model is correct about the security mechanics and wrong about the noun. Two features assume a
different meaning of "session":

- `GET /auth/sessions` — "your active devices", which the user reads and recognises.
- `DELETE /auth/sessions/{id}` — "sign out my old laptop".

With a 15-minute access token, a refresh happens roughly every 15 minutes. A single 30-day login
therefore produces on the order of a thousand rows. A naive "list my sessions" returns a thousand
entries for one device; a correct one has to `GROUP BY family_id` and pick the newest row per group
in every query. And `DELETE /auth/sessions/{id}` addressed at a row id revokes **one link in a
chain**, not the login — it happens to work today only because there is exactly one unrotated token
at any moment, which is a coincidence of the rotation rule rather than a property of the API.

In other words, the family was already the real entity; it just had no row.

## Decision

**Split the table. A `sessions` row is one login (one device). A `refresh_tokens` row is one issued
token.**

```
sessions
  id · user_id · created_at · last_used_at
  expires_at        -- absolute session lifetime, 30 days
  revoked_at · revoked_reason
  ip · user_agent   -- first seen, for the device list

refresh_tokens
  id · session_id → sessions.id
  token_hash (sha256, unique) · parent_id
  issued_at · expires_at · rotated_at · used_at
```

What this makes true:

- **Revocation is one row update.** `sessions.revoked_at` invalidates every token in the chain at
  once — logout, per-device revoke, and replay response are all the same single write, with no
  bulk update and no chance of missing a row.
- **Replay detection is a check, not a scan:** presenting a token whose `rotated_at` is set, or any
  token whose session is revoked, is a replay → revoke the session, write a HIGH audit event.
- **The device list is a plain `SELECT`** over `sessions WHERE revoked_at IS NULL` — no grouping, no
  window function, and it shows the user exactly what they expect to see.
- **Absolute session lifetime has an obvious home** (`sessions.expires_at`) instead of being
  recomputed from the oldest token in a chain.
- `DELETE /auth/sessions/{id}` now genuinely takes a session id.

Invariant 9 is restated: *at most one **unrotated** refresh token per session.* The ~10-second grace
window returns the *same* child token on a double-tab refresh, so it does not violate this.

The token chain is kept (`parent_id`) rather than collapsed to a single mutable row, because the
chain is the forensic record of a replay: it shows which token was reused and how far back the
attacker's copy came from.

## Consequences

- One extra table and one extra join on the refresh path. The refresh path is already doing a hash
  lookup and two writes; a join on a primary key is not the cost that matters.
- Retention has two steps rather than one: `refresh_tokens` are hard-deleted 7 days after expiry,
  `sessions` 90 days after revocation. Deleting a session cascades to its tokens.
- `sessions.last_used_at` is updated on every refresh, which is a write to the parent row on a hot
  path. Acceptable at this scale; if it ever matters, it can be updated lazily.
- The Phase 3 schema must be built this way from the start — retrofitting it after the auth service
  exists means rewriting the part of the codebase with the most security-relevant tests.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **One table, `family_id` column** (the original plan) | The family is the real entity but has no row, so revoking it is a bulk update, listing devices needs a group-and-pick-newest in every query, and the "session" the API exposes is not the row the table stores. |
| **One table, and expose the *first* row of each family as the session** | Makes the session id stable, but revocation is still a bulk update and the device list is still a grouped query. Solves the naming, not the mechanics. |
| **Rename the table `refresh_tokens` and derive sessions as a view** | Honest naming and no new table, but a view over a grouped query is what every device-list and revoke call would pay for, and revocation still touches N rows. |
| **Store only the current token, overwriting on rotation** | Simplest, and destroys replay detection entirely — an overwritten token is indistinguishable from one that never existed. |

## References

- `docs/plan.md` §6 (Authentication), §9.1, §10, §12 (invariants 9–10)
- ADR-0003 (self-built authentication)
