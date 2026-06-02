# Convergence rubric (loop mode)

The coordinator checks convergence after each round (from round 2 onward) to decide whether to keep iterating or stop early.

## Primary metric: new-findings ratio

Word count is a poor signal here: the round-context template deliberately tells each agent to be terse and *not* restate prior findings, so a later round is naturally far shorter than round 1 even when it surfaced valuable new material. Counting **what's actually new** measures the marginal value of another round directly.

You already parse each worker's findings for synthesis, so reuse that parse:

For each agent's round-N output, count its **new** findings — those it did not already raise in any of its own prior rounds. Treat two findings as the same when they point at the same location (file + symbol/area) and the same underlying issue, even if worded differently; ignore pure restatements.

```
new_count   = Σ over agents of (distinct new findings that agent raised this round)
prior_count = distinct findings the panel produced in round N-1
ratio       = new_count / max(1, prior_count)
```

If `ratio < convergence_threshold` (default 0.3) → **converged**, stop the loop.

## Rationale

When the panel is still finding real problems, each round adds a meaningful number of genuinely new findings. When it has mined the area out, agents either repeat themselves (those don't count as new) or explicitly report "no further findings." A sharp drop in *new* findings — independent of how much prose each agent wrote — is the honest signal that another round's marginal value is low.

## Edge cases

- If an agent crashed or produced nothing this round, it contributes 0 new findings; exclude it from `prior_count` only if it was also empty last round. If *no* agent produced any findings this round → converged (nothing left to find).
- If a single agent goes quiet but others keep surfacing new findings, the panel-wide `new_count` governs — do not converge.
- Restatements with a sharper fix or stronger evidence are a judgment call: count one as "new" only if the added evidence/impact/fix is itself a substantive finding, not just nicer wording.
- If `new_count` holds steady or grows round over round, the panel is still productive — keep iterating.

## Tuning

- More aggressive (stop earlier): raise threshold toward 0.5.
- More conservative (keep iterating): lower threshold toward 0.1.

The default 0.3 says "stop when this round's new findings are fewer than 30% of last round's finding count" — i.e. the panel is mostly repeating itself.
