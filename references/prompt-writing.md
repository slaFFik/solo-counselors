# Discovery + prompt-writing phase (both modes)

Before fanning out, the coordinator (the skill itself in run mode, the detached agent in loop mode) turns the user's task into a sharp, self-contained **execution prompt** that is fanned out — identically — to every agent in the panel. In loop mode this prompt is built once and reused for every round. Two phases:

## 1. Repo discovery

Explore the scoped project with read-only tools (Read, Grep, Glob) to gather the context the panel needs. Budget this tightly — aim to spend at most about a third of the time left until the deadline (half at the very most). The panel still needs time to run, and the collector only waits until that same deadline, so overspending here starves the workers.
- Locate the area named or implied by the user's task (paths, symbols, recent changes).
- Note the stack, key files, and entry points relevant to the task.
- If a preset is supplied, follow its **Discovery focus** section.
- If you cannot read the repo (no filesystem access), skip to prompt-writing using only the user's task, and note the limitation in the progress scratchpad.

## 2. Prompt-writing

Synthesize discovery into ONE execution prompt for the panel. It must:
- Restate the task crisply and name the concrete files/areas to examine (with paths).
- Fold in the preset's **What the written prompt should emphasize** section, if a preset is set.
- Be self-contained: an agent reading only this prompt should know exactly what to do.
- Describe what to investigate, **not** what to conclude — never prescribe findings.

Keep the written prompt focused (a few hundred words). The caller then appends the standard execution boilerplate to it.

## When this phase is skipped

If the prompt came from `-f`/stdin, or `--no-inline-enhancement` was set, do **not** run discovery or rewriting. Use the user's prompt verbatim as the base execution prompt. (The execution boilerplate is still appended.)
