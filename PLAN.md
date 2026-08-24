# SolutionBuilder — Delivery Plan

An agent team that knows its technical domain deeply, tuned until it produces
consistent, production-grade ASP.NET + Azure microservices — first locally in
Claude Code, ultimately in Microsoft Foundry.

`.claude/idea.md` is a **reference example** of the kind of solution we want the
team to deliver. It is not the specification. The team, once tuned, is expected
to improve on it.

---

## 0. Operating model

### 0.1 Where knowledge lives

| Layer | Carries | Rule |
|---|---|---|
| Model weights | Clean Architecture, SOLID, DDD, MediatR, Azure patterns | Already excellent — **never re-teach it** |
| Role prompt (`.claude/agents/*.md`) | Judgment, priorities, refusals, output contract | ~40 lines. Not a description of expertise |
| ADR (`standards/adr/*.md`) | Structural decisions, with alternatives | One decision per file |
| `standards/conventions.md` | Micro-decisions (naming, ctor vs factory) | One line per row, not an ADR each |
| Reference solution (`sb-core-service-template`) | Ground truth | Agents **copy** it, they don't recall it |
| Architecture tests / hooks | Enforcement | A rule that doesn't fail a build is advisory |
| Skill (`SKILL.md`) | The procedure | ~30 lines. A **pointer** to `sb-core-service-template`, never a tutorial |

The differentiator is not the prompts. It is the repo artifacts. Claude already
knows Clean Architecture; it does not know *our* Clean Architecture — the ~50
places where the pattern is genuinely ambiguous and we must fix a choice.

### 0.2 The tuning loop

When an agent produces something wrong, diagnose the layer. Never fix by
appending prose to `CLAUDE.md`.

| Symptom | Broken layer | Fix |
|---|---|---|
| Chose differently than we wanted, but defensibly | Missing decision | Write an **ADR** or a `conventions.md` row |
| Violated a decision it could have read | Missing enforcement | Write an **architecture test or hook** |
| Right output, wrong order, missed a step | Skill | Fix the procedure |
| Wrong priorities, over-engineered, didn't push back | Role prompt | Fix the judgment section |
| Genuinely didn't know a fact | Weights | Add an MCP source — not prose |

### 0.3 Two constraints that shape every phase

1. **Subagents cannot converse.** Any "X approves with Y" flow in this plan is
   orchestrated: an artifact is produced, a reviewing agent reads it and emits a
   verdict file. The main thread relays. Design handoffs as **artifacts**, never
   as conversations.
2. **Distribution is a plugin.** `1 repo = 1 service` means a copy-pasted
   `.claude/` folder diverges within a quarter. The team ships as our own
   private plugin from our own marketplace; service repos carry a 10-line
   `settings.json`.

### 0.4 Repository layout

`1 repo = 1 service` holds. The governance repo carries **no .NET services**.

| Repo | Contains | .NET |
|---|---|---|
| `sb-foundry` | agents, skills, hooks, ADRs, conventions | **none** |
| `sb-core-service-template` | the golden reference service — **GitHub template repo** | yes |
| `sb-core-service-authorization`, `sb-core-service-orchestrator`, … | CORE and business services, created from the template | yes |
| `sb-iaac` | deployment variables, App Config / Key Vault provisioning contract | no |

```
sb-foundry/                          <- this repo, the governance repo
  PLAN.md
  .claude-plugin/marketplace.json    <- our private marketplace
  plugins/
    sb-team/                         <- agents, skills, hooks, MCP config
      agents/  skills/  hooks/
  standards/
    adr/NNNN-*.md                    <- structural decisions (MADR)
    conventions.md                   <- micro-decisions table
    tech-radar.md                    <- Phase 4
  requirements/                      <- Phase 2
  docs/
```

**Why `sb-core-service-template` is its own repo.** A reference exists to be trustworthy
ground truth. Living in a subfolder here, its CI, its deploy, and its repo shape
would all be artificial — and ADR 0003 is precisely about repo-level decisions:
branch protection, Actions permissions, secret provisioning, naming, asset
tagging, environment bindings. **A reference that is not a real service repo
cannot validate the repo-level decisions it is supposed to encode.** It also
makes `gh repo create <name> --template sb-core-service-template` the scaffolding
mechanism, which only works on a repo.

