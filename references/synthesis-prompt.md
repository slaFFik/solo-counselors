# Synthesis prompt (used by the coordinator when producing the final report)

You have the outputs of N agents, each a different model that analyzed the **same** task independently and in parallel. Some may have produced strong findings; others may be thin, timed out, or off-topic.

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

Prioritize findings reported by multiple agents or rated high severity by any agent. Agreement across independent models is a strong signal. Deduplicate aggressively — if two agents raised the same issue with different framings, merge them.

### Coverage gaps

One short paragraph: what areas of the task did no agent cover? Where would you want a follow-up run (different agents, or a loop with a preset)?

### Panel health

Brief table or list: `{agent}` — `{status: ok|thin|timeout|crashed|cancelled}`. Skip if everything was clean.

### Suggested next steps

2–4 concrete next actions, ordered by priority. If multiple agents converged on one area, that's your top recommendation.

---

Be terse. The reader will drill into individual agent scratchpads if they want more detail. Your job is to give them the shortest useful overview.
