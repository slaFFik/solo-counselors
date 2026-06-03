# Synthesis prompt (used by the coordinator when producing the final report)

You have the outputs of N agents, each a different model that analyzed the **same** task independently and in parallel. Some may have produced strong findings; others may be thin, timed out, or off-topic.

You may also have **verifier verdicts**: when verification ran, each agent's findings were independently re-checked against the code by a different agent, which ruled every finding `confirmed`, `refuted`, or `uncertain` (with an optional severity adjustment), keyed by finding id. Use these verdicts to weight and label findings as described below. If you were given no verdicts (verification was off or skipped), omit every verification-specific element — no per-finding verification line, no Disputed section, no verification row in Panel health.

Produce a concise synthesis with this structure:

## Synthesis: {{TASK_SHORT_DESCRIPTION}}

**Panel**: {{N}} agents over {{ROUNDS}} round(s) — agents: {{AGENTS_CSV}}{{PRESET_NOTE}}.

### Top findings

For each of the 3–7 most important findings:

- **[severity/confidence]** **{finding title}** — `{file_path}:{symbol}`
  - **Evidence**: short quote or paraphrase
  - **Impact**: user/system consequence
  - **Fix**: minimal change
  - **Reported by**: {comma-separated agents/models}
  - **Verification**: `confirmed by {verifier}` / `uncertain ({verifier})` / `unverified` — include this line **only when verdicts exist**. Apply any verifier severity adjustment to the `[severity/confidence]` above and say so (e.g. "severity lowered to medium per verifier: gated behind admin flag").

Prioritize **confirmed** findings, then findings reported by multiple agents or rated high severity. A finding independently confirmed by a *different* model is the strongest signal there is — stronger than two agents merely raising it, since multiple models can be confidently wrong the same way. Deduplicate aggressively — if two agents raised the same issue with different framings, merge them (and merge their verdicts). Do **not** list a `refuted` finding here; it goes in the Disputed section.

### Disputed / likely false-positive

*(Include this section only when verdicts exist and at least one finding was refuted.)* List each **refuted** finding so the reader can judge it, not silently drop it:

- **{finding title}** — `{file_path}:{symbol}` — raised by {agent}, **refuted by {verifier}**: {one-line reason the verifier gave, e.g. "input is intval()-cast before the query — no injection"}.

Keep these terse. They are findings the panel raised but an independent model could not stand behind; surfacing them with the refutation is more useful than hiding them.

### Coverage gaps

One short paragraph: what areas of the task did no agent cover? Where would you want a follow-up run (different agents, or a loop with a preset)?

### Panel health

Brief table or list: `{agent}` — `{status: ok|thin|timeout|crashed|cancelled}`. Skip if everything was clean.

When verification ran, add one line on **verification coverage**: who verified whom (e.g. `gemini→claude, claude→gemini`), and the confirmed/refuted/uncertain tally. If verification was skipped (single-model or <2 finding-producing workers), say so in one line — it tells the reader the findings are unchecked.

### Suggested next steps

2–4 concrete next actions, ordered by priority. If multiple agents converged on one area, that's your top recommendation.

---

Be terse. The reader will drill into individual agent scratchpads if they want more detail. Your job is to give them the shortest useful overview.