`sb-core-service-template` is **not** a CORE service. It is one entity, one slice, one gRPC
method, one REST endpoint — small enough to stay copyable. CORE services are
built *from* it.

**How agents reach it.** The reference is a *seed*, not a runtime dependency:

- **New service** → created from the template; the patterns arrive in the repo.
- **New slice in an existing service** → copy the sibling slice already in that
  repo. Never leaves the repo.
- **"What is canonical?"** → ADRs and `sb-core-service-template` via the `github` MCP.
  No local clone, always current.

Skills therefore carry *pointers* (repo + path), never copies. Nothing to sync.

### 0.5 The architecture tests

`SB.Core.ServiceTemplate.ArchitectureTests` lives **inside
`sb-core-service-template`**, not in the governance repo. `gh repo create
--template` copies it into every scaffolded service, so enforcement arrives with
the service and costs no feed, no package, and no publish pipeline.

The `sb-foundry` repo therefore carries **no .NET at all**.

**What we give up.** An ADR and its enforcement now live in different repos, so
they cannot change in one commit, and a rule change does not reach services
already created. That is the exact problem a shared package solves — and at one
or two services it is not yet a problem. Compensating discipline until then:

- Every ADR's `Enforced by:` line names the **test method**, not a description.
- The ADR is not `accepted` until that test exists and is green.

**Extraction trigger:** when the third service repo is created, or the first
time an arch-test change has to be hand-copied between repos — whichever comes
first — extract to `SB.Core.ArchitectureTests` on GitHub Packages, referenced by
every service. Do not do it earlier; do not put it off past that point.

---

## Phase 1 — CORE

**Goal:** a Tech Lead and Solution Architect that know our stack, a reference
solution that is ground truth, and enough enforcement that consistency survives
context loss.

**Team:** Solution Architect (agent), Tech Lead (agent), Developer (main thread).

### 1.1 Create the agents

Two files, ~40 lines each. Judgment-heavy, expertise-light.

- `solution-architect.md` — Azure, microservice patterns, DAPR, multi-tenancy,
  service boundaries, contracts, identity, observability. Opus. Read + web +
  `microsoft-docs` + `azure` MCP + write to `standards/` and `docs/`.
- `tech-lead.md` — Clean Architecture, SOLID, DDD, vertical slices, MediatR,
  EF Core, testability. Opus. Read + Bash + LSP.
- Developer stays the **main thread** — that is where LSP diagnostics,
  checkpoints, and the human live. Subagents return text, not reviewable diffs.

A role prompt earns its place through **priority order when principles
conflict**, **explicit refusals**, and an **output contract**. "You are an expert
in X" is near-noise.

Install before first use — an SA interrogated without first-party Azure docs
answers from memory, and Azure moves faster than any model cutoff:

- `csharp-lsp@claude-plugins-official` (needs `csharp-ls` on PATH)
- `microsoft-docs@claude-plugins-official`
- `azure@claude-plugins-official`
- `github@claude-plugins-official`
- `plugin-dev@claude-plugins-official`
- `claude-security@claude-plugins-official` — **now, not Phase 3.** See 1.7.

### 1.2 Interrogate, then decide

Extraction prompt for each architect:

> List every decision point where &lt;domain&gt; is genuinely ambiguous — where
> two competent teams would diverge. For each: the options, the tradeoff, your
> recommendation for our constraints, and the cost of getting it wrong later.

Expect ~30 from each. **Read them properly** — this is the step where the team
surfaces what we didn't know to ask.

Then decide. Two tiers, because 50 ADRs is unusable:

**Tier 1 — ADRs.** Structural, expensive to reverse, ~20 files.

| ADR | Owner | Decides |
|---|---|---|
| 0001 | SA/DevOps | IaC language — Bicep vs Terraform *(gates whether `terraform@claude-plugins-official` joins the bundle)* |
| 0002 | SA | ACA topology: environments, ingress, revisions, DAPR component set |
| 0003 | SA | GitHub layout: repo naming ↔ asset, IaC repo schema, App Config / Key Vault provisioning contract |
| 0004 | TL | Solution structure, project layout, Clean Arch boundaries, slice shape |
| 0005 | TL | NuGet: Azure Artifacts feed, shared-package policy, versioning |
| 0006 | SA | Contract strategy: `.proto` layout, gRPC-internal / REST-public boundary |
| 0007 | SA | Identity: Managed Identity everywhere, Entra ID, service-to-service authN |
| 0008 | SA | Observability: OTel Collector → AppInsights / Log Analytics, tenant-scoped logs |
| 0009 | TL | Error model: `Result<T>` vs exceptions |
| 0010 | TL | Persistence: repository per aggregate vs `DbContext` in handlers |
| 0011 | SA | Multi-tenancy: tier model, isolation axes (compute / logs / storage) |
| 0012 | TL | Validation and the MediatR pipeline behavior set + order |

