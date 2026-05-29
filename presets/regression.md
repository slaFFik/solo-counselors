---
name: regression
defaultRounds: 3
defaultReadOnly: strict
---

# Preset: regression

A review workflow for auditing regression risk — behavior changes that break existing users or dependent systems.

## Discovery focus

When exploring the repo, concentrate on the observable surface and what recently changed:
- Public surface: function signatures, return shapes, event payloads, CLI flags, API responses callers depend on
- Guards that may have weakened: validation, authorization, null checks
- Control-flow / ordering changes: initialization, retry, cleanup timing
- Partial migrations where old and new paths diverge
- Error handling: swallowed exceptions, changed status codes, missing rollback
- Feature-flag / config default changes; tests asserting implementation details rather than behavior

## What the written prompt should emphasize

Instruct the panel to:
- Surface high-blast-radius behavior changes over low-impact style.
- For each: file + function, the user-visible regression risk, and a concrete failing test that would catch it.
