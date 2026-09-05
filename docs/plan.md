# Plan — the source of truth

Every decision in this document is settled. It is the **what**: technology, phases, every table with
its intent, lifecycle APIs, and the invariants. The **why** behind the contested choices lives in
[`docs/adr/`](adr/) — read the ADR index before reopening one of them.

**How to use:** when starting a task, read §7 (the phase), §9 (the tables it touches), and §4–§6
(the rules). [`CLAUDE.md`](../CLAUDE.md) is the compressed version of §4–§6 for everyday work;
[`docs/conventions.md`](conventions.md) covers branches, commits and what "done" means.

---

## 1. Product

A restaurant ordering platform. Customers browse restaurants and menus, build a cart, and pay from
an in-app wallet topped up via Stripe (test mode). Orders are **pickup or delivery**. Restaurant
owners apply to onboard, get approved by an admin, then manage hours, availability, menu, stock, and
incoming orders. Admins oversee the platform and pull reports. The platform takes a commission.

Not a commercial product — but architecture, data integrity, transactions, and security are treated
as if it were.

### Actors

| Role | Intent |
| --- | --- |
| `CUSTOMER` | Browses, orders, holds a wallet, manages their own account and addresses. |
| `RESTAURANT_OWNER` | Owns one or more restaurants; full control over their own, nothing else. |
| `ADMIN` | Approves applications, suspends restaurants/users, reads reports and the audit log, refunds up to a cap. |
| `SUPER_ADMIN` | Everything an admin can, plus: grant/revoke `ADMIN`, change platform settings, GDPR erasure, unlimited refunds, restore archived entities. Bootstrapped by script only. |

Roles are a many-to-many (`user_roles`) — one person can be a customer *and* an owner.
Restaurant-level access is separate and lives in `restaurant_members` (`OWNER`/`MANAGER`/`STAFF`).

---

## 2. Locked decisions

| Decision | Value |
| --- | --- |
| Currency | **USD**, single-currency enforced in the app; `currency CHAR(3)` stored everywhere anyway. |
| Money storage | `BIGINT` **minor units**; every money column ends in `_minor` (`price_minor`, `amount_minor`, `total_minor`). Never floats, never "cents". |
| DB access | **SQLAlchemy 2.0 async** + asyncpg. Settled, no sync fallback. |
| Restaurant ownership | `restaurant_members` join table, **not** `restaurants.owner_id`. |
| Fulfilment | Pickup **and** delivery. No couriers, dispatch, or live tracking — the restaurant marks out-for-delivery and delivered. |
| Commission | Yes. Rate from `platform_settings`, **snapshotted onto each order** at placement. |
| Cart | One cart per (customer, restaurant). No multi-restaurant checkout — but `orders.order_group_id` exists so it stays additive. |
| Wallet | A **double-entry ledger**, not a mutable balance column. There is no `wallets` table — a wallet is a `ledger_account` of type `USER_WALLET`. |
| Auth | Self-built. Short-lived JWT access token + opaque rotating refresh token, chained under a `sessions` row, with **session-wide replay detection**. |
| Type checking | **mypy** in CI (the gate) + **basedpyright** in the editor (the feedback loop). |
| Object storage | **MinIO natively from Phase 1** (S3 API). One storage implementation ever; no local-filesystem backend. |

---

## 3. Technology

### Backend

| Tech | Intent |
| --- | --- |
| Python 3.13/3.14 | Latest stable that all deps support. |
| **uv** | One tool for Python versions, venv, lockfile, and script running. |
| **FastAPI** | HTTP layer, dependency injection, and OpenAPI generation. |
| **Pydantic v2** + `pydantic-settings` | Request/response DTOs, typed error payloads, and env config. |
| **SQLAlchemy 2.0 async** + `asyncpg` | ORM and query builder; `Mapped[]` gives native typing with no mypy plugin. |
| **Alembic** | Versioned migrations, including reference-data migrations. |
| **Postgres 18** | The database. Native `uuidv7()`, full-text search, `pg_trgm`. |
| **Argon2id** via `pwdlib[argon2]` | Password hashing. (`passlib` is unmaintained — do not use.) |
| **PyJWT** | Access token signing/verification. |
| **ARQ** + Redis | Background jobs: emails, reports, thumbnails, retention, order expiry. |
| **Redis** | Job queue, rate limiting, caching. |
| **MinIO** (S3 API) via `aioboto3` | Object storage for images, report artifacts, GDPR exports. |
| **Mailpit** + `aiosmtplib` | Local SMTP catcher with a web UI; real SMTP flow, no external account. |
| **Stripe** test mode + Stripe CLI | Wallet top-ups and refunds with real API semantics; `stripe-mock` in CI. |
| **WeasyPrint** + Jinja2 | HTML → PDF for reports. CSV for anything data-shaped. |
| **structlog** | JSON logs with request-id correlation and a PII-redaction processor. |
| **Ruff** | Lint + format (replaces black, isort, flake8). |
| **mypy** (strict) | CI type gate. |
| **import-linter** | Enforces the layering rules in §4 in CI, so they're real and not aspirational. |
| **pytest** + `pytest-asyncio` + `httpx.AsyncClient` | Test suite. |
| **testcontainers** | Real Postgres for tests. Never SQLite. |
| **polyfactory** | Test data factories. |
| **blockbuster** | Fails tests that make blocking I/O calls inside `async def`. |
| **pre-commit** | Runs ruff, mypy, alembic checks, secret scan, commit-message lint. |
| **just** | Task runner at repo root. |

### Frontend

| Tech | Intent |
| --- | --- |
| **Vite** + TypeScript (strict) | Build tooling. Non-SSR. |
| **React 19** | UI. |
| **TanStack Router** | Type-safe routes *and* search params — this app is full of filter/sort URLs. |
| **Redux Toolkit** + **RTK Query** | Server state and caching; a few slices for auth/UI state only. |
| **`@rtk-query/codegen-openapi`** | Generates typed hooks *and typed error shapes* from the backend's OpenAPI schema. The backend is the single source of truth. |
| **React Hook Form** + **Zod** | Forms and client-side validation; maps directly onto the field-keyed error payload (§5). |
| **Tailwind v4** + **shadcn/ui** | Components without spending time on design. |
| **Vitest** + React Testing Library + **MSW** | Unit/component tests. |
| **Playwright** | A handful of E2E happy paths, run against seeded data. |
| **ESLint 9** + Prettier | Lint/format. |
| Node 24 LTS + pnpm | Runtime and package manager. |

### Repo shape

```
backend/     python app + tests (own pyproject.toml)
frontend/    vite app (own package.json)
docker/      Dockerfiles + compose (Phase 12)
docs/        plan.md · adr/ · conventions.md · erd.md · retention.md · runbook.md
justfile · README.md · CLAUDE.md
```

---

## 4. Architecture

