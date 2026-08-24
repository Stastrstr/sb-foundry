# Conventions

Micro-decisions. One row each, no ceremony. Structural decisions go in
[`adr/`](adr/) instead — see the `write-adr` skill for the tier test.

**Rule:** where two options are otherwise equal, prefer the one an
`.editorconfig`, an analyzer, or an architecture test can enforce. A convention
nobody can check is a convention nobody follows.

Status: `Open` → `Decided`. Fill `Decision` and `Enforcement` together — a
decided row with no enforcement is a rule that will quietly rot.

---

## Domain

| # | Convention | Options | Decision | Enforcement | Status |
|---|---|---|---|---|---|
| D1 | Entity / VO construction | public ctor · private ctor + static `Create()` · both | | | Open |
| D2 | Property mutability | private setters · `init` only · readonly record | | | Open |
| D3 | Invariant violations | throw guard clause · return `Result` | | ADR 0009 | Open |
| D4 | Value object shape | `record` · `record struct` · class | | | Open |
| D5 | Domain events — raised where | inside entity · in handler | | | Open |
| D6 | Domain events — dispatched where | `SaveChanges` interceptor · pipeline behavior · explicit | | | Open |
| D7 | Aggregate root marker | marker interface · base class · neither | | NetArchTest | Open |
| D8 | Strongly-typed IDs | yes · no | | | Open |
| D9 | ID primitive | `Guid` v7 · ULID · `int` · `string` | | | Open |
| D10 | Fixed sets of values | `enum` · smart enum · lookup table | | | Open |
| D11 | Domain project dependencies | zero external · allow MediatR contracts only | | NetArchTest | Open |

## Application

| # | Convention | Options | Decision | Enforcement | Status |
|---|---|---|---|---|---|
| A1 | Slice folder granularity | one folder per use case · one per aggregate | | | Open |
| A2 | Slice folder naming | `CreateOrder/` · `Orders/Create/` | | | Open |
| A3 | Type naming | `CreateOrderCommand` + `CreateOrderHandler` · `Command` + `Handler` nested | | analyzer | Open |
| A4 | DTO location | `Application` · `Api` · shared `Contracts` | | NetArchTest | Open |
| A5 | DTO suffix | `Response` · `Dto` · `Model` | | analyzer | Open |
| A6 | Mapping | manual · Mapster · AutoMapper | | | Open |
| A7 | Transaction boundary | UoW pipeline behavior · explicit in handler | | | Open |
| A8 | Handler registration | assembly scan · explicit per slice | | | Open |
| A9 | Query return shape | entity · projection to DTO in the query | | NetArchTest | Open |

## Infrastructure and API

| # | Convention | Options | Decision | Enforcement | Status |
|---|---|---|---|---|---|
| I1 | HTTP surface | Minimal API · controllers | | | Open |
| I2 | Endpoint registration | per-slice static class · central `MapX()` | | | Open |
| I3 | EF configuration | fluent, one file per entity · attributes · inline | | | Open |
| I4 | gRPC service impl location | `Api` · dedicated `Grpc` project | | NetArchTest | Open |
| I5 | Error response shape | `ProblemDetails` · custom envelope | | contract test | Open |
| I6 | API versioning | URL segment · header · none yet | | | Open |
| I7 | Infrastructure → Application direction | interfaces in Application, impl in Infrastructure | | NetArchTest | Open |

## Cross-cutting

| # | Convention | Options | Decision | Enforcement | Status |
|---|---|---|---|---|---|
| X1 | Namespaces | file-scoped · block | | `.editorconfig` | Open |
| X2 | Nullable reference types | enabled everywhere · per project | | `Directory.Build.props` | Open |
| X3 | Implicit usings | on · off | | `Directory.Build.props` | Open |
| X4 | `Async` method suffix | always · only when an overload collides · never | | analyzer | Open |
| X5 | Type sealing | `sealed` by default · only when needed | | analyzer | Open |
| X6 | Test naming | `Method_Scenario_Expected` · `Should_X_When_Y` | | review | Open |
| X7 | Folder ↔ namespace | must match · free | | analyzer | Open |
| X8 | Project naming | `SB.{Asset}.{Name}.{Layer}` · other | `SB.{Asset}.{Name}.{Layer}` — e.g. `SB.Core.ServiceTemplate.Domain` | NetArchTest | **Decided** |
| X9 | Warnings | `TreatWarningsAsErrors` on · off | | `Directory.Build.props` | Open |
| X10 | Analyzer set | .NET built-in · + StyleCop · + SonarAnalyzer | | `Directory.Build.props` | Open |

