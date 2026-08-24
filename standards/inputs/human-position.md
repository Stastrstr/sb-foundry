# Human position — input to the Phase 1.2 interrogation

**This file is not a standard.** It is the human's starting position, recorded so
the interrogation has something concrete to push against.

## How agents must treat this

- **Challenge every item.** For each, say whether you agree, and if not, what
  you would do instead and what it costs. Agreement is a finding too — say
  *why* it holds, not just that it does.
- **Nothing here is accepted.** Items become real only as an ADR or a
  `conventions.md` row, with enforcement named.
- **Do not cite this file as authority.** If a decision matters, it has an ADR.
- Confidence markers below are the human's, not yours:
  `[decided]` the human has committed · `[position]` a preference, argue with it
  · `[TBA]` explicitly unresolved, needs clarification before it can be decided.

---

## Project intent and non-goals `[decided]`

**The deliverable is the process, not the stack.** The goal is a system that
consumes an RFP, explores it, works with a Product Owner, and builds a solution
from the resulting requirements. The stack is deliberately narrow to keep that
demonstrable — narrowness is a feature, not an oversight.

**Explicit non-goals for the POC.** Do not abstract over these; they are future
options, deliberately deferred:

- Mongo as an alternative to Cosmos
- PostgreSQL as an alternative to Azure SQL
- Any provider-agnostic persistence abstraction

*An agent proposing a repository or provider abstraction "so we can swap the
database later" is proposing the "we will need this later" generality the Tech
Lead role prompt already refuses. Cite this section.*

**No cost requirements for the POC.** Cost is being handled in a separate
dedicated session. Do not let cost drive stack choices now; do record cost
consequences where they are non-obvious (metric cardinality, Durable Task
Scheduler, Cosmos RUs) so the later session has them.

**Commercial and licence decisions are deferred, not resolved.** MediatR is
`[decided]` for the POC — use the existing implementation, do not reimplement
it. The dual-licence question is revisited **before any production use**, not
now. Recorded as a revisit trigger in ADR 0014, not as a settled choice.

---

## Tech Lead scope — inside one service

| Item | Confidence | Lands as |
|---|---|---|
| Result pattern. Do not raise exceptions where the result can be handled and tracked and it is not a show-stopper | `[position]` | ADR 0009 |
| **No reflection anywhere without strong rationale** | `[decided]` | cross-cutting; constrains ADR 0009 and A8 |
| FluentValidation for validation | `[position]` | ADR 0012 |
| Minimal API | `[position]` | conventions I1 |
| **No I/O SDK types outside Infrastructure** — Cosmos SDK, Redis SDK, Azure SDKs. They live in Infrastructure only, behind our own interface | `[decided]` | new ADR |
| **Orchestration logic belongs to the Application layer, not Presentation** | `[position]` | new ADR |
| Telemetry uses native .NET only — see below | `[position]` | same ADR as the SDK rule |

### Telemetry

Drop `TelemetryClient` and every Azure-specific SDK from application code.

| Purpose | Old | Use instead |
|---|---|---|
| Logs and exceptions | `TrackTrace()`, `TrackException()` | `ILogger.LogInformation()`, `ILogger.LogError()` |
| Custom spans and timings | `TrackDependency()`, `TrackEvent()` | `ActivitySource.StartActivity()` |
| Custom metrics | `TrackMetric()` | `System.Diagnostics.Metrics.Meter` |

`ILogger` for application events, debug statements, exceptions. `Activity` for
custom performance profiling — timing a block, custom waterfall steps, tags the
Collector can filter on. `Meter` for numbers and rates — active users, queue
depth.

No vendor library reaches core application code; OpenTelemetry delivers to
AppInsights, the OTel Collector, or the local Aspire dashboard.

*This is the same rule as "no SDK types outside Infrastructure" — telemetry is
just its most-violated instance. Write it once. Open: `ILogger`,
`ActivitySource`, and `Meter` are BCL, not vendor, so the rule permits them
everywhere — but should they be allowed in `Domain`? That is a conventions row.
The Tech Lead interrogation answered **no**, on testability and object-graph
purity grounds rather than the vendor rule — see N-rows in `conventions.md`.*

### Collector-side processing `[position]`

All shaping happens in the **OTel Collector**, not in application code — so the
app stays vendor-neutral and policy changes without redeploying services.