Every ADR carries an `Enforced by:` line naming the test, analyzer, or hook.
An ADR with no enforcement is a wish.

**Tier 2 — `conventions.md`.** Micro-decisions. One row each, no ceremony.
Starter checklist to work through:

*Domain* — constructors vs static `Create()` factories · private setters vs
init-only · guard clauses throw vs `Result` · value objects as `record` /
`record struct` / class · domain events raised where, dispatched where ·
aggregate root marker interface vs base class · strongly-typed IDs yes/no ·
`Guid` v7 vs ULID vs int · enums vs smart enums.

*Application* — one folder per use case, naming (`CreateOrder` vs `Create`) ·
`Command` / `Query` / `Handler` suffixes · DTO location and suffix
(`Response` / `Dto`) · mapping manual vs Mapster · transaction boundary
(UoW behavior vs explicit) · where handler registration lives.

*Infrastructure / Api* — Minimal API vs controllers · endpoint registration
per-slice static class · EF fluent configuration one file per entity ·
gRPC service implementation location · `ProblemDetails` shape · API versioning.

*Cross-cutting* — file-scoped namespaces · nullable enabled · implicit usings ·
`Async` suffix policy · `sealed` by default · test naming convention ·
folder-equals-namespace · project naming `SB.{Asset}.{Name}.{Layer}`.

Enforce every row that an analyzer or `.editorconfig` can enforce. The rest go
in `conventions.md` and become architecture tests where feasible.

### 1.3 Build the reference from the ADRs

The Tech Lead implements `sb-core-service-template` from the decisions — not from memory.
This round-trip is most of the value: writing the code exposes which ADRs were
under-specified, because the code can't be written without guessing. Those gaps
come back as new ADRs or convention rows.

Create `sb-core-service-template` as an empty repo first, then build inside it. Order:
`src/` → `contracts/` → `tests/` → `iac/` last (blocked on ADR 0001, so
everything else proceeds in parallel with that decision).

Mark it a **GitHub template repository** once the convergence drill passes —
not before, or someone scaffolds a service from an unproven template.

### 1.4 The convergence drill — Phase 1 acceptance test

Build the **same** CRUD slice three times. This is how we prove alignment lives
in the artifacts and not in the conversation.

| Run | When | Expect | Means |
|---|---|---|---|
| 1 | From ADRs alone, before `sb-core-service-template` exists | Divergence | Each gap = a missing ADR or convention row |
| 2 | After `sb-core-service-template` exists, same session | Matches reference | Reference is copyable |
| 3 | **Fresh session, zero context**, in a repo scaffolded from the template | Matches run 2 | ✅ Knowledge is in the artifacts, not the chat |

Run 3 is the gate. If a fresh-context agent diverges, the standard is
incomplete — loop back to 1.2. Do not proceed to CORE until run 3 is clean.

### 1.5 Enforcement layer

- `NetArchTest` in `tests/Architecture` — one test per structural ADR.
  *`Domain` may not reference `Microsoft.EntityFrameworkCore`* is a failing
  build, not a paragraph.
- Roslyn analyzers + `.editorconfig` for conventions.
- Hooks in the plugin: `PostToolUse` on Edit/Write → `dotnet format`;
  `Stop` → build + architecture tests must be green.
- IaC policy test: fail on any connection string carrying a secret
  (Managed Identity only, ADR 0007).

### 1.6 Write the skills — last, and small

Only after the reference exists and the drill passes. A skill that **points at**
`sb-core-service-template` cannot drift; a skill that **describes** it drifts the day the code
changes. Never generate skills from the reference.

`dotnet-slice` · `dotnet-service-scaffold` · `write-adr` · `contract-change`.

### 1.7 Shift-left correction

Security and QA *roles* arrive in Phase 3, but their **tooling arrives now** —
otherwise Phase 3 becomes a retrofit of CORE.