---

## How to add a row

Append. Do not renumber existing rows — ADRs, tests, and review comments cite
these IDs.

---

## Gaps found by the Tech Lead interrogation

Added 2026-08-24 from `inputs/interrogation-tech-lead.md`. All Open. These are
the rows a fresh-context agent would otherwise improvise — the exact drift the
§1.4 convergence drill exists to catch.

| # | Convention | Options | Decision | Enforcement | Status |
|---|---|---|---|---|---|
| N1 | Slice-to-project topology | folder in Application only · same-named folder per project it touches · true VSA, one folder one project | | NetArchTest | Open |
| N2 | Cross-slice reference isolation | allowed · forbidden, shared code to `Common` on 3rd occurrence | | NetArchTest namespace-pair test | Open |
| N3 | Slice type visibility | public · `internal sealed` + `InternalsVisibleTo` for unit tests · `internal`, no IVT | | compiler + CA1852 | Open |
| N4 | When a slice may skip Domain | never · queries bypass, commands never · slices may be declared CRUD | | NetArchTest on command handlers | Open |
| N5 | Enum persistence | int · string | | model test | Open |
| N6 | ID generation site | client-side in the factory · database-generated | | model test: PK `ValueGenerated == Never` | Open |
| N7 | Aggregate concurrency token | none · store-generated (`rowversion`/`_etag`) · domain-owned `long Version` | | model test + increment unit test | Open |
| N8 | Tenant scoping mechanism | EF global query filter + ambient context · explicit `.Where()` · container per tenant | | model test + ban `IgnoreQueryFilters` | Open |
| N9 | Soft delete | hard delete + domain event · soft delete flag | | model test | Open |
| N10 | Pagination envelope | `{items,nextCursor,total?}` · offset-only shape · none in template | | contract snapshot test | Open |
| N11 | Error catalogue shape | free-form `Error` · static class per slice, `{slice}.{reason}` · central registry | | reflection test on code format + uniqueness | Open |
| N12 | gRPC error mapping | status code only · status + error code in trailers · `Result` in payload | | snapshot test both transports | Open |
| N13 | Edge request types | bind the command directly · `*Request` in Api mapped to command | | NetArchTest: no binding attributes in Application | Open |
| N14 | Response compile safety | plain classes · positional records · `required` members | | analyzer on `*Response` shape | Open |
| N15 | Ambient time seam | `TimeProvider` · own `IClock` · `DateTime.UtcNow` | | BannedSymbols.txt | Open |
| N16 | Current user / tenant seam | `IHttpContextAccessor` in Application · own `ICurrentUser`/`ITenantContext` | | NetArchTest | Open |
| N17 | Authorization placement | endpoint only · pipeline behavior · coarse at endpoint + resource-level in handler | | none automatic; 401/403 contract test per endpoint | Open |
| N18 | Migration execution | `Database.Migrate()` at startup · separate deploy step · versioned SQL scripts | | ban `Database.Migrate`/`EnsureCreated` | Open |
| N19 | Package allowlist per layer | none · allowlist file per csproj | | CI package-allowlist check | Open |
| N20 | `ActivitySource`/`Meter` declaration + naming | per project · per slice; naming scheme | | review | Open |
| N21 | `[LoggerMessage]` source generator | mandatory · optional | | analyzer | Open |
| N22 | Metric tag cardinality | `tenantId` on metric tags · on spans and log scopes only | | review | Open |
| N23 | Architecture-test library | NetArchTest.Rules · NetArchTest.eNhancedEdition · ArchUnitNET | | CI | Open |
| N24 | Test project topology | per layer · Unit + Integration + Architecture | | CI | Open |
| N25 | Integration test entry level | handler-level · through the transport | | review | Open |
