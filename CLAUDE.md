# CLAUDE.md

## Project

A restaurant ordering platform: customers order (pickup or delivery) from restaurants and pay from
an in-app wallet topped up via Stripe test mode; owners onboard by application and manage hours,
menu and incoming orders; admins approve, moderate and report. The platform takes a commission.
Learning project — but architecture, data integrity, transactions and security are production grade.

**Read [`docs/plan.md`](docs/plan.md) before starting any task.** It is the source of truth: tech,
phases, every table with its intent, lifecycle APIs, and invariants. This file is its compressed
form for everyday work — if the two disagree, the plan wins.

Decisions already taken, with the reasoning and the rejected alternatives, are in
[`docs/adr/`](docs/adr/) — read the index before reopening one. Branch, commit and
phase-completion conventions are in [`docs/conventions.md`](docs/conventions.md).

**Current state:** Phase 1 — backend base setup. No application code exists yet.
*(Update this line at every phase boundary — it is what tells a new session where the project is.)*

**Actors:** `CUSTOMER` · `RESTAURANT_OWNER` · `ADMIN` · `SUPER_ADMIN`, a many-to-many in `user_roles`
(one person can be a customer *and* an owner). Restaurant-level access is separate and lives in
`restaurant_members` (`OWNER`/`MANAGER`/`STAFF`).

## Stack

Backend: Python 3.13+, uv, FastAPI, Pydantic v2, **SQLAlchemy 2.0 async** + asyncpg, Alembic,
Postgres 18, Redis, ARQ, MinIO (S3 API), Mailpit, Stripe test mode, Argon2id via `pwdlib`,
structlog, Ruff, mypy (CI) + basedpyright (editor), import-linter, pytest + testcontainers.

Frontend: Vite, React 19, TypeScript strict, TanStack Router, Redux Toolkit + RTK Query with
OpenAPI codegen, React Hook Form + Zod, Tailwind v4 + shadcn/ui, Vitest + MSW, Playwright.

Commands go through `just` at the repo root (`just dev`, `just test`, `just lint`, `just migrate`,
`just seed`, `just gen-api`).

## Architecture

`Router → Schema (DTO) → Service → Repository → Model`

1. Routers never import SQLAlchemy or touch the session. Services never see `Request`/`Response`/
   status codes — they raise domain exceptions, one handler maps them to HTTP.
2. Services never build response schemas; the router does, via `model_validate(obj)`.
3. Repositories **never commit**. They may `flush()` only to get a DB-generated ID.
4. The **service owns the transaction** via an explicit `UnitOfWork`. Only the outermost use case
   opens one. `get_db` rolls back and closes; it never commits. No auto-commit middleware.
5. **Never hold a DB transaction open across an external HTTP call** (Stripe, S3, SMTP) — split into
   two transactions, made safe by `idempotency_keys` and `outbox_events`.
6. **`lazy="raise"` on every relationship**; repositories declare `selectinload()` per use case.
   `expire_on_commit=False` on the session factory. No blocking I/O in `async def`. Never share one
   `AsyncSession` across concurrent tasks.
7. No `dict[str, Any]` or `Any` in service signatures. Aggregates return a frozen dataclass or
   Pydantic model — never a `Row` or a dict.
8. `core/` is the bottom of the dependency graph: it imports nothing from `models/`, `repositories/`,
   `services/`, `api/`. Enforced by import-linter.
9. Authorization is always `require_permission("restaurant:approve")` — never `require_role(ADMIN)`.
   Restaurant-scoped access always goes through `restaurant_members`.
10. Repositories filter soft-deleted rows in the base query, never at the call site.
11. Services are FastAPI dependencies; **repositories are not** — a service builds its own from the
    session. External adapters are built once in the lifespan and held on `app.state`.
12. Interfaces (`Protocol`, or `ABC` when sharing behaviour) for `EmailSender`, `PaymentGateway`,
    `ObjectStorage`, `TaskQueue`, `CacheBackend`, `RateLimiter`, `Clock`, `PasswordHasher`.
    Do not abstract repositories for DB-swapping, and do not build a DI container.

## Layout

```
backend/     python app + tests (own pyproject.toml)
frontend/    vite app (own package.json)
docker/      Dockerfiles + compose (Phase 12)
docs/        plan.md · adr/ · conventions.md · erd.md · retention.md · runbook.md
justfile · README.md · CLAUDE.md
```

```
backend/src/app/
  main.py                app factory, middleware, exception handlers, lifespan
  api/
    deps/                db.py · auth.py · pagination.py · services.py · common.py
                         (exports Annotated aliases: SessionDep, CurrentUser, OrderServiceDep …)
    v1/routers/          auth · users · restaurants · menu · cart · orders · wallet · jobs · admin
    v1/schemas/          request/response DTOs per domain
  core/                  config · logging · errors(base) · clock · types · constants
    security/            hashing.py · tokens.py   (pure, know nothing about User)
  db/                    session · base · mixins · uow · pagination · filtering
    migrations/          alembic
  models/ repositories/ services/
  workers/               arq task definitions
  scripts/               create_superadmin.py · seed_demo.py
tests/                   unit/ · integration/ · api/ · conftest.py
```

New code goes in the existing directory for its layer. If a change seems to need a new top-level
package, that is a design question — raise it rather than inventing one.

## Conventions

- **Money:** `BIGINT` minor units, every column suffixed `_minor` (`price_minor`, `amount_minor`).
  Never floats, never "cents". `currency CHAR(3)` alongside. USD, enforced in the app.
- **Time:** all `TIMESTAMPTZ`, stored UTC. "Is it open now" is computed in the restaurant's own IANA
  timezone. All time reads go through the `Clock` interface.
