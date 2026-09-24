# ADR 0002 — LangGraph with deterministic evidence gathering

- **Status**: accepted (2026-09-24)

## Context

A free-form ReAct agent, which decides which tool to call at every step, is the default pattern. In
this problem it has two costs:

- **Reproducibility**: the path through the logs changes from run to run.
- **Small local models**: they call tools unreliably, and v1 must work with them.

## Decision

The investigation is an explicit LangGraph `StateGraph`:

1. understand;
2. gather evidence;
3. build timeline;
4. analyze;
5. verify;
6. report.

**The model decides what to look for and what it means**:
- Node 1 turns the question into a typed query.
- Node 4 either states hypotheses that cite evidence ids, or asks for more evidence.

**Code does the looking and the checking**:
- Nodes 2 and 3 query the log source and correlate `order_id`, `payment_id` and `request_id`.
- Node 5 verifies that every citation exists.

Two bounded loops provide the agentic behaviour:
- **Analyze → gather**: at most 2 extra rounds of evidence.
- **Verify → analyze**: 1 reflection round when citations fail.

## Consequences

- Every node except 1 and 4 is unit-testable without a model.
- Hallucinated citations are caught structurally, not by prompting.
- Less open-ended than a ReAct agent. Questions that need a source other than the logs are out of
  scope until that source exists behind the `LogSource` port.
