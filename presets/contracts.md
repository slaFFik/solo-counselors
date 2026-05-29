---
name: contracts
defaultRounds: 3
defaultReadOnly: strict
---

# Preset: contracts

A review workflow for auditing API contract drift across server handlers, shared types, clients, validators, and tests.

## Discovery focus

When exploring the repo, line up each producer against its consumers:
- Producer/consumer pairs: API handlers and their callers/clients
- Schemas, runtime validators, and language types (TS/PHP/Python) for the same data
- Enums and status values across models, serializers, and client assumptions
- Error contracts: status codes and error payload shapes
- Serialization: date/time formats, number/string coercion, nullability
- Docs/examples/spec files vs actual implementation

## What the written prompt should emphasize

Instruct the panel to:
- Find request/response field mismatches, optional-vs-required drift, enum/status drift, inconsistent error contracts, versioning/back-compat breaks, serialization mismatches, and stale docs.
- For each: producer and consumer locations, the concrete mismatch, the runtime impact, and a contract/integration test that fails before the fix and passes after.