| Now (Phase 1, cheap) | Phase 3 (deliberate) |
|---|---|
| `claude-security` plugin scanning every change | Security Architect: Azure Policy, Defender, tool strategy |
| `/security-review` on PRs | Threat model, PCI-DSS forward-look |
| Testcontainers integration tests in `sb-core-service-template` | QA: test strategy, coverage policy, perf baselines |
| Architecture tests | Formal NFR-driven test plan |

### 1.8 CORE release scope

Built from `sb-core-service-template`, once the drill passes.

**Platform:** ACA environment · DAPR components (pubsub, state, secrets,
bindings) · Front Door + API Gateway · App Config + Key Vault · Redis · Blob ·
Cosmos · OTel → AppInsights · asset tagging · GH Actions per repo.

**Core services:**

- **Authorization service** — the tenancy substrate. Organization / Workspace / Folder
  containers, user assign/unassign, tenant tiering hooks for the future
  dedicated compute / logs / storage split.
- **Orchestrator service** — Durable Functions in Docker, generic state-flow
  engine with no business states baked in.

**Risk, stated plainly:** generic infrastructure built with zero consumers gets
built wrong. **Mitigation:** the SA writes the DocGen *consumption contract* as
a paper design during CORE — which containers, which permissions, which
orchestration states, which storage moves. Implement none of it. CORE must
simply satisfy that document. Costs a day, keeps the abstractions honest, and is
a genuine dry run of the Phase 2 pipeline.

### 1.9 ACA now, AKS later — make the port cheap

- All service-to-service via **DAPR service invocation**. No hostnames in app
  code. DAPR is the portability seam.
- All config from **App Configuration + Key Vault**, never ACA revision env
  vars. Env vars are the thing that doesn't port.
- Split IaC into a portable **workload contract** (image, scale rules, DAPR
  components, identity, secret refs) and a thin **platform binding** (ACA today,
  AKS Helm later). Same input, two renderers — AKS becomes an additive PR.
- Keep an **AKS conformance checklist** ADR that CI can't verify yet. Written
  down while the context is fresh.

### 1.10 Phase 1 done when

Split, because Azure is not yet provisioned and must not gate the code work.

### Phase 1a — GitHub + .NET, no Azure

Owners: Tech Lead, Solution Architect (code-side decisions only).

- [x] Own marketplace + plugin installs into a scratch repo
- [x] Vendor plugins installed, `csharp-ls` on PATH, GitHub authenticated
- [ ] `sb-foundry`, `sb-core-service-template`, `sb-iaac` repos created
- [ ] TL interrogation → ADRs 0004, 0005, 0009, 0010, 0012 + `conventions.md`
- [ ] `SB.Core.ServiceTemplate` compiles on `net10.0`: Domain, Host, and whatever
      else ADR 0004 decides
- [ ] Reference CRUD slice + `.proto` + OpenAPI
- [ ] `SB.Core.ServiceTemplate.ArchitectureTests` green, one test per structural ADR
- [ ] Testcontainers integration test for the slice
- [ ] CI on push: build + test + arch tests
- [ ] **Convergence drill run 3 passes** ← the real gate, and it needs no Azure
- [ ] `sb-core-service-template` marked a GitHub template repository

### Phase 1b — Azure, IaC, pipelines

Owner: DevOps Architect (introduced here). Blocked on a subscription.

- [ ] Azure subscription decided and provisioned
- [ ] Capability check: ACA + DAPR, Cosmos, Redis, Blob, App Config, Key Vault,
      Front Door available and within quota
- [ ] ADRs 0001, 0002, 0003, 0007, 0008, 0011
- [ ] `iac/` in the template — workload contract + ACA binding (§1.9)
- [ ] Nightly ACA deploy green
- [ ] CORE release (§1.8)
- [ ] CORE deployed multi-tenant on ACA
- [ ] `claude-security` clean on CORE

---

## Phase 2 — Business Analyst and the RFP process

**Goal:** raw feature text → an agreed solution proposal → PBIs, repeatably.
Test bed: `idea.md` Part 1 (template-driven document generation).

**Adds:** Business Analyst (agent). Product Owner is played by the human.

### 2.1 The pipeline

