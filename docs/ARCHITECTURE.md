# Architecture and Engineering Standards

## 1. Architecture style
Use a **pragmatic layered monolith** in one ASP.NET Core API project. Organize code by feature/use case, with clear domain, application, and infrastructure responsibilities. Do not create multiple deployable services or a large Clean Architecture solution for two primary entities and eight endpoints.

```text
backend/
  SchoolAdmission.Api/
    Features/
      Classes/        # controller, request/response DTOs, class use cases
      Holds/          # controller, request/response DTOs, reservation use cases
    Domain/
      Entities/
      Enums/
    Infrastructure/
      Persistence/    # DbContext, EF configurations, migrations
      Background/     # hold expiration worker
    Common/            # ProblemDetails, validation, shared primitives
  SchoolAdmission.Tests/

frontend/src/
  features/classes/
  features/holds/
  features/student/
  features/admin/
  shared/api/
  shared/components/
  shared/types/
```

Adapt names to the actual generated solution, but preserve the responsibilities.

## 2. SOLID, applied pragmatically
- **SRP:** Controllers translate HTTP; application services implement one use case; entities enforce their own valid state transitions; the expiration worker schedules expiry work.
- **OCP:** Keep volatile behavior (hold duration/configuration and future payment mechanism) behind existing service/configuration boundaries. Do not add plugin abstractions for hypothetical future features.
- **LSP:** Implementations of interfaces must preserve their documented behavior; do not create interfaces with surprising restrictions.
- **ISP:** Prefer small interfaces for real boundaries (for example, `IAdmissionService`, `IClassService`, `TimeProvider`). Do not create broad “manager” interfaces or interfaces for every class by default.
- **DIP:** Controllers depend on application-service abstractions registered with built-in DI. Application logic receives its persistence dependency through constructor injection. Keep HTTP types out of domain entities.

## 3. Patterns to use
- **Dependency Injection:** built-in ASP.NET Core container.
- **Application Service / Use Case:** `ClassService` and `AdmissionService` own orchestration and business workflows.
- **Options pattern:** bind and validate `HOLD_DURATION_SECONDS` and other configuration.
- **Background Service:** `HoldExpirationWorker` performs automatic expiry.
- **DTO + explicit mapping:** API contracts are separate from EF entities.
- **State transition methods:** keep allowed transitions explicit (`HELD -> CONFIRMED`, `HELD -> EXPIRED`, `HELD -> CANCELLED`).
- **Unit of Work / Repository:** use `DbContext` directly; EF Core already supplies these capabilities. Do not add a generic repository wrapper.

Do not apply Strategy, Factory, Mediator, CQRS, Specification, or event-bus patterns unless a concrete requirement makes one necessary. Patterns are not goals by themselves.

## 4. Entity Framework Core and PostgreSQL
- Use EF Core Code First, one `ApplicationDbContext`, Fluent API configurations, and committed migrations.
- Use compatible stable .NET 10 / EF Core 10 / Npgsql EF Core 10 versions. Do not mix major versions or use preview packages.
- Use `DateTimeOffset`/UTC for `CreatedAt`, `ExpiresAt`, and `ConfirmedAt`; use injected `TimeProvider` so time-dependent logic is testable.
- Store `HoldStatus` as its exact string value (`HELD`, `CONFIRMED`, `EXPIRED`, `CANCELLED`).
- Use database constraints for basic invariants (positive `TotalSeats`, required fields, foreign key) and a PostgreSQL partial unique index on `(ClassId, StudentId)` for `HELD`/`CONFIRMED` rows as defense in depth.
- Do not persist `HeldSeats`, `ConfirmedSeats`, or `AvailableSeats` as separate counters. Derive them from application records.
- Use `AsNoTracking` for read-only queries, projections for API responses, async EF APIs, and pass `CancellationToken`.
- Do not use `EnsureCreated` in place of migrations.

## 5. Concurrency protocol — critical
All state-changing operations for a given class that can affect active seat occupancy must follow one PostgreSQL transaction protocol:
1. Begin a `ReadCommitted` transaction.
2. Acquire a row lock on the corresponding `AdmissionClass` using parameterized `SELECT ... FOR UPDATE` (EF Core transaction + Npgsql SQL is acceptable).
3. Re-read the relevant hold/application rows after acquiring the lock.
4. Reconcile holds whose `Status = HELD` and `ExpiresAt <= now` to `EXPIRED` before allocation/transition decisions.
5. Validate ownership, status, duplicate rule, and capacity.
6. Apply exactly one state transition and commit.

Use this protocol for hold creation, confirmation, cancellation, and expiry processing. Do not replace it with an in-process `lock`, `SemaphoreSlim`, or static dictionary: those do not protect multiple backend processes. Keep transaction scope short; do not call external services while holding the lock.

Capacity calculation: `held = count(HELD where ExpiresAt > now)`, `confirmed = count(CONFIRMED)`, `available = max(0, total - held - confirmed)`. Confirming changes HELD to CONFIRMED, so total occupied capacity must not increase. The worker must recheck expiry after acquiring the same class lock.

## 6. API and error handling
- Preserve routes and JSON field names in `docs/REQUIREMENTS.md`.
- Controllers remain thin; use request/response DTOs and validation.
- Return consistent `ProblemDetails` for errors. Never leak stack traces, SQL, secrets, or raw exception details to clients.
- Map expected business conflicts explicitly (including no seats -> `409`).
- Do not expose database entities or provider-specific details in API responses.

## 7. Frontend standards
- Organize React code by feature; components render UI, hooks own reusable stateful logic, and API modules own HTTP calls.
- Centralize API base URL and `X-Student-Id` handling. Do not construct endpoint strings throughout components.
- Type API request/response contracts explicitly; do not use `any` for convenience.
- Treat API responses as the source of truth. Countdown is derived from `expiresAt`; it cannot authorize confirmation.
- Persist the student identifier only to restore the assignment's no-login student context. Handle missing/invalid ID and backend errors visibly.
- Poll class/hold data at a bounded interval and clean up timers/requests when components unmount.
- Use accessible labels, disabled/loading states, and explicit empty/error states. Do not build decorative screens without working flows.

## 8. Test strategy
- Unit-test pure validation and state-transition rules.
- Integration-test the API and EF behavior against real PostgreSQL (not EF InMemory for database semantics).
- Concurrency test: capacity 10, many distinct student IDs, concurrent requests; assert exactly 10 successful holds and `held + confirmed <= 10`.
- Also test duplicate student/class, expired confirmation, ownership, repeated confirmation, cancellation, automatic expiry, and persistence across restart.
- Each bug fix should add or update a regression test.
