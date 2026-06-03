---
name: invariants
defaultRounds: 3
defaultReadOnly: strict
verify: off
---

# Preset: invariants

A review workflow for finding state-synchronization issues, impossible states, and state-management anti-patterns.

## Discovery focus

When exploring the repo, look at how state is represented and kept consistent:
- Clusters of booleans that encode state (2^n combinations, many impossible)
- Bags of optionals where a discriminated union belongs
- Magic strings used for status/state instead of enums/constants
- Status enums duplicated across DB and code (spelling/count/casing drift)
- The same data stored in multiple places; derived values persisted instead of computed
- Multi-step flows / 4+ value status fields managed with ad-hoc conditionals
- Single-source-of-truth violations (validation, types, permissions duplicated)

## What the written prompt should emphasize

Instruct the panel to:
- Surface drift that causes real runtime bugs, not theoretical concerns.
- For each: file path, the specific pattern, what can go wrong, and the minimal fix (discriminated union, enum extraction, computed getter, or state machine).
