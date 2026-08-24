# NNNN. <Short decision title, imperative>

- **Status:** proposed | accepted | superseded by [NNNN](NNNN-....md)
- **Date:** YYYY-MM-DD
- **Owner:** solution-architect | tech-lead
- **Enforced by:** `<test / analyzer rule / hook / CI step>` — or `nothing — review only`

## Context and Problem Statement

What forces this decision now, and what breaks if we defer it. One paragraph.

## Decision Drivers

- Constraint or quality attribute that actually discriminates between the options
- (Ours, typically: multi-tenant isolation tiers · ACA now, AKS later ·
  Managed Identity only · gRPC internal / REST public · PCI-DSS forward-look ·
  1 repo = 1 service)

## Considered Options

1. **Option A** — one line
2. **Option B** — one line
3. **Option C** — one line

At least two. One option means the decision was assumed, not made.

## Decision Outcome

**Chosen: Option X.**

Because <the driver that decided it>.

### Why not the others

| Option | Rejected because |
|---|---|
| B | |
| C | |

## Consequences

**Good**
-

**Bad / harder now**
-

**Revisit when**
- <the concrete signal that would reopen this>

## Enforcement

How the `Enforced by` line above is actually implemented. Name the file.
If enforcement does not exist yet, note it as a task and link the issue.

## References

- Sources checked (`microsoft-docs`, `azure` MCP, upstream release notes) with dates
