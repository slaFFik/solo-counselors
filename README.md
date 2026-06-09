# /solo-counselors

An agent skill that runs **parallel, multi-agent code review** by orchestrating other AI agents through the **Solo MCP** server.

## Why this exists

The original `counselors` by Aaron spins up reviewers via `claude -p` subprocesses. That ties every run to the Claude binary, to your live session, to one model, and Anthropic's policy around programmatic calls of `claude -p`.

`solo-counselors` moves the orchestration onto Solo MCP instead, which buys three things:

- **Agent independence** — the panel runs on `claude`, `gemini`, `codex`, `copilot`, `antigravity`, or any mix Solo exposes. The whole point is a *second opinion from a different model*.
- **Durability (loop mode)** — a `loop` review is driven by a detached Solo-managed coordinator agent, not your terminal. Close Claude Code mid-loop and the run keeps going; reconnect later with `--status`. (A `run` is short and coordinates inline — see *How it works*.)
- **No `claude -p` dependency** — orchestration uses Solo's spawn/timer/KV/scratchpad primitives, so it doesn't shell out to the Claude CLI.

The point is the same as the original: one reviewer has blind spots, several **different models** working the same task independently catch far more. You get one synthesized report instead of a wall of overlapping notes.

## Installation

`solo-counselors` is a skill — install it by cloning this repo into your `skills/` directory.

**For all projects** (personal skill):

```bash
git clone https://github.com/slaFFik/solo-counselors ~/.claude/skills/solo-counselors
```

**For a single project** (checked in alongside the code):

```bash
git clone https://github.com/slaFFik/solo-counselors .claude/skills/solo-counselors
```

The folder name must be `solo-counselors` so Claude Code (or any other agent that supports skills) can discover the skill. Start a new session (or reload skills if agent supports that) and invoke it with `/solo-counselors`.

### Requirements

