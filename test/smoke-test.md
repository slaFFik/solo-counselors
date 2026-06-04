# Smoke test — one-liner

Fast, folder-agnostic manual check. `--no-inline-enhancement` skips repo discovery (no file
reads, so it doesn't matter which folder Solo is scoped to); each worker emits one fake
finding and stops, so a run finishes in seconds. Those trivial workers go idle almost
instantly — often *before* the collector even arms its idle timer — which is exactly the
case the per-finisher drain must handle via its `already_idle` harvest (the `_any` gotcha).
So a **prompt clean finish is itself the regression check** for the idle-drain.

## One-time setup — save a panel as a group

Saving a group once is faster than listing agents with `--agents` on every run, and it's how
you'd normally invoke the skill:

```
/solo-counselors --save-group smoke=claude,gemini
```

Swap `claude,gemini` for any agents you have configured (two distinct agents avoid the
"single distinct agent" warning; repeat one, e.g. `claude,claude`, for a single-model panel).
This just stores the group and exits — it doesn't run anything.

## Run mode (default, fastest)

```
/solo-counselors "SMOKE TEST — do not read or analyze any files; emit one fabricated low-severity finding titled 'smoke-test-ok' then stop." --group smoke --no-inline-enhancement --read-only off --duration 5m
```

If `smoke` is your last-used group you can drop `--group smoke` entirely — the skill falls
back to the last-used group.

## Verification wave (cross-check plumbing)

```
/solo-counselors "SMOKE TEST — do not read or analyze any files; emit one fabricated low-severity finding titled 'smoke-test-ok' then stop." --group smoke --no-inline-enhancement --read-only off --duration 6m --verify cross
```

Forces the end-of-run verifier wave on top of the run-mode path. After the two workers emit
their fake findings, each agent's finding is cross-checked by the other agent. This exercises
the wave **plumbing** — verifier spawn, collect, `## Verdict index` parse, verify-pad write,
archive — regardless of the verdict content; because the finding is fabricated and points at
no real file, the verdict will usually be `uncertain` (or `refuted`), which is expected here.
Needs the 2-agent `smoke` group: a single-agent panel will report "verification needs ≥2
workers — skipping" and skip the wave (also a valid result to eyeball).

## Loop mode (durability + coordinator reap)

```
/solo-counselors "SMOKE TEST — do not read or analyze any files; emit one fabricated low-severity finding titled 'smoke-test-ok' then stop." --group smoke --mode loop --rounds 2 --no-inline-enhancement --read-only off --duration 8m
```

## Plan only (instant, spawns nothing)

```
/solo-counselors "anything" --group smoke --mode loop --rounds 3 --dry-run
```

## Preset resolution (dry-run, instant)

A preset is data the skill reads at run time, so a dry-run confirms one is wired in without
spawning anything — the front-matter defaults flow through into the printed plan. Swap
`architecture` for whichever preset you're checking:

```
/solo-counselors "anything" --group smoke --preset architecture --dry-run
```

Eyeball the plan: `preset = architecture`, `ENHANCE = on` (a preset always enriches — dry-run
stops *before* discovery actually runs, so this stays instant and folder-agnostic), and
`--read-only = strict` + `--verify = cross` **resolved from the preset's front-matter** with
no explicit flags passed. Because `verify` is `cross` on the 2-agent `smoke` group, the plan
also prints the planned **verifier → target** assignment; on a single-agent panel it should
instead show verification downgraded to `off` ("needs ≥2 workers"). Nothing spawns.

## Real enriched run (manual — reads files, not fast)

The dry-run checks *wiring*; to exercise the actual discovery + architecture lens, point a
real run at a friction-rich repo (not folder-agnostic, takes minutes). Scope the prompt to a
subsystem for the sharpest result:

```
/solo-counselors "find deepening opportunities in <a module you know>" --preset architecture --agents claude,gemini --mode run
```

A healthy result names concrete deepening candidates in the architecture vocabulary
(module / seam / depth), ranks them by recommendation strength with an execution sequence,
classifies each candidate's dependencies, and keeps any incidental correctness bugs in a
separate lane. With `--verify cross` (the preset default) each finding is re-checked by a
different model before synthesis.

## What a healthy run looks like (eyeball this)

- Finishes promptly with no hang, and prints a synthesis plus a `run_id`.
- Per-finisher drain: the progress pad's `done` lines appear **as each worker finishes**, not all at once — the faster agent's output is scraped and its process closed before the slower one goes idle (with the trivial smoke workers both land within moments, but never as a single all-idle barrier).
- On a clean `done` the skill reports that the per-agent and progress pads were **archived**
  and the per-run KV keys **deleted**, leaving only `counselors.<run_id>.summary`.
- `/solo-counselors --status <run_id>` afterward still shows the summary — KV is gone, so it
  falls back to the summary pad, and it must **not** offer a poll loop.
- Loop only: no `counselor-coord-<run_id>` process is left running afterward.
- Loop only: each worker runs its rounds **independently** — `counselors.<run_id>.worker.r1.<agent>-<i>` then `…r2.…` pads appear **per worker** (the smoke panel does up to `--rounds 2` each, then converges on the repeated fake finding), and a faster agent can reach `r2` before a slower one finishes `r1`. Progress `done` lines are per-worker-per-round, not lockstep.
- Verify only: a `counselors.<run_id>.verify.<verifier>-<target_index>` pad appears per
  cross-check during the run; on clean `done` it's archived alongside the worker pads (the
  summary stays the only visible pad). The synthesis carries a per-finding **Verification**
  line and a Panel-health verification-coverage line.
