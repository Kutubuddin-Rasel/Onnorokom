# Implementation Plan

Work in one phase at a time. A phase is not complete because code was generated; its exit checks must pass.

| Phase | Scope | Exit criteria |
|---|---|---|
| 0. Specification | Read requirements; confirm assumptions, API contract, architecture and repo state. | Open decisions are recorded; no invented requirements. |
| 1. Foundation | .NET 10 solution, React/Vite/TS, PostgreSQL, environment config, root Compose skeleton. | Services build/start; backend connects to PostgreSQL. |
| 2. EF Core model | Two core entities, enum, DbContext, Fluent configs, constraints/indexes, initial migration. | Migration applies to PostgreSQL; schema and constraints verified. |
| 3. Class APIs | Create/list/get class; calculate seats from holds. | Endpoints match contract and return correct seat counts. |
| 4. Reservation engine | Create hold, get hold, confirm, cancel, list student holds; transactional class-row locking. | Sequential flows and business invariants pass integration tests. |
| 5. Expiration | Configured background worker; clock abstraction; race-safe expiration and seat release. | Holds expire without API calls; expired holds cannot be confirmed. |
| 6. Backend quality | Integration and PostgreSQL concurrency tests; error handling. | Minimum three tests including concurrency pass; invariants hold. |
| 7. Frontend | Student context, class list, admin create class, apply, hold detail/countdown, payment simulation, cancel, my applications, polling. | All workflows operate through UI and survive refresh. |
| 8. Delivery | Compose all services, persistent volume, README, assumptions, AI usage, final regression. | Fresh clone runs with `docker compose up`; restart preserves data; docs and Git history are ready. |

## Working method for each phase
1. Read the relevant requirements and current progress.
2. Inspect existing files before creating or replacing anything.
3. Write a short task plan and list changed files.
4. Implement the smallest complete vertical slice.
5. Run focused tests/build and any necessary database/runtime checks.
6. Review the diff for unrelated changes and contract drift.
7. Update `PROGRESS.md` with verified results only.

Do not start the next phase while a blocking exit criterion in the current phase is failing. If time is limited, reduce polish and optional work—not correctness, concurrency, persistence, or required documentation.
