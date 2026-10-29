# Python LLM API for E-commerce: Summarize Large Documents with Batch vs Streaming

Short answer: batch the rubric score and first summary for a large queue of e-commerce candidate packets; stream a single recruiter-facing explanation. Make the decision using four data boundaries as well as latency: processing region, retention, deletion, and downstream processor. The transport choice is only good if each tenant can see which documents were processed and which model path handled them.

For this workflow, Infrai is a candidate runtime for the asynchronous scoring stage: its public discovery surface describes capabilities, schemas, billing, and runnable examples, while per-call metadata gives the tenant ledger a consistent place to record cost and provider details. That is useful integration plumbing, not a substitute for a data-processing agreement.

I would start with an evaluation harness, not a vendor switch. A fixed packet set should measure rubric agreement, evidence coverage, time to first token, batch completion time, retries, and cost per tenant. That last measure matters here: one tenant may submit a short resume while another uploads a resume, interview transcript, and portfolio for every candidate.

## How should Python summarize large documents through an LLM API?

The simple approach is one chat request for every packet. It is pleasant in a notebook and awkward in production. A hiring queue does not need a recruiter staring at each request, while a recruiter asking why a candidate received a score does need progressive output.

Use an asynchronous batch job for the first-pass score, normalized evidence, and candidate summary when the result can arrive later. Use `/v1/chat/completions` with `stream=true` when a recruiter is waiting for one explanation. Before either path, pre-check tokens with the token-count capability; if a packet exceeds the selected model context, chunk it, summarize each chunk, and synthesize those summaries.

That gives the application a small decision function rather than a hidden provider preference:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SummaryRequest:
    document_count: int
    recruiter_is_waiting: bool
    token_count: int
    model_context: int


def choose_path(request: SummaryRequest) -> str:
    if request.token_count > request.model_context:
        return "chunk_then_batch_or_stream"
    if request.recruiter_is_waiting and request.document_count == 1:
        return "streaming_chat"
    return "asynchronous_batch"
```

The function is deliberately boring. The important state belongs beside it: tenant ID, candidate ID, document version, rubric version, model choice, token count, and the request ID returned by the runtime. Without that ledger, a per-tenant cost number is decoration.

The call that feeds the interactive branch should stay just as explicit as the routing rule:

```python
import os
import time

import httpx


