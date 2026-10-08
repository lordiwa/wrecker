# Wrecker

Adversarial wargaming over MCP. Wrecker takes a **structured use case** (a short
Markdown + YAML file: actor, preconditions, main flow, alternate and exception
flows) and attacks it: it plays sequences against the spec and returns
**candidates** (counterexamples, gaps, dead ends) while spending as few tokens
as possible. Lint and dry run call no model at all.

Its output is candidates, not verdicts: severity and promotion to a finding are
your call.

## Install (Claude Code)

```
/plugin marketplace add lordiwa/wrecker
/plugin install wrecker@wrecker-marketplace
```

Then run `/mcp`: the `wrecker` server should be connected with 8 tools.

## What you need

- **Claude Code 2.1.78 or newer** (the plugin keeps your runs in
  `${CLAUDE_PLUGIN_DATA}`, added in 2.1.78).
- **Node.js 20 or newer** on your `PATH`.
- **Your own `claude` CLI, logged in**, only for `wargame_run_session`: it calls
  models through `claude -p` on **your own Claude subscription**. Wrecker never
  asks for or reads an API key. Without the CLI, lint and dry run still work and
  `wargame_run_session` tells you what is missing. If `claude` is not on the
  `PATH` the server sees, set `WARGAME_CLAUDE_BIN` to the full path of the
  executable.
- **Chromium**, only for the browser runner (`anchor.adapter: "runner"`):
  run `npx playwright install chromium` once. If your project does not have
  Playwright yet, add it with `npm i -D playwright`. Without it the runner says
  so; everything else keeps working.

Tested on Windows (PowerShell and Git Bash).

## Tools

| Tool | What it does | Spends tokens? |
|---|---|---|
| `wargame_spec_guide` | How to write a spec, with a minimal example. | No |
| `wargame_lint_spec` | Checks a spec before anything is spent. | No |
| `wargame_dry_run` | The attack loop with **no model at all** (deterministic, seeded). | No |
| `wargame_run_session` | The attack loop with models, via your `claude` CLI. | Yes, on your subscription |
| `wargame_status` | Status of a run; with no `session_id`, the running release. | No |
| `wargame_findings` | The compressed candidates of a run. | No |
| `wargame_export` | Report (`md`) or candidates (`json`) of a run. | No |
| `wargame_replay` | Re-runs a session by its `random_seed` and compares hashes. | No |

## Try it

The plugin ships an example spec, `fixtures/UC-07-pagar-pedido.md` (pay for an
order). Ask Claude Code:

> Run `wargame_lint_spec` on `fixtures/UC-07-pagar-pedido.md`, then
> `wargame_dry_run` with `random_seed: 7`, and show me the candidates.

(`wargame_dry_run` uses the example when you give it no `spec_path`.)

For your own use case, call `wargame_spec_guide`, write the spec, and pass its
path (relative to your project, or absolute) as `spec_path`.

## Where results go

Runs are written under `out/<session-id>/` in the plugin's data folder
(`${CLAUDE_PLUGIN_DATA}`), so they survive plugin updates. `wargame_export`
returns the exact path.

## License

MIT, see `LICENSE`.
