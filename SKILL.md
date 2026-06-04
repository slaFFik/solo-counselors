---
name: solo-counselors
description: Run parallel multi-agent code review via Solo MCP — replicates the `counselors` tool (https://github.com/aarondfrancis/counselors) without depending on `claude -p`. Use when the user asks for "counselors", "second opinion", "panel of experts", "parallel review", or "multi-agent analysis" — fan the same task out to several AI agents (claude, gemini, codex, copilot, antigravity) in parallel and get a synthesized report. Both modes enrich the prompt via a repo-discovery + prompt-writing phase (optionally shaped by a preset: bughunt, security, regression, contracts, invariants, hotspots, architecture) before fanning out; `--mode run` does it inline and fans once, `--mode loop` spawns a durable coordinator that iterates over rounds.
allowed-tools: Read, Grep, Glob, Bash, AskUserQuestion, mcp__solo__identify_session, mcp__solo__list_projects, mcp__solo__select_project, mcp__solo__list_agent_tools, mcp__solo__spawn_agent, mcp__solo__send_input, mcp__solo__get_process_output, mcp__solo__get_process_status, mcp__solo__stop_process, mcp__solo__close_process, mcp__solo__timer_fire_when_idle_all, mcp__solo__timer_fire_when_idle_any, mcp__solo__timer_set, mcp__solo__timer_list, mcp__solo__timer_cancel, mcp__solo__kv_get, mcp__solo__kv_set, mcp__solo__kv_list, mcp__solo__kv_delete, mcp__solo__scratchpad_read, mcp__solo__scratchpad_write, mcp__solo__scratchpad_tail, mcp__solo__scratchpad_list, mcp__solo__scratchpad_archive
---

# solo-counselors

Parallel multi-agent code review via Solo MCP. The skill spawns the **worker** agents in parallel — **one per agent in the panel** — and fans the **same** prompt out to all of them. The panel's diversity comes from running **different models** on the identical task. Outputs are collected and — optionally — run through an **adversarial cross-verification** wave (each agent's findings re-checked by a different model) before being written up as a single synthesis; the user sees a concise synthesis in chat plus pointers to per-worker scratchpads for drill-down.

**Who coordinates depends on the mode** (see Step 7):

- **`run`** — the skill **coordinates inline**: your current Claude Code session is the coordinator. It builds the enriched prompt, spawns the workers, waits, collects, synthesizes — no separate coordinator process. Cheaper and lower-latency, but tied to this session (close it mid-run and the run is orphaned). Run is single-shot, so that's an acceptable trade.
- **`loop`** — the skill spawns a **detached Solo-managed coordinator** agent that does the orchestration. The run survives this session closing; reconnect with `--status`. Loop is long and multi-round, so durability matters and the heavy intermediate output stays out of your session.

The two paths share their dispatch/collect/synthesize/archive mechanics via `references/orchestration-core.md` — one source of truth, called by the inline run path here and by the detached loop coordinator.

Two modes:

- **`run`** — enriched single-shot fan-out. The skill builds an enriched execution prompt (repo-discovery → prompt-writing) and fans it **once** to one worker per agent. Single round, inline.
- **`loop`** — enriched multi-round. A detached coordinator builds the same enriched prompt, then iterates: rounds 2+ feed each agent **its own** prior output back (never another agent's — no cross-model pollution) so each model refines its own analysis. Convergence stops it early.

**Both** modes enrich by the same policy: a `--preset` (one per run) shapes the discovery + prompt-writing; enrichment runs by default for inline prompts (skip with `--no-inline-enhancement`) and is skipped for `-f` file prompts that have no preset. The run/loop difference is **iteration + durability**, not enrichment.

## Argument grammar

Parse the user's invocation. Positional argument is the prompt (or use `-f/--file <path>`).

**Common flags** (both modes):

| Flag | Meaning | Default |
|---|---|---|
| `<prompt>` or `-f/--file <path>` | The task | required (unless a management flag below) |
| `--mode <run\|loop>` | One-shot or iterative | `run` |
| `--agents <a,b,c>` (alias `-a`) | Agents from `list_agent_tools` — **one worker each**, duplicates allowed | first available |
| `--group <name>` | Use a saved group from KV (alternative to `--agents`) | last-used group if no `--agents` |
| `--read-only <strict\|best-effort\|off>` | Worker mutation policy | preset default or `best-effort` |
| `--verify <off\|cross>` | Adversarial cross-verification of findings before synthesis | preset `verify` or `off` |
| `--context <paths>` | Files to attach to the prompt (comma-sep) | — |
| `--preset <name>` | One preset from `presets/` to shape discovery + prompt-writing | none |
| `--no-inline-enhancement` | Skip discovery + prompt-writing for an inline prompt (send it raw) | off |
| `--duration <e.g. 30m>` | Total time budget | `15m` run / `45m` loop |
| `--dry-run` | Print dispatch plan, no spawn | off |
| `--status <run_id>` | Reconnect to an in-flight run | management |
| `--cancel <run_id>` | Cancel a run | management |
| `--list-groups` | List saved groups, exit | management |
| `--save-group <name>=<a,b,c>` | Define/update a group, exit | management |
| `--delete-group <name>` | Remove a group, exit | management |
| `--list-presets` | List available presets, exit | management |

**Loop-only flags** (ignored in `run` mode — warn if passed there; run is single-round):

| Flag | Meaning | Default |
|---|---|---|
| `--rounds <N>` | Max rounds | preset `defaultRounds` or 3 |
| `--convergence-threshold <ratio>` | Early-stop ratio | 0.3 |
| `--on-timeout <abort\|continue>` | Loop timeout policy | `continue` |

If the user invokes the skill with no flags and no positional prompt, ask (briefly) what they want to analyze and which agents — but lean on defaults aggressively. Don't bombard them with questions. **Note**: both modes accept `--preset` and enrich the prompt; run differs from loop only in being single-shot and inline.

## Step 1: Preflight (run on every invocation, including management flags)

Call these in sequence; abort with a clear error on any failure. Cache the result for the rest of this invocation.

1. `mcp__solo__identify_session(external={name: "claude-code", agent_id: "cc-external"})` — confirms Solo MCP is reachable and establishes an external-actor identity for this Claude Code session. On error / tool-not-found: tell the user "Solo MCP server is not reachable. Add `solo` to your MCP config and restart Claude Code." and stop. The response includes `effective_project_id` / `effective_project_name` if a project is already scoped to this session. **Note**: `mcp__solo__whoami` does NOT work for external callers — it's for Solo-managed processes only. Always use `identify_session` here.
2. If the `identify_session` response shows no `effective_project_id`: call `mcp__solo__list_projects`. If exactly one, `mcp__solo__select_project(project_id=...)` it. If multiple, use AskUserQuestion to ask the user to pick (show project name + path). If zero, tell the user "No Solo projects configured — add one before using counselors" and stop. Note: project scope is the namespace for groups and run state — different repos can have different counselor groups.
3. `mcp__solo__kv_get(key="counselors.preflight")` — probes the KV store now that a project is scoped. Expected: returns `{entry: null}` for a fresh setup, or the stored probe value. On error referencing KV being disabled: tell the user "Solo KV store is not enabled — required for group storage and run state. Enable it in your Solo configuration." and stop. On error about no project selected: re-check step 2.
4. `mcp__solo__list_agent_tools` — must return ≥1 entry with `enabled: true`. If empty: "No Solo agent tools configured. Add at least one agent (claude, gemini, codex, copilot, antigravity) via Solo before running counselors." and stop.

## Step 2: Management flag handlers (short-circuit, no run)

- `--list-groups`: `mcp__solo__kv_list(prefix="counselors.group.")`; print as a table `name | agents | last_used?`. Highlight `counselors.group.last` value.
- `--save-group <name>=<a,b,c>`: validate each agent in csv against the agent-tools list (case-insensitive match on `name` field); reject if any unknown. `mcp__solo__kv_set("counselors.group." + name, csv)`. If no `counselors.group.last` exists, set it to `name`.
- `--delete-group <name>`: `mcp__solo__kv_delete("counselors.group." + name)`. If it was `last`, delete `counselors.group.last` too.
- `--list-presets`: list `presets/*.md` filenames (each is a valid `--preset` value); for each, read its front-matter and show `name | defaultRounds | defaultReadOnly | verify`. Presets apply to **both** modes (`defaultRounds` is used only by loop).
- `--status <run_id>`: `mcp__solo__kv_get("counselors." + run_id + ".status")`. To read any pad you need its id — `scratchpad_list(tags=[run_id])` returns this run's *visible* pads (names + ids). Then branch on status:
  - **`done`** (a finished — possibly detached — run that wasn't cleaned up yet): `scratchpad_read` the `counselors.{run_id}.summary` pad and present it; then reap the coordinator (`kv_get coordinator_pid` → `close_process`, ignore "not found") **and** run the clean-up from Step 9 (delete the per-run KV keys).
  - **absent / null** (a clean `done` whose KV was already deleted, or an unknown id): `scratchpad_list(tags=[run_id])` — a visible `counselors.{run_id}.summary` ⇒ the run completed and was already cleaned up, so read + present it; nothing visible ⇒ tell the user there's no live or recorded run by that id.
  - **`running` / `spawning`**: find `counselors.{run_id}.progress` and `scratchpad_tail(scratchpad_id, lines=20)` to show the last lines. Then branch on the coordinator via `kv_get("counselors." + run_id + ".coordinator_pid")`:
    - **pid present** (a **loop** run): show its `get_process_status` and offer to enter the poll loop — it's live.
    - **no pid** (an inline **run** whose session ended): run mode never spawns a separate coordinator and coordinates inline, so a stuck `running` means the originating session died mid-run. There is **no live process to poll** and nothing will ever flip the status — tell the user it can't be resumed and they should re-run it. Don't offer the poll loop.

    Either way, don't reap or clean — the state is left as-is for inspection.
  - **`partial` / `timeout` / `error` / `cancelled`**: present the summary + tail progress, reap the coordinator, but **do not** delete the KV keys — keep them (and the visible pads) for inspection.
- `--cancel <run_id>`: `kv_set("counselors." + run_id + ".cancel", "1")`; look up coordinator pid via `kv_get("counselors." + run_id + ".coordinator_pid")` and `mcp__solo__stop_process(pid)`. Confirm to the user.

After handling any of these, stop. Do not proceed to the run.

## Step 3: Resolve agents (the panel)

Precedence (highest first):
1. `--agents <a,b,c>` (or `-a`) — split CSV, validate each against the agent-tools list (case-insensitive match on the tool's `name` field). Duplicates are allowed and meaningful (e.g. `claude,claude` runs two independent claude instances).
2. `--group <name>` — `kv_get("counselors.group." + name)`. If missing, error: "Group '{name}' not found. Run `/solo-counselors --list-groups` to see what's saved."
3. `kv_get("counselors.group.last")` → fall back to that group's agents if it exists.
4. **First-run UX**: if no group exists at all (kv_list shows no `counselors.group.*` keys): use AskUserQuestion to ask the user to pick agents from the `list_agent_tools` results (multi-select — encourage 2+ for a real panel), then ask for a name to save as. Allow "skip" to use only the first available agent without persisting. If they save, `kv_set("counselors.group." + name, csv)` and `kv_set("counselors.group.last", name)`.
5. Final fallback: first agent from `list_agent_tools`.

For each agent, resolve its `agent_tool_id` (the `id` field) from the agent-tools listing (needed for `spawn_agent`), and its `tool_type` (used by the strict read-only gate — see *Read-only enforcement*). **Validate against `enabled == true` entries only**, on every resolution path (explicit `--agents`, a `--group`, or `last`): a name that exists in the listing but is `enabled: false` is not spawnable, so reject it with a clear message — "Agent '{name}' exists in Solo but is disabled; enable it in Solo or drop it from the panel." — rather than letting it fail at spawn. (Preflight already guarantees ≥1 enabled agent exists overall; this guards the specific panel the user asked for.)

**Single-agent warning (run mode)**: if the resolved panel has only one distinct agent and the user is in `run` mode, warn: "Run mode's diversity comes from different models — a one-agent panel is just a single opinion. Add another agent, or repeat one (e.g. `--agents {b},{b}`), for a real panel." Then proceed anyway.

## Step 4: Resolve enrichment + mode settings

**Common to both modes:**
- Determine `prompt_source`: `file` if `-f/--file` was used, else `inline`.
- Resolve `--preset` (singular): if given, verify `presets/<name>.md` exists (else error listing valid presets); read its front-matter for `defaultRounds` / `defaultReadOnly` / `verify`.
- Compute `ENHANCE` (whether to run discovery + prompt-writing):
  - preset given → `on` (a preset is a discovery/prompt-writing workflow; `--no-inline-enhancement` does not disable it).
  - else no preset, `prompt_source == file` → `off` (use the file as provided).
  - else no preset, `prompt_source == inline` → `on`, unless `--no-inline-enhancement` → `off`.
- Resolve `--read-only`: explicit value, else preset `defaultReadOnly`, else `best-effort`.
- Resolve `--verify` (`off` | `cross`): **explicit `--verify` flag wins**, else the preset's `verify` front-matter value, else `off`. (So `security`/`bughunt`/`architecture` default to `cross`; the other presets and no-preset runs default to `off`; and an explicit `--verify` always overrides the preset.) Then sanity-check the panel: `cross` needs **≥2 workers** to cross-check (a different model is preferred but not required). If `verify == cross` but the dispatch list has only one worker, **downgrade to `off`** with a one-line warning ("verification needs ≥2 workers — skipping; add or repeat an agent for a cross-check") — don't error. Verification adds a wave of agent processes, so note when it's on that the run will spawn extra (verifier) processes after collection.

**Loop only:**
- Resolve `--rounds`: explicit value, else preset `defaultRounds`, else 3. Take `--convergence-threshold` / `--on-timeout` as given.
- In **run** mode, if the user passed `--rounds` / `--convergence-threshold` / `--on-timeout`, warn that they're ignored — run is single-round.

## Step 5: Build dispatch list

One worker **per agent entry** (duplicates allowed), all receiving the same prompt. E.g. agents=[claude, gemini, claude] → `[(claude, 0), (gemini, 1), (claude, 2)]`, each entry carrying its `agent_tool_id`.

Every worker spawns at once — there is no cap (in loop mode this is the same one agent panel each round). If the panel is unusually large (say >8 workers), warn the user before spawning, since each worker is a full agent process.

## Step 6: Dry-run handling

If `--dry-run`: print the dispatch plan as a table (one row per worker: index, agent), plus the resolved values for `--mode`, `--read-only`, `--verify`, `--duration`, `--preset`, `ENHANCE`, and — in loop mode — `--rounds` (a **per-worker** cap — each model advances through its own rounds independently), `--convergence-threshold` (per-worker), `--on-timeout`. When `--verify cross`, also print the planned **verifier → target** assignment (apply the cyclic, model-diversity-preferring rule from VERIFY-WAVE to the resolved panel: each worker checked by the next, interleaving models where possible) so the user can see who would check whom before spawning. Exit without spawning anything.

## Step 7: Launch the run

**Common to both modes** first:
1. Generate `run_id` — short uuid (8 hex chars is fine; `Bash` can produce it via `openssl rand -hex 4` or `python3 -c "import uuid; print(uuid.uuid4().hex[:8])"`).
2. `kv_set("counselors." + run_id + ".args", JSON.stringify({mode, agents, preset, enhance, verify, rounds, ...}))`.
3. Compute `duration_ms` from `--duration` by arithmetic (`30m` → `1_800_000`) — a **relative** budget, not a wall-clock deadline (Solo has no clock tool; see *Timekeeping* in `orchestration-core.md`). Compute `agents_csv` = the resolved panel joined (e.g. `claude,gemini,claude`).

Then branch by mode.

### Step 7a — Run mode: coordinate inline (no separate coordinator)

In run mode **you are the coordinator** — do the work in this session; do **not** spawn a separate coordinator agent. Read what you'll need: `Read(references/orchestration-core.md)`, `Read(references/worker-preamble.md)`, and — when `ENHANCE == on` — `Read(references/prompt-writing.md)`, `Read(references/execution-boilerplate.md)`, plus `Read(<preset_path>)` if a preset is set; and — when `verify != off` — `Read(references/verifier-preamble.md)`. Then drive the core routines yourself:

1. `kv_set("counselors." + run_id + ".status", "running")`. `scratchpad_write("counselors.{run_id}.progress", "[coordinator-start mode=run inline enhance={ENHANCE}]", tags=["counselor","{run_id}","progress"])` — **record `progress_id`**.
2. **SPAWN-WORKERS** (core) — **prewarm**: `dispatch`, `read_only_policy`, `name_suffix = ""` → `spawned`. Boot the panel **now** so the workers' CLI cold-start overlaps the discovery in the next step. **Fire the spawns and move straight to step 3 — do not wait for boot or `get_process_status` first; the CLIs boot in the background while you run discovery.** (With `ENHANCE == off` there's no discovery to overlap, so this only hides the boot; the batched spawn is still correct.)
3. **BUILD-EXECUTION-PROMPT** (core): `user_prompt` (resolve `-f`; fold in `--context`), `prompt_source`, `ENHANCE`, `preset_path`, the prompt-writing text, the execution-boilerplate text, `progress_id`, `duration_ms`. → `execution_prompt` + **`prompt_id`**. When `ENHANCE == on` you run repo discovery here with **Read/Grep/Glob** (you have full file access) **while the workers boot**; cap discovery at ~⅓ of `duration_ms`.
4. **DISPATCH-WAVE** (core): `spawned` (from step 2), `execution_prompt`, `prior_round_context_by_index = {}` (none — run is single-round), the worker-preamble text. Sends the task to all prewarmed workers in one batch. → `workers`.
5. **COLLECT-WAVE** (core): `workers`, `round_seg = ""`, `extra_tags = []`, `progress_id`, `wave_budget_ms` = the whole `duration_ms` when `verify == off`; when `verify != off`, leave a margin (give collection ~⅔ and reserve ~⅓ for the verify wave). No `deadline_timer_id` (run mode lets COLLECT-WAVE arm and poll its own wave-deadline cutoff) and no `on_finish` (run is single-round). Its per-finisher `_any` drain scrapes and closes each worker the instant it idles — fast workers aren't held behind slow ones. → outputs (in memory) + worker `scratchpad_id`s.
6. **VERIFY-WAVE** (core) — **only when `verify != off` and the drain finished within budget** (skip it if the cutoff fired with workers unfinished — that's the `partial` case and the budget is spent): `findings_by_worker` (the outputs you just collected, keyed by worker), `panel = dispatch`, `verify_mode = verify`, `read_only_policy`, the verifier-preamble text, `progress_id`, `wave_budget_ms` = the reserved remainder. → `verdicts` + `verify_ids`. When skipped, both are empty.
7. **SYNTHESIZE** (core): outputs, `agents_csv`, `rounds = 1`, `preset_note` (` · preset {name}` or empty), `verdicts` (from step 6). → `summary_id`.
8. **Finalize** — call **CLASSIFY-AND-FINALIZE** (core): `worker_outcomes` (from COLLECT-WAVE), `cut_short_status` (`partial` if the drain's `duration_ms` cutoff fired with workers still unfinished; empty if the drain collected cleanly — run mode has no separate `timeout` exit), `cancelled` (true only if the user interrupted this session mid-run — `stop_process` any live workers first, then SYNTHESIZE what you hold), the worker `scratchpad_id`s, the `verify_ids` (from step 6; empty if verification was off or skipped), `progress_id`, and `prompt_id`. It settles `done`/`partial`/`cancelled`, archives **only** on a clean `done` (so 1 of 3 agents failing while the other 2 succeed is still `done`; verify pads archive alongside the worker pads), and writes the status.
9. Present **inline** per Step 9 — you already hold the summary. **There is no poll loop; the loop-only Step 8 does not apply to run mode.**

> **Run-mode idle-wait.** `COLLECT-WAVE` runs its per-finisher `_any` drain from this **external** session (your live Claude Code session, not a Solo-managed agent), so apply the *Idle-drain pattern* with the external-session fallbacks **on every iteration**. Each `timer_fire_when_idle_any` arm: first harvest the response's `already_idle` workers and collect them immediately — no body needed, and this also covers the fast-worker race where someone finished before you armed. Then, only if some are still busy, the `WORKER-IDLE` body is the primary wake — but delivery to an external session is **not guaranteed**, so if it doesn't arrive within a short grace, stop waiting and re-check each still-running worker's `get_process_status`, pacing with a short `Bash("sleep …")` between checks, then re-scan and re-arm on whoever's left. Because the drain processes finishers as they idle, a fast worker is scraped and closed without waiting on a slow peer. The wave's clock-free **cutoff** is a one-shot `timer_set` deadline that COLLECT-WAVE arms and polls via `timer_list` (its firing doesn't depend on body delivery either). The **VERIFY-WAVE** (when `verify != off`) drains the same way from this same external session. Loop mode's detached coordinator gets reliable body delivery, so it simply ends its turn between arms.

### Step 7b — Loop mode: spawn a detached coordinator

1. `kv_set("counselors." + run_id + ".status", "spawning")`.
2. Pick the coordinator agent: prefer `claude-code` from the agent-tools list (it needs Read/Grep/Glob for discovery); fall back to the first available with a warning to the user.
3. `mcp__solo__spawn_agent(agent_tool_id=<coordinator_tool_id>, name="counselor-coord-" + run_id, include_agent_instructions=true)` → `coord_pid` (keep the response's optional `agent_instructions` — the coordinator's own bootstrap).
4. `kv_set("counselors." + run_id + ".coordinator_pid", coord_pid)`.
5. `Read(references/coordinator-loop-prompt.md)`; substitute all `{{...}}` placeholders with concrete values from this invocation:
   - `{{RUN_ID}}` ← run_id
   - `{{DISPATCH_JSON}}` ← JSON of dispatch list with `agent`, `agent_tool_id`, `index`
   - `{{USER_PROMPT_OR_PATH}}` ← user's prompt text (read from `-f` file if provided; fold in `--context` file contents if given)
   - `{{READ_ONLY_POLICY}}` ← resolved read-only policy
   - `{{VERIFY_MODE}}` ← resolved verify (`off` | `cross`)
   - `{{DURATION_MS}}` ← duration_ms (relative budget; the coordinator arms its own deadline timer from it)
   - `{{ORCHESTRATION_CORE_PATH}}` ← absolute path to references/orchestration-core.md
   - `{{WORKER_PREAMBLE_PATH}}` ← absolute path to references/worker-preamble.md
   - `{{VERIFIER_PREAMBLE_PATH}}` ← absolute path to references/verifier-preamble.md
   - `{{PROMPT_SOURCE}}` ← `inline` | `file`
   - `{{ENHANCE}}` ← `on` | `off`
   - `{{PRESET_PATH}}` ← absolute path to `presets/<name>.md`, or empty string
   - `{{ROUNDS}}` ← resolved rounds
   - `{{CONVERGENCE_THRESHOLD}}` ← threshold
   - `{{ON_TIMEOUT}}` ← policy
   - `{{ROUND_CONTEXT_TEMPLATE_PATH}}` ← absolute path to references/round-context-template.md
   - `{{CONVERGENCE_RUBRIC_PATH}}` ← absolute path to references/convergence-rubric.md
   - `{{PROMPT_WRITING_PATH}}` ← absolute path to references/prompt-writing.md
   - `{{EXECUTION_BOILERPLATE_PATH}}` ← absolute path to references/execution-boilerplate.md
6. `mcp__solo__send_input(process_id=coord_pid, input=<the `agent_instructions` (if any) then a blank line, followed by the substituted coordinator text>, submit=true, wait_ms=2000)`.
7. Verify via `mcp__solo__get_process_status(coord_pid)` that the coordinator is alive and accepted input. If not, error and surface what went wrong.
8. Tell the user: `Coordinator spawned (pid={coord_pid}, run_id={run_id}). Polling…` Then proceed to Step 8.

## Step 8: Poll loop (loop mode only)

Run mode skips this — it already holds the result.

The coordinator is detached and **timer-driven** (it arms its own deadline timer and wakes itself on worker-idle), so this poll is only to stay attached and surface progress — the run does not depend on it. Timer-body delivery to an external session like yours isn't *guaranteed* (it sometimes arrives, but don't count on it to drive the poll), so pace the re-check with a short `sleep` (e.g. `Bash("sleep 15")`) between calls rather than busy-looping. Each tick: `mcp__solo__kv_get("counselors." + run_id + ".status")`. Stop on `done`, `cancelled`, `error`, `timeout`, `partial`. Hard ceiling ≈ `duration_ms` + 30s grace (count your own ticks — don't read a clock).

While polling, optionally tail progress for a one-liner update if many minutes pass: `scratchpad_list(tags=[run_id])` → find `counselors.{run_id}.progress` → `scratchpad_tail(scratchpad_id, lines=5)`.

If you reach the ceiling with status still `running`, surface a warning and offer to either keep polling or run `--cancel`. (Detaching is safe — the coordinator keeps going; reconnect later with `--status`.)

## Step 9: Present results

Both modes land here. **Run mode** already holds the summary text (and wrote `counselors.{run_id}.summary`) — show it verbatim now. **Loop mode** reads it back: reads are by `scratchpad_id`, so `scratchpad_list(tags=[run_id])` → find `counselors.{run_id}.summary` → `scratchpad_read(scratchpad_id)` → show verbatim.

Then, by final status:

- **`done`**: the summary is the only visible pad (everything else was archived). Tell the user the per-agent outputs, any **verification verdicts**, progress, and (loop) the built prompt were **archived**, not deleted — they remain in Solo and can be browsed there to drill into an individual agent's review or the verdicts on it. Mention the run_id so the user can `--status` or share it later.
- **`partial`**: nothing was archived — `scratchpad_list(tags=[run_id])` also returns the per-agent pads (`counselors.{run_id}.worker.{agent}-{index}`, with `.r{N}.` per round in loop mode) and any verify pads (`counselors.{run_id}.verify.{verifier}-{target_index}`). Read the summary and list those names for drill-down.
- **`cancelled` / `error` / `timeout`**: report status, find `counselors.{run_id}.progress` in the list and `scratchpad_tail` it, and list whatever worker pads exist (none are archived in these states).

**Reap the coordinator (loop mode only).** Once you've presented — on *any* terminal status — the loop coordinator is dead weight (its summary/status/pads are all persisted). Close it: `mcp__solo__close_process(coordinator_pid)` (use the pid you hold from Step 7b, or `kv_get("counselors." + run_id + ".coordinator_pid")`). It may have already despawned itself or been `stop_process`ed by `--cancel` — **ignore an "already closed / not found" error**. Run mode has no separate coordinator to reap (the workers were already closed during collection).

**Clean up run state (clean `done` only, both modes).** Mirroring the scratchpad archiving, after presenting a clean `done` (and, loop mode, reaping the coordinator *first* — the reap needs `coordinator_pid`), delete this run's transient KV keys: `kv_list(prefix="counselors." + run_id + ".")` → `mcp__solo__kv_delete` each returned key (covers `.status`, `.args`, `.coordinator_pid`, `.cancel`, and any future per-run key). They're pure operational state with no value once the run is over, and otherwise accumulate per activation. The visible `counselors.{run_id}.summary` scratchpad is then the run's **only** surviving artifact. **Never** touch the persistent `counselors.group.*` / `counselors.preflight` keys — different namespace. On `partial` / `timeout` / `cancelled` / `error`, **keep** the KV keys (just as nothing is archived) so the run stays inspectable via `--status`.

## Scratchpad naming & lifecycle

All of a run's pads live under the `counselors.{run_id}.` prefix and carry the `{run_id}` tag, so `scratchpad_list(tags=["{run_id}"])` enumerates a run (returning ids + names; **archived pads are excluded from the list**). Reads/tails/archives are keyed by `scratchpad_id`, not by name — list first, then read by id.

| Pad | Name | Lifecycle |
|---|---|---|
| Master report | `counselors.{run_id}.summary` | **stays visible** — the deliverable |
| Progress log | `counselors.{run_id}.progress` | archived on a clean `done` |
| Built prompt (loop) | `counselors.{run_id}.prompt` | archived on a clean `done` |
| Per-agent output | `counselors.{run_id}.worker.{agent}-{index}` (loop: `…worker.r{N}.{agent}-{index}`) | archived on a clean `done` |
| Verifier verdicts | `counselors.{run_id}.verify.{verifier}-{target_index}` (only when `--verify cross`) | archived on a clean `done` |

On a clean `done` the coordinator archives everything except the summary, so a finished run shows a **single** pad. Archiving hides, it does not delete — archived pads remain in Solo. On `partial` / `timeout` / `cancelled`, nothing is archived, so the per-agent pads stay visible for inspection.

## Read-only enforcement

The worker preamble enforces this in prompt; under `--read-only strict` additionally pass an agent-specific allowlist via `extra_args` when calling `spawn_agent` for workers (Claude Code: `["--allowedTools", "Read,Glob,Grep,WebFetch,WebSearch"]`). The `--allowedTools` flag is Claude-CLI-specific, so gate it on the agent's `tool_type` from `list_agent_tools`: apply the allowlist only when `tool_type == "claude"`. For any other `tool_type` (`gemini`, `codex`, `copilot`, `generic`, …) under `strict`, **don't refuse it and don't ask the user** — automatically **downgrade that one agent to best-effort** (the in-prompt READ-ONLY block, which binds every backend), note the per-agent downgrade, and keep going. This preserves the full multi-model panel — the whole point of counselors — while `claude`-type agents still get the hard allowlist. So `strict` means "hard sandbox where supported, in-prompt enforcement everywhere else," and a preset defaulting to `strict` (e.g. `bughunt`, `security`) never collapses a mixed panel to Claude-only.

**Verifiers** (`--verify cross`) are restricted exactly like workers — they only re-read code to check claims — so the same policy applies to them: the verifier preamble carries the in-prompt READ-ONLY block, and under `strict` the `tool_type`-gated `--allowedTools` allowlist is wired in for Claude verifiers, with the same per-agent best-effort downgrade for non-Claude backends.

Note: the **coordinator** is NOT read-only — it needs to call MCP tools to orchestrate, and it uses Read/Grep/Glob for the discovery phase (both modes). Only workers and verifiers are restricted. (In run mode the coordinator is *this skill session*, which already has full access; in loop mode it's the spawned coordinator agent.)

## Operating notes

- Be terse in chat. Brief updates between major steps. Don't narrate every Bash/MCP call.
- **Durability is loop-only.** The detached loop coordinator survives outer Claude Code being closed — kill CC mid-loop and reconnect with `/solo-counselors --status <run_id>`. **Run mode coordinates inline**, so closing CC mid-run orphans it (the spawned workers keep running with nobody to collect them); run is short, so this is an accepted trade. If a user needs a long review to survive disconnection, that's what loop mode is for. Once a loop reaches a terminal state the coordinator **despawns itself** (and the skill / `--status` reaps it as a backstop), so no idle agent lingers after completion.
- All run state lives under `counselors.<run_id>.*` (KV keys + scratchpads). On a clean `done` the intermediate pads are **archived** (by the coordinator — the skill in run, the detached agent in loop) and the per-run **KV keys are deleted** (by the skill / `--status`, after presenting; see Step 9), leaving the single visible `counselors.<run_id>.summary` pad as the run's only artifact. Archived pads are recoverable; deleted KV is gone (pure operational state). On non-clean endings nothing is archived or deleted — everything stays for inspection. The persistent `counselors.group.*` keys are a separate namespace and are never touched. (Edge case: a loop you detach from and never reconnect to leaves its KV until the next `--status`, which cleans it.)
- The diversity axis is **models**, not personas. To get more perspectives, add agents (or repeat one). For a domain-focused review use `--preset <name>`; pair it with `--mode loop` for a multi-round deep-dive.
- **Per-finisher pipelining.** Workers are **prewarmed** (spawned before discovery so their CLI boot overlaps it) and collected with an idle-**drain**: each worker is scraped and closed the moment it goes idle, never blocked on a slower peer. In **loop** mode this extends across rounds — each model advances to its next round the instant it finishes the current one (its next round depends only on its **own** prior findings, so there's no cross-worker barrier), making `--rounds` a per-worker cap; a fast model can complete more rounds than a slow one inside the same budget. The whole drain is bounded by a single clock-free deadline timer (Solo has no clock).
- **Verification** (`--verify cross`) turns the panel's model diversity into a checks-and-balances step: after collection, each agent's findings are re-checked by a *different* agent (cyclic assignment, model-diverse where possible) that rules them confirmed / refuted / uncertain. The synthesis prioritizes confirmed findings and segregates refuted ones into a "Disputed / likely false-positive" section instead of dropping them. It runs once, end-of-run (after the final loop round), and needs ≥2 finding-producing workers. Because it spawns an extra wave of processes, it defaults **on** for `security`/`bughunt`/`architecture` (costly false positives, or judgement calls worth a second model) and **off** elsewhere; an explicit `--verify off|cross` overrides the preset.
- The 7 built-in presets live in `presets/` and apply to **both** modes. The user can add more by dropping new markdown files with the same front-matter shape (`name`, `defaultRounds`, `defaultReadOnly`) and the `Discovery focus` / `What the written prompt should emphasize` sections.
