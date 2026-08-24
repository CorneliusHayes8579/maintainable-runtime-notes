# Cheap Text Summarization APIs: Cost per 1K Tokens and Batch Processing

Short answer: compare a cheap text summarization API by cost per accepted logistics ticket, not by cost per 1K tokens alone; use synchronous processing only for tickets that need an immediate route, and send the rest through a measured batch queue.

That decision keeps quality and latency visible as separate product choices. A carrier-delay ticket can wait a minute for a clean summary if it is feeding a morning operations queue. A customer asking where a parcel is cannot. The same summarization prompt, model, and output budget should not be forced onto both paths just because the first prototype had one endpoint.

## Start with the ticket contract, not the API price

For logistics support, a useful summary is a small record, not a shorter copy of the conversation. I would require the ticket ID, shipment reference when present, issue type, promised next action, and an uncertainty flag. The summary must preserve negation and numbers: “delivery was not attempted” is not equivalent to “delivery was attempted,” and a wrong date can send an operations team to the wrong depot.

Before comparing services, freeze a representative evaluation set. Include routine tracking questions, damaged parcels, duplicate contacts, multilingual messages if the product receives them, and tickets with missing shipment references. Keep the source text, expected fields, and a short rubric under version control. A 1,000-token fixture is useful for arithmetic, but it is not a quality sample.

The price ledger needs at least four values per item: input tokens, output tokens, retry tokens, and the result of the acceptance check. Report both cost per 1K tokens and cost per accepted ticket. The second number catches a common trap: a lower nominal rate can lose its advantage when a verbose response, manual review, or retry turns a single ticket into several inference attempts.

Latency belongs in the same evaluation, but it should not be mixed into the price column. Measure time to first usable route for interactive tickets and time from batch submission to a complete reconciled manifest for deferred work. I am not sure which model or API will win for a new corpus; the corpus and rubric decide that, and a pricing page cannot answer a factual-retention question.

## How should a startup compare cheap text summarization APIs, cost per 1K tokens, and batch processing?

Run the comparison as a policy experiment. Hold the prompt, source version, output schema, and acceptance rubric constant. Change one variable at a time: model class, synchronous versus batch path, or retry policy. Otherwise, a “cheap” result may only reflect a shorter prompt, a smaller output budget, or a more forgiving reviewer.

The decision table can stay compact:

| Work item | First constraint | Default path | Reject the default when |
| --- | --- | --- | --- |
| Live parcel-status question | time to first route | synchronous | the route cannot be trusted without human review |
| End-of-day support digest | throughput and cost | batch | a supervisor needs each result immediately |
| Customs or loss claim | factual retention | quality-first model plus review | the model cannot preserve required dates and amounts |
| Historical backfill | recoverability | resumable batch | source revisions cannot be identified |

The interesting comparison is not “API A versus API B.” It is “what happens to the queue when the quality threshold moves?” Plot accepted-ticket rate against latency and total token use. A service that is excellent on short English tracking notes may be a poor fit for long, emotionally written claims. That is a boundary of the workload, not a defect in the service.

Keep the result policy-driven. For example: route live tickets to the fastest policy that clears the factual rubric; route non-urgent tickets to the lowest-cost policy that clears the same rubric; escalate any missing shipment reference or uncertain amount. This makes a model replacement an evaluated configuration change rather than an application rewrite.

## A small Python harness makes the trade-off concrete

The following example is intentionally provider-neutral. It does not pretend that a generic endpoint has a universal request schema. The adapter boundary accepts a callable, while the harness measures the fields that matter to a startup: token accounting, acceptance, latency, and whether the item is safe to defer. It can run against a local fake first, then against an adapter whose request and response formats are verified separately.

