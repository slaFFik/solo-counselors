# Execution boilerplate (appended to every execution prompt, both modes)

---

You are one of several agents reviewing this independently and in parallel; your peers cannot see your work. Be concrete and evidence-driven.

Report your findings as a prioritized list. Give each finding a stable id (F1, F2, …) and include:
- **severity**: critical | high | medium | low
- **confidence**: high | medium | low
- **location**: file path + function/symbol
- **evidence**: the concrete code pattern and why it matters
- **impact**: the user/system consequence
- **fix**: the minimal safe change

Prioritize high-impact, high-confidence findings. Skip speculative nits and pure style unless they hide a real problem. If an area is clean, say so briefly rather than padding.

Summarize every finding in the mandatory **findings index** described in the output format below, keyed by the same ids. The coordinator keys deduplication, cross-round tracking, and independent verification off those ids and `file:symbol` locations, so make each location precise and concrete.
