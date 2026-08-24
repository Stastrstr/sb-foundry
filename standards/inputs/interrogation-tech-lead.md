# Tech Lead interrogation — decision register

Output of the Phase 1.2 interrogation, 2026-08-24. **Nothing here is decided.**
This is input to the human decision pass; items become real only as an ADR or a
`conventions.md` row with enforcement named.

Every *Recommendation* below is the agent's, explicitly not a rule.
`[NEW]` marks decisions `conventions.md` had no row for — 13 of them, and they
are the most valuable part of this output.

Run conditions: role prompt inlined into a general-purpose agent (the plugin
agent type needs a session restart to register). 39 decision points returned
against a 25–35 target.

---

## Priority reading — the five that change other decisions

**#31 EF Core vs the SDK rule** `[NEW]` — the highest-value collision found.
The rule "no SDK types outside Infrastructure" is violated on day one by the
chosen stack: MediatR and FluentValidation in Application, ASP.NET in
Presentation, EF Core wherever queries live. *A rule the reference
implementation breaks immediately is a rule nobody believes, and it gets quoted
selectively in review to win arguments.*

*Recommendation: replace the principle with three tiers plus a package
allowlist file —*

| Layer | May reference |
|---|---|
| Domain | BCL only. Zero `PackageReference`. No `ILogger`, no `ActivitySource` |
| Application | Pure-CPU, in-memory-testable packages: mediator abstractions, FluentValidation. **No I/O packages**: Azure SDKs, Redis, Dapr client, `HttpClient` wrappers |
| Infrastructure / Presentation | Anything |

*Whether EF Core sits inside or outside the Application line is the open half,
and it must be settled in the same sitting as ADR 0010. Option (c) —
`IReadDbContext` exposing `IQueryable` — was explicitly refused: `IQueryable` is
BCL but `ToListAsync()`, `Include()`, `AsNoTracking()` are EF extension methods,
so EF returns through the side door and no project-reference test can see it.
Enforced by: CI package-allowlist check per csproj. Same mechanism as #38.*

**#13 Aggregate version** `[NEW]` — *a TL decision the SA's design depends on.*
Options: no concurrency control · store-generated (`rowversion` / Cosmos
`_etag`) · domain-owned `long Version` as the concurrency token.
*Recommendation: domain-owned `Version`. The deciding argument is not the TL's:
the version-stamped-events design in `human-position.md` needs a monotonic
aggregate version **available to the outbox writer before commit**. Choosing
`rowversion` silently breaks that keystone, and it surfaces during CORE, not
now.* Corollary: the template ships the **version** but **not** an outbox — a
version is a data migration to retrofit, an outbox table is not.
Tier: ADR (propose 0013). Enforced by: model test that every `IAggregateRoot`
has a concurrency token; unit test that it increments on every state change.

**#37 NetArchTest may be unmaintained** `[NEW]` — *challenges PLAN.md §0.5/§1.5
directly.* The whole enforcement strategy rests on this library, and ~20 ADRs
will name test methods in it. Public sources say `NetArchTest.Rules` has had no
release since ~2023; `NetArchTest.eNhancedEdition` is a maintained fork;
ArchUnitNET is actively maintained with a larger rule vocabulary.
*Recommendation: smoke-test all three against `net10.0` **before** any ADR cites
a test by name. Choosing after writing them is a 20-file edit.*

**#34 Migration execution** `[NEW]` — *contradicts an SA `[decided]` rule.*
Options: `Database.Migrate()` at startup · separate deploy step / ACA job ·
versioned SQL scripts.
*Recommendation: separate deploy step. Not a preference — startup migration
races across ACA replicas and creates a hard DB dependency at boot, which
breaks the already-decided "start degraded" rule. Someone will write the startup
version because it is the tutorial default.*
Enforced by: BannedApiAnalyzers on `Database.Migrate`/`EnsureCreated`.

