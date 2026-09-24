# ADR 0004 — No retrieval-augmented generation in v1

- **Status**: accepted (2026-09-24)

## Context

The only source in v1 is structured logs. They are best searched by exact filters (`request_id`,
`order_id`, `payment_id`, event, level, time window), not by embedding similarity. Semantic search
over log lines would add a vector store and an embedding model, and would retrieve less precisely
than a filter.

## Decision

- No vector store in v1. Retrieval over logs is deterministic filtering plus correlation hops
  (ADR 0002).
- Domain knowledge the model needs, such as which states are expected, comes from the event
  descriptions in the log contract, injected into the analysis prompt.

## Revisit when

v2 adds the observed system's **documentation** as a second source: specs, ADRs and runbooks. Those
are unstructured text, where RAG fits. The candidate design is Chroma with a retriever over
Revestimientos' `docs/`, queried by the analyze node to explain *why* the system behaves as it
did, citing the document. Hybrid retrieval over logs (filters + semantic search on event
descriptions) is also a v2 candidate if the evaluation shows questions that filters cannot resolve.