| Signal | Treatment |
|---|---|
| Requests | as-is |
| Exceptions | as-is |
| Custom events | as-is |
| Dependencies | aggregate over N minutes, sample the remainder, record duration anomalies as-is — split per type: SQL, Service Bus, etc. |
| Traces | aggregate before sampling, archive to Blob |

*Three points for the SA to resolve in ADR 0008:*

1. ***Metric cardinality is a different mechanism from log volume.*** *A `Meter`
   with a `tenantId` tag creates one time series per tenant per metric per
   other-tag-combination — a storage and billing multiplier, not a volume one.
   The Collector can strip the tag, but then per-tenant metrics are gone, which
   may be wanted for tiering and billing. The decision is **which few metrics
   keep `tenantId`** — bounded and deliberate — not a blanket setting.*
2. ***Traces in Blob are write-only unless something can query them.*** *Without
   a retrieval pipeline it is paying to store data nobody can reach during an
   incident. **Tail-based sampling** usually gets the same value: keep 100% of
   traces containing an error or breaching a latency threshold, sample the rest
   at N%.*
3. ***Tail sampling constrains the deployment.*** *The Collector must buffer all
   spans of a trace before deciding, so **every span of a trace must reach the
   same Collector instance**. With multiple ACA replicas that needs the
   `loadbalancing` exporter routing by trace ID in front of the sampling tier —
   a two-tier Collector topology. Decide before ADR 0008 assumes a single tier.*
4. *Anomaly detection needs a definition — p99 breach, absolute threshold, or
   both, per dependency type. Currently unspecified.*

---

## Solution Architect scope — crosses service boundaries

| Item | Confidence |
|---|---|
| **Aspire for service discovery and related microservice logic — scoped to local dev, wiring, and OTel only. Deployment stays hand-written IaC** | `[decided]` |
| DAPR for any communication it supports. Where unsupported, decide case by case | `[position]` + `[TBA]` |
| Simple API → ASP.NET. Stateful orchestration → Durable Functions | `[position]` |
| Durable Functions deployed as a **Docker image, not the SaaS/serverless model** | `[position]` |
| Durable Functions are the Orchestrator Services | `[position]` |
| Service to service: **gRPC** | `[position]` |
| Service to service pub/sub: **Service Bus Topic** | `[position]` |
| Service to service one-way: **Service Bus Queue** | `[position]` |
| Session support, where a sequence of events must hold: via DAPR | `[position]` + `[TBA]` |
| Public access: **REST** | `[position]` |
| Denormalized services → **Cosmos DB** | `[position]` |
| Relational needs → **Azure SQL** | `[position]` |
| Files → **Azure Blob** | `[position]` |
| Health checks — Aspire defaults for now | `[TBA]` |
| Retries and similar — probably covered by DAPR | `[TBA]` |

### SDK rule — clarified, and it converges with the interrogation

**Human's clarification (2026-08-24):** the rule was always about **I/O SDKs** —
Cosmos SDK, Redis SDK, Azure clients. Not MediatR, not FluentValidation.

*This is the Tech Lead's proposed Application tier almost exactly. The
interrogation disagreed with the literal wording, not the intent; the intent was
enforceable all along. Adopt the three-tier allowlist from
`interrogation-tech-lead.md` #31 — it now expresses what was meant:*

| Layer | May reference |
|---|---|
| Domain | BCL only. Zero `PackageReference` |
| Application | Pure-CPU, in-memory-testable: mediator abstractions, FluentValidation. **No I/O packages**: Cosmos SDK, Redis, Azure SDKs, Dapr client, `HttpClient` wrappers |
| Infrastructure / Presentation | Anything |

**Still open — and it is the expensive half: is EF Core an I/O SDK?** By the
clarified logic it is, which bans it from Application and pushes every read
query into Infrastructure, making a slice span three projects. The alternative
keeps EF in Application and treats it as the one carved-out I/O package.
`IReadDbContext` exposing `IQueryable` was already refused: `ToListAsync`,
`Include`, and `AsNoTracking` are EF extension methods, so EF re-enters through
a side door no project-reference test can see. **Decide with ADR 0010, in the
same sitting.**

Enforced by: CI package-allowlist check per csproj — the same mechanism as #38.

