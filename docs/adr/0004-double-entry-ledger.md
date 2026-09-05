# ADR-0004 — A double-entry ledger, not a stored wallet balance

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

Customers hold an in-app wallet: topped up via Stripe, debited when an order is placed,
credited back on refund. The platform also takes a commission, which must be split out of each
completed order and reported on later.

The obvious implementation is a `wallets` table with a `balance_minor` column, updated on each
transaction. It is also the implementation that goes wrong in the specific ways money goes
wrong: a balance that drifts from its history has no authority to be reconciled against, a
crash between "write transaction row" and "update balance" leaves the two permanently
disagreeing, and concurrent updates need row locking that is easy to get subtly wrong.

Money is also the part of this system with the strongest correctness requirement — it is
auditable, it is disputed by users, and it is the one place where "mostly right" is worthless.

## Decision

**The wallet is a double-entry ledger. There is no stored balance anywhere, and there is no
`wallets` table.**

Three tables:

- **`ledger_accounts`** — `account_type` ∈ `USER_WALLET` / `RESTAURANT_PAYABLE` /
  `PLATFORM_REVENUE` / `PLATFORM_CASH` / `STRIPE_CLEARING`, plus `owner_type` / `owner_id`
  (nullable for platform accounts) and `currency`. Unique on
  `(account_type, owner_id, currency)`.
  **A user's wallet *is* their `USER_WALLET` account** — one source of truth, no second table to
  keep in step.
- **`ledger_transactions`** — one balanced movement of money, with a `kind`, a
  `reference_type`/`reference_id`, and an `idempotency_key`.
- **`ledger_entries`** — the atoms: `transaction_id`, `account_id`, `direction`
  (`DEBIT`/`CREDIT`), `amount_minor` (> 0), `currency`.

**`ledger_entries` are immutable — never updated, never deleted.** A correction is a
compensating entry; a refund is a compensating transaction. The history is append-only, which
is what makes it evidence.

**A balance is always derived:** `SUM(credits) − SUM(debits)` over an account.

Invariants, each of which becomes a test (§12 of the plan):

1. Every transaction's debits equal its credits.
2. The whole ledger sums to zero.
3. Every account's balance equals the sum of its entries — there is nothing to drift.
4. All entries within a transaction share a currency (enforced by a DB `CHECK`).

`ledger_accounts` is a **table with a type column, not an enum**, so that loyalty points or a
promotions account are additive rather than a migration of the money model.

## Consequences

- Reading a balance is an aggregate query rather than a column read. At this project's scale
  that is a non-issue with an index on `(account_id)`.
- **If balance reads ever get slow, the answer is a snapshot/checkpoint table plus a
  reconciliation job — never a mutable balance column.** Reintroducing the column reintroduces
  every problem this ADR exists to avoid.
- Every money operation must be expressed as a balanced pair of entries. Crediting a wallet
  "from nowhere" is impossible by construction: a top-up debits `STRIPE_CLEARING` and credits
  `USER_WALLET`.
- Order payment can debit the wallet inside the placement transaction; commission is split into
  `RESTAURANT_PAYABLE` and `PLATFORM_REVENUE` at completion using
  `orders.commission_rate_snapshot`.
- A ledger transaction may reference multiple orders, which is what keeps multi-restaurant
  checkout additive later.
- More rows and more code than a balance column. That is the price of a wallet that can be
  audited and reconciled, and it is the intended lesson of the project.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **`wallets.balance_minor`, updated in place** | Drifts irrecoverably, has no authority to reconcile against, needs careful locking, and cannot answer "why is my balance this number". |
| **Balance column *plus* a transaction log** | Two sources of truth that disagree the first time a write is interrupted. The log is then either authoritative (making the column redundant) or not (making it useless). |
| **Single-entry transaction log** | Records that money moved but not where from and where to; commission splits and platform revenue become inferred rather than recorded. Cannot self-check by summing to zero. |
| **Cached balance column with a reconciliation job from the start** | Premature. Correct as a *later* optimisation on top of the ledger, and explicitly held in reserve above. |

## References

- `docs/plan.md` §2 (locked decisions), §9.7, §12 (invariants 1–2)
