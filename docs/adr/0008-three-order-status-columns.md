# ADR-0008 — Order state is three independent columns, not one status enum

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

An order has a status. The default model is one `status` column with a list of values —
`PENDING`, `PAID`, `PREPARING`, `DELIVERED`, `CANCELLED`, `REFUNDED` — and a transition table.

It falls apart on the first real combination. An order that was delivered and then partially
refunded is *both* `DELIVERED` and `PARTIALLY_REFUNDED`. A single column forces one of three bad
answers: invent a combined value (`DELIVERED_PARTIALLY_REFUNDED`, and then the cross product of
every fulfilment state with every refund state), overwrite the fulfilment state with the refund
state (destroying the fact that the food was delivered), or add a boolean beside the enum and admit
the enum was never the whole story.

The underlying reason is that payment, fulfilment and refund are **three independent state machines
driven by three different actors** — the customer's wallet, the restaurant's kitchen, and an
admin or automated policy — that happen to be attached to the same row.

## Decision

**`orders` carries three status columns, each its own state machine:**

| Column | Values | Driven by |
| --- | --- | --- |
| `payment_status` | `PENDING` · `PAID` · `FAILED` | Wallet debit at placement |
| `fulfillment_status` | `PENDING_PAYMENT` → `PLACED` → `ACCEPTED` → `PREPARING` → `READY` → `COMPLETED` / `OUT_FOR_DELIVERY` → `DELIVERED`; terminal `REJECTED` · `CANCELLED_BY_CUSTOMER` · `CANCELLED_BY_RESTAURANT` · `EXPIRED` | Restaurant, customer, expiry jobs |
| `refund_status` | `NONE` · `PARTIAL` · `FULL` | Refunds, derived from the ledger |

Allowed `fulfillment_status` transitions are **declared as data** and enforced in the service
layer. Every transition writes an `order_status_history` row and emits an `outbox_events` row in the
same transaction (invariant 6).

Each column is stored as `VARCHAR` + `CHECK` rather than a native enum — see ADR-0009.

Rules that keep the three from drifting into each other:

- `refund_status` is a **projection of the ledger**, never an independent source of truth. If it
  disagrees with the sum of compensating entries, the ledger is right.
- Cancellation policy is a **single function** over `(fulfillment_status, actor, clock)` — customer
  free until `ACCEPTED`, with a fee until `PREPARING`, not after; restaurant may reject until
  `PREPARING`.
- `REJECTED` and `EXPIRED` both auto-refund, which is a fulfilment transition *causing* a refund
  transition — the two machines interact, but neither owns the other's column.

## Consequences

- "Show me the order's status" is a presentation concern: the UI derives one human label from three
  columns. That derivation lives in one place on the frontend.
- Queries are more explicit — `WHERE fulfillment_status = 'READY'` rather than a value that also
  encodes payment and refund state. Indexes are per-column and narrower.
- Three transition tables to test instead of one, but each is small and each is honest.
- Partial refunds, refunds after delivery, and "paid but the restaurant never accepted" are all
  representable without a schema change. That is the whole return on the decision.
- A reviewer meeting this for the first time will ask why there is no `status` column. This ADR is
  the answer.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **One `status` enum** | Cannot express "delivered and partially refunded" without a cross-product of values. The list grows multiplicatively and every transition table grows with it. |
| **One enum + boolean flags** | Concedes the point while keeping the worst part: the enum still lies, and the flags are unconstrained. |
| **A separate `order_states` event table, status derived** | Event sourcing. Correct, more general, and disproportionate — every read becomes a fold. `order_status_history` already gives the audit trail without making it the source of truth. |
| **Status on related tables only** (`deliveries.status`, `refunds.status`) | Already the case for delivery mechanics, but the *order* still needs a queryable state for listings and the restaurant's incoming-orders board. |

## References

- `docs/plan.md` §9.6, §12 (invariants 3, 6)
- ADR-0009 (`VARCHAR` + `CHECK` over native enums)
