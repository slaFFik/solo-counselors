---
name: security
defaultRounds: 3
defaultReadOnly: strict
verify: cross
---

# Preset: security

A review workflow for finding exploitable vulnerabilities with an attacker's mindset — real exploits, not theoretical concerns.

## Discovery focus

When exploring the repo, trace untrusted data from source to sink — and adapt the taxonomy to the stack you actually find (a CLI, library, or data pipeline has different sinks than a web app). Trace the area the task names first; sweep the rest of this list only if discovery budget remains:
- Entry points where untrusted input enters: request handlers, CLI args, file/network reads, deserialization
- Sinks: queries, shells/`eval`/subprocess, template rendering, HTML output, file paths (path traversal, archive extraction/zip-slip, symlink/TOCTOU, temp-file races), outbound request targets (SSRF), and regex engines fed untrusted input (ReDoS)
- Auth and access control: session handling, authorization checks, object ownership (IDOR), privilege boundaries, CSRF on state-changing endpoints, mass assignment / over-posting
- Secrets and crypto: hardcoded keys, weak algorithms, token generation, storage/transport encryption; secrets or PII leaking into logs and error output
- Config and error handling: verbose errors, debug flags, CORS, default credentials

## What the written prompt should emphasize

The standard per-finding fields (including `reachability`) and the severity/confidence anchors come from the execution boilerplate — don't restate them. On top of those, instruct the panel to:
- Find and rank exploitable issues: injection (SQL/NoSQL/OS/LDAP/template), broken authentication, broken access control / IDOR / path traversal, SSRF, CSRF, mass assignment, sensitive-data exposure, XSS (reflected/stored/DOM), insecure deserialization, ReDoS, security misconfiguration, missing cryptographic controls. Tag each finding with its vulnerability class (or CWE) so the panel can be deduplicated.
- For each finding, state **why the existing controls do not stop it** — the auth/validation/sanitization/encoding already on the path and why it is insufficient — plus the concrete payload or request. (The `reachability` field carries the source→sink path.)
- Prioritize by exploitability: a real unauthenticated injection or RCE outranks a missing header or an admin-gated issue.