```
requirements/NNN-feature/request.md      <- pasted raw text
   |
   +- BA  -> interview loop (2.2) -> answers.md
   |         functional-requirements.md  (numbered, testable)
   |         acceptance-criteria.feature (Gherkin)
   |         open-questions.md           <- HARD GATE
   |
   +- SA  -> nfr.md · asf.md · service-decomposition.md
   |         contracts/*.proto + openapi.yaml   <- generated
   |         cost-model.md (2.4) · monitoring-plan.md (2.5)
   |         adr-delta.md · solution-plan.yaml  <- schema-validated
   |
   +- HUMAN (PO) -> solution-proposal.md signed (2.6)
   |
   +- PBI generation (2.7)
   |
   +- TL / QA fan out from solution-plan.yaml
```

Two rules make this repeatable:

- **`solution-plan.yaml` is the contract, not prose.** Services, owned data,
  DAPR building blocks, endpoints, ACA/AKS flag, tenancy tier, asset assignment.
  Schema-validated, so a malformed plan fails before code is written.
- **Contract-first.** SA emits `.proto` and OpenAPI *before* implementation.
  The only reliable way to keep parallel service work from diverging.

### 2.2 The PO question channel

The BA must not read chat — chat doesn't port to headless. **The file is the
record; the terminal is the fast path.**

```
loop until open-questions.md is empty:
  BA drafts from request.md + answers.md
  BA emits up to 5 open questions
    - discrete choice  -> AskUserQuestion (structured, in-terminal, fast)
    - open-ended       -> appended to open-questions.md, PO edits in VS Code
  every answer is written back into answers.md, with its question
  BA re-reads and revises
```

`answers.md` is the durable elicitation record. Because it is a file, the same
BA runs unattended in Phase 5 with a different front end — GitHub Issue
comments, Teams, a Foundry thread — and the pipeline is unchanged. Only the
adapter changes.

**Tune the BA to maximise surfaced ambiguity, then stop.** A BA that guesses at
ambiguity is worse than useless. A non-empty `open-questions.md` blocks the SA.

### 2.3 BA ↔ SA

Not a conversation (see 0.3). A two-pass artifact loop the main thread runs:
BA drafts FRs → SA reads and returns NFR-driven questions as a file → BA
revises → SA designs. Encode as a Workflow script once the shape is stable.

### 2.4 Costs

**LLM-estimated Azure pricing is unreliable.** Ground it:

- SA derives SKU choices per service per tenant tier from `solution-plan.yaml`.
- Prices come from the **Azure Retail Prices API**
  (`https://prices.azure.com/api/retail/prices`, public, no auth) — a script,
  not a guess.
- Output `cost-model.md`: per-tenant-tier monthly estimate, the scaling driver
  behind each line, and the three assumptions most likely to be wrong.
- Cross-check against the `azure` MCP for what actually exists in the
  subscription.

### 2.5 Monitoring

SA produces `monitoring-plan.md` from the NFRs: SLIs and SLOs per service, OTel
spans and attributes (tenant-scoped, ADR 0008), AppInsights dashboards, alert
rules and thresholds, log retention per tenant tier. Every NFR with a number
must map to a metric that exists — otherwise it isn't an NFR, it's a wish.

### 2.6 Customer agreement

`solution-proposal.md` — generated, not hand-written. Contains: scope, FRs,
NFRs, architecture summary, cost model, monitoring plan, **assumptions**,
**explicitly out of scope**, and open risks. This is the signed artifact.
Publishing it as a shareable Artifact page is the natural delivery format for a
real customer.

### 2.7 PBI generation

**Recommendation: GitHub Issues.** We already run GitHub as the backbone —
1 repo per service, IaC repo, Actions — the `github` plugin is validated vendor,
PBIs live next to code, and sub-issues give hierarchy. Reconsider Azure DevOps
Boards or Jira (`atlassian` plugin) only if real portfolio management is needed.

Generate from `solution-plan.yaml`, not from prose, so every PBI traces to an FR
and an acceptance criterion. One epic per service, one issue per slice,
acceptance criteria copied verbatim from the `.feature` file.

### 2.8 Phase 2 done when

- [ ] `idea.md` Part 1 runs end to end: raw text → signed proposal → PBIs
- [ ] The SA independently challenges the "templates will be XML" assumption and
      lands a reasoned ADR *(the acceptance test for whether the SA is genuinely
      tuned — a tuned SA compares XSL-FO vs OOXML + OpenXML SDK vs
      Razor/Handlebars + a typesetting pipeline, scored against our four output
      formats and multi-tenancy)*
