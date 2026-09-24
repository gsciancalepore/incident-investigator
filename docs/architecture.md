# Architecture — v1

> Status: **defined, not implemented** (2026-09-24). v1 intentionally has no MCP, no agent SDK and
> no infrastructure beyond a Python process and a language model.

## What it is

`incident-investigator` is an AI agent that answers questions about production incidents
("what happened to order 123?", "why did the payment webhook fail last night?") by reading an
observed system's logs. It returns a report where every claim cites the log entry that supports it.

It is an **external system**. The system it observes, Revestimientos (a Laravel e-commerce store),
never imports, calls or knows about it. The two systems share one thing: a **versioned log
contract**.

## System context

```
┌──────────────────────────────┐        log contract v1         ┌───────────────────────────────┐
│ Revestimientos (observed)    │  JSON Schema + event catalog   │ incident-investigator         │
│ Laravel 12                   │ ─────────────────────────────▶ │ Python 3.12                   │
│                              │  app-YYYY-MM-DD.jsonl files    │                               │
│ • request_id on every line   │  in a directory                │ • reads only that directory   │
│ • no personal data (removed  │                                │ • no DB, no network access to │
│   before writing)            │                                │   the observed system         │
└──────────────────────────────┘                                └──────────────┬────────────────┘
                                                                               │ chat model interface
                                                                  ┌────────────▼──────────────┐
                                                                  │ Language model (swappable)│
                                                                  │ default: Gemini free tier │
                                                                  └───────────────────────────┘
```

The observed side owns the contract. It lives in Revestimientos at
`docs/observabilidad/log-schema.v1.json`, and a copy is vendored here under
`contracts/revestimientos/`, recording the commit it was taken from. Revestimientos validates
every emitted line against the schema in its own test suite. This project validates its fixtures
against the vendored copy. If either side drifts incompatibly, a test fails.

**Compatibility rule (set by the contract)**: new events and new optional attributes are
compatible. The investigator keeps unknown events as generic evidence and ignores unknown
attributes. Removing, renaming or retyping anything requires a new `schema_version`, which the
investigator refuses until it supports it. The contract also warns that a missing event does not
prove the fact did not happen (the producer may drop lines rather than break a request). Reports
must not treat absence as proof.

## Inside the investigator

Hexagonal layout: the investigation logic depends on ports, never on files or model vendors.

```
src/incident_investigator/
├── contract/     LogEntry, EventCatalog: Pydantic models generated from / checked against the schema
├── ports/        LogSource (query logs), ChatModel (LangChain's BaseChatModel is the port)
├── adapters/     FileLogSource (jsonl directory), model factory (ollama | anthropic | …)
├── graph/        LangGraph StateGraph: nodes + state
├── report/       Report model → Markdown and JSON renderers
└── cli.py        `investigate --logs DIR "question"`
eval/
├── scenarios/    one folder per incident: fixture logs + question + expected findings
└── run.py        runs every scenario against a model, scores it, writes a results table
```

### `LogSource` port

```python
class LogQuery(BaseModel):
    request_ids: list[str] = []
    order_ids: list[int] = []
    payment_ids: list[str] = []
    events: list[str] = []
    levels: list[Level] = []
    since: datetime | None = None
    until: datetime | None = None
    limit: int = 500

class LogSource(Protocol):
    def query(self, q: LogQuery) -> list[LogEntry]: ...
    def schema_version(self) -> str: ...
```

v1 has one adapter, `FileLogSource`. It reads `app-*.jsonl` from one directory, streams
line by line, validates every line against the contract (tolerating additive changes), and refuses to run if it finds a
`schema_version` it does not support. Later sources (a Render log API, Grafana Loki) are new adapters
behind the same port. The graph does not change.

### The investigation graph (LangGraph)

```
          ┌────────────────────┐
question ─▶ 1. understand      │  LLM → IncidentQuery (structured output)
          └─────────┬──────────┘   identifiers, time window, symptom
                    ▼
          ┌────────────────────┐
          │ 2. gather evidence │  deterministic: LogSource queries + correlation hops
          └─────────┬──────────┘   order_id → request_ids → every line of those requests
                    ▼              payment_id ↔ order_id
          ┌────────────────────┐
          │ 3. build timeline  │  deterministic: sort, dedupe, compact
          └─────────┬──────────┘
                    ▼
          ┌────────────────────┐   LLM → hypotheses, each citing evidence ids,
          │ 4. analyze         │◀─┐ or a request for more evidence
          └─────────┬──────────┘  │
          more evidence needed? ──┘ (max 2 extra rounds, back through node 2)
                    ▼
          ┌────────────────────┐   deterministic: every citation must exist in the
          │ 5. verify          │   evidence set; unsupported claims are dropped or
          └─────────┬──────────┘   downgraded; on failure → back to 4 once (reflection)
                    ▼
          ┌────────────────────┐
          │ 6. report          │  Report (Pydantic) → report.md + report.json
          └────────────────────┘
```

