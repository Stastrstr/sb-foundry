# Wiring a service repo to the SolutionBuilder team

Every service repo carries the same short `.claude/settings.json`. The team is
versioned centrally here; service repos never hold their own copy of agents,
skills, or standards.

## 1. Copy the settings template

Copy [`service-repo-settings.json`](service-repo-settings.json) to
`.claude/settings.json` in the service repo. It already points at the real
governance repo, `Stastrstr/sb-foundry`.

## 2. Install the plugins

**`enabledPlugins` alone does not install anything.** From Claude Code v2.1.195
onward, a plugin whose source is external — a GitHub repo, an npm package —
is *enabled* by project settings but is not *installed* until each developer
installs it. Claude Code reports it as not installed and prints the command.

Each developer runs once per machine:

```
claude plugin install sb-team@solutionbuilder
```

The `claude-plugins-official` marketplace is supposed to be added automatically
on first interactive start, but it was **not** present on the first machine we
set up. If `claude plugin install` reports `not found in marketplace
"claude-plugins-official"`, register it explicitly first:

```
claude plugin marketplace add anthropics/claude-plugins-official
```

Then the vendor plugins install directly:

```
claude plugin install csharp-lsp@claude-plugins-official
claude plugin install microsoft-docs@claude-plugins-official
claude plugin install azure@claude-plugins-official
claude plugin install github@claude-plugins-official
claude plugin install claude-security@claude-plugins-official
claude plugin install commit-commands@claude-plugins-official
```

`plugin-dev@claude-plugins-official` is for **authoring** the team plugin, not
for using it. Install it in the governance repo only, and disable it elsewhere —
it costs ~2.3k tokens on every session for skills a service repo never invokes.

Installs made through `claude plugin install` do not affect an already-running
session. Run `/reload-plugins` or restart.

### Context cost

Measured with `claude plugin details`. MCP tool schemas resolve at runtime and
are not counted; LSP runs out of process and costs nothing.

| Plugin | Always-on | Notes |
|---|---|---|
| `csharp-lsp` | ~0 | LSP only. Needs the `csharp-ls` binary |
| `github` | ~0 | MCP only |
| `commit-commands` | ~103 | |
| `sb-team` | ~103 | Grows as we add agents and skills |
| `microsoft-docs` | ~485 | 3 skills + MS Learn MCP |
| `claude-security` | ~642 | 7 agents, invoked on demand |
| `plugin-dev` | ~2,349 | Governance repo only |
| `azure` | ~6,282 | 28 skills. The largest single cost — see below |

`azure` is expensive because it ships 28 skills, most of which we will never
use. It stays because `azure-cost`, `azure-reliability`, `azure-compliance`,
`azure-kubernetes`, and `appinsights-instrumentation` sit directly on our path
and the MCP server is the SA's grounding. Skills cannot be cherry-picked from a
plugin — it is all or nothing. Revisit if session budget becomes tight.

## 3. Binaries the plugins expect

`csharp-lsp` does **not** install its language server. Without it the plugin
loads but reports `Executable not found in $PATH`, and you lose the compiler
diagnostics that make it the highest-ROI plugin in the set.

```
dotnet tool install --global csharp-ls
```

Confirm `csharp-ls` resolves on `PATH`, then restart Claude Code.

## Before the governance repo is on GitHub

During Phase 0 and Phase 1 the marketplace is local. Add it by path instead of
by `extraKnownMarketplaces`:

```
/plugin marketplace add d:/Study/ClaudeCode/AI-Foundry
/plugin install sb-team@solutionbuilder
```

Local marketplaces have auto-update **disabled** by default, so re-run
`/plugin marketplace update solutionbuilder` after changing plugin contents, then
`/reload-plugins`.

## Verifying

```
claude plugin details sb-team@solutionbuilder
```

lists the commands, skills, agents, hooks, MCP and LSP servers the plugin
actually contributes. If a skill you added does not appear, clear the cache with
`rm -rf ~/.claude/plugins/cache`, restart, and reinstall.
