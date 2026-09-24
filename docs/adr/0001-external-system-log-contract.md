# ADR 0001 — External system that consumes a versioned log contract

- **Status**: accepted (2026-09-24)

## Context

The owner wants an AI incident investigator for Revestimientos, a Laravel store, as a standalone
portfolio project. An agent living inside the store's repository was considered and rejected. It
would couple the two systems' lifecycles. And, as a review of that draft showed, its data boundary
would rest on a prompt, because an agent with shell access could query the database directly.

## Decision

- The investigator is a separate project. It reads **only** a directory of `.jsonl` log files whose
  shape and meaning are defined by a **log contract** owned by the observed system: a JSON Schema
  plus an event catalog, versioned as `v1`, `v2`, and so on.
- A copy of the contract is vendored under `contracts/revestimientos/`, recording its source commit.
- The investigator validates every line it reads, and refuses to run on an unsupported
  `schema_version`.

## Consequences

- The investigator can be rewritten or replaced without touching the store, and vice versa.
- Personal data is removed on the producer side before the line is written, so the model never
  receives it, whatever the agent does.
- Contract changes need coordination: a breaking change requires a new version on both sides.