**#14 Tenant scoping** `[NEW]` — the in-service half of ADR 0011.
Options: EF global query filter + ambient `ITenantContext` · explicit `.Where()`
per query · container-per-tenant.
*Recommendation: global query filter using EF Core 10 named query filters, so a
filter can be disabled selectively and visibly. Two enforcements: model test
that every `ITenantScoped` entity has a filter, and BannedApiAnalyzers on
`IgnoreQueryFilters`. Hard rule: tenant id comes from the token, never the
request body or route.* Same row decides soft delete — *recommendation: hard
delete plus a domain event; soft delete leaks into every query forever and is
effectively irreversible once data exists.*
Failure mode: cross-tenant disclosure. Not reversible — it is an incident.

---

## Structure

| # | Decision | Recommendation | Tier | Enforced by |
|---|---|---|---|---|
| 1 | Layer topology | Four projects (Domain/Application/Infrastructure/Api). *Violates "least ceremony" — wins on priority 1, since a project reference is the only enforcement surviving a dev who never read the ADR.* gRPC impls in Api (I4); Api is the only composition root | ADR 0004 | NetArchTest layer tests |
| 2 | Slice topology + **cross-slice isolation** `[NEW]` | Same-named folder per project it touches. **Add the isolation test nobody writes**: no type in `Features.X` may reference `Features.Y`. Without it, "deletable in one commit" is aspiration | ADR + convention | NetArchTest namespace-pair test |
| 3 | Slice visibility `[NEW]` | `internal sealed` + `InternalsVisibleTo` for unit tests only — makes cross-slice coupling a *compile* error, free enforcement of #2 | convention | compiler + CA1852 |
| 4 | When a slice may skip Domain `[NEW]` | Queries bypass Domain; commands never do. *Rejected "declare a slice CRUD" — unenforceable, so always answered yes under deadline* | ADR | NetArchTest: command handlers may not touch `DbContext` |

## Domain

| # | Decision | Recommendation | Tier | Enforced by |
|---|---|---|---|---|
| 5 | Construction (D1) | Private ctor + static `Create()` **on aggregate roots only**; VOs get public ctors when they have no rules. *Uniform `Create()` everywhere gets copied 40 times* | convention | NetArchTest: no public ctors on `IAggregateRoot` |
| 6 | State shape (D2/D4) | Entities = class + private setters + methods; VOs = `sealed record`. `record struct` only after profiling | convention | review + sealed-or-abstract test |
| 7 | Invariant violations (D3) | Throw in Domain, `Result` in Application. **The line is the whole decision**: if a valid use case produces it → `Result`; if only a bug produces it → throw | ADR 0009 | **nothing — review only** |
| 8 | Domain events (D5/D6) | Raise in the entity, dispatch in a post-commit behavior. **Collision nobody writes down: D11's zero-dependency rule forbids events implementing `INotification`** — most reference solutions violate this silently. *Template ships **no** domain event* | ADR 0004 | NetArchTest: Domain ⇸ MediatR |
| 9 | Aggregate marker (D7) | Marker interface. Base class is reuse-not-polymorphism | convention | NetArchTest |
| 10 | IDs (D8/D9) + **generation site** `[NEW]` | `Guid` v7, **client-generated**, primitive in the template. *Generation site is non-negotiable: DB-generated breaks Cosmos, breaks outbox stamping (no ID before commit), and makes handler tests need a database.* Strongly-typed is genuinely open — decide deliberately, the template wins by inertia | ADR (site) + convention | EF model test: every PK has `ValueGenerated == Never` |
| 11 | Enums (D10) + **persistence** `[NEW]` | `enum` stored as **string**. *The persistence half is missing and is the expensive half — int→string later is a data migration* | convention | model test: enum properties have string conversion |
| 12 | Domain dependencies (D11) + telemetry | Domain references **nothing**, including `Logging.Abstractions` and `DiagnosticSource`. *Answers your open question: **no** ILogger in Domain — for testability and object-graph purity, not the vendor rule* | convention | NetArchTest + CI check: zero `PackageReference` |

## Application

