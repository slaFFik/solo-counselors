# Convergence rubric (loop mode)

In the pipelined loop, convergence is decided **per worker**, the moment that worker finishes a round (from its **round 2** onward), to decide whether to re-dispatch it for another round or stop it. There is **no panel-wide round barrier** — a fast model may converge and stop after 2 rounds while a slow one is still on round 1. Each worker's decision uses **only its own** findings across its own rounds (no cross-model pollution), so it is a clean per-worker signal.

## Primary metric: per-worker new-findings ratio

Word count is a poor signal here: the round-context template deliberately tells each agent to be terse and *not* restate prior findings, so a later round is naturally far shorter than round 1 even when it surfaced valuable new material. Counting **what's actually new** measures the marginal value of another round directly.

Each worker ends its output with a machine-readable `## Findings index` (one line per finding: `F<n> | severity | confidence | file:symbol | claim`). Use it to make this count mechanical rather than a prose judgment call.

For the worker that just finished its round `r`, count its **new** findings — those it did not already raise in any of **its own** prior rounds (`r-1`, `r-2`, …). Match on the index's `file:symbol` location plus the underlying issue: treat two findings as the same when they point at the same location and the same issue even if worded differently; ignore pure restatements.

```
new_count   = that worker's distinct new findings in round r
prior_count = that worker's distinct findings in round r-1
ratio       = new_count / max(1, prior_count)
```

If `ratio < convergence_threshold` (default 0.3) → **that worker has converged**: stop it (`done_reason = "converged"`) and do not re-dispatch it. Other workers are unaffected and keep going at their own pace.

Convergence counts **raw** findings — verification is a single end-of-run wave (see VERIFY-WAVE) that runs after the last worker stops, so no verdicts exist yet while a worker is deciding whether to do another round. That's intentional: convergence measures whether *that worker* is still surfacing new material; verification later decides which of the whole panel's findings survive.

## Rationale

When a worker is still finding real problems, each of its rounds adds a meaningful number of genuinely new findings. When it has mined its area out, it either repeats itself (those don't count as new) or explicitly reports "no further findings." A sharp drop in *that worker's* new findings — independent of how much prose it wrote — is the honest signal that another round from it has low marginal value. Deciding per worker means a productive model isn't stopped just because a peer went quiet, and a mined-out model isn't dragged through more rounds just because a peer is still productive.

## Edge cases

- A worker that crashed or produced nothing this round contributes 0 new findings → it converges immediately (the coordinator records `done_reason = "empty"`); never re-dispatch a crashed worker.
- Restatements with a sharper fix or stronger evidence are a judgment call: count one as "new" only if the added evidence/impact/fix is itself a substantive finding, not just nicer wording.
- If a worker's `new_count` holds steady or grows round over round, it is still productive — keep re-dispatching it (up to its `rounds` cap).
- The `rounds` value is the hard per-worker ceiling: a worker that never converges still stops after `rounds` rounds (`done_reason = "maxrounds"`).

## Tuning

- More aggressive (stop each worker earlier): raise threshold toward 0.5.
- More conservative (let workers keep iterating): lower threshold toward 0.1.

The default 0.3 says "stop a worker when its round's new findings are fewer than 30% of its previous round's finding count" — i.e. that model is mostly repeating itself.
