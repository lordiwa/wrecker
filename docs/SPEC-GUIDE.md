# Writing a Wrecker spec

Wrecker attacks a use-case spec: it plays many action sequences against it and reports where the spec
has a gap, a dead end, a contradiction or a counterexample. You (or your AI) write the spec; Wrecker
checks it with `wargame_lint_spec` (free) and attacks it with `wargame_dry_run` (free). Every error
names the line or field, what was expected, and the section of this guide to read.

## File layout

One markdown file: a YAML header between two `---` lines, then a body with `##` sections.

```
---
id: UC-LOGIN
...header fields...
---
## Main flow
1. [action_token] Actor: what happens
## Alternate flows
## Exception flows
```

Body sections are recognised by heading: `## Main flow`, `## Alternate flows`, `## Exception flows`.
Any other `##` section (notes, referenced states) is ignored. Older specs use Spanish keywords; they
parse the same as the English ones in this guide.

## Header

Required:
- `id`: short unique id; it names sessions and findings.
- `name`: short name of the use case.
- `actor`: who performs the main flow. Wrecker uses it to tell the actor's choices from system faults.
- `spec_version`: a label you change when the spec changes; findings and resumed sessions carry it.
- `success_postcondition`: a condition (see Conditions) that must hold when a play reaches `end_success`.
- `failure_postcondition`: a condition that must hold when a play reaches `end_failure`.

Optional:
- `level` (default `user_goal`), `scope`, `trigger`, `preconditions` (list of text): context for the
  model roles and the report.
- `fog`: see Fog.
- `initial_world`: see World and paths.
- `referenced_states`: list of every world path a `requires` or `applies_when` reads (lint checks it).
- `injects`: see Injects.

## Fog

`fog` maps a header field to the lowest fog level that may see it: `fog: { injects: 4 }`. Levels are
whole numbers 1 to 4; the default is 3. Fields: `id`, `name`, `actor`, `level`, `scope`,
`spec_version`, `trigger`, `preconditions`, `main_flow`, `alternate_flows`, `exception_flows`,
`referenced_states`, `initial_world`, `injects`, `success_postcondition`, `failure_postcondition`.
Attackers see less or more of the spec: the naive player sees fog 1, the expert 2, the integrator 3
and the adversary 4. Hide what a real user would not know.

## World and paths

`initial_world` is the state every play starts from, as YAML: `{ signed_in: false, cart: { items: 2 } }`.
Conditions and effects address it with dotted paths: `cart.items`. A path under `input.` (for example
`input.card`) is what the actor types in that step: a condition only on `input.*` is the actor's choice.

## Conditions

Used by `success_postcondition`, `failure_postcondition`, `requires`, `applies_when`.
- `{ op: eq, path: P, value: V }`: the value at P equals V (`true` and `"true"` compare equal).
- `{ op: exists, path: P }`: P is set and not null. `{ op: absent, path: P }`: P is unset or null.
- `{ op: count, path: P, value: N }`: the list or string at P has length N (a number at P is used as is).
- `{ op: matches, path: P, value: "regex" }`: the string at P matches the regular expression.
- `{ all: [c1, c2] }`: every condition holds. `{ any: [c1, c2] }`: at least one holds.
There is no greater-than: use `any` of `eq`. A path that does not exist makes `eq`, `count` and
`matches` undetermined, not false.

## Effects

`effect: { path: value, ... }` sets world paths when the step runs. `{ path: { inc: 1 } }` adds to a
number (missing counts as 0). `path: null` clears it. Nothing else.

## Steps

One line per step, then its keywords indented under it:

```
2. [submit_password] User: submits the password → the system signs the user in.
   - requires: { op: eq, path: account_locked, value: false }
   - effect: { signed_in: true }
   - goto: end_success
```

- Step id: a number, plus a letter and number inside flows (`2a1`). Each id appears once.
- `[action_token]`: lowercase snake_case, unique per step. Tokens are the moves a play can make.
- `Actor:` then free text.
- `requires`: the condition under which the step may run. When it does not hold the step does not run,
  and if no flow anchored at that step covers the case, Wrecker reports a gap.
- `effect`: see Effects.
- `input` (optional): a note on what the actor types; Wrecker ignores it.
- `goto`: the next step id, `end_success` or `end_failure`. In the main flow a missing `goto` means
  the next main step (the last one goes to `end_success`). Inside a flow every step needs a `goto`.

## Flows

Alternate and exception flows branch from a step of any flow:

```
- 2a. condition: the password is wrong
  - anchor: 2
  - applies_when: { op: eq, path: account_locked, value: false }
  - recovery: the user can retry
  - 2a1. [submit_wrong_password] User: ...
    - goto: 2
```

- Flow id: step number plus a letter (`2a`), then `condition:` in words.
- `anchor`: the step id where the flow can start instead of that step.
- `applies_when` (optional): the condition under which the flow may start; without it, it is the
  actor's free choice.
- `recovery`: required in exception flows: what the user can do after the failure.
- Its steps follow, each ending in a `goto` back to a step or to an end.

## Ends

`end_success` and `end_failure` are the two ends. Reaching `end_success` without
`success_postcondition` holding (or `end_failure` without `failure_postcondition`) is a
counterexample. From every step some end must be reachable.

## Injects

Faults Wrecker may inject in the middle of a play, besides its generic catalog:

```
injects:
  - name: session_expired
    family: session
    applies_when: { op: eq, path: signed_in, value: true }
    effect: { signed_in: false }
```

`name` identifies it in findings; `family` is a free label that groups related faults (two injects
of the same family may also be tried together); `applies_when` says when it can fire; `effect`
changes the world (see Effects).

## Lint checks

`wargame_lint_spec` reports: a `goto` or `anchor` to a missing step; a flow with no steps or whose last
step has no `goto`; an exception without `recovery`; a condition path missing from `referenced_states`
or set by nothing (no effect, inject or top-level `initial_world` key); a step from which no end is
reachable; one action token on two steps. Fix them all before `wargame_dry_run`.

## Example

A minimal sign-in use case: signing in works, a wrong password shows an error, and the fifth wrong
password in a row locks the account. It passes `wargame_lint_spec` with no findings.
`wargame_dry_run` still reports `gap` candidates on it: that is normal, they are questions for the
spec owner (what happens if the actor does X here?). Also read the coverage and `steps_never_reached`
of the run: they name paths the attacker never reached, and `never_fired_why` says for each one what
its condition needs (for example `needs failed_attempts == 4; no play got there`). On this example
every transition fires, the lockout step 2b1 included: the attacker repeats the wrong-password loop
until the counter condition holds.

{{EXAMPLE}}