- [ ] Cost model is API-grounded, not estimated
- [ ] `answers.md` is complete enough that a fresh BA reproduces the FRs

---

## Phase 3 — QA and Security Architect

**Adds:** Security Architect (agent), QA (agent, `isolation: worktree`).

### 3.1 Responsibilities

| Role | Owns | Approves via |
|---|---|---|
| **SecA** | Tool selection, Azure Policy set, threat model, PCI-DSS forward-look, secret and identity policy | `security-plan.md` reviewed by SA (architecture fit) and BA (business impact) — artifact + verdict files, not a meeting |
| **SA** | NFRs with numbers, the test plan those NFRs imply, tool choices | `test-strategy.md` |
| **QA** | Executing all of it: integration, performance, security, regression | Test suites in-repo, gates in CI |

### 3.2 Test layers

| Layer | Tooling | Gate |
|---|---|---|
| Architecture | NetArchTest | PR blocking (already Phase 1) |
| Unit | xUnit | PR blocking |
| Integration | Testcontainers — Cosmos, Redis, Azurite, DAPR sidecar | PR blocking |
| Contract | `buf breaking` on `.proto`, OpenAPI diff | PR blocking |
| E2E | `playwright@claude-plugins-official` (Microsoft) | Nightly |
| Performance | k6 or NBomber; Azure Load Testing for scale | Nightly, against SLOs from 2.5 |
| SAST | `claude-security` + CodeQL | PR blocking |
| Dependencies | Dependabot + Trivy on images | PR blocking |
| Cloud posture | Defender for Cloud, Azure Policy, Microsoft Security DevOps action | Continuous |

### 3.3 Contract Steward

Introduce here if not earlier. With 1 repo = 1 service and internal gRPC, a
silently broken `.proto` is the most likely production incident, and it is fully
mechanizable. Agent + hook, `buf breaking` in CI.

### 3.4 Phase 3 done when

- [ ] Every numbered NFR has a test that measures it
- [ ] Security gates block merges; Azure Policy denies at deploy
- [ ] `security-plan.md` approved by SA and BA
- [ ] Perf baselines recorded per tenant tier

---

## Phase 4 — Continuous upgrade

**Goal:** the standard tracks upstream instead of decaying into tech debt.

**Adds:** Standards Curator (scheduled routine, Sonnet).

```
weekly scheduled routine
  -> Curator agent
  -> microsoft-docs MCP + WebSearch: .NET, ASP.NET, Azure, DAPR, ACA/AKS,
     Cosmos, GitHub Actions release notes
  -> diff against standards/adr/ + conventions.md + sb-core-service-template
  -> opens a PR proposing changes           <- never a direct write
  -> SA / TL agent reviews -> human merges
```

**Hard rule: external information enters as a proposal, never as an edit.**
Otherwise the standard drifts silently and we have built a hallucination
amplifier on a cron trigger.

Also in scope: Dependabot and .NET/Azure version drift; deprecation warnings;
`standards/tech-radar.md` (adopt / trial / assess / hold) as the durable record
of what we looked at and rejected, so the same debate isn't reopened quarterly.

**Done when:** at least one upstream-driven ADR has been proposed, reviewed, and
merged without a human noticing the change first.

---

## Phase 5 — Microsoft Foundry

Claude models went **GA in Microsoft Foundry on 2026-06-29**, Azure-hosted or
Anthropic-hosted, under Azure auth, billing, and governance, MACC-eligible. The
destination exists and the model family does not change.

### 5.1 What ports

| Where logic lives | Ports? |
|---|---|
| Skills (`SKILL.md` + scripts) | ✅ Agent SDK reads the same `.claude/` |
| Agent definitions (markdown) | ✅ |
| **MCP servers** | ✅ Claude Code *and* Foundry Agent Service both speak MCP |
| Hooks | ❌ Claude Code only — rebuild as pipeline steps |
| Slash commands | ❌ UX affordance, not portable logic |

**Therefore, from Phase 3 onward, wrap every capability that isn't plain file
editing as an MCP server** — standards lookup, ADR store, `solution-plan.yaml`
validator, contract linter, cost calculator. MCP is the portability layer.
Design for it before it is needed.

### 5.2 Ladder

