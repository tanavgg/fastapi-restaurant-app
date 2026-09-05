# ADR-0002 — SQLAlchemy 2.0 async, with no sync fallback

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

FastAPI's central advantage is an async request path. A sync endpoint is not *broken* —
Starlette runs `def` handlers in a bounded threadpool (~40 threads by default) — but each
thread carries real memory and context-switch cost, and the ceiling is reached long before the
database's.

The specific reason it matters for this application: request paths call **several independent
external services** — Postgres, Stripe, MinIO, SMTP. Async lets independent calls overlap
instead of serialising.

The standing objection to async SQLAlchemy is debuggability: a lazy load that fires outside an
active session surfaces as `MissingGreenlet` somewhere unrelated to the cause. That objection is
real, and it is also almost entirely addressable by configuration rather than by discipline.

An escape hatch ("start sync, switch later if debugging hurts") was explicitly considered and
rejected — a half-committed choice means writing every adapter twice and getting the benefit of
neither.

## Decision

**SQLAlchemy 2.0 async with `asyncpg`. Settled, with no sync fallback plan.**

Four rules make the async footguns loud instead of subtle. They are non-negotiable and are
mirrored in `CLAUDE.md`:

1. **`lazy="raise"` is the default on every relationship.** This is the single most important
   configuration line in the backend. It turns a lazy load outside the session into an
   immediate, obvious error naming the exact attribute, at development time. Repositories then
   declare `selectinload()` per use case — which also kills N+1 queries by construction.
2. **`expire_on_commit=False` on the session factory.** Otherwise every attribute expires after
   commit and the next attribute access attempts a reload outside the transaction.
3. **No blocking I/O inside `async def`.** Async clients, or `asyncio.to_thread`. `blockbuster`
   in the test suite fails the build on violations, so this is enforced rather than remembered.
4. **One session per request; never share an `AsyncSession` across concurrent tasks.**
   `asyncio.gather` over the same session is a bug — gather over separate sessions, or over
   non-DB work.

Honest limit of the claim: async does **not** make Postgres faster. The connection pool is
usually the real ceiling. Async buys concurrency under I/O wait, not throughput against a
saturated database.

## Consequences

- Every I/O-touching dependency must have an async client: `asyncpg`, `aiosmtplib`, `aioboto3`,
  `httpx`, ARQ. A library without one has to be pushed to a thread, deliberately.
- Alembic uses the async template; migrations still run synchronously inside it.
- `lazy="raise"` means every new use case must state what it loads. This is intended: it is
  extra keystrokes in exchange for no accidental queries.
- Test fixtures, factories and the `AsyncClient` harness are all async, which raises the cost of
  the Phase 1 test scaffolding but pays back from Phase 2 onward.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **Sync SQLAlchemy** | Simpler debugging, but forfeits the reason FastAPI was chosen, serialises independent external calls, and caps concurrency at the threadpool. |
| **SQLModel** | Fuses ORM models with Pydantic schemas, destroying the model/schema separation this architecture depends on (§4). Thinner feature surface and lags SQLAlchemy releases. |
| **Tortoise / Piccolo / Ormar** | Async-native, but small ecosystems, weaker migration stories, and near-zero industry presence — a tool learned once and never seen again. |
| **Raw `asyncpg`, no ORM** | Maximum SQL exposure, but hand-rolled identity mapping, relationship loading and migrations. *Adopted selectively instead*: Phase 11 reporting queries are written in SQLAlchemy Core or raw SQL. |

## References

- `docs/plan.md` §2 (locked decisions), §3 (technology), §4 (architecture rules 8–10)
