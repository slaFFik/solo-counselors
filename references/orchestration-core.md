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
- **Fence extraction**: pull the text between `<<<COUNSELOR-OUTPUT-BEGIN>>>` and `<<<COUNSELOR-OUTPUT-END>>>`. If the markers are missing, take the last ~3000 chars and prefix `[no fence markers detected — raw tail]`.
- **Read-only spawn**: under `read_only_policy == strict`, pass an agent-specific tool allowlist via `extra_args` at spawn (Claude Code: `["--allowedTools","Read,Glob,Grep,WebFetch,WebSearch"]`). Agents with no allowlist flag: refuse to spawn under `strict` and note it in progress.

## Routine: BUILD-EXECUTION-PROMPT

Turns the user's task into the (usually enriched) prompt that every worker receives. Both modes call this once before dispatching — run inline, loop in the coordinator.

**Inputs**: `user_prompt` (text or path), `prompt_source` (`inline`|`file`), `enhance` (`on`|`off`), `preset_path` (or empty), `prompt_writing_text`, `execution_boilerplate_text`, `progress_id`, `deadline_ts_ms`.

1. If `enhance == on`: follow `prompt_writing_text`. Run the **discovery** phase — explore the scoped repo with Read/Grep/Glob, guided by the preset's *Discovery focus* when `preset_path` is set; budget discovery tightly against the deadline. Then the **prompt-writing** phase to produce a sharp, self-contained `base_prompt` (a longer, repo-grounded prompt — not the raw one-liner). If you cannot read the repo, fall back to `base_prompt = user_prompt` and note it in progress.
2. Else (`enhance == off`): `base_prompt = user_prompt` (resolve from file if it is a path), verbatim.
3. `execution_prompt = base_prompt + "\n\n" + execution_boilerplate_text`.
4. `scratchpad_write(name="counselors.{run_id}.prompt", content=execution_prompt, tags=["counselor","{run_id}","prompt"])` → **record `prompt_id`**. Append `[execution-prompt-built enhance={enhance} preset={name|none} words={wc}]` to progress.

**Returns**: `execution_prompt`, `prompt_id`. (The `prompt_id` is archived with the other working pads on a clean `done`.)

## Routine: DISPATCH-WAVE

**Inputs**: `dispatch` (list of `{agent, agent_tool_id, index}`), `execution_prompt`, `prior_round_context` (empty string when none), `name_suffix` (`""` for run, `-r{N}` for loop), `read_only_policy`, `worker_preamble_text`.

For each entry (call `spawn_agent` per entry — each returns immediately, so workers run in parallel):

1. `spawn_agent(agent_tool_id={entry.agent_tool_id}, name="counselor-{agent}-{index}{name_suffix}", include_agent_instructions=true)` → `worker_pid`. Under `strict`, add the `extra_args` allowlist.
2. Build the composite first turn by substituting into `worker_preamble_text`:
   - `{{EXECUTION_PROMPT}}` ← `execution_prompt` — **the same text for every worker in the wave**.
   - `{{PRIOR_ROUND_CONTEXT}}` ← `prior_round_context` (empty string when there is none).
3. `send_input(process_id=worker_pid, input=composite, submit=true, wait_ms=2000)`.
4. `get_process_status(worker_pid)` — verify alive + accepted input. On failure, append an error line to progress and **continue** (one failed spawn does not abort the wave).
5. Track `{agent, index, worker_pid}`.

**Returns**: the list of spawned workers `[{agent, index, worker_pid}]`.

## Routine: COLLECT-WAVE

**Inputs**: `workers` (from DISPATCH-WAVE), `round_seg` (`""` or `r{N}.`), `extra_tags` (`[]` for run, `["round-{N}"]` for loop), `progress_id`, `deadline_ts_ms`.

1. `max_wait_ms = max(0, deadline_ts_ms - now)`. Schedule `timer_fire_when_idle_all(processes=[all worker_pids], max_wait_ms=max_wait_ms, body="wave-complete")` and **inspect the response — do not blindly wait for the body**:
   - `status == "already_satisfied"` → every worker was *already* idle when you scheduled, so Solo created **no** timer and will fire **no** body. Go straight to step 2 now. This is the fast-worker race ("the agent finished before the timer"): if you wait for a `wave-complete` that will never arrive, you hang.
   - otherwise a timer is pending → end your turn and resume when the `wave-complete` body wakes you (or the `max_wait_ms` guard fires). The response's `already_idle` workers are already counted toward the all-idle condition; `waiting_on` is who you're still blocked on. An idle worker = its turn ended, whether it *finished* or *asked a question* — step 2 reads each output and tells them apart, so you never block on a worker that paused to ask.
2. For each worker:
   - `get_process_status(worker_pid)` — distinguish clean idle from crash/timeout.
   - `get_process_output(worker_pid, lines=4000)`.
   - **Fence-extract** (see conventions).
   - `scratchpad_write(name="counselors.{run_id}.worker.{round_seg}{agent}-{index}", content=extracted, tags=["counselor","{run_id}","worker","{agent}", ...extra_tags])` → **record the returned `scratchpad_id`** in your working-pads list. Keep the extracted text **in memory** (needed for synthesis and, in loop, the next round's prior-context).
   - Append to progress (you hold `progress_id`): `[{round_seg}{agent}-{index} done | status={status} | words={wc}]`.
   - `close_process(worker_pid)`.

**Returns**: collected outputs (in memory, per worker) + the recorded worker `scratchpad_id`s + per-worker final status.

## Routine: SYNTHESIZE

**Inputs**: the collected outputs you already hold (no re-read), `agents_csv`, `rounds`, `preset_note`.

Produce the summary following `references/synthesis-prompt.md`, substituting `{{AGENTS_CSV}}`, `{{N}}`, `{{ROUNDS}}`, `{{PRESET_NOTE}}`, `{{TASK_SHORT_DESCRIPTION}}`. Then:

`scratchpad_write(name="counselors.{run_id}.summary", content=summary_md, tags=["counselor","{run_id}","summary"])` → record `summary_id`. **Never archive the summary** — it is the deliverable and stays visible.

## Routine: ARCHIVE-WORKING-PADS  *(clean `done` path only)*

**Inputs**: all worker `scratchpad_id`s (every wave/round), `progress_id`, and — loop only — `prompt_id`.

For each of those ids: `scratchpad_archive(scratchpad_id=...)`. This leaves `counselors.{run_id}.summary` as the **only visible** pad for the run. Archiving **hides, does not delete** — archived pads stay in Solo. Note that `scratchpad_list(tags=["{run_id}"])` *excludes* archived pads, so to re-open one later you read it by the `scratchpad_id` you recorded (or un-archive it in Solo).

**Only run this on a clean `done`.** For `partial` / `timeout` / `cancelled`, archive nothing — leave the per-agent pads visible for inspection.
