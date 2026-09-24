# ADR 0003 — Swappable language model

- **Status**: accepted, with the default provider **open** (2026-09-24)

## Context

The owner's budget is free tiers only. The model has to be replaceable, both to stay free and to
compare quality in the evaluation suite.

The development machine (WSL2) currently exposes **2 GB of RAM and no GPU**. That is not enough to
run a 7–8B model locally with Ollama. It is enough for 1–3B models, whose reasoning is weak for this
task.

## Decision

- The graph depends on LangChain's `BaseChatModel`. A factory builds the concrete model from
  configuration (`provider`, `model`, `temperature=0`).
- Every model call uses **structured output** (Pydantic), so providers are interchangeable at the
  output level.
- Supported providers in v1: **Ollama** (local, free) and **Anthropic** (hosted, paid, optional
  extra, used for quality comparison with an explicit spending cap).

## Open: default provider

To be decided before implementation. Options:

1. Give WSL more memory (`.wslconfig`), or run Ollama natively on the Windows host and reach it
   from WSL, then use a 7–8B model.
2. Use a free hosted tier through an OpenAI-compatible endpoint. This adds a provider and a key.
3. Accept a small local model for development and use the evaluation suite to measure the gap.

## Consequences

- Evaluation results are reported per model. The README shows the comparison, not a single number.
