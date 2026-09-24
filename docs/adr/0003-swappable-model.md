# ADR 0003 — Swappable language model

- **Status**: accepted (2026-09-24). Default provider decided the same day: **Gemini API, free tier**

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

## Default provider: Gemini API free tier (owner's decision, 2026-09-24)

The machine cannot run a useful local model (see Context), so the default is a hosted free tier:

- **Integration**: `langchain-google-genai`, with the API key read from `GOOGLE_API_KEY`, never
  committed. `temperature=0` and structured output, like every provider.
- **Model id is configuration**, not code. Default: a current *Flash* model with a free tier.
  Model ids change every few months, so the one in use is recorded with each evaluation run.
- **Condition: no personal data, ever.** On the free tier, Google **may use prompts and responses
  to improve its products**. This is acceptable only because the log contract guarantees that
  personal data is removed before a line is written (ADR 0001). If the investigator ever gains a
  source without that guarantee, the free tier stops being an option for it.
- **Rate limits**: the free tier is rate-limited, and the limits change. The model adapter retries
  with backoff on rate-limit errors. The evaluation runner throttles between scenarios. Limits are
  checked on the day the suite runs, not written down here.
- **Other providers stay supported**: Ollama (offline, small models, for the evaluation
  comparison), and Anthropic as an optional extra with a spending cap.

## Consequences

- Evaluation results are reported per model. The README shows the comparison, not a single number.
