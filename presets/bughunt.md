---
name: bughunt
defaultRounds: 3
defaultReadOnly: strict
verify: cross
---

# Preset: bughunt

A review workflow for hunting real correctness bugs, edge-case failures, and missing tests that allow regressions.

## Discovery focus

When exploring the repo, map the areas most likely to harbor runtime failures (go deep on the area the task names first; sweep the rest of this list broadly only if discovery budget remains):
- Control flow with conditionals, loops, and early returns — where off-by-one and wrong-default bugs hide
- Inverted logic and wrong operators: flipped `&&`/`||`, negation/De Morgan mistakes, loose-equality and type-coercion surprises
- Error and boundary paths: empty inputs, max/min values, partial failures, cleanup/rollback
- Null/undefined handling and unchecked inputs at trust boundaries
- Data handling: date/time/timezone math, numeric overflow / float rounding / divide-by-zero, encoding/locale/case-folding
- Concurrency and ordering: shared mutable state, async state transitions, TOCTOU
- Resource lifecycle: handles, locks, listeners, exceptions swallowed in `finally`
- Risky branches with thin or no test coverage: error handlers, retries, fallbacks, migrations

## What the written prompt should emphasize

The standard per-finding fields (including `reachability`) and the severity/confidence anchors come from the execution boilerplate — don't restate them. On top of those, instruct the panel to:
- Prioritize correctness failures with real blast radius over style. Rate by impact, not visibility: silent data corruption, bad persistence, billing/accounting errors, and background-job damage count even when no user sees them directly.
- For each finding, give a concrete failing **case**: the inputs (or call sequence) and the expected-vs-actual behavior. This is a read-only review — describe the case, do not run it. If the project has a test suite, also name the existing test that should have caught it and why it didn't; if the project has no tests, skip the test angle entirely and rely on the failing case alone.
- Skip trivial style comments unless they hide a correctness bug.