### Outbox store `[decided]`

**The outbox lives in whatever store the service already has. Never introduce a
store for it.** Standing up SQL purely to hold an outbox table reintroduces the
dual write the pattern exists to eliminate.

| Service store | Outbox mechanism |
|---|---|
| Azure SQL | Outbox table, same transaction as the business write |
| Cosmos DB | **Explicit event documents in the same container, same partition key**, written in one transactional batch, read off the **change feed** |

*The Cosmos form is stronger than it appears — the change feed is already a
durable, replayable, per-partition-ordered stream with checkpointing, so it is
an outbox reader out of the box. Two things must be right: write **explicit
event documents**, never infer events from state diffs; and put a **TTL** on
them so they do not accumulate.*

*The Cosmos **partition key is the ordering scope**, which makes it the same
decision as the Service Bus session key. Decide them together.*

*Verify first: DAPR has a transactional outbox on state stores and Cosmos is a
supported one. It may collapse most of this into component configuration.*

### Reliable messaging — the "cannot lose this message" pattern `[TBA]`

Where two services must stay near-perfectly in sync — a write side and a read
side — the human's position is **outbox** for sending, **Service Bus sessions**
for ordering, a **dead-letter listener**, and a **resync API** driven by schedule
or on dead-letter. Wanted as a **reusable pattern**, not a one-off.

Worked example: assigning or unassigning a person to a folder, where a separate
service renders a document grid and must not miss the change.