**Design rule: the model decides what to look for and what it means. Code does the looking and the
checking.** Nodes 2, 3 and 5 are plain Python. That keeps the agent reproducible, testable without
a model, and usable with small local models, which are unreliable at free-form tool calling (ADR
0002).

### Report

Fixed sections, in order:
1. **Symptom**.
2. **Timeline**.
3. **Evidence**: every entry cited by file, `timestamp` and `request_id`.
4. **Hypotheses**: each with high, medium or low confidence and the evidence behind it.
5. **Root cause**: only if the evidence is conclusive; otherwise it says so.
6. **Next steps**.

When the logs hold nothing relevant, the report says "no evidence found" and lists what it searched.
It does not fill gaps with guesses.

### Domain knowledge without RAG

v1 does not retrieve documents (ADR 0004). What the model knows about the observed system comes from
the **contract itself**. Every event in the catalog has a description written by the observed side,
and it states expected behaviour. For example, `order.stock_negative` is a **known, accepted state**
(the store sells beyond stock and restocks from the manufacturer), not a bug. Those descriptions go
into the analysis prompt.

## Trust boundaries

| Boundary | Rule |
|---|---|
| Personal data | Removed by the observed system **before** the line is written. The investigator never sees it, so what the model sees is already clean. This is also what makes the Gemini free tier acceptable, since it may use prompts to improve Google's products (ADR 0003) |
| Log content → prompt | Log values are **untrusted data**. Some come from outside the store (headers such as MercadoPago's `x-request-id`, payment references). They go into the prompt only inside delimited, typed evidence blocks, never as instructions. The contract caps the length of externally sourced strings |
| Model output | Parsed into Pydantic models. Anything that fails validation is retried once, then reported as an analysis failure, never passed through as free text |
| Side effects | None. The investigator reads one directory and writes one report |

## Evaluation (the part that makes it a portfolio piece)

Each scenario in `eval/scenarios/` is an incident reproduced on purpose in Revestimientos, with the
resulting log lines committed as fixtures and the known answer written down beforehand:

| Scenario | Question | Expected |
|---|---|---|
| Invalid webhook signature | "Did MercadoPago notify payment X?" | `webhook.signature_invalid`, cause identified |
| Payment preference failure | "Customer says the pay button did nothing — order N" | `checkout.mp_preference_failed` |
| Payment rejected | "Why is order N still pending?" | `webhook.payment_not_approved` with status |
| Stuck after payment redirect | "Order N never got paid" | `checkout.payment_started` with nothing afterwards, reported as **"abandoned at MercadoPago or webhook lost: the logs cannot tell which"** (a declared limit of contract v1) |
| Checkout rejected by stock | "I couldn't buy" (time window) | `checkout.rejected`, reason `dominio` |
| Negative stock | "Stock went negative for product P" | recognised as **expected** behaviour, not a bug |
| Unknown order | "What happened to order 99999?" | "no evidence found" |

Metrics per model:
- root-cause accuracy;
- **citation validity** (must be 100%);
- evidence recall;
- **invented identifiers** (must be 0);
- correct "no evidence" answers;
- latency and cost.

The same suite runs against every configured model, and the comparison table goes in the README.

## Stack

| Concern | Choice |
|---|---|
| Language | Python 3.12 |
| Packaging | `uv` |
| Orchestration | LangGraph |
| Model interface | LangChain chat models: `langchain-google-genai` (**default, Gemini free tier**), `langchain-ollama`, `langchain-anthropic` (optional extra) |
| Data models | Pydantic v2 |
| Schema validation | `jsonschema` |
| CLI | Typer |
| Quality | pytest, ruff, mypy (strict) |

## What v1 deliberately does not do

It has none of these:
- MCP;
- an agent SDK;
- a vector store or RAG;
- a UI beyond the CLI;
- scheduled or alert-triggered runs;
- database access;
- any hosted infrastructure.

Each one is a later version with its own decision record (see `roadmap.md`).