| # | Decision | Recommendation | Tier | Enforced by |
|---|---|---|---|---|
| 15 | **Mediator library + licence** `[NEW]` | *Explicitly refused to resolve — turns on revenue vs the vendor threshold, not a TL call.* MediatR 13 is dual-licensed; 12.x stays Apache-2.0. Technically **cheaper to reverse than it feels** if the mediator surface stays in handler signatures + one registration line | ADR (propose 0014) | CI package allowlist |
| 16 | Pipeline behaviours (0012) | **Exactly two, ordered: validation → transaction** (commands only). No logging or timing behavior — that is OTel's job, and a second inconsistent story | ADR 0012 | test asserting the registered sequence equals an expected array |
| 17 | Slice naming (A1/A2/A3) | `Features/{Aggregate}/{UseCase}/` + suffixed types. *Picked the shape the enforcement can see* | convention | name↔namespace test |
| 18 | Command return + DTOs (A4/A5) | Commands return the ID; queries return DTOs. **No shared `Contracts` project inside the service** — shared-kernel trap at service scale, breaks slice deletability | convention | NetArchTest on `*Response` location |
| 19 | Projection (A9) + **pagination envelope** `[NEW]` | Project inside the query. **Define the envelope now** — `{ items, nextCursor, total? }` — and implement offset behind it. *The envelope is a public REST contract; the implementation is not. Cheapest future optionality in the whole list* | convention | contract snapshot test |
| 20 | Mapping (A6) + **compile safety** `[NEW]` | Manual mapping **plus**: `Response` types are positional records or have `required` members. *Without that, manual mapping buys nothing over Mapster — object initializers drop fields just as silently* | convention | analyzer on `*Response` shape |
| 21 | Transactions + registration (A7/A8) | UoW behavior on commands; `SaveChangesAsync` banned in handlers. Assembly scan — *wins on **deletability**, not ceremony; the only reason to accept startup reflection* | convention | BannedApiAnalyzers |
| 22 | **Ambient context: time, user, tenant** `[NEW]` | `TimeProvider` (never `IClock` — BCL ships the seam and the fake). Own `ICurrentUser`/`ITenantContext` — *survives the abstraction-for-one refusal because it keeps `IHttpContextAccessor` out of Application and has a real test seam*. Ban `DateTime.UtcNow` | convention | BannedSymbols.txt |
| 23 | **Authorization placement** `[NEW]` | Coarse at the endpoint (native in both transports); resource-level inside the handler returning a domain `Error`. *Two competent teams genuinely diverge — the behavior approach is defensible* | ADR (security weight) | **nothing automatic**; mandatory 401/403 contract test per endpoint |

## Errors and validation

| # | Decision | Recommendation | Tier | Enforced by |
|---|---|---|---|---|
| 24 | Result type (0009) | Hybrid, **hand-rolled** `Result`/`Error` in Domain (~60 lines). A library is a vendor type in Domain, colliding with your own SDK rule. **Cost nobody mentions: `Result` + mediator pipeline do not compose** — a validation behavior short-circuiting `IRequest<Result<T>>` cannot construct `Result<T>` for unknown `T` without reflection. *Decide the mechanism in the ADR or the first implementer uses `Activator.CreateInstance` and it ships to 40 repos* | ADR 0009 | analyzer on handler return types |
| 25 | **Error catalogue** `[NEW]` | Static error class per slice, codes `{slice}.{reason}`. *A central registry is a file every slice edits — defeats deletability* | convention | reflection test: codes match regex, unique |
| 26 | Error→transport (I5) + **gRPC half** `[NEW]` | `ProblemDetails` for REST; gRPC status + same error code in trailers; **one** mapping table defined once in Api. *I5 covers REST only — without the gRPC row the two transports report the same failure differently* | convention | snapshot tests per category, both transports |
| 27 | Validation (0012) | FluentValidation in a behavior; **do not** call .NET 10's `AddValidation()`. *Deciding argument: two transports on day one, so validation sits on the command, not the endpoint. For a REST-only service the built-in is genuinely better — record that rather than pretending FV is universal.* **Sub-rule: validators are structural only** — uniqueness/existence belong in the handler with a unique index, because a DB-querying validator is impure and a TOCTOU bug | ADR 0012 | package allowlist; test that every `*Command` has a validator |

## Presentation

