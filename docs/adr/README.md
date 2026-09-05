# Architecture Decision Records

Decisions that a competent reader might reasonably question, recorded with the reasoning that
produced them. [`docs/plan.md`](../plan.md) is the **what**; these are the **why**. Format, naming
and lifecycle rules are in [ADR-0001](0001-record-architecture-decisions.md).

**Numbers are never reused or renumbered. ADRs are never deleted — a changed mind becomes a new
ADR, and the old one is marked `Superseded by ADR-NNNN`.**

Update this table in the same commit as a new ADR.

| # | Decision | Status | Date |
| --- | --- | --- | --- |
| [0001](0001-record-architecture-decisions.md) | Record architecture decisions | Accepted | 2026-09-05 |
| [0002](0002-async-sqlalchemy.md) | SQLAlchemy 2.0 async, with no sync fallback | Accepted | 2026-09-05 |
| [0003](0003-self-built-authentication.md) | Build authentication in-house rather than adopt a provider or library | Accepted | 2026-09-05 |
| [0004](0004-double-entry-ledger.md) | A double-entry ledger, not a stored wallet balance | Accepted | 2026-09-05 |
| [0005](0005-no-domain-orm-mapper-layer.md) | No separate domain entities and no ORM↔domain mapper layer | Accepted | 2026-09-05 |
| [0006](0006-no-column-level-phone-encryption.md) | No column-level encryption of phone numbers | Accepted | 2026-09-05 |
| [0007](0007-restaurant-members-join-table.md) | Restaurant ownership lives in `restaurant_members`, not `restaurants.owner_id` | Accepted | 2026-09-05 |
| [0008](0008-three-order-status-columns.md) | Order state is three independent columns, not one status enum | Accepted | 2026-09-05 |
| [0009](0009-varchar-check-over-native-enums.md) | `VARCHAR` + `CHECK` for evolving statuses; native enums only for fixed sets | Accepted | 2026-09-05 |
| [0010](0010-minio-from-phase-1.md) | MinIO from Phase 1; no local-filesystem storage backend | Accepted | 2026-09-05 |
| [0011](0011-audit-log-pii-scope.md) | The audit log stores IP and user agent; "no PII" means no direct or free-text PII | Accepted | 2026-09-05 |
| [0012](0012-sessions-and-refresh-tokens.md) | A `sessions` parent row per login, with `refresh_tokens` as its children | Accepted | 2026-09-05 |

## Writing a new one

1. Take the next free number — check the table, not the filesystem.
2. Copy the section structure from ADR-0001: Status · Date · Context · Decision · Consequences ·
   Alternatives considered · References.
3. State the decision in the title as an outcome ("A double-entry ledger, not a stored wallet
   balance"), not as a topic ("Wallet design").
4. The **Alternatives considered** section is the part that earns the file. An ADR with no
   rejected alternatives is describing a default, not a decision.
5. Add the row here.