```
Router      thin: parse, authorize, call ONE service, serialize the response
  ▼
Schemas     pydantic DTOs. ORM models never cross the wire.
  ▼
Service     business rules, orchestration, OWNS THE TRANSACTION
  ▼
Repository  SQLAlchemy queries only. no business rules.
  ▼
Model       ORM tables, relationships, constraints
```

### Non-negotiable rules

1. Routers never import SQLAlchemy and never touch the session.
2. Services never see `Request`, `Response`, or status codes. They raise domain exceptions; one handler maps those to HTTP.
3. Services never construct response schemas — the router does, via `model_validate(obj)`.
4. Repositories **never commit**. They **may `flush()`** solely to obtain a DB-generated ID.
5. Only the **outermost** use case opens a Unit of Work. Inner services receive it and never open their own (use `begin_nested()` for a savepoint if genuinely needed).
6. `get_db` yields a session and **rolls back + closes** on exit. It never commits. No auto-commit middleware.
7. **Never hold a DB transaction open across an external HTTP call.** Split into two transactions, made safe by the idempotency and outbox tables.
8. **`lazy="raise"` is the default on every relationship.** Repositories declare `selectinload()` for what each use case needs. This is the rule that makes async SQLAlchemy painless.
9. `expire_on_commit=False` on the session factory.
10. Never blocking I/O inside `async def`. Never share one `AsyncSession` across concurrent tasks.
11. No `dict[str, Any]` or `Any` in any service signature. Aggregate/computed results return a frozen dataclass or Pydantic model — never a `Row` or a dict.
12. **`core/` is the bottom of the dependency graph**: domain-agnostic infrastructure that imports nothing from `models/`, `repositories/`, `services/`, or `api/`. Enforced by import-linter.
13. Authorization is always `require_permission("restaurant:approve")` — a string permission, never `require_role(ADMIN)`.
14. Repositories filter soft-deleted rows **at the base-query level**, never at the call site.

### Backend layout

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

**Dependency wiring:** services are FastAPI dependencies; **repositories are not** — a service
constructs its own repos from the session, keeping the graph shallow (router → service → repos).
External adapters (email, storage, payments, cache, queue) are built **once in the lifespan**, held
on `app.state`, and overridden in tests via `dependency_overrides`. No DI container.

### Swappable interfaces

`Protocol` for anything faked in tests; `ABC` only when sharing base behaviour.

| Interface | Implementations |
| --- | --- |
| `EmailSender` | SMTP/Mailpit · Fake (records messages) |
| `PaymentGateway` | Stripe · Fake |
| `ObjectStorage` | S3/MinIO · Fake |
| `TaskQueue` | In-process · ARQ |
| `CacheBackend` / `RateLimiter` | In-memory · Redis |
| `Clock` | Real · Frozen — makes open-now, expiry, and grace-period logic deterministically testable |
| `PasswordHasher` | Argon2 · dummy (tests run far faster) |

Repositories are **not** abstracted for DB-swapping; they get a `Protocol` only so services can be
unit-tested with fakes.

---

## 5. Cross-cutting conventions

### API
- Everything under `/api/v1`. UUIDs in URLs. All timestamps UTC ISO-8601 (`TIMESTAMPTZ` in the DB).
- Pagination envelope: `{ items: [...], page_info: { next_cursor, has_more, total? } }`.
- Query grammar (fixed **encoding**, per-resource **fields**): `?limit=&cursor=&sort=-created_at&q=`, `-` prefix = descending.
- Sortable and filterable fields are **whitelisted per resource** in a typed Pydantic query model. Not pedantry — arbitrary sort fields mean unindexed scans and column-existence leaks.
- Offset pagination is acceptable for small admin lists; cursor (keyset) pagination for public listings.

### Errors — one envelope, typed payloads
RFC 9457 `application/problem+json`, with an optional `errors` member whose shape is declared per
exception class via a Pydantic model on a generic base
(`class AppError(Exception, Generic[TErrors])` with `status`, `code`, `errors`).

```json
{ "type": "...", "title": "...", "status": 409, "detail": "...", "instance": "...",
  "code": "CART_INVALID", "trace_id": "01J8...", "errors": { ... } }
```

| Exception | `errors` shape |
| --- | --- |
| `ValidationError` | `dict[str, list[{code, message, params}]]` — field-keyed, feeds RHF `setError` directly |
| `InsufficientFunds` | `{required_minor, available_minor, currency}` |
| `CartInvalid` | `{items: [{item_id, reason: PRICE_CHANGED\|UNAVAILABLE\|OUT_OF_STOCK, old_price_minor, new_price_minor}]}` |
| `OrderTransitionNotAllowed` | `{from, to, allowed: [...]}` |
| `RestaurantClosed` | `{opens_at, timezone}` |
| `OutsideDeliveryArea` | `{distance_km, radius_km}` |

Every error carries a machine-readable **`code`**; the frontend renders from `code` + params, never
by parsing `detail`. Declare them in routes (`responses={409: {"model": ...}}`) so they reach OpenAPI
and the RTK Query codegen produces typed errors end-to-end.

### Database naming
- `snake_case`, **plural** table names (`user` is a reserved SQL word — that settles it). Class names singular.
- FKs `<singular>_id` · booleans `is_`/`has_` · timestamps `_at` · join tables `restaurant_cuisines`.
- `MetaData(naming_convention=...)` for `pk_`/`fk_`/`uq_`/`ix_`/`ck_` **set in the first migration**, so Alembic autogenerate produces stable names.
- **`VARCHAR` + `CHECK` for statuses that will evolve** (order statuses, job types). Native PG enums only for genuinely fixed sets (`food_type`, `day_of_week`). Native enums are painful to alter.
- Primary keys: **UUIDv7**, DB-generated. Sortable like an int, safe to expose.
- Unique indexes on soft-deletable tables are **partial** (`WHERE deleted_at IS NULL`) so a re-create isn't blocked.

### Money
`BIGINT` minor units + `currency CHAR(3)`. USD, enforced in the app. `CHECK` that all ledger entries
sharing a `transaction_id` share a currency. Never assume 2 decimal places in shared helpers.

### Time
All `TIMESTAMPTZ`, stored UTC. Each restaurant carries its own IANA `timezone`; "is it open now" is
computed in **the restaurant's** local time. All time reads go through the `Clock` interface.

---

## 6. Security & compliance requirements