| # | Decision | Recommendation | Tier | Enforced by |
|---|---|---|---|---|
| 28 | HTTP surface (I1/I2) | Minimal API + `IEndpoint` scan, and **one `RouteGroupBuilder`** carrying auth, versioning, ProblemDetails. *The group is the important half — without a single composition point every endpoint copy-pastes conventions and they drift by slice five* | convention | test: every endpoint reachable and under the versioned group |
| 29 | **Edge request types** `[NEW]` | Separate `*Request` in Api mapped to the command. *Follows from your own SDK rule — generated protobuf types are vendor types that cannot reach Application, so binding commands directly would give the two transports different rules* | convention | NetArchTest: no ASP.NET binding attributes in Application |
| 30 | Versioning (I6) | URL segment from day one, at the route group. *"None yet" always becomes "v1 implied", and retrofitting is a breaking change for every consumer. Strongest cost/benefit ratio in the list* | convention (ADR-adjacent) | test: no route outside the versioned group |

## Persistence

| # | Decision | Recommendation | Tier | Enforced by |
|---|---|---|---|---|
| 32 | Repository vs DbContext (0010) | Repository per aggregate for commands; direct query access for reads. Generic `IRepository<T>` refused — one implementation, no polymorphism, hides the aggregate boundary #9 and #13 depend on | ADR 0010 | NetArchTest |
| 33 | EF configuration (I3) | `IEntityTypeConfiguration<T>` per entity, assembly-scanned. *Honest cost: persistence ignorance means EF configures private setters and backing fields by convention, and when that breaks it breaks confusingly* | convention | model test |
| 35 | **Template provider** `[NEW]` | SQL Server + Testcontainers, **plus an explicit acknowledgement in ADR 0004 that Cosmos services diverge** and need their own template variant. *Cosmos has no migrations, no scaffolding, partition-scoped batching only — it diverges day one regardless. Abstracting over both is exactly the "we will need this later" generality to refuse* | ADR 0004 | nothing — caught by the convergence drill |

## Testing, packaging, enforcement

