# ADR-0003 — Build authentication in-house rather than adopt a provider or library

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

The platform needs registration, login, email verification, password reset, session revocation,
role- and permission-based authorization, and restaurant-scoped ownership checks. The usual
advice — "never roll your own auth" — is about cryptographic primitives (hashing, token
signing), not about session bookkeeping.

This is also a learning project whose stated goal is production-grade security understanding.
Delegating auth to Auth0/Clerk/Supabase would remove precisely the part with the most to teach,
and would put an external dependency in front of the local development loop.

The genuine risk is not "we cannot write a login endpoint" — it is refresh-token handling,
revocation, and the dozen small leaks (user enumeration, timing, unbounded login attempts) that
distinguish working auth from safe auth.

## Decision

**Authentication is built in-house, on top of vetted primitives. No auth provider, and no
framework-level auth library (`fastapi-users`, `authlib`'s session layer).**

Primitives are *not* hand-written:

- **Argon2id** via `pwdlib[argon2]` with tuned parameters. (`passlib` is effectively
  unmaintained and is not used.)
- **PyJWT** for access-token signing and verification.
- `secrets` for opaque token generation; SHA-256 for refresh-token storage.

The design, in full:

- **Access token:** JWT, 15 minutes, carrying `jti`, `sub`, roles, `iat`. Held **in memory** on
  the frontend — never in `localStorage`.
- **Refresh token:** opaque high-entropy random, stored **SHA-256 hashed** (a fast hash is
  correct for high-entropy secrets; Argon2 here would only add latency). Delivered in an
  httpOnly `Secure` `SameSite` cookie scoped to the refresh path, and *also* accepted in the
  request body so a future mobile client works without redesign.
- **Rotation on every refresh**, with a constant `family_id` across the chain. One row in
  `sessions` per issued token; the chain sharing a `family_id` is one login.
- **Replay detection:** presenting an already-rotated or revoked token revokes the **entire
  family**, writes a HIGH-severity audit event, and forces re-login.
- **~10 second grace window:** a token rotated moments ago returns the *same* child token, so
  two browser tabs refreshing simultaneously do not log the user out. The frontend also does
  single-flight refresh.
- **Absolute session lifetime** of 30 days on top of per-token TTL.
- **Instant revocation** via `users.tokens_valid_from`: any access token with `iat` earlier than
  that value is rejected. Bumping one column is logout-everywhere and password-change. `user.status`
  is checked on every request too.
- IP and user-agent are recorded but **never hard-enforced** — mobile IPs change constantly.
- Login throttling: per-account lockout (`failed_login_count`, `locked_until`) plus per-IP rate
  limiting. Auth responses are deliberately generic so they do not reveal whether an email exists.

Authorization is separate and is always by **string permission**
(`require_permission("restaurant:approve")`), never by role name — see `CLAUDE.md`.

## Consequences

- We own the security bugs. This obliges: a table-driven 401/403 matrix test covering every
  endpoint (Phase 3 exit criterion), explicit IDOR probes, and the concurrency test for double
  refresh.
- Revocation is a real feature rather than a token-expiry approximation, because sessions are
  rows we control.
- No external dependency in the local dev loop, and no per-MAU cost.
- Refresh-token statefulness means a `sessions` table with a retention job (90 days) — accepted;
  it is also what makes per-device revoke possible.
- If this ever became a real product, the auth surface is the first thing to re-audit or replace.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **Auth0 / Clerk / Supabase Auth** | Removes the highest-learning part of the project, adds a network dependency to local development and to CI, and makes the `users` table a mirror of someone else's. Per-MAU pricing at scale. |
| **`fastapi-users`** | Opinionated about models, sessions and routers in ways that conflict with the layering rules in §4, and hides exactly the mechanics worth understanding. |
| **Stateless JWT only, no refresh table** | Simple, but revocation becomes impossible without a denylist, which is the stateful thing being avoided. Long-lived stateless tokens are the classic mistake. |
| **Session cookies only, no JWT** | Perfectly respectable and simpler, but forecloses the future mobile client and teaches less about token handling. |
| **Argon2 hashing of refresh tokens** | Unnecessary: refresh tokens are high-entropy random, so a fast hash is not the weak link — it only adds latency to every refresh. |

## References

- `docs/plan.md` §6 (Authentication), §9.1, §12 (invariants 9–11, 14)
- ADR-0012 (sessions and refresh tokens)
