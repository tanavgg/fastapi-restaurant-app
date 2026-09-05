# Restaurant Ordering Platform

A restaurant ordering platform. Customers browse restaurants and menus, build a cart, and pay
from an in-app wallet topped up via Stripe (test mode). Orders are **pickup or delivery**.
Restaurant owners apply to onboard, get approved by an admin, then manage hours, availability,
menu, stock and incoming orders. Admins oversee the platform and pull reports. The platform
takes a commission.

Not a commercial product — but architecture, data integrity, transactions and security are
treated as if it were.

> **Status: Phase 0 complete — decisions and repo hygiene.** No application code exists yet.
> The roadmap is §7 of [`docs/plan.md`](docs/plan.md); see [Project status](#project-status).

## Contents

- [Stack](#stack)
- [Architecture](#architecture)
- [Repository layout](#repository-layout)
- [Getting started](#getting-started)
- [Commands](#commands)
- [Testing](#testing)
- [Documentation](#documentation)
- [Project status](#project-status)
- [Licence](#licence)

## Stack

**Backend** — Python 3.13+ · [uv](https://docs.astral.sh/uv/) · FastAPI · Pydantic v2 ·
SQLAlchemy 2.0 async + asyncpg · Alembic · Postgres 18 · Redis · ARQ · MinIO (S3 API) · Mailpit ·
Stripe test mode · Argon2id (`pwdlib`) · structlog · Ruff · mypy · import-linter ·
pytest + testcontainers

**Frontend** — Vite · React 19 · TypeScript (strict) · TanStack Router · Redux Toolkit +
RTK Query with OpenAPI codegen · React Hook Form + Zod · Tailwind v4 + shadcn/ui ·
Vitest + MSW · Playwright

Why each of the contentious ones was chosen: [`docs/adr/`](docs/adr/).

## Architecture

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

The layering rules are enforced in CI by **import-linter**, not by good intentions. The full set
lives in [`CLAUDE.md`](CLAUDE.md).

Load-bearing choices:

- The wallet is a **double-entry ledger** with no stored balance ([ADR-0004](docs/adr/0004-double-entry-ledger.md)).
- Money is `BIGINT` **minor units**, every column suffixed `_minor`. Never floats.
- Order state is **three independent columns** — `payment_status`, `fulfillment_status`,
  `refund_status` — because one enum cannot express "delivered and partially refunded".
- Orders snapshot name, price and food type; a menu change never alters a past order.
- Auth is self-built: 15-minute JWT access tokens plus opaque rotating refresh tokens with
  family-based replay detection ([ADR-0003](docs/adr/0003-self-built-authentication.md)).

## Repository layout

```
backend/     python app + tests (own pyproject.toml)      [Phase 1]
frontend/    vite app (own package.json)                  [Phase 5]
docker/      Dockerfiles + compose                        [Phase 12]
docs/        plan.md · adr/ · conventions.md · erd.md · retention.md · runbook.md
justfile     task runner — the entry point for everything
```

## Getting started

> Filled in during **Phase 1**. The project runs natively first (no Docker); containers arrive in
> Phase 12 and both paths are verified from a clean clone in Phase 14.

Prerequisites (native path): **Python 3.13**, [uv](https://docs.astral.sh/uv/),
[just](https://github.com/casey/just), Postgres 18, Redis, MinIO, Mailpit, Node 24 + pnpm,
Stripe CLI.

```bash
git clone <repo> && cd fastapi-restaurant-app
cp backend/.env.example backend/.env    # then fill it in — the app fails fast on missing config
just setup                              # install dependencies
just migrate                            # apply migrations
just seed                               # idempotent demo data
just dev                                # API on :8000, /docs for OpenAPI
```

## Commands

Everything goes through `just` at the repository root, and every recipe works from any
directory.

| Command | Does |
| --- | --- |
| `just dev` | Run the API (and, from Phase 5, the frontend) |
| `just test` | The whole suite — real Postgres via testcontainers, never SQLite |
| `just lint` | Ruff, mypy, import-linter, ESLint |
| `just migrate` | Apply Alembic migrations |
| `just seed` | Idempotent demo data (`scripts/seed_demo.py`) |
| `just gen-api` | Regenerate the frontend's typed API client from the backend's OpenAPI schema |

The backend's OpenAPI schema is the **single source of truth** for frontend types. Run
`just gen-api` after changing any request or response schema. Both `openapi.json` and the generated
client are **committed**, and CI asserts that regenerating them produces no diff.

## Testing

| Layer | What |
| --- | --- |
| Unit | Pure logic with faked repositories and a frozen `Clock`. No DB. |
| Integration | Repositories and services against **real Postgres** (testcontainers). |
| API | Full request→response through `httpx.AsyncClient`, including a 401/403 matrix per endpoint. |
| Concurrency | Explicit races: last-item contention, double-submitted checkout, cancel vs confirm. |
| Invariant | Ledger sums to zero · balances equal the sum of entries · stock never negative. |
| E2E | Playwright over seeded data. |

Target ~80% on `services/` and `repositories/`. **Never test against SQLite.**

## Documentation

| Document | Contents |
| --- | --- |
| [`docs/plan.md`](docs/plan.md) | **The source of truth** — tech, phases, every table with its intent, lifecycle APIs, invariants |
| [`CLAUDE.md`](CLAUDE.md) | The rules that apply to every change — layering, conventions, security, data integrity |
| [`docs/adr/`](docs/adr/) | Architecture decisions and the reasoning behind them |
| [`docs/conventions.md`](docs/conventions.md) | Branch, commit, PR and phase-completion conventions |
| `docs/erd.md` | Entity-relationship diagram *(Phase 2)* |
| `docs/retention.md` | Data retention policy and the jobs that enforce it *(Phase 13)* |
| `docs/runbook.md` | Operational runbook *(Phase 14)* |

## Project status

| Phase | | |
| --- | --- | --- |
| 0 | Decisions & repo hygiene | ✅ |
| 1 | Backend base setup (no Docker) | **next** |
| 2 | Data foundations | |
| 3 | Auth core | |
| 4 | Auth extended, account management & GDPR | |
| 5 | Frontend foundation + auth integration | |
| 6 | Restaurants: onboarding & management | |
| 7 | Menu & images | |
| 8 | Discovery | |
| 9a | Cart & order placement | |
| 9b | Fulfilment, delivery & cancellation | |
| 10 | Wallet, Stripe, commission & refunds | |
| 11 | Jobs & reports | |
| 12 | Dockerisation | |
| 13 | Hardening, retention & performance | |
| 14 | CI/CD & documentation | |

## Licence

[MIT](LICENSE) © 2026 Tanav Gupta