| # | Decision | Recommendation | Tier | Enforced by |
|---|---|---|---|---|
| 36 | **Test topology** `[NEW]` | Three projects; integration tests enter **through the transport**. *Handler-level integration tests cannot catch binding, auth, serialization, or route conventions — which is where slice bugs actually are.* Template ships **one** integration test | convention | CI |
| 38 | Packages (0005) | CPM on from day one. Shared packages **only** for infrastructure plumbing, never domain types, contracts, or a `Common` grab-bag. The template depends on no `SB.*` package — it is copied, not referenced. *Same distinction the SA drew for `SB.Core.Messaging` vs `SB.Contracts` — keep both ADRs consistent or one gets cited to justify the other* | ADR 0005 | CI package allowlist (same mechanism as #31) |
| 39 | Build strictness (X4/X5/X9/X10) | `TreatWarningsAsErrors` **everywhere**, `AnalysisLevel=latest-recommended`, plus **BannedApiAnalyzers** — the workhorse behind ~8 decisions above. **No StyleCop** — generates a suppression file within a month and teaches people to suppress | convention | `Directory.Build.props` + `BannedSymbols.txt` |

---

## Part 2 — challenges to the human position

| Item | Verdict |
|---|---|
| Result pattern | **Agree.** Value is that failure appears in the *signature*, not "avoiding exceptions". Six under-specified points listed below |
| FluentValidation | **Agree**, but one of your reasons is now weaker — .NET 10 ships built-in minimal API validation. Justify FV by *two transports*, not self-evidence |
| Minimal API | **Agree.** Costs to record: lose `[ApiController]`'s automatic 400+ProblemDetails; conventions must go through a route group or drift |
| **No SDK types outside Infrastructure** | **Agree with intent, disagree with the rule as written** — see #31. Violated day one by your own stack |
| Orchestration in Application | **Agree, but "orchestration" is undefined so the rule cannot be applied.** Operative sentence proposed below |
| Telemetry | **Agree, strongly** — best-specified item in the file. Five gaps below |

### Result — six things under-specified enough to block writing code

1. Does Domain return `Result` or throw? (D3 and ADR 0009 must be decided together.)
2. One error per `Result` or many? Validation produces many, a domain rule one.
3. Hand-rolled or library? A library is a vendor type in Domain — collides with your SDK rule.
4. Does `Result` cross the transport boundary? *Recommendation: mapped at the edge — a `Result` on the wire is a custom error envelope contradicting `ProblemDetails`.*
5. **"Tracked" is doing a lot of work.** Log every failed `Result` at Error → alert noise; none → silent failures. *Recommendation: behavior logs failures at Warning with the error code as a structured field; exceptions at Error.*
6. The `Result` + pipeline composition problem (see #24).

### Orchestration — the operative sentence

*Proposed: **an endpoint performs exactly one call into Application, plus binding
and status mapping. Two calls means it belongs in a handler.*** Applicable in
review in five seconds.

May an endpoint: bind and map request→command (**yes**, translation) · call two
handlers (**no**) · choose 201 vs 200 from a `Result` (**yes**, transport) ·
build a `Location` header (**yes**) · open a transaction (**no**) · read
tenant/user from `HttpContext` onto the command (***this is where competent
teams diverge*** — recommendation **no**, go through `ICurrentUser` so both
transports share one path).

*Honest consequence: if a use case genuinely needs two aggregates in two
transactions, the answer is a process manager or outbox — SA territory. Say so
in the ADR or the rule gets worked around rather than escalated.*

### Telemetry — five gaps

- **Where `ActivitySource`/`Meter` instances are declared.** Their *names* become
  the filter keys every dashboard uses — a public-ish contract. Missing entirely.
- **Is `[LoggerMessage]` mandatory?** *Recommendation: yes — compile-checked
  templates, no allocation, source generator so no runtime dependency.*
- **Event ID and message-template conventions.** Without them the logs are not
  queryable, defeating the move to structured logging.
- **Metric cardinality, and it has a bill attached.** `tenantId` as a `Meter` tag
  multiplies every time series by tenant count. *Recommendation: tenant id on
  **spans** (sampled) and log scopes, never metric tags unless the tenant count
  is bounded and small. Decide before CORE, not after the first invoice.*
- **Manual spans in the template: no.** Auto-instrumentation covers the one slice.

---

## Cross-cutting flags

1. **Three items in your position collide with your own SDK rule** — MediatR and
   FluentValidation in Application, EF Core wherever queries live. Decide the SDK
   ADR and ADR 0010 in one sitting or they will contradict.
2. **#13 is a TL decision the SA's design depends on.** Cross-reference it in
   both ADRs.
3. **PLAN.md's enforcement rests on NetArchTest, which appears unmaintained.**
   Smoke-test before ~20 ADRs name test methods in it.
4. **13 missing `conventions.md` rows**: slice-to-project topology · cross-slice
   isolation · pagination envelope · error catalogue · tenant scoping ·
   aggregate version/concurrency · ID generation site · enum persistence ·
   migration execution · telemetry naming and cardinality · edge request types ·
   required-member responses · package allowlist. *That is where a fresh-context
   agent improvises — exactly what the §1.4 convergence drill is built to catch.*

---

## Verification log

**Checked (web search, August 2026):** MediatR 13 dual-licensed RPL-1.5 /
commercial, free below a stated revenue threshold; 12.x remains Apache-2.0 ·
ASP.NET Core 10 ships `AddValidation()` over DataAnnotations with per-endpoint
`DisableValidation()` (secondary sources, not confirmed on learn.microsoft.com) ·
FluentValidation 12 Apache-2.0; `FluentValidation.AspNetCore` deprecated,
auto-validation removed after 11.1.0 · EF Core 10 named query filters,
`LeftJoin`/`RightJoin`, complex types to JSON, structs as complex types · EF
Cosmos: no migrations, no scaffolding, atomicity only within a single-partition
batch (docs describe default batching "starting with EF 11" — do not assume on
EF 10) · NetArchTest.Rules last release ~2023; eNhancedEdition fork and
ArchUnitNET maintained (secondary sources, **not** run against net10.0).

**Asserted from memory, NOT verified:** `Guid.CreateVersion7()` availability
(believed .NET 9+) · `dotnet ef migrations has-pending-model-changes` (believed
EF 9+) · AutoMapper moving to the same commercial licence · BannedApiAnalyzers
compatibility with the .NET 10 SDK.
