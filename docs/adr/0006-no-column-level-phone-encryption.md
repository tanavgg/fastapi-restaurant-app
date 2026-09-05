# ADR-0006 — No column-level encryption of phone numbers

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

The system stores phone numbers in three places: `users.phone`, `user_addresses.phone`, and the
per-order snapshot `order_delivery_details.recipient_phone`. Phone numbers are personal data
under GDPR, and application-level (column) encryption is a frequently recommended control for
sensitive columns.

The question is whether encrypting these specific columns buys enough to justify what it costs.

The threat it defends against is narrow: an attacker who obtains the database contents — a
stolen backup, a dump, a read-only replica leak — but *not* the application's key material. It
does **not** defend against the far more likely paths in an application like this one: SQL
injection through the app, a compromised application host (which holds the key), a leaked
credential with app-level access, or an over-permissioned admin endpoint.

## Decision

**Phone numbers are stored in plaintext columns. No column-level or application-level
encryption.**

The protections actually relied on, all of which cover the realistic threats better:

- **Encryption at rest** at the volume/disk level (managed Postgres or LUKS), which covers the
  stolen-media and stolen-backup cases that column encryption also covers.
- **TLS in transit**, everywhere.
- **PII never reaches the logs** — a structlog redaction processor covering `email`, `phone`,
  `password`, `token`, `address`, `card`, `authorization`, **with a test that asserts it**.
- **`audit_log` contains no PII — IDs only**, which is what makes erasure possible while keeping
  the security trail.
- **Anonymisation on erasure:** on GDPR erasure `users.phone` is nulled, `user_addresses` rows
  are hard-deleted, and `order_delivery_details` is scrubbed (`scrubbed_at`), while the order
  and ledger records survive.
- **Data minimisation:** no date of birth, no gender, nothing collected without a feature behind
  it.
- Authorization enforced in the service layer with explicit IDOR tests — the control that
  actually stops a user reading another user's phone number.

## Consequences

- Phone numbers remain searchable, indexable, sortable and joinable. Support lookup by phone
  and deduplication stay possible without a deterministic-encryption scheme and its own
  weaknesses.
- No key management, no key rotation procedure, no re-encryption migration, and no class of
  outage where the key is unavailable and the application cannot read its own data.
- A database dump obtained *without* application access exposes phone numbers, mitigated only
  by disk-level encryption. This is an accepted risk for a non-commercial project and is
  recorded here explicitly rather than left implicit.
- If this ever handled real customer data at scale, this ADR should be revisited — the
  conclusion may well change, and the revisit is a new ADR superseding this one.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **Randomised column encryption (AES-GCM per value)** | Destroys equality search and indexing on the column. Lookup-by-phone would need a full scan and decrypt of every row. |
| **Deterministic encryption / blind index** | Restores equality lookup, but leaks equality (identical numbers produce identical ciphertext), still breaks prefix and range search, and adds a key-management burden for a marginal gain. |
| **`pgcrypto` in the database** | Puts the key next to the data for the threat model that matters, so it defends against very little that disk encryption does not already cover. |
| **Tokenisation via an external vault** | Correct at scale, wildly disproportionate here — an entire additional service for three columns. |

## References

- `docs/plan.md` §6 (Privacy / GDPR), §9.1
- ADR-0011 (audit log PII scope)
