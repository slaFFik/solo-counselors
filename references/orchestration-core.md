# Orchestration core — shared dispatch / collect / synthesize / archive routines

This file is the **single source of truth** for the mechanics of a counselors run. It is read by **both** executors:

- the **skill itself**, acting as the *inline coordinator* for `run` mode (executed in your current Claude Code session), and
- the **detached loop coordinator** agent, for `loop` mode (a separate Solo-managed process).

Each caller supplies the parameters below and wraps these routines with its own control flow:

- **run** = one wave, raw prompt, no rounds (see `SKILL.md`).
- **loop** = discovery + prompt-writing, then one wave per round, convergence, timeout (see `coordinator-loop-prompt.md`).

Identity, status lifecycle, cancellation polling, round control, and done-signaling belong to the **caller**, not here. These routines start once identity and parameters are established.

## Shared conventions

- **Scratchpad gotcha**: `scratchpad_write` takes a `name` and **returns a `scratchpad_id`**. Every *other* op — `scratchpad_read`, `scratchpad_tail`, `scratchpad_archive` — is keyed by `scratchpad_id`, NOT by name. Enumerate a run's *visible* pads with `scratchpad_list(tags=["{run_id}"])` (returns ids + names; archived pads excluded). **Record the `scratchpad_id` of every pad you write** — you need ids to append, read back, and archive.
- **Worker-pad naming**: `counselors.{run_id}.worker.{round_seg}{agent}-{index}`, where `round_seg` is `""` (run) or `r{N}.` (loop round N) — yielding `…worker.{agent}-{index}` or `…worker.r{N}.{agent}-{index}`. Tags: `["counselor","{run_id}","worker","{agent}"]`, plus `"round-{N}"` in loop.
- **Fence extraction**: pull the text between `===COUNSELOR-OUTPUT-BEGIN===` and `===COUNSELOR-OUTPUT-END===`. If the markers are missing, take the last ~3000 chars and prefix `[no fence markers detected — raw tail]`.
- **Read-only spawn**: under `read_only_policy == strict`, pass an agent-specific tool allowlist via `extra_args` at spawn (Claude Code: `["--allowedTools","Read,Glob,Grep,WebFetch,WebSearch"]`). For an agent with **no known allowlist flag** (currently anything other than the Claude family), do **not** refuse it and do **not** ask the user — automatically **downgrade just that agent to best-effort** (the worker preamble's in-prompt READ-ONLY block, which binds every backend) and note the per-agent downgrade in progress. Model diversity is the whole point of the panel, so keep every agent in the run: `strict` means "hard sandbox where supported, in-prompt enforcement everywhere else," never a silently shrunk panel.
- **Timekeeping**: Solo has **no read-the-clock tool**, and timer waits are **relative**, so a run carries a relative `duration_ms` budget — the skill derives it from `--duration` by arithmetic (`30m` → `1_800_000`), never from a wall clock — instead of an absolute deadline. Express every wait as a relative `*_ms` value. The detached **loop** coordinator is a Solo agent, so it enforces the total budget with a one-shot Solo timer (`timer_set(delay_ms=duration_ms, …)` set once at start) and waits on workers with idle timers — never a sleep loop or `now`-arithmetic. The inline **run** coordinator is an *external* session (timer-body delivery to it isn't *guaranteed* — though it often arrives in practice), so it schedules the wave's idle-timer as the primary wake but doesn't depend on the body: `already_satisfied` and a short sleep-poll grace cover the cases where it doesn't show up. It passes `duration_ms` straight through as the single wave's `max_wait_ms` guard.

## Routine: BUILD-EXECUTION-PROMPT

Turns the user's task into the (usually enriched) prompt that every worker receives. Both modes call this once before dispatching — run inline, loop in the coordinator.

**Inputs**: `user_prompt` (text or path — the caller has **already folded in any `--context` file contents**, so treat it as the complete task input; context handling is the caller's job, not this routine's), `prompt_source` (`inline`|`file`), `enhance` (`on`|`off`), `preset_path` (or empty), `prompt_writing_text`, `execution_boilerplate_text`, `progress_id`, `duration_ms`.

1. If `enhance == on`: follow `prompt_writing_text`. Run the **discovery** phase — explore the scoped repo with Read/Grep/Glob, guided by the preset's *Discovery focus* when `preset_path` is set; **cap discovery at roughly a third of `duration_ms` (half at the very most)** — the worker wave shares the same budget, so an over-long discovery starves it. Then the **prompt-writing** phase to produce a sharp, self-contained `base_prompt` (a longer, repo-grounded prompt — not the raw one-liner). If you cannot read the repo, fall back to `base_prompt = user_prompt` and note it in progress.
2. Else (`enhance == off`): `base_prompt = user_prompt` (resolve from file if it is a path), verbatim.
3. `execution_prompt = base_prompt + "\n\n" + execution_boilerplate_text`.
4. `scratchpad_write(name="counselors.{run_id}.prompt", content=execution_prompt, tags=["counselor","{run_id}","prompt"])` → **record `prompt_id`**. Append `[execution-prompt-built enhance={enhance} preset={name|none} words={wc}]` to progress.

**Returns**: `execution_prompt`, `prompt_id`. (The `prompt_id` is archived with the other working pads on a clean `done`.)

## Routine: DISPATCH-WAVE

**Inputs**: `dispatch` (list of `{agent, agent_tool_id, index}`), `execution_prompt`, `prior_round_context_by_index` (map from a worker's `index` to **that worker's own** prior-round block; empty/absent ⇒ no prior context — run mode, or loop round 1), `name_suffix` (`""` for run, `-r{N}` for loop), `read_only_policy`, `worker_preamble_text`.

For each entry (call `spawn_agent` per entry — each returns immediately, so workers run in parallel):

1. `spawn_agent(agent_tool_id={entry.agent_tool_id}, name="counselor-{agent}-{index}{name_suffix}", include_agent_instructions=true)` → `worker_pid` **and the response's optional `agent_instructions`** (the agent's own bootstrap — keep it). Under `strict`, add the `extra_args` allowlist if this agent supports one; if it doesn't (non-Claude), spawn it anyway under best-effort enforcement (see *Read-only spawn*) — never skip an agent or pause to ask.
2. Build the worker turn by substituting into `worker_preamble_text`:
   - `{{EXECUTION_PROMPT}}` ← `execution_prompt` — **the same text for every worker in the wave**.
   - `{{PRIOR_ROUND_CONTEXT}}` ← `prior_round_context_by_index[entry.index]` (empty string if absent). **Each worker gets only its OWN prior-round findings — never another agent's. No cross-model pollution: round N of `claude-0` carries only round N-1 of `claude-0`.**
   Then form the **composite first turn**: if `agent_instructions` is non-empty, prepend it (then a blank line) so the agent gets its own bootstrap before our preamble; otherwise the composite is just the substituted preamble.
3. `send_input(process_id=worker_pid, input=composite, submit=true, wait_ms=2000)`.
4. `get_process_status(worker_pid)` — verify alive + accepted input. On failure, append an error line to progress and **continue** (one failed spawn does not abort the wave).
5. Track `{agent, index, worker_pid}`.

**Returns**: the list of spawned workers `[{agent, index, worker_pid}]`.

## Routine: COLLECT-WAVE

**Inputs**: `workers` (from DISPATCH-WAVE), `round_seg` (`""` or `r{N}.`), `extra_tags` (`[]` for run, `["round-{N}"]` for loop), `progress_id`, `wave_budget_ms` (relative ceiling for this wave — run passes `duration_ms`; loop passes its per-round cap, with the coordinator's deadline timer as the authoritative global backstop).

1. `max_wait_ms = wave_budget_ms`. Schedule `timer_fire_when_idle_all(processes=[all worker_pids], max_wait_ms=max_wait_ms, body="wave-complete")` and **inspect the response — do not blindly wait for the body**:
   - `status == "already_satisfied"` → every worker was *already* idle when you scheduled, so Solo created **no** timer and will fire **no** body. Go straight to step 2 now. This is the fast-worker race ("the agent finished before the timer"): if you wait for a `wave-complete` that will never arrive, you hang.
   - otherwise a timer is pending → end your turn and resume when the `wave-complete` body wakes you (or the `max_wait_ms` guard fires). The response's `already_idle` workers are already counted toward the all-idle condition; `waiting_on` is who you're still blocked on. An idle worker = its turn ended, whether it *finished* or *asked a question* — step 2 reads each output and tells them apart, so you never block on a worker that paused to ask.
2. **First pass** — for each worker: `get_process_status(worker_pid)` (distinguish clean idle from crash/timeout), `get_process_output(worker_pid, lines=4000)`, then **fence-extract** (see conventions). Mark a worker **needs-nudge** if it is idle but produced **no closing fence** (it stalled or asked a question instead of finishing). Don't close workers yet.
3. **Nudge pass (at most once)** — if any worker needs-nudge and the budget allows: `send_input(worker_pid, "You went idle without emitting your analysis. Proceed with your best read-only analysis now and wrap your final answer in ===COUNSELOR-OUTPUT-BEGIN=== / ===COUNSELOR-OUTPUT-END===.", submit=true)` for each such worker, then `timer_fire_when_idle_all` on just those workers with a short cap (a small fraction of `wave_budget_ms`; honor `already_satisfied` exactly as in step 1), re-read each, and re-extract. Nudge each worker **only once**.
4. **Record & close** — for each worker: if the closing fence is still missing, fall back to the raw tail (prefixed `[no fence markers detected — raw tail]`). `scratchpad_write(name="counselors.{run_id}.worker.{round_seg}{agent}-{index}", content=extracted, tags=["counselor","{run_id}","worker","{agent}", ...extra_tags])` → **record the returned `scratchpad_id`** in your working-pads list; keep the extracted text **in memory** (needed for synthesis and, in loop, the next round's prior-context); append to progress (you hold `progress_id`): `[{round_seg}{agent}-{index} done | status={status} | words={wc} | nudged={yes|no}]`; `close_process(worker_pid)`.

**Returns**: collected outputs (in memory, per worker) + the recorded worker `scratchpad_id`s + per-worker final status.

## Routine: SYNTHESIZE

**Inputs**: the collected outputs you already hold (no re-read), `agents_csv`, `rounds`, `preset_note`.

Produce the summary following `references/synthesis-prompt.md`, substituting `{{AGENTS_CSV}}`, `{{N}}`, `{{ROUNDS}}`, `{{PRESET_NOTE}}`, `{{TASK_SHORT_DESCRIPTION}}`. Then:

`scratchpad_write(name="counselors.{run_id}.summary", content=summary_md, tags=["counselor","{run_id}","summary"])` → record `summary_id`. **Never archive the summary** — it is the deliverable and stays visible.

## Routine: CLASSIFY-AND-FINALIZE

Settles the run's terminal status and archives **only** on a clean `done`. Both modes call this once, after SYNTHESIZE — the single source of truth for the done/partial/cancelled decision so the two callers can't drift. Each caller owns *detecting* the cut-short/cancel signals; this routine owns what they *mean*.

**Inputs**: `worker_outcomes` (every worker's final status across the whole run — all waves/rounds), `cut_short_status` (empty if the run reached its natural end; otherwise the label for a run stopped *as a whole* before collection finished — `timeout` when the budget was exhausted via the deadline timer, `partial` when loop `on_timeout == abort` fired after a round timed out, or run-mode's wave guard expired with workers unfinished), `cancelled` (true if the user cancelled), `worker_ids` (all worker `scratchpad_id`s), `progress_id`, and — loop only — `prompt_id`.

1. `any_usable` = at least one worker produced a real extracted analysis (not an empty/crash tail).
2. Classify:
   - **`cancelled`** — if `cancelled`. (Synthesis already ran upstream on whatever was held.) **Do not archive.**
   - **`cut_short_status`** (`timeout` | `partial`) — else if `cut_short_status` is set: the run was stopped as a whole. **Do not archive** — leave the per-agent pads visible for inspection.
   - **`partial`** — else if not `any_usable`: collection finished but nobody produced usable output. **Do not archive.**
   - **`done`** — otherwise: collection finished and at least one worker produced usable output. Individual workers that crashed or timed out *among* successful peers do **not** downgrade the run — they're reported in the synthesis Panel-health section.
3. On **`done`** only: **ARCHIVE-WORKING-PADS** with `worker_ids` + `progress_id` (+ `prompt_id` in loop).
4. `kv_set("counselors.{run_id}.status", <classified status>)`.

**Returns**: the terminal status. (KV-key cleanup on a clean `done` is the *skill's* job — Step 9 — not this routine's.)

## Routine: ARCHIVE-WORKING-PADS  *(clean `done` path only)*

**Inputs**: all worker `scratchpad_id`s (every wave/round), `progress_id`, and — loop only — `prompt_id`.

For each of those ids: `scratchpad_archive(scratchpad_id=...)`. This leaves `counselors.{run_id}.summary` as the **only visible** pad for the run. Archiving **hides, does not delete** — archived pads stay in Solo. Note that `scratchpad_list(tags=["{run_id}"])` *excludes* archived pads, so to re-open one later you read it by the `scratchpad_id` you recorded (or un-archive it in Solo).

**Only run this on a clean `done`.** For `partial` / `timeout` / `cancelled`, archive nothing — leave the per-agent pads visible for inspection.
