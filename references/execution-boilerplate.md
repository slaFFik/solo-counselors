# Execution boilerplate (appended to every execution prompt, both modes)

---

You are one of several agents reviewing this independently and in parallel; your peers cannot see your work. Be concrete and evidence-driven.

Report your findings as a prioritized list. Give each finding a stable id (F1, F2, …) and include:
- **severity**: critical | high | medium | low
- **confidence**: high | medium | low
- **location**: file path + function/symbol — the precise point where the issue manifests
- **evidence**: the concrete code pattern and why it matters
- **reachability**: how the issue is actually reached and why it bites — the entry point or caller, the trigger/inputs/conditions, and any guard, check, or sanitizer already on the path that does **not** prevent it. If you cannot identify a path that reaches it, lower **confidence** to match.
- **impact**: the user/system consequence
- **fix**: the minimal safe change

Calibrate consistently — you are one of several *different models*, and shared anchors keep the panel comparable:
- **severity** — critical: data loss or corruption, a security breach, or a crash/failure on a common path; high: wrong behavior on a common path, or an exploit that needs a precondition; medium: limited blast radius or an edge-case-only path; low: minor, with real but small impact.
- **confidence** — high: verified against the code with nothing seen that would prevent it; medium: likely, but depends on caller/runtime context you cannot fully see; low: plausible but the evidence is thin or the path that reaches it is unconfirmed.

Prioritize high-impact, high-confidence findings. Skip speculative nits and pure style unless they hide a real problem. If an area is clean, say so briefly rather than padding.

Summarize every finding in the mandatory **findings index** described in the output format below, keyed by the same ids. The coordinator keys deduplication, cross-round tracking, and independent verification off those ids and `file:symbol` locations, so make each location precise and concrete.