- **DB naming:** snake_case, **plural** tables, singular classes. UUIDv7 PKs. FKs `<singular>_id`,
  booleans `is_`/`has_`, timestamps `_at`. `MetaData(naming_convention=...)` so Alembic produces
  stable constraint names. Partial unique indexes on soft-deletable tables.
- **Enums:** `VARCHAR` + `CHECK` for anything that will evolve (order statuses, job types); native
  Postgres enums only for fixed sets (`food_type`, `day_of_week`).
- **API:** `/api/v1`, UUIDs in URLs, UTC ISO-8601. Pagination envelope
  `{items, page_info: {next_cursor, has_more, total?}}`; grammar `?limit=&cursor=&sort=-created_at&q=`.
  The **encoding is fixed; sortable/filterable fields are whitelisted per resource** in a typed
  Pydantic query model.
- **Errors:** RFC 9457 `problem+json` with a machine-readable `code` and an **optional typed `errors`
  payload declared per exception class** (`AppError(Exception, Generic[TErrors])`). The frontend
  renders from `code` + params, never by parsing `detail`. Declare them in `responses={...}` so they
  reach OpenAPI and the RTK Query codegen.
- **Migrations:** every model change ships with its Alembic migration in the same commit.
  Autogenerate is a **draft** — always read and edit the result (autogenerate misses server
  defaults, `CHECK`s, partial indexes and data moves). **Never edit a migration that has already
  been applied or committed** — write a new one. Reference data (cuisines) is a migration too.
- **Frontend:** TypeScript strict, no `any`. The generated API client (`just gen-api`) is never
  hand-edited — fix the backend schema instead. All list/filter/sort/pagination state lives in
  **typed URL search params**, so views are deep-linkable and the back button is correct. Errors
  are rendered from the machine-readable `code`, never by parsing `detail`.
- **Docstrings:** Google style on every public service and repository function.

## Security rules

- Access JWT 15 min, held in memory on the frontend. Refresh token opaque, SHA-256 hashed at rest,
  rotated every use. **A `sessions` row is one login/device; `refresh_tokens` are its chained
  children** — replay revokes the session, which is one row update (ADR-0012). ~10s grace window
  for double-tab refreshes. Instant revocation via `users.tokens_valid_from`.
- **No PAN, CVV or expiry may ever enter the API.** Wallets are credited **only** from a
  signature-verified, deduped Stripe webhook — never from a client success callback.
- `Idempotency-Key` required on order placement and wallet top-up.
- **`audit_log` carries no direct or free-text PII** — subject and target are IDs, `metadata` holds
  enumerable facts, never user-supplied strings. `ip` and `user_agent` are the one declared
  exception (ADR-0011). Written in the same transaction as the action.
- **PII never reaches the logs** (structlog redaction processor, with a test).
- GDPR erasure is **anonymisation in place**, never row deletion, after a 30-day grace period.
  Per-order address snapshots live in `order_delivery_details` precisely so they can be scrubbed
  while `orders` and the ledger survive.
- Uploads: presigned direct-to-storage only (bytes never pass through the API), magic-byte
  validation, EXIF stripping, dimension caps, generated keys.

## Data integrity

- The wallet is a **double-entry ledger**. There is no `wallets` table and no stored balance —
  a balance is always `SUM(credits) − SUM(debits)`. Ledger entries are **immutable**; corrections are
  compensating entries.
- Orders snapshot name, price and food type onto `order_items`. A menu change must never alter a
  past order.
- Order state is **three columns** — `payment_status`, `fulfillment_status`, `refund_status` — because
  one enum cannot express "delivered and partially refunded".
- Order placement is one transaction that locks stock rows in deterministic ID order.
- `orders.commission_rate_snapshot` is captured at placement; never read the live rate for reporting.
- The full invariant list is §12 of 003 — treat each as a required test.

## Testing

Unit (faked repos + frozen `Clock`, no DB) · integration (real Postgres via testcontainers) ·
API (`httpx.AsyncClient`, including a 401/403 matrix per endpoint) · concurrency races ·
invariant tests · Playwright E2E over `seed_demo.py`. **Never test against SQLite.**
Target ~80% on `services/` and `repositories/`; don't chase 100%.

## Working agreements

- Do not commit or push unless explicitly asked. Branches and commits follow
  [`docs/conventions.md`](docs/conventions.md): `phase-NN-slug` / `feat/slug`, Conventional Commits
  with an imperative subject, body explaining *why*.
- Do not add features beyond the current phase — new ideas go to the backlog, not the branch.
- **No documentation or application file references `personal/` or anything inside it.** That
  directory is the human's private working notebook (analyses, change notes, decision logs) and is
  git-ignored, so a fresh clone does not have it — nothing a reader of this repository sees may
  depend on it. Durable content belongs in `docs/plan.md`, `docs/adr/`, or `CLAUDE.md`.
  The agent workflow skills in `.claude/skills/` are the one exception: they write the notebook,
  and they create the directory on first use rather than assuming it.
- Prefer amending `docs/plan.md` over re-deriving a settled decision in conversation.
- Changes to git-tracked files follow the `workflow-change` skill; analysis requests follow
  `workflow-analyze`.
- Record non-obvious decisions as an ADR in `docs/adr/` (numbered, never renumbered, superseded
  rather than deleted), and update `docs/adr/README.md` in the same commit.
- **Dependencies:** `uv add` / `uv add --group dev` for the backend, `pnpm add` for the frontend.
  Never `pip install`, never hand-edit a lockfile. Lockfiles are committed.
- **New commands go in the `justfile`** — if a task needs a remembered incantation, it is a recipe.
- Never run a destructive database command against `restaurant_dev` without asking;
  `restaurant_test` is disposable and owned by the test harness.