### Authentication
- Argon2id password hashing with tuned parameters. Password policy enforced server-side.
- Access token: JWT, 15 min, carries `jti`, `sub`, roles, `iat`. Held **in memory** on the frontend.
- Refresh token: opaque random, **SHA-256** hashed at rest (high-entropy — a fast hash is correct here), in an httpOnly `Secure` `SameSite` cookie scoped to the refresh path. Also accepted **in the request body** so a future mobile client works.
- **Rotation on every refresh.** A login is one row in `sessions`; each issued refresh token is one row in `refresh_tokens` pointing at it. Rotation appends a child token to the same session.
- **Replay detection:** presenting an already-rotated/revoked token revokes the **whole session** (one row update, which invalidates every token in the chain), writes a HIGH audit event, and forces re-login.
- **~10 second grace window:** a token rotated moments ago returns the *same* child token, so a double-tab refresh doesn't log people out. Frontend also does single-flight refresh.
- Absolute session lifetime (30 days) on top of per-token TTL.
- IP and user-agent recorded, **never hard-enforced** (mobile IPs change constantly).
- **Instant revocation** via `users.tokens_valid_from`: reject any access token with `iat <` that value. Bumping one column kills all of a user's access tokens — that is logout-everywhere and password-change. `user.status` is checked on every request too. A Redis `jti` denylist is optional in Phase 12 for per-device granularity.
- Login throttling: per-account lockout (`failed_login_count`, `locked_until`) plus per-IP rate limiting. Auth responses are generic so they don't leak whether an email exists.

### Authorization
- String permissions, checked as dependencies. Role → permission matrix in code (movable to the DB later without touching call sites).
- **Ownership checks are separate from role checks** and always enforced in the service layer, tested explicitly for IDOR.
- An admin cannot grant themselves `SUPER_ADMIN`. The last `SUPER_ADMIN` cannot be demoted or deleted.

### Privacy / GDPR
- **Erasure = anonymisation, never row deletion** (orders and ledger entries are financial records; GDPR Art. 17(3) covers this).
- `DELETE /users/me` starts a **30-day grace period**: account deactivated and sessions revoked immediately, anonymised by a scheduled job afterwards, cancellable by logging in. Protects against account-takeover-driven destruction.
- On anonymisation: PII on `users` overwritten (`email → deleted+{id}@deleted.invalid`, name → `Deleted User`, phone/password nulled), `user_addresses` hard-deleted, sessions and tokens deleted, uploaded avatars deleted from storage, `order_delivery_details` scrubbed. Orders, ledger entries and the audit log survive.
- ⭐ **The audit log carries no direct or free-text PII.** Subject and target are IDs — never an email, name, phone, address, or card detail. `ip` and `user_agent` **are** retained (see §9.1): they are personal data, kept under legitimate interest for security and fraud investigation, retention-limited to 2 years, and out of scope for erasure. This is what makes erasure possible while keeping the security trail intact.
- ⭐ **Per-order address snapshots live in `order_delivery_details`, not on `orders`** — so they can be scrubbed without touching the financial record. This is a hard requirement, not a preference.
- Right of access/portability: `POST /users/me/export` → a background job → a ZIP of JSON/CSV behind a short-TTL presigned link.
- Consent: `terms_accepted_at`, `marketing_consent`, and a `consent_log`. Marketing email checks consent; transactional email is exempt.
- Data minimisation: no DOB, no gender, nothing without a feature behind it.
- **PII never reaches the logs** — a structlog redaction processor for `email`, `phone`, `password`, `token`, `address`, `card`, `authorization`. There is a test for this.
- Column-level encryption of phone numbers: **considered and rejected** (breaks search/indexing for marginal gain over disk encryption). Record as an ADR.

### Payments
- **No PAN, CVV, or expiry may ever enter the API.** Stripe Elements/Checkout only; store `payment_method_id` + `last4`/`brand` for display.
- Webhooks: signature verification, event-id dedupe table, idempotent handlers, replay-safe.
- **Never credit a wallet from the client's success callback** — only from a verified webhook.
- `Idempotency-Key` header required on order placement and wallet top-up.