1. **Local** — Claude Code + VS Code, human in the loop (Phases 1–4)
2. **Headless CI** — Agent SDK in GitHub Actions, same `.claude/` folder.
   BA/SA run on issue-open, QA on PR. Human still approves.
   *(`agent-sdk-dev@claude-plugins-official`)*
3. **Containerized** — Agent SDK service on ACA, MCP servers as sidecars,
   Managed Identity, OTel → AppInsights. The team dogfoods our own stack.
4. **Foundry Agent Service** — Claude via Foundry, our MCP servers registered as
   tools, Entra ID governance. Skills and agent definitions carry over; hooks
   become pipeline stages.

---

## Validated-vendor plugins by phase

All from `claude-plugins-official` (Anthropic-curated: first-party or vetted
partner). Install with `/plugin install <name>@claude-plugins-official`.

| Phase | Plugin | Vendor | Why |
|---|---|---|---|
| 1 | `csharp-lsp` | Anthropic | Highest ROI item. Real compiler diagnostics after every edit. Needs `csharp-ls` on PATH |
| 1 | `microsoft-docs` | Microsoft | MS Learn MCP — grounds Azure/.NET in first-party docs, not memory |
| 1 | `azure` | Microsoft | Azure MCP + Azure skills. Live resource introspection |
| 1 | `github` | GitHub | Repos, PRs, issues across the estate |
| 1 | `plugin-dev` | Anthropic | Authoring our own team plugin |
| 1 | `claude-security` | Anthropic | Shift-left scanning from day one |
| 1 | `commit-commands`, `code-review` | Anthropic | Commit / PR workflow, multi-agent review |
| 1* | `terraform` | HashiCorp | **Only if ADR 0001 chooses Terraform.** Bicep has no vendor plugin |
| 3 | `playwright` | Microsoft | E2E for both portals |
| 5 | `mcp-server-dev`, `agent-sdk-dev` | Anthropic | MCP-ification and the Foundry bridge |
| — | `atlassian` / `linear` | Vendor | Only if PBIs move off GitHub Issues |

**No validated vendor exists — we author these ourselves.** This is the real
content of the standard: DAPR building blocks and component YAML · Bicep ·
AKS/ACA parity · Cosmos modeling (partition keys, RU budgeting) ·
Clean Architecture / DDD / MediatR / vertical slices in .NET · NuGet +
Azure Artifacts · multi-tenancy tiering · OTel Collector → AppInsights ·
Durable Functions in Docker · Managed-Identity-only enforcement.

The `azure` plugin gives *runtime access* to Azure. It does not give us *our
authoring opinions*. That gap is what the team is for.

---

## Open decisions

| # | Decision | Owner | When |
|---|---|---|---|
| 1 | Bicep vs Terraform | DevOps/SA agent → human | ADR 0001, Phase 1 |
| 2 | Does the Authorization service own the tenant/container model, or project it from a separate tenancy service? *(drives the dedicated-storage tier)* | SA | Phase 1, before CORE |
| 3 | Is "Folder" in Organization/Workspace/Folder a hierarchy level or a cross-cutting tag? | SA | Phase 1, before CORE |
| 4 | PBI target — GitHub Issues (recommended) vs Azure DevOps vs Jira | Human | Phase 2 |
| 5 | Template format for DocGen — XML/XSL-FO vs OOXML vs Razor + typesetting | SA, as an ADR | Phase 2 |
| 6 | k6 vs NBomber for performance | QA / SA | Phase 3 |

---

## Risk register

| Risk | Mitigation |
|---|---|
| Building agents before a reference exists | Phase 1 order: interrogate → decide → build → drill |
| Prose standards with no enforcement | Every ADR carries `Enforced by:` |
| Standard drift across 40 repos | Plugin distribution, not copied `.claude/` folders |
| Curator writing directly to standards | PR-only, human merge |
| CORE abstractions built with no consumer | DocGen paper consumption contract (1.8) |
| Security / QA retrofitted in Phase 3 | Tooling shifts left to Phase 1 (1.7) |
| Agents designed to "talk to each other" | Artifact + verdict files, orchestrated by the main thread |
| ACA assumptions leaking into app code | DAPR seam, App Config, two-layer IaC (1.9) |
| Logic locked into hooks, unportable to Foundry | MCP-first from Phase 3 (5.1) |

---

**Phases 1 and 2 are the whole game.** Skip the convergence drill and this
becomes a very expensive way to generate inconsistent .NET.
