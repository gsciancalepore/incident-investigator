# Roadmap

| Version | Adds | Depends on |
|---|---|---|
| **v1** | CLI, LangGraph investigation over local `.jsonl` logs, swappable model, evaluation suite | Revestimientos log contract v1 (spec `observabilidad-01-contrato-logs`) |
| v2 | RAG over Revestimientos documentation (Chroma); optional hybrid search over logs | v1 evaluation results |
| v3 | Staging logs through a new `LogSource` adapter (Render API or a free log backend) | Revestimientos observability stage 2 |
| v4 | Tools exposed over MCP; a Gradio UI for demos | — |
| v5 | Alert-triggered runs (agent SDK or a scheduled runner) | Revestimientos observability stage 5; a budget decision, because hosted model calls are paid |

Nothing past v1 is committed. Each version starts with its own decision record.

## v1 — before writing code

1. ~~Revestimientos merges log contract v1~~: done (PR #39). The schema is vendored in
   `contracts/revestimientos/`.
2. ~~Decide the default model provider~~: Gemini free tier (ADR 0003). Create a `GOOGLE_API_KEY`.
3. Reproduce the evaluation scenarios in Revestimientos and commit their log fixtures.
