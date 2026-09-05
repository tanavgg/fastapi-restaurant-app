# ADR-0010 — MinIO (S3 API) from Phase 1; no local-filesystem storage backend, ever

- **Status:** Accepted
- **Date:** 2026-09-05
- **Deciders:** Tanav Gupta
- **Supersedes:** the earlier plan to run a local-filesystem `ObjectStorage` implementation until Docker arrives in Phase 12

## Context

Images (menu items, restaurant logos and covers, avatars), report artifacts and GDPR export ZIPs all
need object storage. Docker does not arrive until Phase 12, and the original plan was to write a
filesystem-backed `ObjectStorage` implementation for Phases 1–11 and swap it for S3/MinIO later —
the interface exists precisely to make that swap cheap.

That reasoning is sound about the *interface* and wrong about the *feature*. The upload design is
not "put bytes somewhere and get them back". It is:

1. `POST /uploads` returns a **presigned PUT URL**; the client uploads directly to storage.
2. `POST /uploads/{id}/complete` does a **HEAD** against the object to verify size, content type and
   existence before marking it `READY`.
3. Downloads of reports and exports use **short-TTL presigned GET URLs**.
4. Bytes never pass through the API at all.

Presigned URLs are not an S3 implementation detail that a filesystem backend can approximate — they
are the entire security model. A filesystem implementation cannot produce them, so it would have to
be a *different flow* (bytes through the API, a route serving files, a different upload endpoint on
the frontend). That is not one interface with two implementations; it is two features, one of which
gets deleted in Phase 12 along with its tests, and the real one gets written late and under time
pressure at the point where the CORS, content-type and signature details all bite at once.

## Decision

**MinIO runs natively from Phase 1 and is the only `ObjectStorage` implementation that will ever
exist**, other than a `FakeObjectStorage` used in tests.

- Native install for Phases 1–11 (a single binary; `minio server ./.data/minio`), containerised in
  Phase 12. The *deployment* changes; the code does not.
- Access via `aioboto3` against the S3 API, so moving to real S3, R2 or Backblaze later is an
  endpoint and a credential.
- `FakeObjectStorage` (in-memory, records calls, returns fake presigned URLs) is the test double.
  It exists because tests should not hit the network — **not** as an alternative deployment.
- The `ObjectStorage` protocol is still worth having: it is what makes the fake possible and keeps
  `aioboto3` out of the service layer. Its purpose is testability and boundary hygiene, not
  swappability.

## Consequences

- One more thing to install before `just dev` works. The README must say so, and `/readyz` must
  check MinIO (Phase 13).
- The presigned-upload flow, its CORS configuration and its magic-byte/EXIF/dimension validation are
  exercised from Phase 7 onward instead of being written blind in Phase 12.
- No throwaway code, no throwaway tests, and no phase where the frontend upload path has to be
  rewritten.
- Local development stores real objects on disk under MinIO's data directory — git-ignored, and
  disposable.
- If MinIO's native binary ever becomes inconvenient to install, the answer is Docker for that one
  service, not a filesystem backend.

## Alternatives considered

| Alternative | Why not |
| --- | --- |
| **Filesystem backend until Phase 12** | Cannot produce presigned URLs, so it forces a different upload flow, different frontend code and different tests — all of which are deleted later. Defers the genuinely fiddly part (CORS, signatures, content types) to the busiest phase. |
| **Real AWS S3 from the start** | Real credentials, real cost, network dependency in tests and in CI, and a shared bucket for a learning project. MinIO is the same API with none of that. |
| **Store images as `BYTEA` in Postgres** | Bloats the database and its backups, defeats CDN caching, and makes streaming and presigning impossible. |
| **Defer images entirely to a later phase** | Images are core to a restaurant product; a menu without them is not the app being built. |

## References

- `docs/plan.md` §2, §7 (Phase 1, Phase 7), §9.4
