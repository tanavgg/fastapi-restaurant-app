# ADR-0005 — No separate domain entities and no ORM↔domain mapper layer

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta

## Context

The purist reading of clean/hexagonal architecture keeps a set of framework-free domain entities
in the centre, with mappers translating ORM rows into entities on the way in and back out on the
way out. The stated benefits are testability and independence from the persistence library.

This project already commits to a strict layering (`Router → Schema → Service → Repository →
Model`) with rules that services never see HTTP and repositories never hold business logic. The
question is whether to add a *sixth* representation of every concept: request schema, response
schema, ORM model, domain entity, and two mappers between them.

The specific fear driving the alternative is real: services that pass `dict[str, Any]` around,
and ORM models leaking into HTTP responses. But the mapper layer is not the only way to prevent
that, and it is by some distance the most expensive.

## Decision

**Services operate on SQLAlchemy ORM models directly. There are no separate domain entities and
no mapping layer.**

Four rules make this safe, and matter more than the layer would have:

1. **Nothing in a service signature is `dict[str, Any]` or `Any`.** Enforced by mypy strict. If
   a dict is being built to move data between layers, a type is missing.
2. **Input:** the router passes the validated Pydantic request model straight into the service,
   or a small dedicated command object when the service needs more than the request carries
   (`actor_id`, `idempotency_key`). Never `**kwargs`, never a dict.
3. **Output:** returning ORM models from services is fine. The **router** converts, via
   `ResponseModel.model_validate(obj)` with `from_attributes=True`. Services never construct
   response schemas — that would couple business rules to HTTP shape.
4. **When an ORM model is not enough** — aggregates and computed results such as
   "restaurant + distance + open_now + item_count" — the repository returns a **frozen dataclass
   or Pydantic model**, mapped at the repository boundary. Never a raw SQLAlchemy `Row`, never a
   dict, upward. This is the precise point at which codebases slide into dicts; the typed result
   object means the slide never starts.

The async-specific risk of passing ORM models around — a lazy load firing after the session
closes — is neutralised by ADR-0002: `lazy="raise"` + explicit `selectinload()` +
`expire_on_commit=False`.

## Consequences

- Roughly half the code of the mapped alternative, and no class of bug that lives in mapping
  functions.
- ORM models carry persistence concerns into the service layer. Accepted: SQLAlchemy is not
  being swapped, and the repository is already the seam that would matter if it were.
- Business logic is tested against real model instances (or fakes conforming to repository
  `Protocol`s) rather than against pure entities. Unit tests still need no database.
- Rule 4 must be applied consistently. The first time a `Row` or a dict is returned upward from
  a repository, this ADR has been broken and should be repaired rather than tolerated.
- If the ORM ever *did* need replacing, the work is real — but it is bounded by the repository
  layer, and it is work that was hypothetical the whole time.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **Domain entities + ORM models + mappers** | Roughly doubles the code for a database swap that will not happen. Mapping layers are where bugs hide, and every new field must be added in five places. |
| **SQLAlchemy imperative mapping** (map ORM onto plain domain classes) | Genuinely elegant, and avoids the mapper functions — but it is a niche configuration style with thin documentation, and it fights `Mapped[]` typing, which is one of the main reasons SQLAlchemy 2.0 was chosen. |
| **Pydantic models as the domain layer** | Re-introduces the SQLModel problem: validation, persistence and transport concerns fused into one class. |
| **Return dicts from repositories** | The failure mode this ADR exists to prevent. |

## References

- `docs/plan.md` §4 (architecture rules 3, 11)
- ADR-0002 (async SQLAlchemy)