### Uploads
Magic-byte validation (not the client's `Content-Type`), pixel-dimension caps (decompression bombs),
EXIF stripping (GPS), explicit `Content-Type` on the stored object, generated keys — never the
client's filename.

### Retention (`docs/retention.md`, enforced by jobs)
Sessions 90d · used/expired tokens 7d · job artifacts 7d · pending uploads 1h · audit log 2y ·
orders and ledger entries forever.

---

## 7. Task order

Each phase ends with: green tests, updated README/docstrings, OpenAPI descriptions, a merged branch,
and a moved Trello card. Ideas arriving mid-phase go to the backlog, not the branch.

### Phase 0 — Decisions & repo hygiene ✅
Agree this document. `.gitignore`, `.gitattributes`, `.editorconfig`, `.python-version` (3.13),
`LICENSE` (MIT), README skeleton, `CLAUDE.md`, `docs/conventions.md` (branch, commit, PR and
phase-completion conventions), `docs/adr/` with ADR-0001 (the format) plus one ADR per non-obvious
choice — async SQLAlchemy, self-built auth, ledger over balance column, no domain/ORM mapper layer,
no phone encryption, `restaurant_members`, three order status columns, `VARCHAR`+`CHECK` enums,
MinIO from Phase 1, audit-log PII scope, sessions/refresh-token split.

### Phase 1 — Backend base setup (no Docker)
uv project (**Python 3.13**) with dependency groups · native Postgres 18 (`restaurant_dev`,
`restaurant_test`) · **native MinIO** + the `ObjectStorage` implementation · `Settings` from env with
`backend/.env.example`, failing fast · app factory, `/healthz`, `/readyz`, CORS, request-id
middleware · structlog JSON with the PII redaction processor · error envelope, `AppError` generic
base, handlers · async engine, session factory (`expire_on_commit=False`), `get_db`, `UnitOfWork` ·
Alembic with the naming convention, one baseline migration · Ruff (`line-length = 100`), mypy,
basedpyright, import-linter, pre-commit (`conventional-pre-commit`, `gitleaks`) · pytest harness
(testcontainers Postgres, migrations once per session, per-test rollback, `AsyncClient` fixture,
blockbuster) · `justfile`.

**Watch for:** the baseline migration must `CREATE EXTENSION IF NOT EXISTS citext` and `pg_trgm`
(`users.email` is `citext`; §8 search needs `pg_trgm`). `uuidv7()` is core in Postgres 18 and needs
no extension — but the testcontainers image must be pinned to `postgres:18`, never `postgres:latest`,
or it will not exist.

**Done when:** `just dev` serves `/docs`, `just test` is green against real Postgres, `pre-commit run -a` is clean.

### Phase 2 — Data foundations
ERD in `docs/erd.md` (Mermaid) · base mixins (UUIDv7 PK, timestamps, soft delete) · identity +
restaurant + platform tables from §9 · real constraints (NOT NULL, CHECK, partial uniques,
deliberate FK `ON DELETE`) · `BaseRepository` + `db/pagination.py` (cursor encode/decode, `Page[T]`,
`paginate()`) + filter/sort whitelisting · **reference-data migrations** (cuisines) ·
`platform_settings` + a cached typed `SettingsService` · **`scripts/seed_demo.py`** (idempotent,
fixed random seed) and `scripts/create_superadmin.py` · **`docs/retention.md`** written now (§6
already fixes the numbers) so the Phase 13 jobs are implemented against a document rather than a
memory · a test asserting no pending autogenerate diff.

### Phase 3 — Auth core
Argon2id hashing · register, login, refresh, logout, logout-all, `GET /users/me` · access JWT +
opaque refresh tokens chained under a `sessions` row, replay detection, and the grace window · `tokens_valid_from`
revocation · cookie strategy + documented CSRF reasoning · string-permission dependencies, the
role→permission matrix, and the ownership helper · audit entries for every auth event.
**Done when:** the 401/403 matrix is a table-driven test and every row passes.

### Phase 4 — Auth extended, account management & GDPR
Email verification via Mailpit · password reset (single-use, hashed, expiring; revokes all sessions) ·
change password (re-auth) and change email (verify the new address first) · login throttling and
rate limiting behind the `RateLimiter` protocol · session list + per-device revoke · profile CRUD ·
addresses · self-deactivation · **GDPR export job** and the **30-day erasure flow** · `consent_log`.

### Phase 5 — Frontend foundation + auth integration
`frontend/.env.example` (`VITE_API_URL`) · Vite + React 19 + TS strict · ESLint/Prettier/Vitest/Playwright in pre-commit and the justfile ·
TanStack Router with public/authenticated/role-guarded layouts · Redux store + RTK Query with
**OpenAPI codegen** (`just gen-api`) · access token in memory, **single-flight** silent refresh on
401, protected routes, role-aware nav · pages: register, verify email, login, forgot/reset password,
profile, addresses, sessions · **shared UI kit built once here**: RHF+Zod field wrappers that consume
the field-keyed error payload, data table, pagination controls, toast, error boundary, loading and
empty states.
**Done when:** register → verify → login → refresh past expiry → logout works in a browser.

### Phase 6 — Restaurants: onboarding & management
Application = a `DRAFT`/`PENDING_APPROVAL` restaurant row + a review-metadata row; approval is a
**status transition, not a data copy** · admin review queue with approve/reject + reason ·
`restaurant_members` and ownership enforcement (IDOR-tested) · weekly hours including overnight spans
and split shifts · closures · pause with `accepting_orders_paused_until` · cuisines (m2m) ·
`is_pure_veg` · lat/long · delivery config (`supports_pickup`/`supports_delivery`,
`delivery_radius_km`, fee, minimum) · `is_open_now` in the restaurant's timezone, with tests around
midnight, overnight hours and DST · status lifecycle enforcement · FE: application form, owner
dashboard, admin review queue.

### Phase 7 — Menu & images
Categories (ordered, soft delete) · items with `food_type`, `price_minor`, availability, optional
stock tracking · **presigned direct-to-storage upload flow** (`uploads` table, complete + verify,
orphan reaper) · thumbnail worker job producing WebP variants · upload security (magic bytes, EXIF,
dimension caps) · bulk availability toggle · FE: owner menu editor, public menu view.

### Phase 8 — Discovery
The reusable query builder: whitelisted filters and sorts, stable tie-breaking, cursor pagination ·
Postgres full-text (`tsvector` + GIN) on restaurant and item names/descriptions, `pg_trgm` for fuzzy
matching · filters: cuisine, open-now, pure-veg, food type, price range, fulfilment type, distance
(haversine from lat/long — **no PostGIS**), admin-only status · `EXPLAIN ANALYZE` every listing query,
add the indexes it asks for, record before/after in an ADR · FE: search, filter panel, sort, paged
list, **all state in typed URL search params** (deep-linkable, back-button correct).

### Phase 9a — Cart & order placement
Server-side cart, one per (customer, restaurant) · checkout revalidation against current prices and
availability, surfacing changes via `CartInvalid` rather than silently repricing · **placement in one
transaction**: lock stock rows in deterministic ID order → validate restaurant status/open/accepting →
validate delivery area and minimum → snapshot prices and food types → create order + items +
adjustments → decrement stock → debit the wallet ledger → write commission snapshot → commit ·
`Idempotency-Key` with a stored-response table · `order_number` from a sequence ·
`order_delivery_details` snapshot · concurrency tests: two customers racing the last item, double
submit, cancel racing confirm.

### Phase 9b — Fulfilment, delivery & cancellation
The three status columns and the transition table (§9) · every transition writes
`order_status_history` **and emits an outbox event** · `deliveries` rows for delivery orders ·
cancellation policy as a single function · scheduled jobs: expire unaccepted orders, expire unpaid
orders · outbox publisher worker (first consumer: email) · FE: customer order history and detail,
owner incoming-orders board with status actions.

### Phase 10 — Wallet, Stripe, commission & refunds
`ledger_accounts` / `ledger_transactions` / `ledger_entries`, balanced by construction · top-up:
create a Stripe PaymentIntent → client confirms → **webhook** credits the wallet · webhook signature
verification, `processed_webhook_events` dedupe, idempotent handlers · order payment debits inside
the placement transaction; **commission is a two-step**: the *rate* is snapshotted onto the order at
placement, and the *split* into `RESTAURANT_PAYABLE` and `PLATFORM_REVENUE` is posted at completion
using that snapshotted rate. A refund after completion posts compensating entries that reverse the
customer debit **and** the split proportionally — the commission is never silently kept · refunds as **compensating entries**, never deletions ·
transaction history endpoint · invariant tests: whole ledger sums to zero; every account balance
equals the sum of its entries · FE: wallet page, top-up, statement.
**Stretch:** `payouts` (batch settlement to owners).

### Phase 11 — Jobs & reports
Generic `jobs` + `job_artifacts` with discriminated-union params/results · owner sales reports
(by day, item, status) and admin platform reports (GMV, take rate, net revenue, top restaurants,
new users) using `GROUP BY`, window functions and date bucketing · **write these queries in
SQLAlchemy Core / raw SQL** for the SQL depth · CSV + PDF (WeasyPrint) artifacts · FE polls job
status, then downloads via a presigned link · date-range and timezone correctness in aggregates.

### Phase 12 — Dockerisation
Multi-stage Dockerfiles (uv for the backend; frontend built and served by nginx/Caddy), non-root
users, healthchecks, `.dockerignore` · `compose.yaml`: api, worker, db, redis, mailpit, minio, web ·
`compose.override.yaml` for hot reload · swap the in-process implementations for the real ones —
**ARQ on Redis and the Redis rate limiter (in-memory limiting is simply wrong with >1 worker)** ·
migrations as an explicit step, never on app start · env/secrets strategy that maps onto a cloud
deploy · README verified from a clean clone, both natively and in Docker.

### Phase 13 — Hardening, retention & performance
Security pass: authz matrix retested, IDOR probes, mass-assignment check, security headers, CORS
tightened, upload validation, `pip-audit`/`pnpm audit`, secret scanning · **the PII-in-logs
redaction test** · retention jobs from §6 · N+1 hunt (`lazy="raise"` should have caught these
already), `EXPLAIN` sweep, index review, pool sizing · load test the ordering path (k6/Locust),
confirm no deadlocks under contention · Prometheus metrics, slow-query logging, a `/readyz` that
actually checks Postgres, Redis and MinIO · graceful shutdown, request timeouts, payload caps.

### Phase 14 — CI/CD & documentation
GitHub Actions: lint → typecheck → backend tests (service-container Postgres) → frontend tests →
Playwright → build images; plus a "no pending autogenerate diff" check. Coverage reported, red PRs
blocked · Renovate/Dependabot · README (architecture, both setup paths, env table, common tasks,
troubleshooting) · `docs/adr/`, `docs/erd.md`, `docs/retention.md`, `docs/runbook.md` · Google-style
docstrings on every public service and repository function, with ruff's pydocstyle rules on ·
a written deploy sketch.

---

## 8. Testing strategy

| Layer | What |
| --- | --- |
| **Unit** | Pure logic with faked repositories (`Protocol`) and a frozen `Clock`: transition rules, cancellation policy, open-now, pricing, permission matrix. No DB. |
| **Integration** | Repositories and services against a **real Postgres** (testcontainers): constraints, transactions, locking, cursor pagination, soft-delete filtering. |
| **API** | Full request→response through `httpx.AsyncClient`: happy paths plus the **401/403 matrix for every endpoint**. |
| **Concurrency** | Explicit races: last-item contention, double-submitted checkout, cancel vs confirm, double refresh. |
| **Invariant** | Ledger sums to zero · account balance equals sum of entries · order total equals sum of adjustments · stock never negative. |
| **E2E** | Playwright over `seed_demo.py` data: register→order→pay, owner accepts→completes, admin approves. |

Aim ~80% on `services/` and `repositories/`. Don't chase 100%.

---

## 9. Data model

Conventions: every table has `id UUID` (uuidv7, DB-generated) unless stated; `created_at`/`updated_at`
`TIMESTAMPTZ` via mixin; money columns are `BIGINT` minor units paired with `currency CHAR(3)`;
`deleted_at` means soft delete and is filtered at the repository base-query level.

### 9.1 Identity & access

**`users`**
`email` (citext, unique) · `password_hash` (nullable after anonymisation) · `full_name` · `phone` ·
`status` (`ACTIVE`/`INACTIVE`/`SUSPENDED`/`DELETED`) · `email_verified_at` · **`tokens_valid_from`** ·
`failed_login_count` · `locked_until` · `locale` · `marketing_consent` · `terms_accepted_at` ·
`deletion_requested_at` · `anonymized_at` · `avatar_upload_id`
*Intent:* the person. **Never hard-deleted** — orders and ledger entries depend on it.
*Lifecycle:* `deactivate` (self, reversible by logging in) · `suspend`/`unsuspend` (admin) ·
`DELETE /users/me` → 30-day grace → **anonymise in place**.

**`user_roles`** — `user_id` + `role` (composite PK), role ∈ `CUSTOMER`/`RESTAURANT_OWNER`/`ADMIN`/`SUPER_ADMIN`.
*Intent:* platform-wide roles; many-to-many so one person can be customer and owner. Grants/revokes are audited. The last `SUPER_ADMIN` cannot be removed.

**`user_addresses`** — `user_id` · `label` · `recipient_name` · `phone` · `line1`/`line2`/`city`/`state`/`postal_code`/`country` · `latitude` · `longitude` · `is_default`
*Intent:* the customer's saved addresses, used to prefill checkout.
*Lifecycle:* **hard delete** — safe only because orders snapshot into `order_delivery_details`.

**`sessions`** — `user_id` · `created_at` · `last_used_at` · `expires_at` (absolute lifetime, 30d) · `revoked_at` · `revoked_reason` · `ip` · `user_agent`
*Intent:* **one row per login**, i.e. per device. This is the thing the user sees in "your active sessions" and the thing `DELETE /auth/sessions/{id}` revokes. Revocation is a single row update, which is also what makes replay handling correct and cheap.
*Lifecycle:* revoked (soft) → hard-deleted by the retention job after 90 days.

**`refresh_tokens`** — `session_id` · `token_hash` (sha256, unique) · `parent_id` · `issued_at` · `expires_at` · `rotated_at` · `used_at`
*Intent:* one row per **issued** refresh token. Rotation appends a child to the chain within one session, so a 30-day session accumulates rows while the user still sees exactly one device.
⭐ The parent/child split exists because a rotating token produces a new row every 15 minutes: without it, "list my sessions" returns dozens of entries for one login and "revoke this session" revokes one link in a chain instead of the login. See ADR-0012.
*Rules:* at most one **unrotated** token per session (the 10s grace window returns the *same* child, so this holds). Presenting a token whose `rotated_at` is set, or one belonging to a revoked session, is a replay → revoke the session.
*Lifecycle:* hard-deleted 7 days after expiry, with their session.

**`email_verification_tokens`** — `user_id` · `token_hash` · `new_email` (set when verifying an email *change*) · `expires_at` · `used_at`
*Intent:* single-use, hashed, expiring proof of email ownership. Used for both signup verification and email change.

**`password_reset_tokens`** — `user_id` · `token_hash` · `expires_at` · `used_at` · `requested_ip`
*Intent:* single-use, hashed, expiring reset proof. Consuming it revokes all of the user's sessions.

**`consent_log`** — `user_id` · `consent_type` (`TERMS`/`MARKETING`/`PRIVACY`) · `granted` · `version` · `ip` · `user_agent` · `occurred_at`
*Intent:* append-only proof of what was consented to and when. Required to defend a marketing-email claim.

**`audit_log`** — `occurred_at` · `actor_user_id` (nullable) · `actor_type` (`USER`/`ADMIN`/`SYSTEM`/`WEBHOOK`) · `action` · `target_type` · `target_id` · `result` (`SUCCESS`/`FAILURE`) · `ip` · `user_agent` · `request_id` · `metadata` JSONB
*Intent:* who did what to whom, when, from where — for security- and money-relevant actions only. **Not** change-data-capture, **not** application logs, **not** `order_status_history`.
*Rules:*
- ⭐ **No direct or free-text PII.** Subject and target are **IDs**. Never an email address, name, phone number, postal address, card detail, token, or a free-text `metadata` field that could carry one. `metadata` holds enumerable facts (`{"from": "PENDING", "to": "ACCEPTED"}`), never user-supplied strings.
- **`ip` and `user_agent` are the deliberate exception** and are stored in full. They are personal data, they are also the two fields that make a security log usable in an investigation, and every real audit log keeps them. Basis: legitimate interest (security and fraud prevention); retention: 2 years, same as the row; **excluded from erasure** under GDPR Art. 17(3)(b)/(e). Recorded as ADR-0011.
- Written in the **same transaction** as the action · append-only, no delete API, trimmed after 2 years.

### 9.2 Restaurants

**`restaurants`**
`slug` (unique) · `name` · `description` · **`status`** · `is_pure_veg` · `phone` · `email` ·
address fields · `latitude` · `longitude` · **`timezone`** (IANA) · `supports_pickup` ·
`supports_delivery` · `delivery_radius_km` · `delivery_fee_minor` · `delivery_min_order_minor` ·
`avg_prep_minutes` · **`is_accepting_orders`** · **`accepting_orders_paused_until`** ·
`logo_upload_id` · `cover_upload_id` · `archived_at`
*Intent:* the sellable entity. Also serves as the onboarding application while in `DRAFT`/`PENDING_APPROVAL`.
*Status:* `DRAFT` (owner drafting) → `PENDING_APPROVAL` → `REJECTED` (fixable, resubmittable) / `ACTIVE` → `INACTIVE` (owner-deactivated, reversible) / `SUSPENDED` (admin-deactivated, admin-only lift) → `ARCHIVED` (terminal soft delete, admin-only restore).
*`is_accepting_orders`:* an orthogonal **momentary pause** — still listed and visible, checkout blocked. `accepting_orders_paused_until` lets it auto-expire.
*Lifecycle:* never hard-deleted (orders reference it).

**`restaurant_members`** — `restaurant_id` · `user_id` · `role` (`OWNER`/`MANAGER`/`STAFF`) · unique `(restaurant_id, user_id)` · **partial unique on `(restaurant_id) WHERE role = 'OWNER'`**
*Intent:* who may act on a restaurant. One owner → many restaurants, and (later) many staff → one restaurant, without a retrofit. **Every restaurant-scoped authorization check goes through this table.**

**`restaurant_applications`** — `restaurant_id` · `submitted_at` · `submitted_by` · `submitted_snapshot` JSONB · `decision` (`PENDING`/`APPROVED`/`REJECTED`) · `reviewed_by` · `reviewed_at` · `rejection_reason` · `reviewer_notes`
*Intent:* **review metadata only** — the restaurant row holds the actual data, so approval is a status transition rather than a data copy. One row per submission gives a resubmission history. `submitted_snapshot` is the immutable record of exactly what the admin approved.
*Lifecycle:* never deleted.

**`restaurant_hours`** — `restaurant_id` · `day_of_week` (0–6) · `opens_at` (time) · `closes_at` (time)
*Intent:* the weekly schedule. Multiple rows per day allow split shifts (lunch/dinner). `closes_at < opens_at` means the span crosses midnight.

**`restaurant_closures`** — `restaurant_id` · `starts_at` · `ends_at` · `reason` · `created_by`
*Intent:* **planned, forward-looking** closures (holidays, renovation) layered on top of the weekly hours. **Not an audit table** — `is_accepting_orders` changes go to `audit_log`.

**`cuisines`** — `name` · `slug` (unique) · `is_active`
*Intent:* a curated, admin-managed vocabulary so cuisine filtering actually works. Seeded via a **reference-data migration**.

**`restaurant_cuisines`** — `restaurant_id` + `cuisine_id` (composite PK). *Intent:* multi-cuisine support.

### 9.3 Menu

**`menu_categories`** — `restaurant_id` · `name` · `description` · `position` · `deleted_at`
*Intent:* menu sections, explicitly ordered.
*Lifecycle:* **soft delete** — its items fall back to "Uncategorised".

**`menu_items`** — `restaurant_id` · `category_id` (nullable) · `name` · `description` ·
**`food_type`** (`VEG`/`NON_VEG`/`EGG`/`VEGAN`, native enum, required) · `price_minor` · `currency` ·
`is_available` · `is_stock_tracked` · `stock_quantity` · `image_upload_id` · `position` · `deleted_at`
· `CHECK (NOT is_stock_tracked OR stock_quantity >= 0)`
*Intent:* the sellable item. `is_available` is a transient "sold out" toggle (still visible);
`deleted_at` is permanent removal from the menu.
*Lifecycle:* **never hard-deleted** — past orders and reports must still resolve it.

### 9.4 Uploads

**`uploads`** — `uploaded_by` · `purpose` (`MENU_ITEM_IMAGE`/`RESTAURANT_LOGO`/`RESTAURANT_COVER`/`AVATAR`) · `status` (`PENDING`/`READY`/`FAILED`/`DELETED`) · `storage_key` · `content_type` · `size_bytes` · `checksum` · `width` · `height` · `variants` JSONB · `completed_at` · `deleted_at`
*Intent:* the record that makes **presigned direct-to-storage uploads** safe — orphan reaping, quota enforcement, thumbnail keys, and reference counting. Bytes never pass through the API.
*Flow:* `POST /uploads` (validate, create PENDING, return presigned PUT) → client PUTs to storage → `POST /uploads/{id}/complete` (HEAD-verify, mark READY, enqueue thumbnails) → link to the entity.
*Lifecycle:* PENDING rows older than 1h are reaped with their objects; soft delete then async object deletion once nothing references it.

### 9.5 Cart

**`carts`** — `customer_id` · `restaurant_id` · `fulfillment_type` · unique `(customer_id, restaurant_id)`
*Intent:* server-side cart, **one per restaurant**; several may be open at once. No multi-restaurant checkout.
*Lifecycle:* **hard delete** — cleared on checkout, no historical value.

**`cart_items`** — `cart_id` · `menu_item_id` · `quantity` · `notes` · unique `(cart_id, menu_item_id)`
*Intent:* a reference to the live menu item — **prices are read live and only snapshotted at placement**, so the cart always reflects current reality. Revalidation at checkout raises `CartInvalid` rather than silently repricing.
*Lifecycle:* hard delete.

### 9.6 Orders

**`orders`**
`order_number` (unique, from a sequence — `ORD-2026-000123`) · `customer_id` · `restaurant_id` ·
`order_group_id` (nullable, no FK — reserved so multi-restaurant checkout stays additive) ·
`fulfillment_type` (`PICKUP`/`DELIVERY`) · **`payment_status`** (`PENDING`/`PAID`/`FAILED`) ·
**`fulfillment_status`** · **`refund_status`** (`NONE`/`PARTIAL`/`FULL`) · `currency` ·
`subtotal_minor` · `total_minor` · **`commission_rate_snapshot`** · `commission_amount_minor` ·
`placed_at` · `accepted_at` · `ready_at` · `completed_at` · `cancelled_at` · `cancelled_by` ·
`cancellation_reason` · `idempotency_key` · `customer_notes`
*Intent:* the financial and fulfilment record. **Never deleted, never scrubbed** — it holds no PII by design.
*Why three status columns:* one enum cannot express "delivered **and** partially refunded". Payment, fulfilment and refund are three independent state machines.
*Why the rate snapshot:* reading the live commission rate at reporting time would silently rewrite history.
*Why `idempotency_key` is here **and** in `idempotency_keys`:* deliberate belt and braces. `idempotency_keys` is the general middleware that replays the original response; the column carries a **partial unique index** (`WHERE idempotency_key IS NOT NULL`) so that a duplicate order is impossible at the database level even if that middleware is bypassed, misconfigured, or raced. The table is the feature; the column is the backstop.
*`order_number`:* one **global, non-resetting** Postgres sequence. The year in `ORD-2026-000123` comes from `placed_at` and is cosmetic — the counter does **not** restart in January. Per-year numbering would need a sequence created by a job and has a race at midnight on 1 January, for no benefit.

`fulfillment_status` transitions:
```
PENDING_PAYMENT → PLACED → ACCEPTED → PREPARING → READY ─┬→ COMPLETED          (pickup)
                                                         └→ OUT_FOR_DELIVERY → DELIVERED
terminal: REJECTED · CANCELLED_BY_CUSTOMER · CANCELLED_BY_RESTAURANT · EXPIRED
```
Allowed transitions are declared as data and enforced in the service. `REJECTED` (restaurant declined)
and `EXPIRED` (never accepted in time, job-driven) both auto-refund. Cancellation policy is a single
function: customer free until `ACCEPTED`, with a fee until `PREPARING`, not after; restaurant may
reject until `PREPARING`.

**`order_items`** — `order_id` · `menu_item_id` · **`name_snapshot`** · **`food_type_snapshot`** · **`unit_price_minor`** · `quantity` · `line_total_minor` · `notes`
*Intent:* an immutable snapshot. A later menu price change must never alter a past order.

**`order_adjustments`** — `order_id` · `type` (`ITEM_SUBTOTAL`/`DELIVERY_FEE`/`SERVICE_FEE`/`TAX`/`TIP`/`DISCOUNT`) · `label` · `amount_minor` (signed; discounts negative) · `sort_order`
*Intent:* the order total as a **list of lines** rather than fixed columns, so a coupon later becomes one more row instead of a pricing rewrite. These sum exactly to `total_minor`.
*Note:* platform commission is **not** an adjustment — it is a settlement split, held in `commission_amount_minor` and the ledger, not something the customer pays.

**`order_delivery_details`** — `order_id` (PK, 1:1) · `recipient_name` · `recipient_phone` · address fields · `latitude` · `longitude` · `delivery_instructions` · **`scrubbed_at`**
*Intent:* ⭐ the per-order address snapshot, **deliberately separated from `orders`** so GDPR erasure can scrub the PII while the financial record survives intact. This split is a hard requirement.

**`deliveries`** — `order_id` (unique, 1:1) · `status` (`PENDING`/`OUT_FOR_DELIVERY`/`DELIVERED`/`FAILED`) · `dispatched_at` · `delivered_at` · `failure_reason` · `distance_km_snapshot` · `notes`
*Intent:* delivery state as **its own table** rather than columns on `orders`, so couriers, dispatch and live tracking can be added beside it later. Today the restaurant drives the transitions.

**`order_status_history`** — `order_id` · `from_status` · `to_status` · `actor_user_id` · `actor_type` · `reason` · `occurred_at`
*Intent:* user-visible domain history ("Accepted 14:02, Ready 14:20"). Distinct from `audit_log`.

**`idempotency_keys`** — `key` · `user_id` · `endpoint` · `request_hash` · `response_status` · `response_body` JSONB · `expires_at` · unique `(user_id, endpoint, key)`
*Intent:* a retried `POST /orders` or top-up returns the **original** response, never a duplicate order or double charge. `request_hash` catches a reused key with a different body.

**`outbox_events`** — `aggregate_type` · `aggregate_id` · `event_type` · `payload` JSONB · `occurred_at` · `published_at` · `attempts`
*Intent:* written **in the same transaction** as the state change, so a committed order can never lose its email. First consumer is email; a live SSE feed later is just another subscriber.

### 9.7 Money (double-entry ledger)

**`ledger_accounts`** — `account_type` (`USER_WALLET`/`RESTAURANT_PAYABLE`/`PLATFORM_REVENUE`/`PLATFORM_CASH`/`STRIPE_CLEARING`) · `owner_type` · `owner_id` (nullable for platform accounts) · `currency` · unique `(account_type, owner_id, currency)`
*Intent:* **a table, not an enum**, so loyalty points and other account types are additive.
⭐ **There is no `wallets` table** — a user's wallet *is* their `USER_WALLET` account. One source of truth.

**`ledger_transactions`** — `kind` (`TOPUP`/`ORDER_PAYMENT`/`ORDER_REFUND`/`COMMISSION`/`PAYOUT`/`ADJUSTMENT`) · `reference_type` · `reference_id` · `occurred_at` · `created_by` · `memo` · `idempotency_key`
*Intent:* one balanced movement of money. Can reference multiple orders, which is what keeps group checkout additive.

**`ledger_entries`** — `transaction_id` · `account_id` · `direction` (`DEBIT`/`CREDIT`) · `amount_minor` (> 0) · `currency`
*Intent:* the atoms. ⭐ **Immutable — never updated, never deleted.** Corrections are compensating entries.
*Invariants:* debits equal credits within every transaction · the whole ledger sums to zero · all entries in a transaction share a currency (CHECK) · an account balance is **derived** (`SUM(credits) − SUM(debits)`), never stored.
*Note:* if balance reads get slow, add a snapshot table plus a reconciliation job — never a mutable column.

**`payment_intents`** — `user_id` · `stripe_payment_intent_id` (unique) · `amount_minor` · `currency` · `status` · `succeeded_at` · `failure_reason` · `ledger_transaction_id` (nullable)
*Intent:* tracks a Stripe top-up. ⭐ The wallet is credited **only from a verified webhook**, never from the client's success callback.

**`processed_webhook_events`** — `stripe_event_id` (PK) · `event_type` · `received_at` · `processed_at` · `result`
*Intent:* dedupe. Stripe retries and replays; handlers must be idempotent and this is what makes them so.

**`refunds`** — `order_id` (nullable) · `payment_intent_id` (nullable, **FK to `payment_intents.id`** — the local row, never the Stripe id; the Stripe identifier lives only on `payment_intents.stripe_payment_intent_id`) · `amount_minor` · `reason` · `status` · `stripe_refund_id` · `requested_by` · `ledger_transaction_id`
*Intent:* order refunds go to the wallet; top-up refunds go back through Stripe. Both write compensating ledger entries.

**`payouts`** *(Phase 10 stretch)* — `restaurant_id` · `period_start` · `period_end` · `amount_minor` · `status` · `ledger_transaction_id` · `paid_at`
*Intent:* batch settlement of `RESTAURANT_PAYABLE` to owners.

### 9.8 Platform

**`platform_settings`** — `key` (PK) · `value` JSONB · `description` · `updated_by` · `updated_at`
*Intent:* things that change **without a deploy** — commission rate, service fee, min order value, max items per order, wallet top-up cap, order acceptance timeout, feature flags. Read through a cached `SettingsService` with a **typed accessor per key** (`commission_rate() -> Decimal`), never a stringly-typed lookup. Changes are audited.
*The dividing line:* runtime-changeable → this table; environment-specific (DB URL, Stripe keys, S3 endpoint) → env vars. Never mix.

**`jobs`** — `type` · `status` (`QUEUED`/`RUNNING`/`SUCCEEDED`/`FAILED`/`EXPIRED`) · `requested_by` · `params` JSONB · `result` JSONB · `error` JSONB · `progress` · `attempts` · `started_at` · `finished_at` · `expires_at`
*Intent:* **user-visible** job state so the frontend can poll and yesterday's report is still findable. Generic, not report-specific — types include sales report, platform report, **GDPR export**, bulk import. ARQ's Redis queue tracks *execution*; this tracks *state*. Both exist.
*JSONB is correct here* — `params`/`result` are genuinely polymorphic. Type them at the edges with a **discriminated Pydantic union** on `type`.

**`job_artifacts`** — `job_id` · `storage_key` · `filename` · `content_type` · `size_bytes` · `expires_at`
*Intent:* separate because one job produces several files (CSV *and* PDF) that expire independently. Downloaded via short-TTL presigned links.
*Lifecycle:* hard-deleted with their objects after `expires_at` (7 days).

---

## 10. Lifecycle & deletion API matrix

**Governing rule:** anything touching **money, legal, or audit** is immutable or soft-deleted;
**transient working state** is hard-deleted; **PII** is a third category, handled by anonymisation.

| Entity | API | Effect |
| --- | --- | --- |
| User | `POST /users/me/deactivate` | `INACTIVE`, all sessions revoked; logging in reactivates. |
| | `DELETE /users/me` | 30-day grace, then **anonymise in place**. Never a row delete. |
| | `POST /admin/users/{id}/suspend` \| `/unsuspend` | Admin ban/unban; sessions revoked, login refused with a reason. |
| Restaurant | `POST /restaurants/{id}/deactivate` \| `/activate` | Owner delists/relists; fully reversible. |
| | `POST /admin/restaurants/{id}/suspend` \| `/unsuspend` | Admin delists punitively; owner cannot self-lift. |
| | `POST /restaurants/{id}/archive` | Terminal soft delete; delisted everywhere, orders preserved, admin-only restore. |
| | `POST /restaurants/{id}/pause` \| `/resume` | Flips `is_accepting_orders`; still listed, checkout blocked. |
| Menu category | `DELETE /menu-categories/{id}` | Soft delete; items fall back to "Uncategorised". |
| Menu item | `PATCH /menu-items/{id}/availability` | Transient "sold out"; still visible. |
| | `DELETE /menu-items/{id}` | Soft delete; off the menu, still resolvable by past orders and reports. |
| Cart / cart item | `DELETE` | **Hard delete.** |
| Address | `DELETE /users/me/addresses/{id}` | **Hard delete** — safe only because of `order_delivery_details`. |
| Order | — | **Never deleted.** Cancel / reject / refund are state transitions. |
| Ledger entry | — | **Never deleted or updated.** Compensating entries only. |
| Session (a login/device) | `DELETE /auth/sessions/{id}` | `sessions.revoked_at` set — one row update, which invalidates every refresh token in its chain. Hard-deleted by retention after 90 days. |
| | `POST /auth/logout-all` | Revokes every session **and** bumps `tokens_valid_from`. |
| Refresh token | — | Marked `rotated_at`/`used_at`; hard-deleted 7 days after expiry, with its session. |
| Verification / reset tokens | — | Marked `used_at`; hard-deleted 7 days after expiry. |
| Upload | `DELETE /uploads/{id}` | Soft delete, then async object deletion once unreferenced. |
| Audit log | — | **No delete API.** Trimmed after 2 years by retention. |
| Job + artifacts | — | Hard-deleted with their files after `expires_at`. |
| Restaurant application | — | Never deleted; resubmission creates a new row. |

---

## 11. Deferred — and the measures that keep them additive

Not being built. The measures in bold are already in §9 and cost nothing now.

| Deferred | Measure taken |
| --- | --- |
| Coupons / promotions | ⭐ **`order_adjustments`** rows instead of fixed total columns. |
| Editable RBAC | ⭐ **String permissions** (`require_permission("restaurant:approve")`), never `require_role`. |
| Geospatial / "near me" | ⭐ **`latitude`/`longitude` stored now.** The pain is backfilling by geocoding, not the query. PostGIS is one later migration. |
| Couriers, dispatch, live tracking | ⭐ **`deliveries` as its own table**, not columns on `orders`. |
| Loyalty points | ⭐ **`ledger_accounts` as a table** with an `account_type`. |
| Live order feed (SSE) | **Domain events in `outbox_events`** on every status change — a feed is just another subscriber. Use SSE, not websockets. |
| Multi-restaurant checkout | **`orders.order_group_id`** column reserved; a ledger transaction may already span multiple orders. |
| Multi-currency | `currency` column everywhere, `_minor` naming, never assume 2 decimals. |
| Reviews & ratings | None needed — purely additive. |
| Mobile app | `/auth/refresh` accepts the token in the **body** as well as the cookie. |
| i18n | Machine-readable error **`code`s** rendered on the frontend; `users.locale`. |
| Restaurant change re-approval | Would be a `restaurant_change_requests` table. Not designed for. |
| Multi-tenant / white-label | **No measure.** Deliberately not designed for. |

---

## 12. Invariants (each becomes a test)

1. Every `ledger_transaction`'s debits equal its credits; the whole ledger sums to zero.
2. An account's balance equals the sum of its entries — there is no stored balance to drift.
3. `orders.total_minor` equals the sum of its `order_adjustments`.
4. `orders.subtotal_minor` equals the sum of its `order_items.line_total_minor`.
5. Stock never goes negative — enforced by a DB `CHECK` **and** `SELECT … FOR UPDATE` in the placement transaction, locking rows in deterministic ID order to avoid deadlocks.
6. An order only moves along declared transitions; every move writes `order_status_history` and an outbox event.
7. Orders can only be placed at a restaurant that is `ACTIVE`, `is_accepting_orders`, and open in **its own timezone** at that moment.
8. A delivery order is only placed within `delivery_radius_km` and above `delivery_min_order_minor`.
9. Exactly one **unrotated** refresh token per session; reusing a rotated token, or using any token of a revoked session, revokes the whole session. (The ~10s grace window returns the *same* child token, so it does not break this.)
10. An access token issued before `users.tokens_valid_from` is rejected.
11. Retrying `POST /orders` with the same `Idempotency-Key` returns the original order, never a second.
12. A wallet is only credited by a Stripe webhook that passed signature verification and was not already in `processed_webhook_events`.
13. Exactly one `OWNER` per restaurant (partial unique index).
14. The last `SUPER_ADMIN` cannot be demoted or deleted; no admin can escalate their own privileges.
15. `audit_log` rows contain no direct or free-text PII: no value matching an email, phone or postal-address shape, and no value equal to any of the actor's PII columns. `ip` and `user_agent` are the declared exception (§9.1).
16. An anonymised user's orders and ledger entries remain intact and correct.
17. Soft-deleted rows never appear in any listing endpoint.
