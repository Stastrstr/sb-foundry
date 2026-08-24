---
name: interrogate
description: Extract the decision space from a specialist agent before writing any
  standards. Use at the start of a phase, when onboarding a new role (tech-lead,
  solution-architect, devops-architect, qa, security-architect), or whenever you
  need to know what is genuinely ambiguous in a domain before deciding anything.
---

# Interrogate a specialist

The point is **not** to get recommendations. It is to get the *decision space* —
every place the domain is genuinely ambiguous and two competent teams diverge.
The human decides; the agent's job is to make sure nothing is decided by
accident.

## Procedure

1. Confirm the agent has its grounding tools before starting. An architect
   interrogated without `microsoft-docs` answers from memory, and Azure moves
   faster than any training cutoff.
2. Point it at `standards/inputs/human-position.md` and any existing ADRs.
3. Run the extraction prompt below.
4. Do **not** decide during the interrogation. Collect first, decide after.
   Deciding as you go anchors every later answer to the earlier ones.
5. Feed the output into the human's decision pass, then `write-adr`.

## The extraction prompt

> List every decision point in <domain> where the answer is genuinely ambiguous
> — where two competent teams would diverge, not where there is a settled best
> practice. For each, give:
>
> - **Decision** — one line
> - **Options** — at least two, real ones actually in use
> - **Tradeoff** — what each option buys and costs
> - **Recommendation** — yours, with the reason, marked as a recommendation
> - **Cost of getting it wrong** — and how expensive it is to reverse later
> - **Tier** — ADR (structural, expensive to reverse) or convention (naming,
>   syntax, idiom)
> - **Enforceable by** — the test, analyzer, or CI gate that could check it,
>   or "nothing" if none exists
>
> Then, separately: go through `standards/inputs/human-position.md` item by item.
> For each, say whether you agree and **why** — agreement needs its own
> reasoning, not a nod. Where you disagree, say what you would do instead and
> what it costs.
>
> Do not explain the domain to me. Do not restate patterns I can look up. I want
> the places where judgement is actually required.

## What a good result looks like

- **~25–35 decision points.** Fewer than 15 means it collapsed ambiguity into
  convention and you will discover the missing decisions later, as drift.
- Most are **conventions**, a minority are ADRs. If everything is an ADR, the
  tier test was not applied.
- Recommendations are **marked as recommendations**, not stated as facts.
- It **disagrees with `human-position.md` somewhere.** Total agreement means it
  is anchoring on the human rather than thinking, and the result is worthless.
- Version-specific claims **cite a source with a date**.

## Failure modes, and what each means

| Symptom | Diagnosis | Fix |
|---|---|---|
| 8 confident recommendations instead of 30 ambiguities | Prompt's "surface ambiguity" section is too quiet | Strengthen the role prompt |
| Generic, blog-shaped advice | Weights are not enough here after all | Add domain priming or better grounding tools |
| Agrees with every human position | Anchoring, not reasoning | Re-run with the position file withheld, compare |
| Every item is an ADR | Tier test not applied | Point it at the `write-adr` skill first |

The first two are the bet this whole approach rests on. If either shows up, the
role prompt is wrong — fix the prompt, not the output.

## After

The output is **input to a human decision pass**, not a standard. Nothing is
accepted because an agent recommended it. Run `write-adr` for each Tier 1 item
the human decides; append conventions rows for the rest.
