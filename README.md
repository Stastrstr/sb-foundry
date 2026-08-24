# SolutionBuilder

Governance repo for an agent team that delivers ASP.NET + Azure microservices —
first locally in Claude Code, ultimately in Microsoft Foundry.

Start with **[PLAN.md](PLAN.md)**.

## Layout

| Path | What | Phase |
|---|---|---|
| [`PLAN.md`](PLAN.md) | The delivery plan. The source of truth for sequencing | — |
| [`.claude/idea.md`](.claude/idea.md) | The original brief. A *reference example*, not a spec | — |
| [`.claude-plugin/marketplace.json`](.claude-plugin/marketplace.json) | Our private marketplace catalog | 0 ✅ |
| [`plugins/sb-team/`](plugins/sb-team/) | The team: agents, skills, hooks | 0 ✅ |
| [`standards/adr/`](standards/adr/) | Architecture decisions. One per file | 1 |
| [`standards/conventions.md`](standards/conventions.md) | Micro-decisions. One row each | 1 |
| [`requirements/`](requirements/) | BA/SA intake and outputs, one folder per feature | 2 |
| [`docs/`](docs/) | Setup and operational docs | 0 ✅ |

## Repos

`1 repo = 1 service`. This repo carries **no .NET services**.

| Repo | Contains |
|---|---|
| `sb-foundry` (this) | agents, skills, hooks, ADRs, conventions — no .NET |
| `sb-core-service-template` | the golden reference service — GitHub template repo |
| `sb-{asset}-{name}` | CORE and business services, scaffolded from the template |
| `sb-iaac` | deployment variables and provisioning contract |

`sb-core-service-template` is a separate repo because a reference that isn't a real service
repo cannot validate the repo-level decisions it's meant to encode — branch
protection, Actions permissions, secret provisioning, tagging. See
[PLAN.md §0.4](PLAN.md).

Architecture tests live in `sb-core-service-template` and travel with the
template. See [PLAN.md §0.5](PLAN.md) for the extraction trigger.

## Why the team ships as a plugin

`1 repo = 1 service` means a copy-pasted `.claude/` folder diverges within a
quarter. The team is versioned here and installed everywhere from this
marketplace. Service repos carry only a short `.claude/settings.json` — see
[`docs/service-repo-setup.md`](docs/service-repo-setup.md).

## Where knowledge lives

Claude already knows Clean Architecture. It does not know *our* Clean
Architecture. The layers, in order of leverage:

| Layer | Rule |
|---|---|
| Model weights | Never re-teach the theory |
| Role prompt | Judgment, priorities, refusals — not a description of expertise |
| ADR / conventions | The decisions where the pattern is genuinely ambiguous |
| `sb-core-service-template` repo | Ground truth. Agents **copy** it, they don't recall it |
| Architecture tests, hooks | Enforcement. A rule that doesn't fail a build is advisory |
| Skill | A **pointer** to `sb-core-service-template`, ~30 lines, never a tutorial |

When an agent gets something wrong, fix the layer that failed — see the tuning
loop in [PLAN.md §0.2](PLAN.md). Never fix it by appending prose to `CLAUDE.md`.

## Working on the plugin

The marketplace is local until this repo is pushed:

```
/plugin marketplace add d:/Study/ClaudeCode/AI-Foundry
/plugin install sb-team@solutionbuilder
```

After changing anything under `plugins/`:

```
/plugin marketplace update solutionbuilder
/reload-plugins
```

Validate the manifests before committing:

```
claude plugin validate ./plugins/sb-team
claude plugin validate .
```
