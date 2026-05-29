---
name: bughunt
defaultRounds: 3
defaultReadOnly: strict
---

# Preset: bughunt

A review workflow for hunting real correctness bugs, edge-case failures, and missing tests that allow regressions.

## Discovery focus

When exploring the repo, map the areas most likely to harbor runtime failures:
- Control flow with conditionals, loops, and early returns — where off-by-one and wrong-default bugs hide
- Error and boundary paths: empty inputs, max/min values, partial failures, cleanup/rollback
- Null/undefined handling and unchecked inputs at trust boundaries
- Concurrency and ordering: shared mutable state, async state transitions, TOCTOU
- Resource lifecycle: handles, locks, listeners, exceptions swallowed in `finally`
- Risky branches with thin or no test coverage: error handlers, retries, fallbacks, migrations

## What the written prompt should emphasize

Instruct the panel to:
- Prioritize user-visible correctness failures over style; high-blast-radius bugs over speculative nits.
- For each finding give severity, confidence, file + function, the concrete code pattern and why it fails at runtime, user/system impact, the minimal safe fix, and a concrete failing-test scenario (inputs + expected behavior).
- Skip trivial style comments unless they hide a correctness bug.
