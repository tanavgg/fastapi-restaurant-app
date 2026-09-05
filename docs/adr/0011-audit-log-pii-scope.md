# ADR-0011 — The audit log stores IP and user agent; "no PII" means no direct or free-text PII

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

Two rules in the plan were in direct contradiction:

- §6 and invariant 15: *"the audit log contains zero PII — IDs only"*, which is what makes GDPR
  erasure possible while keeping the security trail intact.
- §9.1: `audit_log` has the columns `ip` and `user_agent`.

**An IP address is personal data under the GDPR** — recital 30 names it, and *Breyer* (C-582/14)
settled that dynamic IPs count when the controller has a lawful means to identify the subject. The
same applies to `consent_log.ip`, `sessions.ip` and `password_reset_tokens.requested_ip`.

So invariant 15, taken literally, fails on the first row written. It needed to be either enforced by
removing the columns, or restated honestly.

Removing them is not what real systems do, and not for sentimental reasons. In an actual security
investigation — "was this account taken over?", "did the same actor do this from two places?",
"which admin approved this restaurant, and was it from the office?" — the IP and the user agent are
the two fields that turn a log into evidence. Every mature audit log keeps them, and the regulation
anticipates exactly this: Art. 17(3)(b) and (e) exempt processing needed for legal obligations and
for the establishment or defence of legal claims from the right to erasure.

## Decision

**The rule is restated rather than the columns removed.**

> `audit_log` carries **no direct or free-text PII**. Subject and target are **IDs** — never an
> email address, name, phone number, postal address, card detail, or token. The `metadata` JSONB
> holds enumerable facts (`{"from": "PENDING", "to": "ACCEPTED"}`), never user-supplied strings.
> `ip` and `user_agent` are the **declared exception** and are stored in full.

Terms attached to that exception:

| | |
| --- | --- |
| **Lawful basis** | Legitimate interest — security, fraud prevention, and defence of legal claims (GDPR Art. 6(1)(f); erasure exemption Art. 17(3)(b)/(e)). |
| **Retention** | 2 years, the same as the audit row itself. The retention job is what bounds this, and it is therefore a privacy control, not just housekeeping. |
| **Erasure** | Audit rows are **not** anonymised when a user is erased. The `actor_user_id` points at a `users` row that has been anonymised in place, so the trail survives without resolving to a person. |
| **Access** | Reading the audit log is an admin permission (`audit:read`), and reading it is itself audited. |
| **Documented** | Recorded here and in the privacy notice, because "we keep IPs for two years" is a thing that must be disclosed, not assumed. |

**Invariant 15 becomes testable** and stays a required test:

> No `audit_log` row contains a value matching an email, phone-number or postal-address shape, and
> no value equal to any of the actor's or target's PII column values. `ip` and `user_agent` are
> excluded from the assertion.

That test is worth more than the original absolute rule, because it actually runs.

## Consequences

- The audit log is genuinely useful in an investigation, which is the reason it exists.
- The retention job becomes load-bearing for compliance: if it stops running, IPs accumulate beyond
  the stated period. It needs monitoring in Phase 13, not just implementation.
- A subject access request (Art. 15) must include audit rows relating to the requester. The export
  job needs to know that.
- The **application logs** remain absolutely PII-free — the structlog redaction processor covers
  `email`, `phone`, `password`, `token`, `address`, `card`, `authorization`, with a test. The audit
  log is a deliberate, bounded, documented exception; the log stream is not an exception at all.
- If this ever handled real customer data in the EU, this ADR is the paragraph a DPO would read
  first. It is written to survive that.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **Drop `ip` / `user_agent` from `audit_log`** | Keeps the slogan literally true and makes the log nearly useless for its stated purpose. Account-takeover investigation is exactly the case it was built for. |
| **Truncate IPs (IPv4 /24, IPv6 /48)** | The web-analytics convention, and a genuine privacy improvement — but it destroys the "same actor, same address" correlation that makes a security log worth keeping. Appropriate for analytics, not for audit. |
| **Store a keyed hash of the IP** | Allows equality matching while hiding the value, but kills subnet and geolocation analysis, and the key becomes a secret whose loss destroys the log's value retroactively. |
| **Move `ip`/`user_agent` to a separate `audit_log_context` table that erasure scrubs** | Structurally tidy and keeps the absolute rule — but erasing the context of a *security* event is the one case the Art. 17(3) exemption exists for, and it would let an attacker erase their own trail by requesting deletion. **This is the strongest rejected option; it is rejected because of that last sentence.** |

## References

- `docs/plan.md` §6 (Privacy / GDPR), §9.1, §12 (invariant 15)
- GDPR recital 30; *Breyer v Bundesrepublik Deutschland*, C-582/14; Art. 6(1)(f), Art. 17(3)(b)/(e)
- ADR-0006 (no column-level phone encryption)