**Verified:** DAPR's Azure Service Bus pub/sub component does support sessions
(`requireSessions`, `sessionIdleTimeoutInSec`, `maxConcurrentSessions`). But
there are reported issues that session-enabled topics do not always deliver in
sequence (dapr/components-contrib#3281) and that same-session messages can be
delivered in parallel (dapr/dapr#8834). **Do not make correctness depend on
broker ordering.**

*Recommended addition to challenge — **version-stamped events**. Every event
carries the source aggregate's monotonic version; the consumer stores
`lastAppliedVersion` per entity and discards anything `≤` it.*

*Consequences, and why this is the keystone:*

- *The consumer becomes idempotent **and** order-insensitive at once. Duplicates
  are free, reordering is free.*
- *Sessions become an optimisation rather than the correctness mechanism, so the
  DAPR ordering issues stop being a risk and this path keeps the DAPR
  portability seam instead of being special-cased.*
- *Resync can no longer race live traffic. Without versions, a resync landing
  beside a live message can move state **backwards**. With them, both paths
  converge.*
- *The assign/unassign example is order-sensitive, so out-of-order delivery
  silently inverts the result. Versions make that impossible, not just unlikely.*

Open points:

- **Session key choice is the highest-stakes decision here.** Session on
  `folderId` and folders process in parallel; session on `tenantId` and an entire
  tenant serialises. Use the smallest unit that needs ordering.
- **Do not auto-replay the dead-letter queue** — a poison message loops
  DLQ → retry → DLQ forever. DLQ is an alert; **resync is the recovery path**.
  State-based recovery converges, event-based replay can be arbitrarily stale.
- Verify whether DAPR's **transactional outbox** on state stores covers the
  sending half before hand-rolling it.
- Reusable as `SB.Core.Messaging` — outbox, version tracking, idempotent-consumer
  base. **This is infrastructure plumbing, not domain contracts**, so the
  shared-kernel objection raised against a single `SB.Contracts` does not apply.
  State that distinction explicitly in the ADR, or one will be cited to justify
  the other.

### Microservice invariants — estate-level audit `[decided]`

We must actively verify we are not drastically violating microservice
principles — shared databases, shared business logic, and similar.

**The structural point: per-repo CI cannot see these violations.** A service's
own pipeline has no visibility into another service's connection strings or
package graph. Every enforcement designed so far lives inside one repo; these
invariants need a vantage point *above* the repos. The audit therefore runs in
`sb-foundry` or `sb-iaac`, reading the IaC, the package graph, and the service
catalog — on a schedule and on any IaC change.

**Mechanically detectable — these become estate CI checks:**

| Invariant | Detected from |
|---|---|
| No two services share a logical database or Cosmos container | IaC + App Configuration connection targets |
| No service reads another service's store directly | connection-target audit against the service catalog |
| Shared packages are infrastructure plumbing only — never domain types or business rules | package graph: any `SB.*` package referenced by 2+ services must be on an allowlist |
| Contracts packages contain generated types only | package-content check (see interrogation #38) |
| A service depends only on another's `.Contracts`, never its internals | package graph |
| One writer per dataset | service catalog: data-owned field |
| 1 repo = 1 service, repo name reflects the asset | repo audit against `sb-{asset}-{name}` |
| No shared schema or migrations across services | migration-project ownership audit |

**Needs judgement — periodic SA review, not CI:**

- *Is this shared library infrastructure plumbing or business logic?* The
  `SB.Core.Messaging` versus `SB.Contracts` distinction is the slippery one and
  it is where a distributed monolith actually starts.
- Are two services really one bounded context split wrongly?
- Is a given coupling accidental or essential?
- Synchronous call chains deeper than two (the SA role prompt already refuses
  these; detecting them needs traces or a call graph).

**Owner:** Solution Architect. **Feeds:** the service catalog in `sb-iaac`,
which is now load-bearing rather than nice-to-have — it is the audit's input.

*Add to the SA interrogation scope: which of these are worth automating in
Phase 1a versus deferring, and what the catalog schema needs to contain to make
the mechanical checks possible.*

### Service dependency map `[TBA]`

Human wants a dependency map for three purposes: CI version checks, deployment
ordering, and health checks.

*Recommendation to challenge: only one of the three is a real need, and a map is
not how it is met.*

- ***Deployment ordering is an anti-pattern.*** *If services must deploy in a
  set order, the estate is a distributed monolith. The human has accepted this
  and asked for the standard replacement patterns — see **Independent
  deployability** below. Adopt the rule: **every service starts and becomes
  healthy regardless of whether its dependencies are up**, and any service
  version interoperates with any other. Both are testable.*
- ***Health checks must not consult a dependency map.*** *If B reports unhealthy
  because A is down, the orchestrator restarts or removes B and one failure
  cascades. Liveness = am I running, never external. Readiness = only what is
  needed to serve at all. Dependency health = telemetry and alerts, never a
  status that triggers orchestration.*
- ***CI version checks are real — but derive the map, do not author it.*** *The
  package feed already knows who references which contract package. A
  hand-maintained file drifts within a month and then lies. With additive-only
  contracts, any consumer version can talk to any producer version anyway, so
  the map is for blast-radius visibility on a forced major — which should be
  near-never.*

*What is worth authoring is a **service catalog** — owner, asset, contracts
published, data owned. Metadata, not a graph. Natural fit for `sb-iaac`.*

### Independent deployability `[decided]`

The human has accepted that deployment ordering is an anti-pattern and asked to
adhere to standard microservice practice instead. These patterns replace it and
should become an ADR with each row's enforcement named.

| Pattern | Solves | Enforced by |
|---|---|---|
| **Expand / contract** (parallel change) | Any change that appears to need coordination | Review + contract tests |
| **Additive-only contracts**, tolerant reader | Producer/consumer version skew | `buf breaking` in CI |
| **Deploy ≠ release** via feature flags | "B needs A's new capability first" | Azure App Configuration Feature Management |
| **Start degraded** | "B cannot boot without A" | Startup test run with dependencies down |
| **Consumer-driven contract tests** | Knowing you broke a consumer before deploying | Pact-style verification in producer CI |

**Expand/contract is the universal technique.** Any coordinated-looking change
becomes three independently safe deployments: *expand* (new alongside old, both
work) → *migrate* (consumers move, any order) → *contract* (delete the old once
unreferenced). No step breaks a running version, so no step has a prerequisite.
Applies to protobuf fields, REST endpoints, message schemas, and database
columns alike.

**Feature flags are what actually removes the requirement.** Most real "A before
B" cases are "B's new behaviour needs A's new capability". Deploy both dark in
any order, flip the flag when everything is in place. Azure App Configuration is
already in the stack — no new component.

**Database changes** get their own form: add nullable column → backfill → write
both → read new → stop writing old → drop old. Rule: *a migration may never
break the currently deployed code.*

**Start degraded** means no blocking calls to other services during startup.
Retry with backoff, circuit-break, serve what you can. DAPR provides the
resiliency policies already.

**Escape hatch, stated honestly:** a semantic change where old and new genuinely
cannot coexist becomes a coordinated release with a runbook, treated as
exceptional. *Needing this more than rarely is a signal that service boundaries
are wrong, not that orchestration is missing.* That diagnostic value is part of
why the rule is worth keeping.

---

### Result type and pipeline short-circuit `[position]` — mechanism resolved

Human proposed **ErrorOr** (or an equivalent of our own).

*The composition problem, stated precisely, because it is not a library
problem.* A `IPipelineBehavior<TRequest, TResponse>` short-circuiting on
validation failure must return `TResponse`, but inside that class `TResponse` is
an open type parameter — the compiler does not know it is a `Result<something>`,
so a failed result cannot be constructed. **ErrorOr's implicit conversions do
not fire on an open generic**, so the wall is identical. Concrete-typing the
behavior works but the genericity comes from DI registration
(`typeof(ValidationBehavior<,>)` applied assembly-wide), so per-type behaviors
would mean one class per slice, hand-maintained.

*Resolution, and it satisfies the no-reflection rule outright — **static
abstract interface members** (C# 11, available on .NET 10):*

```csharp
public interface IResultOf<TSelf> {
    static abstract TSelf FromErrors(IReadOnlyList<Error> errors);
}

public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TResponse : IResultOf<TResponse>
{
    // ...
    if (failures.Count > 0) return TResponse.FromErrors(failures);  // compile-time, no reflection
}
```

*Consequence for the library choice: this argues for **hand-rolled** over
ErrorOr — the constraint interface must be declared on our own type, and ErrorOr
is a vendor package in Application, colliding with the human's own SDK rule.
Roughly 60 lines. Record the mechanism in ADR 0009 explicitly, or the first
implementer reaches for a cast or `Activator.CreateInstance`.*

#### Rejected: non-generic marker interface plus a cast

Proposed and worth recording, because it will be proposed again:

```csharp
public interface IRequestHandlerResult { }          // plain marker

where TResponse : IRequestHandlerResult
    IRequestHandlerResult failure = new FailureResult(errors);
    return (TResponse)failure;                       // InvalidCastException at runtime
```

**Why it fails.** The constraint tells the compiler *what `TResponse` is*; it
gives no way to *create* one. `FailureResult` is not `Result<Guid>`, so the cast
compiles and throws at runtime.

**Why it is worse than reflection.** It compiles, it reads as type-safe, and it
fails on the first validation error of an untested slice — in production.
Reflection at least fails where you expect it to. The two workarounds seen in
the wild are `(TResponse)(object)Result<Unit>.From(errors)`, which works for
exactly one response type, and `typeof(TResponse).GetGenericArguments()`, which
is the reflection the `[decided]` no-reflection rule forbids.

The generic self-referential marker above is the same design with the flaw
removed — `static abstract` lets the constraint carry a constructor rather than
just an identity.

#### Alternative, legitimate if chosen deliberately

Throw a `ValidationException` and map it in middleware. Widely used. The
trade-off is explicit: validation failures leave the `Result` channel and become
exception-shaped, so "expected failures are Results" stops being universally
true. Acceptable as a decision; not acceptable as an accident because the
generic felt awkward.

### Contract distribution

| Item | Confidence |
|---|---|
| Contracts for gRPC, REST, and Service Bus messages ship as **NuGet packages on an internal feed** | `[position]` |

Human's stated goal: avoid data-consistency issues between services.

*Correction to carry into the ADR: a package gives **compile-time** consistency,
not data consistency. Service A on v2 while Service B still runs v1 is a runtime
skew the package cannot prevent. Wire-compatibility discipline — protobuf field
numbering, additive-only change — enforced by `buf breaking` in CI is what
prevents data issues. The package is distribution, not safety.*

Open points for the interrogation:

- One package **per producing service** (`SB.{Asset}.{Name}.Contracts`), never a
  single shared `SB.Contracts` — a package everyone references is the
  shared-kernel trap and produces a lockstep upgrade cycle.
- The `.proto` is the source of truth; the package is a **build output**, never
  hand-edited.
- **Generated types only** — no helpers, validators, base classes, or extension
  methods. Contract packages accrete behaviour unless something stops them.
  Enforceable as a package-content check.
- Service Bus message contracts are **additive-only forever** — messages are
  persisted and outlive deployments, so they need stricter rules than gRPC.
- REST: an OpenAPI-generated client package is an internal convenience only.
  External consumers cannot use the feed, so REST stability comes from
  versioning and the OpenAPI document.
- Feed choice: GitHub Packages vs Azure Artifacts. `[TBA]`

### Orchestration engine

Human's reasoning: where several microservices represent bounded contexts and a
flow needs coordinating, do **not** reinvent reliable transaction flow. The
orchestrator is its own service. **Never ASP.NET and Functions in one service.**
Concern: has never used Durable Functions with Aspire.

| Item | Confidence |
|---|---|
| Orchestrator is a standalone service, never mixed with an ASP.NET service | `[decided]` |
| Do not build our own orchestration engine | `[decided]` |
| Which engine | `[TBA]` — see below |

Rolling our own is **not a live option** — checkpointing, replay, durable timers,
idempotent activity dispatch, and failure recovery are a multi-year build.

Three candidates, all derived from the Durable Task Framework:

| Engine | Fits Aspire | Fits the template | Honours "orchestration in Application" |
|---|---|---|---|
| Durable Functions in a container | Functions integration, newer | ✗ Functions app, not ASP.NET | hard |
| **Durable Task SDK standalone** | ✓ plain ASP.NET Core | ✓ | ✓ |
| Dapr Workflow | ✓ | ✓ | ✓ |

*Recommendation to challenge: **Durable Task SDK standalone.** The deciding
argument is the human's own rule — orchestration logic belongs to Application,
not Presentation. Inside a Functions host the orchestrator functions **are** the
entry points, so Durable Functions violates that structurally rather than by
carelessness. The Durable Task SDK gives the identical orchestrator/activity
model inside a plain ASP.NET Core app.*

*Aspire concern is addressed: Aspire 13.3 added Durable Task Scheduler support
with a local emulator wired into the Aspire dashboard.*

*Honest cost: the Durable Task SDKs use **Durable Task Scheduler** as their
backend — an Azure managed service with parts in preview. New dependency, new
cost line, and preview risk against the PCI-DSS forward-look. Verify current
GA status with `microsoft-docs` before deciding.*

**Clarification the human conflated.** Two different things:

- The engine's internal control and work-item queues are its **storage
  backend**. DAPR has no business there, and this is not a communication
  decision at all.
- The orchestrator's **outbound calls to other services** are ordinary
  service-to-service communication and follow the same DAPR rule as every other
  service. Do not special-case them, or the orchestrator loses the ACA→AKS
  portability seam and the resiliency policies everything else gets.

---

## Tensions the interrogation must resolve

Found while reviewing the position. Each needs an explicit answer, not a
preference.

1. **Two orchestration engines.** Durable Functions is built on the Durable Task
   Framework; Dapr Workflow is the same concept reimplemented for Dapr. Adopting
   both means two state stores, two programming models, two failure modes. See
   *Orchestration engine* above for the three-way decision and its cost.

2. **Three things do service discovery.** Aspire, DAPR service invocation, and
   ACA all provide it. The `[decided]` Aspire scope above resolves most of this
   — Aspire for local dev and wiring, DAPR for runtime invocation — but the
   boundary must be written down, not assumed.

3. **Aspire's deployment model vs the two-layer IaC design.** Aspire deploys to
   ACA via `azd`, generating infrastructure. That collides with PLAN.md §1.9's
   portable workload contract plus thin platform binding, which exists to make
   the AKS port cheap. The `[decided]` scope keeps deployment hand-written —
   confirm that survives contact with the DevOps Architect in Phase 1b.

4. **Building-block mapping must be precise.** A Service Bus *queue* for one-way
   messaging is not DAPR pub/sub — that is a DAPR binding. Name the exact DAPR
   building block for every communication decision.

5. **DAPR may already provide the outbox.** DAPR has a transactional outbox
   feature on state stores. Verify against `microsoft-docs` and the DAPR docs
   whether it covers the write/read sync case before designing one by hand — it
   may collapse most of that work into component configuration.

6. **DAPR and Service Bus sessions.** Verify that ordered delivery via sessions
   is actually supported through the DAPR Service Bus component and under what
   constraints. Do not assume.
