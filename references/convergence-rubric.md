# Convergence rubric (loop mode)

The coordinator checks convergence after each round (from round 2 onward) to decide whether to keep iterating or stop early.

## Primary metric: word-count ratio

For each agent:
```
ratio = words(this_round_output) / max(1, words(prior_round_output))
```

`mean_ratio = mean(ratio across all agents with both-round outputs)`.

If `mean_ratio < convergence_threshold` (default 0.3) → **converged**, stop the loop.

## Rationale

When the panel runs out of novel things to say, outputs shrink — agents either repeat (and the round-context instruction tells them to add depth or challenge prior claims, which still produces less text) or they say "no further findings". A sharp drop in word count is a strong signal that the marginal value of another round is low.

## Edge cases

- If `words(prior_round)` is 0 (agent crashed/empty): skip that agent from the ratio computation. If no agents have valid ratios, do not converge — keep iterating until rounds exhausted.
- If a single agent shrinks but others don't: do not converge; the mean across agents governs.
- If word count actually grows (ratio > 1): the panel is still finding new material; keep iterating.

## Tuning

- More aggressive (stop earlier): raise threshold toward 0.5.
- More conservative (keep iterating): lower threshold toward 0.1.

The default 0.3 says "stop when the average agent's new output is less than 30% of what they produced last round."
