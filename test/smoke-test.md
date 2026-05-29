# Smoke test — one-liner

Fast, folder-agnostic manual check. `--no-inline-enhancement` skips repo discovery (no file
reads, so it doesn't matter which folder Solo is scoped to); each worker emits one fake
finding and stops, so a run finishes in seconds. Those trivial workers go idle almost
instantly — the exact case that used to hang the collector — so a **prompt clean finish is
itself the regression check** for the idle-wait fix.

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

## Loop mode (durability + coordinator reap)

```
/solo-counselors "SMOKE TEST — do not read or analyze any files; emit one fabricated low-severity finding titled 'smoke-test-ok' then stop." --group smoke --mode loop --rounds 2 --no-inline-enhancement --read-only off --duration 8m
```

## Plan only (instant, spawns nothing)

```
/solo-counselors "anything" --group smoke --mode loop --rounds 3 --dry-run
```

## What a healthy run looks like (eyeball this)

- Finishes promptly with no hang, and prints a synthesis plus a `run_id`.
- On a clean `done` the skill reports that the per-agent and progress pads were **archived**
  and the per-run KV keys **deleted**, leaving only `counselors.<run_id>.summary`.
- `/solo-counselors --status <run_id>` afterward still shows the summary — KV is gone, so it
  falls back to the summary pad, and it must **not** offer a poll loop.
- Loop only: no `counselor-coord-<run_id>` process is left running afterward.
