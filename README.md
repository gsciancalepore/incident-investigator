# incident-investigator

An AI agent that investigates production incidents by reading an observed system's logs. It answers
questions like *"what happened to order 123?"* with a report where every claim cites the log line
that supports it, and it says "no evidence found" instead of guessing.

The first observed system is **Revestimientos**, a Laravel 12 e-commerce store. The two systems are
independent. They share a versioned log contract (JSON Schema + event catalog), and nothing else.

> **Status**: architecture defined (2026-09-24), implementation not started.

## Design at a glance

- **External by design**: reads a directory of `.jsonl` files. No database, no network access to the
  observed system. Personal data is removed before the logs are written.
- **Explicit graph, not a free-form agent**: LangGraph pipeline where the model decides *what to
  look for* and *what it means*, and plain code does the retrieval, correlation and citation
  checking.
- **Swappable model**: Gemini free tier by default, Ollama or Anthropic as alternatives, behind
  LangChain's chat model interface, always with structured output.
- **Measured**: an evaluation suite of incidents reproduced on purpose, with known answers, scored
  per model: root-cause accuracy, citation validity, invented identifiers.

## Documents

- [Architecture](docs/architecture.md)
- Decision records:
  - [0001 — External system and log contract](docs/adr/0001-external-system-log-contract.md)
  - [0002 — LangGraph with deterministic evidence gathering](docs/adr/0002-langgraph-deterministic-retrieval.md)
  - [0003 — Swappable model](docs/adr/0003-swappable-model.md)
  - [0004 — No RAG in v1](docs/adr/0004-no-rag-in-v1.md)
- [Roadmap](docs/roadmap.md)
