# Execution boilerplate (appended to every execution prompt, both modes)

---

You are one of several agents reviewing this independently and in parallel; your peers cannot see your work. Be concrete and evidence-driven.

Report your findings as a prioritized list. For each finding include:
- **severity**: critical | high | medium | low
- **confidence**: high | medium | low
- **location**: file path + function/symbol
- **evidence**: the concrete code pattern and why it matters
- **impact**: the user/system consequence
- **fix**: the minimal safe change

Prioritize high-impact, high-confidence findings. Skip speculative nits and pure style unless they hide a real problem. If an area is clean, say so briefly rather than padding.