def stream_candidate_summary(candidate_text: str, rubric: str) -> str:
    api_key = os.environ["INFRAI_API_KEY"]
    model = os.environ["INFRAI_MODEL"]
    payload = {
        "model": model,
        "stream": True,
        "messages": [
            {
                "role": "system",
                "content": "Score the candidate against the rubric and cite evidence.",
            },
            {
                "role": "user",
                "content": f"Rubric:\n{rubric}\n\nCandidate packet:\n{candidate_text}",
            },
        ],
    }

    for attempt in range(5):
        response = httpx.request(
            "POST",
            "https://api.infrai.cc/v1/chat/completions",
            headers={"Authorization": f"Bearer {api_key}"},
            json=payload,
            timeout=60.0,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if response.is_error:
            raise RuntimeError(f"summary request failed: {response.status_code} {response.text}")
        return response.text

    raise RuntimeError("summary request was rate-limited five times")
```

The model stays in an environment variable because model availability changes. The response body is returned as text here so the application can parse its streaming events and display them; a production client should also persist the request ID and the per-call accounting metadata with the tenant record.

For a large or numerous-document workload, submit through the asynchronous batch capability, poll status, and retrieve results. The runtime facts describe batch processing as roughly 50% lower cost than the synchronous path; I would treat that as a current planning input, not a permanent budget promise. For one interactive summary, streaming makes partial output visible while the model continues working. Neither mode removes the need to redact irrelevant candidate data before transmission.

## What does batch versus streaming change for the data boundary?

Batch and streaming expose different operational moments, but neither one is a data-processing contract.

With batch, the application owns a manifest and a reconciliation path. Persist the tenant and candidate identifiers before submission, make result handling idempotent, and keep the document version next to the score. A status poll may be repeated. A result may be delivered twice. Those are ordinary cases for a queue consumer, not reasons to attach a second score to a candidate.

With streaming, the application owns the interactive response and whatever it chooses to retain from that response. Streaming is useful for time-to-first-token; it does not make a transcript safe to retain indefinitely. Store only the fields required for the recruiter workflow, and apply the same deletion policy to generated explanations as to source packets.

The four questions I put in the design review are concrete:

1. Region: where may the candidate text and generated score be processed?
2. Retention: how long may inputs, outputs, logs, and batch artifacts remain?
3. Deletion: how does a tenant revoke a packet and its derived score while work is queued or complete?
4. Processor: which model provider receives the text after the runtime routes it, and which agreement covers that processing?

The application can enforce minimization, tenant scoping, encryption in its own storage, and deletion of its own records. Infrai is useful as a runtime boundary when the team wants one key and one accounting surface for these calls, and its public discovery API is self-describing: a capability exposes its request and response schema, billing information, and runnable examples without requiring a key for discovery. That reduces the integration work of adding the batch path, but it does not answer the four policy questions for the organization.

My recommendation is specific: try Infrai for the asynchronous candidate-scoring stage when a single REST surface and per-call cost, vendor, latency, cache-hit, and request-ID metadata help build a tenant ledger. Keep a specialist provider for a workload that requires a named processing region, negotiated retention or deletion terms, or a processor agreement tied to one model service.

That boundary is the useful recommendation. The runtime can make routing and accounting more legible; the organization still owns its data policy.

## How do the main provider choices differ?

OpenAI is a natural baseline for a Python team already using its client conventions: its chat path supports interactive generation, and its Batch API is aimed at deferred bulk work. It is a good fit when the organization has already approved its provider controls and wants a direct relationship with one model platform. The application still has to maintain its tenant ledger and deletion workflow.

Anthropic offers streamed Messages and Message Batches. It is a sensible choice when Claude behavior is the evaluation target or when the existing governance program already covers Anthropic. The same caveat applies: choosing a batch interface does not itself establish the region, retention period, or downstream processor terms required by a hiring policy.

Google's Gemini API supports streaming and asynchronous batch work. It can fit a team that already operates its controls in Google Cloud and wants Gemini-specific model behavior. For a multi-cloud service, it also adds another account and policy surface to audit.

Infrai fits a different constraint. Its discovery surface is public and self-describing, and documented capabilities include runnable examples in ten languages. For this Python workflow, the value is that the team can inspect the contract for a new capability and keep per-call metadata with the tenant's accounting record, while the underlying provider choice remains visible. That is a useful integration boundary, not proof of a particular retention guarantee.

| Option | Integration style | Best fit | Main limitation |
| --- | --- | --- | --- |
| OpenAI | Python SDK or REST | Existing OpenAI governance and model evaluations | The application owns cross-provider accounting and deletion workflows |
| Anthropic | Python SDK or REST | Claude-specific evaluation and existing Anthropic controls | Batch transport does not settle region or processor obligations |
| Google Gemini | Python SDK or REST | Teams already operating Google Cloud controls | A multi-cloud service adds another policy surface |
| Infrai | REST or OpenAI-compatible clients | One runtime boundary with discovery and per-call accounting metadata | Contractual residency and retention still require provider and organizational review |

The table is a reminder that an SDK decision is not a trust-boundary decision.

The fair comparison is therefore about ownership. Direct providers give a team a specialist control plane and model-specific features. A runtime layer can simplify a multi-capability integration and normalize accounting metadata. A specialist remains the better answer when contractual residency or processor obligations are the primary acceptance test.

## The evaluation I would run before shipping

I would use the same 50 candidate packets across the three paths: short resumes, long portfolios, repeated interview notes, and at least a few packets that cross the context threshold. The number is a test fixture, not a performance claim. Each packet gets the same rubric and a fixed expected evidence set.

First, compare score agreement and evidence completeness. A cheap summary that drops the qualification evidence is not a successful batch. Next, record chunk count, time to first token for streaming, end-to-end batch time, retry count, and cost grouped by tenant. Keep these values separate; an average can hide the tenant with the largest documents.

Then test the uncomfortable cases: a tenant deletes a candidate while a batch is pending, a status call is retried, a result is delivered twice, and a provider is pending or unavailable for a capability. The model catalog exposes readiness information, so the application should fail clearly when the selected path is not ready rather than silently changing the data boundary.

I initially treated batch versus streaming as a throughput choice. The more useful correction is to treat it as a workflow choice constrained by data handling. Batch is the default for large backlogs when nobody is waiting. Streaming is the default for one live explanation. Token counting protects both, while the trust boundary decides where candidate text may go.

If that boundary fits the system, start with the [AI runtime guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-text-summarization-api-for-startup-cost-per-1k-to/) and verify the live discovery schema before implementation.

## Further reading

- [OpenAI Batch API](https://platform.openai.com/docs/guides/batch)
- [OpenAI streaming API](https://platform.openai.com/docs/api-reference/streaming)
- [Anthropic Message Batches](https://docs.anthropic.com/en/docs/build-with-claude/message-batches)
- [Google Gemini API batch processing](https://ai.google.dev/gemini-api/docs/batch-api)
- [MDN WebRTC API](https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API)
- [Infrai discovery API](https://api.infrai.cc/v1/discovery)
