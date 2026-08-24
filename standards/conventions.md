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
