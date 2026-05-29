---
name: security
defaultRounds: 3
defaultReadOnly: strict
---

# Preset: security

A review workflow for finding exploitable vulnerabilities with an attacker's mindset — real exploits, not theoretical concerns.

## Discovery focus

When exploring the repo, trace untrusted data from source to sink:
- Entry points where untrusted input enters: request handlers, CLI args, file/network reads, deserialization
- Sinks: queries, shells/`eval`, template rendering, HTML output, file paths
- Auth and access control: session handling, authorization checks, object ownership (IDOR), privilege boundaries
- Secrets and crypto: hardcoded keys, weak algorithms, token generation, storage/transport encryption
- Config and error handling: verbose errors, debug flags, CORS, default credentials

## What the written prompt should emphasize

Instruct the panel to find and rank exploitable issues:
- Injection (SQL/NoSQL/OS/LDAP/template), broken authentication, broken access control / IDOR / path traversal, sensitive-data exposure, XSS (reflected/stored/DOM), insecure deserialization, security misconfiguration, missing cryptographic controls.
- For each: file path, the vulnerable pattern, how an attacker exploits it, and the specific fix. Prioritize by exploitability — a real injection beats a missing header.
