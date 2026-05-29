---
name: hotspots
defaultRounds: 4
defaultReadOnly: strict
---

# Preset: hotspots

A review workflow for finding high-impact performance bottlenecks, with emphasis on asymptotic complexity and scaling behavior.

## Discovery focus

When exploring the repo, identify the hot paths and the data sizes that flow through them:
- Nested scans, repeated sort/filter passes, per-item linear lookups (accidental O(n^2)+)
- N+1 access across database / API / filesystem / queue / cache
- Unbounded traversal/fan-out; repeated expensive work that could be cached/memoized/batched
- Serialization/parsing churn and large allocations/copies in tight loops
- Missing pagination / streaming / chunking / backpressure

## What the written prompt should emphasize

Instruct the panel to:
- Prioritize hot-path issues and large, low-risk wins; prefer evidence over speculation.
- For each: severity, confidence, file + function, the exact costly operation, complexity (define n/m, before/after Big-O), expected latency/throughput/memory effect, the minimal fix (index/map, batching, caching, pagination), and a benchmark/profiling validation idea.
- Skip micro-optimizations unless clearly on a critical hot path.