- [Solo MCP](https://soloterm.com/docs/integrations/mcp-server) configured, with the KV store enabled.
- At least one [Solo agent](https://soloterm.com/docs/agents/what-are-agents) (`claude`, `gemini`, `codex`, `copilot`, `antigravity`, `grok`) — two or more for a meaningful `run` or `loop` panel.

## Usage

```
/solo-counselors "<task>" [flags]
```

### Run flags

These flags are also available in `loop` mode.

| Flag | Meaning | Default |
|---|---|---|
| `<prompt>` or `-f/--file <path>` | The task to review. Either positional text (enriched by default — discovery + prompt-writing runs) or read from a file via `-f`/`--file`. A **file** prompt is sent **as-is**; enrichment is skipped unless you also pass `--preset`. | required |
| `--mode <run\|loop>` | One-shot or iterative, can be omitted | `run` |
| `--agents <a,b,c>` (`-a`) | Agent panel — one worker each, dups allowed | first available |
| `--group <name>` | Use a saved agent group, or create a new one if doesn't exist | last-used |
| `--read-only <strict\|best-effort\|off>` | Worker mutation policy | preset default or `best-effort` |
| `--verify <off\|cross>` | Adversarial cross-check of findings before synthesis (one agent re-checks another's) | preset `verify` or `off` |
| `--context <paths>` | Files to attach to the prompt | — |
| `--preset <name>` | Shape discovery + prompt-writing (one preset) | none |
| `--no-inline-enhancement` | Skip enrichment for an inline prompt (send it raw) | off |
| `--duration <e.g. 30m>` | Total time budget | `15m` run / `45m` loop |
| `--dry-run` | Print plan, don't spawn | off |
| `--status <run_id>` / `--cancel <run_id>` | Reconnect / cancel a run | — |
| `--list-groups` / `--save-group <name>=<a,b,c>` / `--delete-group <name>` | Manage agent groups (list / save / delete) | — |
| `--list-presets` | List available presets, then exit | — |

### Loop-only flags

They are added on top of the `run` flags.

| Flag | Meaning | Default |
|---|---|---|
| `--rounds <N>` | Max rounds before stopping | preset default or 3 |
| `--convergence-threshold <ratio>` | Early-stop when a round's *new* findings fall below this fraction of the prior round's | 0.3 |
| `--on-timeout <abort\|continue>` | What to do if a round times out | `continue` |

### Examples

```
/solo-counselors "review src/auth for races" --agents claude,gemini,codex
/solo-counselors "hunt for bugs in the worker pool" --preset bughunt --agents claude,gemini
/solo-counselors "is this diff safe to merge?" --agents claude,claude --no-inline-enhancement
/solo-counselors "audit the checkout flow" --mode loop --preset security --rounds 4
/solo-counselors "deep review of the payments module" --mode loop --agents claude,gemini
/solo-counselors "review the auth module" --agents claude,gemini,codex --verify cross
/solo-counselors "quick bug scan, skip the cross-check" --preset bughunt --verify off
```

## Presets

A **preset** shapes a review run: it tells the discovery phase *what to look for* and the prompt-writing phase *what to emphasize*. It is a **workflow, not a per-agent persona** — every agent on the panel gets the same preset-shaped prompt. Pass one with `--preset <name>` (one per run); list them anytime with `--list-presets`.

| Name | Description |
|---|---|
| `architecture` | Finds deepening opportunities — refactors that turn shallow modules into deep ones, for better testability and navigability. |
| `bughunt` | Hunts real correctness bugs, edge-case failures, and missing tests that allow regressions. |
| `contracts` | Audits API contract drift across server handlers, shared types, clients, validators, and tests. |
| `hotspots` | Finds high-impact performance bottlenecks, with emphasis on asymptotic complexity and scaling behavior. |
| `invariants` | Finds state-synchronization issues, impossible states, and state-management anti-patterns. |
| `regression` | Audits regression risk — behavior changes that break existing users or dependent systems. |
| `security` | Finds exploitable vulnerabilities with an attacker's mindset — real exploits, not theoretical concerns. |

Beyond the shared read-only `strict` default, some presets carry their own defaults: `architecture`, `bughunt`, and `security` enable adversarial cross-verification (`--verify cross`), and `hotspots` runs 4 loop rounds instead of 3. Any flag you pass overrides the preset default.

**Add your own** — drop a markdown file in `presets/` using the same front-matter shape (`name`, `defaultRounds`, `defaultReadOnly`, `verify`) followed by a discovery-focus and a prompt-emphasis section. It shows up in `--list-presets` automatically.

## Features

- **Models-as-experts fan-out** — one worker per `--agents` entry, all given the identical prompt. Duplicates are allowed and useful (`--agents claude,claude` = two independent claude instances). Agreement across different models is a strong signal in the synthesis.

- **Two execution modes** (both enrich the prompt first):
  - `run` (default) — single-shot. Enriches the prompt, then fans it out once, inline. A fast second opinion.
  - `loop` — multi-round. Builds the same enriched prompt, then rounds 2+ feed each agent **its own** prior findings back (never its peers' — no cross-model pollution) so each model refines instead of repeating. **Each model advances through its rounds at its own pace** — the coordinator re-dispatches a worker the instant it finishes a round rather than waiting for the whole panel, so a fast model goes deeper while a slow one is still early. Runs durably via a detached coordinator; each worker stops early on its own **convergence** (when one of its rounds turns up few genuinely new findings).

- **Presets (both modes)** — seven built-in review workflows that shape *what discovery looks for* and *what the prompt emphasizes*, plus your own. See [Presets](#presets).

- **Prompt enrichment** — for an inline prompt (either mode) the discovery + prompt-writing phase runs by default (skip it with `--no-inline-enhancement`); a `-f` file prompt is used as-is unless a preset is set.

- **Project-context parity** — each CLI auto-loads only its own context file (`CLAUDE.md` for Claude, `AGENTS.md` for codex/antigravity, `GEMINI.md` for gemini; copilot and grok read `CLAUDE.md` too), so a repo documented only in `CLAUDE.md` would leave gemini/codex/antigravity panelists investigating from scratch. Every worker and verifier is instructed to read whichever of those files exist at the repo root before analyzing, and discovery names the ones it found in the enriched prompt.

- **Saved groups** — persist a panel of agents to KV and reuse it (`--save-group`, `--list-groups`, `--group`).

- **Durable & resumable (loop)** — loop state lives in `counselors.<run_id>.*` KV keys and scratchpads, driven by a detached coordinator; reconnect with `--status <run_id>`, stop with `--cancel <run_id>`. A `run` coordinates inline, so it lives and dies with your session — short by design.

- **Read-only enforcement** — `strict` / `best-effort` / `off`. Workers are sandboxed; under `strict`, Claude-family agents get a hard tool allowlist (`--allowedTools`) and any other backend automatically falls back to in-prompt enforcement for that one agent (no prompt, panel kept whole) — so `strict` (including a preset that defaults to it) never silently drops non-Claude members. The coordinator stays writable so it can orchestrate (and read the repo during loop discovery). Verifiers (below) are sandboxed the same way.

- **Adversarial cross-verification (`--verify cross`)** — optionally, after the panel reports, each agent's findings are re-checked by a *different* agent (read-only, deterministic cyclic assignment, model-diverse where possible) that opens the cited code and rules every finding **confirmed / refuted / uncertain**. The synthesis then prioritizes confirmed findings and moves refuted ones into a **"Disputed / likely false-positive"** section instead of shipping them. This turns the panel's model diversity into a checks-and-balances step against self-preferential bias and confident-but-wrong findings — agreement across models isn't the same as an independent model actually standing behind a finding. It runs once (end-of-run, after the final loop round), needs ≥2 finding-producing agents, and is set per preset: **on for `security`/`bughunt`/`architecture`**, off elsewhere; override with `--verify off|cross`.

- **One master report** — every agent's raw findings are deduped and merged (with per-model attribution) into a single human-facing synthesis; the per-agent scratchpads are archived for drill-down. See [How it works](#how-it-works).

- **Guardrails** — preflight verifies Solo is reachable, a project is scoped, KV is enabled, and an agent exists. `--dry-run` prints the dispatch plan without spawning; `--duration` bounds total runtime (a single clock-free deadline timer). The whole panel is prewarmed at once — spawned before discovery so each agent's boot overlaps it.

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

loop only:    each agent re-runs rounds 2..N AT ITS OWN PACE; each sees only ITS OWN
              previous-round findings (never its peers') — to refine and go deeper
              without repeating itself. The coordinator re-dispatches a worker the
              moment it finishes a round (no panel barrier), merging models at synthesis.
```

So the modes differ only after the prompt exists: **`run`** fans it out once; **`loop`** iterates rounds 2..N (and runs durably via the coordinator).

1. The **skill** parses your invocation, runs preflight checks, and resolves the agent panel. In **run** it then coordinates the fan-out itself; in **loop** it spawns a detached coordinator and polls for status.
2. The **coordinator** (your session in run, a separate Solo agent in loop) builds the enriched prompt (repo-discovery → prompt-writing, both modes) — **prewarming the panel during discovery so the agents' boot overlaps it** — fans the prompt to one worker per agent, and **collects each worker the moment it goes idle** (a per-finisher drain, so a fast model is never held behind a slow one), and — when `--verify cross` is in effect — runs an adversarial cross-verification wave (each agent's findings re-checked by a different agent) before writing the synthesis. Building the prompt and the dispatch/collect/verify/synthesize/archive mechanics are all shared via `references/orchestration-core.md`.
3. The **workers** each review the same prompt independently, in read-only mode — differing only by which model they are.

**Scratchpads → one synthesis** — each worker writes its full review into its **own Solo scratchpad** as it goes: raw, per-model, one pad per agent (per round in loop). Those pads are the working notes, not the deliverable. Once the panel finishes (and the cross-verification wave, if `--verify cross` is on), the coordinator reads every pad, dedupes overlapping findings, merges what's left with per-model attribution, and writes a **single combined synthesis** to `counselors.<run_id>.summary`. **That synthesis is the human-facing report** — the one thing you're meant to read and act on, instead of diffing several overlapping walls of notes. On a clean run the per-agent pads are then archived (and the run's transient state deleted), so the synthesis is the only visible artifact — but archived ≠ gone, so you can still open an individual agent's pad in Solo to see the raw reasoning behind any line in the summary.

## Credits

Concept and presets inspired by [`counselors`](https://github.com/aarondfrancis/counselors) by Aaron Francis.
