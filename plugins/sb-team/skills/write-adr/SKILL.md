---
name: write-adr
description: Record an architecture or engineering decision. Use when a decision
  is being made about structure, technology, boundaries, patterns, or conventions —
  or when an agent proposes something the existing standards do not cover. Also use
  when asked to write, update, supersede, or review an ADR.
---

# Record a decision

Two tiers. Pick the tier first — most decisions are Tier 2, and writing them as
ADRs makes the standard unreadable.

| | Tier 1 — ADR | Tier 2 — convention |
|---|---|---|
| Scope | Structure, boundaries, technology, contracts | Naming, syntax, file placement, idiom |
| Reversal cost | Weeks. Touches many services | Minutes. A rename or a formatter run |
| Examples | `Result<T>` vs exceptions · Bicep vs Terraform · repository vs `DbContext` | `Async` suffix · `sealed` by default · ctor vs static `Create()` |
| Goes in | `standards/adr/NNNN-kebab-title.md` | One row in `standards/conventions.md` |
| Target count | ~20 total | Unlimited |

If unsure: does a *new service* have to re-make this decision from scratch?
Yes → Tier 1. No → Tier 2.

## Tier 1 procedure

1. Take the next free number from `standards/adr/`. Never reuse one.
2. Copy `standards/adr/0000-template.md`. Keep every section.
3. **Considered Options needs at least two real options.** One option means the
   decision was not made, it was assumed. If you can only find one, say so
   explicitly under Decision Drivers and mark the ADR `proposed`.
4. Ground technology claims in `microsoft-docs` or the `azure` MCP, not memory.
   Azure moves faster than any model cutoff. Cite what you checked.
5. Fill **Enforced by** with a specific test, analyzer rule, hook, or CI step —
   named, not described. If nothing can enforce it, write
   `Enforced by: nothing — review only` and treat that as a known weakness.
   An ADR with no enforcement is a wish.
6. Status starts `proposed`. **Only a human moves it to `accepted`.**
7. Cross-link: add `Supersedes` / `Superseded by` to both files when replacing an
   earlier decision. Never delete or edit an accepted ADR's decision — supersede it.

## Tier 2 procedure

Append one row to `standards/conventions.md`. Set Status to `Decided` and fill
Enforcement. If it cannot be enforced by `.editorconfig`, an analyzer, or an
architecture test, prefer the option that can be.

## When implementation contradicts a decision

Do not silently deviate. Either the code is wrong, or the ADR is
under-specified — and the second is common while `sb-core-service-template` is being built.
Stop, state which you believe it is, and propose either a fix or a new ADR.

## Review checklist

- [ ] Correct tier
- [ ] Two or more genuine options, with the tradeoff stated
- [ ] Decision Drivers reference our actual constraints (multi-tenant tiers,
      ACA-now/AKS-later, Managed Identity only, PCI-DSS forward-look)
- [ ] `Enforced by` names something specific
- [ ] Status is `proposed` — a human accepts it
- [ ] Consequences include what this makes *harder*, not only what it improves
