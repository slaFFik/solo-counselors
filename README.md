# /solo-counselors

A Claude Code skill that runs **parallel, multi-agent code review** by orchestrating other AI agents through the **Solo MCP** server.

It's a reimplementation of Aaron Francis's [`counselors`](https://github.com/aarondfrancis/counselors) tool — same idea, different engine.

## Why this exists

The original `counselors` spins up reviewers via `claude -p` subprocesses. That ties every run to the Claude binary, to your live session, and to one model.

`solo-counselors` moves the orchestration onto Solo MCP instead, which buys three things:

- **Agent independence** — the panel runs on `claude`, `gemini`, `codex`, `copilot`, `antigravity`, or any mix Solo exposes. The whole point is a *second opinion from a different model*.
- **Durability (loop mode)** — a `loop` review is driven by a detached Solo-managed coordinator agent, not your terminal. Close Claude Code mid-loop and the run keeps going; reconnect later with `--status`. (A `run` is short and coordinates inline — see *How it works*.)
- **No `claude -p` dependency** — orchestration uses Solo's spawn/timer/KV/scratchpad primitives, so it doesn't shell out to the Claude CLI.

The point is the same as the original: one reviewer has blind spots, several **different models** working the same task independently catch far more. You get one synthesized report instead of a wall of overlapping notes.

## How it works

Both modes fan **one prompt** out to the panel — **one worker per agent** — in parallel, then synthesize the outputs. The diversity comes from running **different models** on the **identical** task. Both modes **enrich** the prompt the same way first; they differ only in **who runs the fan-out** and **how many rounds**.

**`run` — your session *is* the coordinator** (the loop's two top boxes collapse into one). It enriches the prompt, then fans it out **once**. Lower-latency and cheaper than loop, but tied to your terminal: close it mid-run and the run is orphaned.

```
         ┌─────────────────────┐
         │  You (skill)        │  ← also the coordinator
         │  • discover + write │     (no separate process)
         │  • fan out 1 prompt │
         │  • collect + synth  │
         └──────────┬──────────┘
                    │ same prompt → one worker per agent
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
┌───────────┐ ┌───────────┐ ┌───────────┐
│  claude   │ │  gemini   │ │   codex   │
└───────────┘ └───────────┘ └───────────┘

different models · identical task · independent · read-only
```

**`loop` — the skill spawns a separate, durable coordinator** that also runs repo-discovery + prompt-writing and survives your session closing:

```
┌──────────────────────┐            ┌──────────────────────┐
│  You (skill)         │            │  Coordinator         │
│  • parse args        │── spawns ─►│  • discover + write  │
│  • spawn coordinator │            │  • fan out / rounds  │
│  • poll status       │◄─summary───│  • collect + synth   │
└──────────────────────┘            └──────────┬───────────┘
                                               │ same prompt → one worker per agent (per round)
                                 ┌─────────────┼─────────────┐
                                 ▼             ▼             ▼
                           ┌───────────┐ ┌───────────┐ ┌───────────┐
                           │  claude   │ │  gemini   │ │   codex   │
                           └───────────┘ └───────────┘ └───────────┘

                           different models · identical task · independent · read-only
```

**How the prompt is built** — the same in both modes:

Neither mode fans out your raw one-liner. Both first build an **enriched execution prompt** — a repo-discovery → prompt-writing pass (guided by `--preset` if set). `-f` file prompts are used as-is, and `--no-inline-enhancement` opts an inline prompt out.

```
both modes:   your task ──► [ repo discovery ] ──► [ prompt-writing ] ──► enriched prompt
                            (guided by --preset, if one is set)

loop only:    rounds 2..N re-run the panel; each agent sees only ITS OWN previous-round
              findings (never its peers') — to refine and go deeper without repeating
              itself. Independence is preserved; the coordinator merges across models
              only at synthesis.
```

So the modes differ only after the prompt exists: **`run`** fans it out once; **`loop`** iterates rounds 2..N (and runs durably via the coordinator).

1. The **skill** parses your invocation, runs preflight checks, and resolves the agent panel. In **run** it then coordinates the fan-out itself; in **loop** it spawns a detached coordinator and polls for status.
2. The **coordinator** (your session in run, a separate Solo agent in loop) builds the enriched prompt (repo-discovery → prompt-writing, both modes), fans it to one worker per agent, waits for them all to finish, scrapes each one's output, and writes a synthesis. Building the prompt and the dispatch/collect/synthesize/archive mechanics are all shared via `references/orchestration-core.md`.
3. The **workers** each review the same prompt independently, in read-only mode — differing only by which model they are.

You see a concise synthesis in chat plus pointers to per-worker scratchpads for drill-down.

## Features

- **Models-as-experts fan-out** — one worker per `--agents` entry, all given the identical prompt. Duplicates are allowed and useful (`--agents claude,claude` = two independent claude instances). Agreement across different models is a strong signal in the synthesis.

- **Two execution modes** (both enrich the prompt first):
  - `run` (default) — single-shot. Enriches the prompt, then fans it out once, inline. A fast second opinion.
  - `loop` — multi-round. Builds the same enriched prompt, then rounds 2+ feed each agent **its own** prior findings back (never its peers' — no cross-model pollution) so each model refines instead of repeating. Runs durably via a detached coordinator; stops early on **convergence** (output shrinks below a threshold).

- **Presets (both modes)** — `bughunt`, `security`, `regression`, `contracts`, `invariants`, `hotspots`. A preset (one per run, via `--preset`) shapes *what the discovery phase looks for* and *what the written prompt emphasizes* — it is a workflow, not a per-agent persona. Add your own by dropping a markdown file in `presets/` with the same front-matter shape.

- **Prompt enrichment** — for an inline prompt (either mode) the discovery + prompt-writing phase runs by default (skip it with `--no-inline-enhancement`); a `-f` file prompt is used as-is unless a preset is set.

- **Saved groups** — persist a panel of agents to KV and reuse it (`--save-group`, `--list-groups`, `--group`).

- **Durable & resumable (loop)** — loop state lives in `counselors.<run_id>.*` KV keys and scratchpads, driven by a detached coordinator; reconnect with `--status <run_id>`, stop with `--cancel <run_id>`. A `run` coordinates inline, so it lives and dies with your session — short by design.

- **Read-only enforcement** — `strict` / `best-effort` / `off`. Workers are sandboxed; under `strict`, Claude-family agents get a hard tool allowlist (`--allowedTools`) and any other backend automatically falls back to in-prompt enforcement for that one agent (no prompt, panel kept whole) — so `strict` (including a preset that defaults to it) never silently drops non-Claude members. The coordinator stays writable so it can orchestrate (and read the repo during loop discovery).

- **One master report** — beyond the original, the coordinator writes a deduped combined summary (with per-model attribution) to `counselors.<run_id>.summary`. On a clean run the per-agent scratchpads are then **archived** (and the run's transient KV state is **deleted**), so a finished run leaves a single visible pad as its only artifact — but archived ≠ deleted, so you can still browse an individual agent's review in Solo if you want to drill in.

- **Guardrails** — preflight verifies Solo is reachable, a project is scoped, KV is enabled, and an agent exists. `--dry-run` prints the dispatch plan without spawning; `--duration` bounds total runtime. The whole panel spawns at once.

## Usage

```
/solo-counselors "<task>" [flags]
```

### Run flags

These flags are also available in `loop` mode.

| Flag | Meaning | Default |
|---|---|---|
| `<prompt>` or `-f/--file <path>` | The task to review. Either positional text (enriched by default — discovery + prompt-writing runs) or read from a file via `-f`/`--file`. A **file** prompt is sent **as-is**; enrichment is skipped unless you also pass `--preset`. | required |
| `--mode <run\|loop>` | One-shot or iterative | `run` |
| `--agents <a,b,c>` (`-a`) | Agent panel — one worker each, dups allowed | first available |
| `--group <name>` | Use a saved agent group | last-used |
| `--read-only <strict\|best-effort\|off>` | Worker mutation policy | preset default or `best-effort` |
| `--context <paths>` | Files to attach to the prompt | — |
| `--preset <name>` | Shape discovery + prompt-writing (one preset) | none |
| `--no-inline-enhancement` | Skip enrichment for an inline prompt (send it raw) | off |
| `--duration <e.g. 30m>` | Total deadline | `15m` run / `45m` loop |
| `--dry-run` | Print plan, don't spawn | off |
| `--status <run_id>` / `--cancel <run_id>` | Reconnect / cancel a run | — |
| `--list-groups` / `--save-group <name>=<a,b,c>` / `--delete-group <name>` | Manage agent groups (list / save / delete) | — |
| `--list-presets` | List available presets, then exit | — |

### Loop-only flags

They are added on top of the `run` flags.

| Flag | Meaning | Default |
|---|---|---|
| `--rounds <N>` | Max rounds before stopping | preset default or 3 |
| `--convergence-threshold <ratio>` | Early-stop when output shrinks below this | 0.3 |
| `--on-timeout <abort\|continue>` | What to do if a round times out | `continue` |

**Examples:**

```
/solo-counselors "review src/auth for races" --agents claude,gemini,codex
/solo-counselors "hunt for bugs in the worker pool" --preset bughunt --agents claude,gemini
/solo-counselors "is this diff safe to merge?" --agents claude,claude --no-inline-enhancement
/solo-counselors "audit the checkout flow" --mode loop --preset security --rounds 4
/solo-counselors "deep review of the payments module" --mode loop --agents claude,gemini
```

## Requirements

- Solo MCP configured in Claude Code, with the KV store enabled.
- At least one Solo agent (`claude`, `gemini`, `codex`, `copilot`, `antigravity`) — two or more for a meaningful `run` panel.

## Credits

Concept and presets inspired by [`counselors`](https://github.com/aarondfrancis/counselors) by Aaron Francis.
