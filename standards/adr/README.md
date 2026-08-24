# Architecture Decision Records

One decision per file. MADR-flavoured. Numbered sequentially, never reused.

- `0000-template.md` — copy this, keep every section.
- Status starts `proposed`. **Only a human moves it to `accepted`.**
- Every ADR carries an `Enforced by:` line. An ADR with no enforcement is a wish.
- Accepted ADRs are never edited in place — they are **superseded** by a new one,
  and both files are cross-linked.

Micro-decisions (naming, syntax, idiom) do **not** belong here. They go as one
row in [`../conventions.md`](../conventions.md). See the `write-adr` skill for
the tier test.

## Planned set — Phase 1

Filled during Phase 1.2 (interrogate → decide). Numbers are reserved now so the
two architects can work in parallel without collisions.

| ADR | Owner | Decides | Status |
|---|---|---|---|
| 0001 | SA | IaC language — Bicep vs Terraform | not started |
| 0002 | SA | ACA topology: environments, ingress, revisions, DAPR component set | not started |
| 0003 | SA | GitHub layout: repo naming ↔ asset, IaC repo schema, App Config / Key Vault provisioning contract | not started |
| 0004 | TL | Solution structure, project layout, Clean Architecture boundaries, slice shape | not started |
| 0005 | TL | NuGet: Azure Artifacts feed, shared-package policy, versioning | not started |
| 0006 | SA | Contract strategy: `.proto` layout, gRPC-internal / REST-public boundary | not started |
| 0007 | SA | Identity: Managed Identity everywhere, Entra ID, service-to-service authN | not started |
| 0008 | SA | Observability: OTel Collector → AppInsights / Log Analytics, tenant-scoped logs | not started |
| 0009 | TL | Error model: `Result<T>` vs exceptions | not started |
| 0010 | TL | Persistence: repository per aggregate vs `DbContext` in handlers | not started |
| 0011 | SA | Multi-tenancy: tier model, isolation axes (compute / logs / storage) | not started |
| 0012 | TL | Validation and the MediatR pipeline behavior set + order | not started |

0013+ are unallocated — the interrogation in Phase 1.2 is expected to add to this
list, and gaps found while building `sb-core-service-template` will add more.

## Open decisions carried from PLAN.md

| Decision | Owner | Becomes |
|---|---|---|
| Does the Authorization service own the tenant/container model, or project it from a separate tenancy service? | SA | new ADR, before CORE |
| Is "Folder" in Organization/Workspace/Folder a hierarchy level or a cross-cutting tag? | SA | new ADR, before CORE |
| Template format for DocGen — XML/XSL-FO vs OOXML vs Razor + typesetting | SA | new ADR, Phase 2 |