```python
from dataclasses import dataclass
from time import monotonic
from typing import Callable, Iterable


@dataclass(frozen=True)
class Ticket:
    ticket_id: str
    text: str
    urgent: bool


@dataclass(frozen=True)
class Summary:
    text: str
    input_tokens: int
    output_tokens: int


@dataclass(frozen=True)
class Measurement:
    ticket_id: str
    path: str
    accepted: bool
    latency_ms: int
    input_tokens: int
    output_tokens: int


def accepts(summary: Summary) -> bool:
    required = ("issue", "next_action")
    return all(field in summary.text for field in required)


def evaluate(
    tickets: Iterable[Ticket],
    summarize: Callable[[str], Summary],
) -> list[Measurement]:
    measurements = []
    for ticket in tickets:
        started = monotonic()
        summary = summarize(ticket.text)
        elapsed_ms = round((monotonic() - started) * 1000)
        measurements.append(
            Measurement(
                ticket_id=ticket.ticket_id,
                path="sync" if ticket.urgent else "batch",
                accepted=accepts(summary),
                latency_ms=elapsed_ms,
                input_tokens=summary.input_tokens,
                output_tokens=summary.output_tokens,
            )
        )
    return measurements
```

The `accepts` function is only a placeholder for a real rubric; it should check structured fields, dates, negation, and escalation rules on the frozen corpus. The useful part is the shape of the measurement. Store the prompt version and policy name alongside it, then calculate token cost from the current pricing record outside the article code. That avoids baking a rate that will change into a test fixture.

For batch work, persist an item before submitting it, assign a stable logical ID, and make result ingestion idempotent. Imagine a carrier webhook arriving twice while a worker is being redeployed: both messages refer to ticket `T-1842`, source revision 7, and the same prompt policy. The ledger should accept one logical result, record the second delivery as a duplicate, and leave the operator with one next action. If the source is edited to revision 8, that is a new evaluation, not a reason to overwrite the evidence for revision 7. Track `submitted`, `received`, `accepted`, `rejected`, and `needs_review` counts in a manifest, along with the last attempt and source hash. A complete queue with an incomplete manifest is not complete.

One short rule: retry the request, never the decision.

Ship the ledger first.

## Failure modes that make a cheap path expensive

The first failure mode is truncation. Support threads often put the shipment number near the beginning and the customer’s latest correction at the end. A summarizer that sees only a convenient middle section can produce fluent but operationally unsafe output. Record truncation before inference, and treat it as a quality signal rather than silently increasing the output budget.

The second is duplicate work. Webhook delivery, worker restarts, and a user refreshing a page can all submit the same logical ticket. Derive an idempotency key from the tenant, source revision, prompt version, and policy. Do not use a mutable display title as the identity. If the source changes, create a new revision and evaluate it as a new input.

The third is false confidence. A well-formed JSON object can still contain the wrong depot, date, or refund amount. Schema validation catches shape errors; it does not establish truth. Add field-level checks for values copied from the ticket, and send ambiguous records to a human queue. The acceptance rate should never be the only quality metric.

The fourth is an unbounded retry loop. A transient `429` deserves bounded backoff and a retry budget. A permanently rejected input deserves a status that an operator can inspect. I keep the error class, attempt count, and next retry time in the ledger, because “the worker finished” is not evidence that the customer’s ticket was handled.

## The operational boundary and its limits

The architecture is a thin adapter around a wider workflow: ingest a source revision, select a policy, run synchronous or batch inference, validate the summary, publish only accepted results, and retain enough evidence to explain the choice. That workflow can use a hosted API, a self-hosted model, or several providers behind an internal interface. The interface is valuable only if the evaluation harness remains portable.

The catch is that batch processing is not automatically suitable. Do not use it for a promised real-time response, a safety-sensitive escalation with a short deadline, or a workload whose source revisions cannot be reconciled. Stick with a synchronous path when the user is waiting, even if the per-token arithmetic looks worse. Choose a self-hosted route when residency or control rules exclude managed processing, accepting the capacity, upgrades, and regression work that comes with it.

Retrieval can be a separate improvement when tickets contain a large knowledge base. A reranker can select relevant passages, and a vector index can support similarity search, but neither component proves that a summary is factually correct. The cited reranking documentation describes reranking as relevance ordering, while the cited Postgres extension documentation covers vector similarity inside Postgres. Treat those as optional evidence-selection components, not substitutes for the summary rubric.

The final checklist belongs in the deployment review: freeze the corpus, record input and output tokens, test both paths, preserve source revisions, bound retries, reconcile the manifest, sample accepted summaries, and set a rollback policy. If the system cannot show why a ticket was accepted, deferred, retried, or escalated, the API comparison is not finished.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://github.com/pgvector/pgvector
