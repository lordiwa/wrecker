---
name: wargame
description: Attack a set of approved use cases with Wrecker and return one JSON summary. For each use case it writes (or reuses) a spec in the current project under design/wargame/, lints it until 0 problems, runs the free dry run with an explicit seed and, in full mode, a model session only when the dry run leaves transitions uncovered (plus the live anchor when a target and an anchor file exist). Use it at a ticket-close wargaming step, or whenever someone asks to wargame approved use cases with Wrecker.
argument-hint: "[mode: full|short] [target: <url>] [seed: <int>] <approved use cases as text, or a path to a file with them>"
allowed-tools: Read, Glob, Write(design/wargame/**), Edit(design/wargame/**), mcp__plugin_wrecker_wrecker__wargame_spec_guide, mcp__plugin_wrecker_wrecker__wargame_lint_spec, mcp__plugin_wrecker_wrecker__wargame_dry_run, mcp__plugin_wrecker_wrecker__wargame_run_session, mcp__plugin_wrecker_wrecker__wargame_status, mcp__plugin_wrecker_wrecker__wargame_findings, mcp__wrecker__wargame_spec_guide, mcp__wrecker__wargame_lint_spec, mcp__wrecker__wargame_dry_run, mcp__wrecker__wargame_run_session, mcp__wrecker__wargame_status, mcp__wrecker__wargame_findings
---

# Wargame approved use cases with Wrecker

You turn approved use cases into Wrecker specs, attack them, and return ONE JSON object.
Wrecker returns candidates, not verdicts: you never decide severity, and you never fix the
system under test.

Hard rules, for the whole run:
- Write and edit files ONLY under `design/wargame/` in the current working directory. Never touch
  any other file of the project. (The tool grants below pre-approve writes inside
  `design/wargame/**` only; this rule is what forbids writing anywhere else.)
- Never simulate, estimate or invent a result. Every number, id and finding in your output comes
  from a Wrecker tool response in this run. If you do not have it, say so with an error (below).
- Use the Write and Edit tools to write specs, never a shell.

Arguments: $ARGUMENTS

## 0. Read the input

From the arguments (and, if they name a readable file, from that file):
- `mode`: `full` (default) or `short`.
- `target`: optional base URL of a live system (full mode only).
- `seed`: optional integer; default `7`. Every run uses it as `random_seed`, explicitly.
- The approved use cases: everything else. Each one has an id if the text gives one (`UC-...`);
  otherwise name them `UC-1`, `UC-2`, ... in order. Never change what a use case says: if it is
  ambiguous, write the narrowest spec that matches its words.

## 1. Check that Wrecker is connected (never fall back)

The tools are `mcp__plugin_wrecker_wrecker__wargame_*`, or `mcp__wrecker__wargame_*` when the
server was added to the project directly; either is fine. If they are not in your tool list, search
your deferred tools for `wargame` once. If `wargame_spec_guide`, `wargame_lint_spec` and
`wargame_dry_run` are still not available, STOP. Do not write specs, do not attack the use cases
yourself. Print exactly this JSON and nothing else:

{"engine":"wrecker","error":"WRECKER_NOT_CONNECTED","message":"The wrecker MCP server is not connected, so nothing was attacked. Install the plugin, restart Claude Code, check /mcp shows wrecker connected, then run this again.","install":["claude plugin marketplace add lordiwa/wrecker","claude plugin install wrecker@wrecker-marketplace"]}

After this check, if any Wrecker tool call fails, errors or is denied, or a Write or Edit of a
spec file fails or is denied, do not retry it with other arguments or another tool and do not
fill in its result: that spec's entry gets `"error":"TOOL_FAILED"` and `"message"` with the tool
name and the error text as returned, and you go on with the next one.

## 2. One spec per use case: reuse it when the use case did not change

The spec file is `design/wargame/<UC-id>.md` in `full` mode and `design/wargame/<UC-id>.short.md`
in `short` mode. Each spec keeps the use case it was written from, verbatim, as comment lines at
the top of its YAML header: one line `# approved: <line>` per line of the use-case text (an empty
line becomes `# approved:`). Comments do not change what Wrecker attacks.

For each use case:
1. If the file exists, compare its `# approved:` lines with this use case's text after
   normalizing both the same way: drop the `# approved:` prefixes; treat line breaks, indentation
   and any run of spaces or tabs as one space; drop list markers at the start of a line (`-`, `*`,
   `+`, `1.`, `1)`); turn curly quotes into straight ones (`“` `”` -> `"`, `‘` `’` -> `'`);
   trim the ends. If the normalized texts are equal, REUSE IT UNCHANGED: do not rewrite it, do not
   reformat it, do not bump anything. Go to step 3 (lint). Any other difference (a word, a
   number, punctuation) means the use case changed.
2. Otherwise write it. Call `wargame_spec_guide` once (no section) before the first spec you write,
   and follow that guide only.
   - `id` is the UC id. `spec_version` is `<UC-id>-v1` for a new file; when you rewrite an existing
     file because its use case changed, take its `spec_version` and add one (`-v1` -> `-v2`).
   - Main flow = the use case's happy path. Every alternate and exception path the use case names
     becomes its own flow (`2a`, `3b`, ...), with a `condition`, an `anchor` and, for exceptions,
     a `recovery`. Do not invent paths the use case does not name.
   - `short` mode keeps the main flow plus at most two alternate or exception paths. Each path you
     drop is listed in a header comment `# dropped in short mode: <path id or name>`, one per line,
     so a reused short spec still names them.
   - The postconditions encode exactly what the use case promises at success and at failure.
3. Use absolute paths when you pass a spec to a tool.

## 3. Lint to zero, bounded

Call `wargame_lint_spec` on each spec. It passes when `valid` is not `false`, `problems` is absent
or empty, and `findings` is empty. Otherwise fix the spec (read the guide section each problem
names) and lint again: at most 3 fixes (4 lint calls). Fixing a spec you reused is a rewrite: bump
its `spec_version`. If it still fails, do not attack that use case: record it with
`"error":"LINT_FAILED"` and the remaining problems, and go on.

## 4. Attack

For each spec that passed lint:
1. `wargame_dry_run` with `spec_path` and `random_seed: <seed>`. Free, no model.
2. `mode: short` stops here: no `wargame_run_session`, no live anchor. Add
   `"model session skipped: short mode"` to that spec's `not_running`, and one line
   `"path <id or name> not attacked: short mode"` for each `# dropped in short mode:` line.
3. `mode: full`: call `wargame_run_session` (same `spec_path` and `random_seed`) only if
   - the dry run's `status.coverage.transitions.pct` is below 100, or
   - `target` was given and `design/anchors/<UC-id>.runner.json` (then `anchor:
     {"mode":"live","adapter":"runner","target":<target>}`) or
     `design/anchors/<UC-id>.translation.json` (then `anchor: {"mode":"live","adapter":"http",
     "target":<target>}`) exists in the current project.
   Otherwise skip it and add `"model session skipped: the dry run covered every transition"` to that
   spec's `not_running`. A target with no anchor file adds `"live anchor skipped: no
   design/anchors/<UC-id>.runner.json or .translation.json"`.
   When the session ran, report its result (its `session_id`, coverage and findings) and keep the
   dry run's id in `dry_run_session_id`.

## 5. Build the JSON from the tool output (never invent fields)

Copy values from the tool results; do not compute or guess them.
- `version`: `status.version.version` of the run (fall back to `status.version.sha`).
- `random_seed`: `status.random_seed`. `mode`: the mode you ran.
- Per spec: `spec_version` from the spec, `spec_sha256` (= `status.spec_sha256`, a hash of the spec
  file as attacked), `reused` (`true` when step 2.1 applied), `session_id`, `coverage`
  (= `status.coverage`), `paths_attacked` (= `status.paths_attacked`), `not_running`
  (= `status.not_running` plus your skip lines).
  In `paths_attacked`, `reached` means every step of that flow was executed by at least one play;
  `attacked` means at least one play executed or tried one of its steps.
- Per finding entry of the summary: `id` = its `stable_id`; `type`, `plausibility`, `verified`,
  `explanation`, `step_cited` as given. A folded entry (it has `stable_ids`) becomes one finding with
  `id` = the first of `stable_ids` and `folded_ids` = all of them.
- Mapping over all specs, with EVERY id: a finding's `id`, and for a folded entry every one of its
  `folded_ids`. `counterexample`, `contradiction` and `dead_end` ids go to `mapping.blocking`
  (candidates to triage); `gap` ids go to `mapping.questions` (questions for the spec owner; they do
  not block).

Finding ids are comparable only within one `spec_version` of a use case: the same id in two runs
means the same candidate on the same spec. `reused: false` means the spec was written or rewritten
in this run, so ids from earlier runs of that use case no longer apply; do not read an old id that
is missing now as resolved.

Your final message is exactly one JSON object, no prose and no code fence:

{"engine":"wrecker","version":"0.2.0","mode":"short","random_seed":7,
 "specs":[{"uc":"UC-1","spec_path":"design/wargame/UC-1.short.md","spec_version":"UC-1-v1",
   "spec_sha256":"<64 hex>","reused":false,"session_id":"S-UC-1-7-r1","coverage":{...},
   "paths_attacked":[...],"not_running":[...],
   "findings":[{"id":"W-UC-1-0123456789ab","type":"gap","plausibility":"medium","verified":false,
     "explanation":"...","step_cited":"2","folded_ids":["W-UC-1-0123456789ab","W-UC-1-ba9876543210"]}]}],
 "mapping":{"blocking":[],"questions":["W-UC-1-0123456789ab","W-UC-1-ba9876543210"]}}

A use case that failed lint appears in `specs` as
`{"uc":...,"spec_path":...,"error":"LINT_FAILED","problems":[...],"findings":[]}`, and one whose
tool call failed as `{"uc":...,"spec_path":...,"error":"TOOL_FAILED","message":"...","findings":[]}`.
