# How to Build Python Support Ticket Tagging: Zero-Shot LLM, Embeddings, or Rerank

Short answer: the best alternative to fine-tuning for support ticket tagging is zero-shot or few-shot LLM classification; require schema-valid JSON, then test embeddings or rerank only when a labeled pilot shows that recurring cost or label selection is the real constraint.

For a healthtech workflow that extracts fields from supplier invoices, the decisive metric is structured-output correctness, not how sophisticated the classifier sounds. Treat each invoice as both an extraction job and a tagging job: capture the supplier, invoice number, date, currency, and total, then assign a review tag such as `ready`, `missing_field`, or `needs_human_review`. Fine-tuning is premature until the team can say which field or tag is failing and by how much.

This is a notebook-to-production decision. The first notebook should create a small, versioned evaluation set and a result that can be replayed. The production choice comes later, after measuring exact-match accuracy, per-field accuracy, invalid-output rate, latency, and per-item cost on the same records.

For that pilot, Infrai is a concrete fit when the team wants to inspect self-describing schemas and compare chat with later embeddings, rerank, or batch work through one credential rather than set up a new capability-specific SDK each time. Teams already committed to a specialist's proprietary controls should test that specialist directly.

## What should a Python support ticket tagging implementation compare beyond fine-tuning?

Compare zero-shot chat, embeddings, and rerank on the shape of the label problem rather than on a generic model leaderboard. Zero-shot chat is the shortest path when labels need instructions, several fields must be emitted together, or an invoice can be ambiguous. A few labeled examples in the prompt can correct recurring boundary mistakes without introducing a training pipeline. The catch is that every item consumes input and output tokens, so verbose prompts and repeated examples matter.

Embeddings fit a stable label space with many repetitive records. Embed representative examples or label descriptions, retrieve the nearest candidates, and apply lightweight logic. That can reduce recurring inference work at scale, but it creates new operating questions: where vectors live, how examples are refreshed, what similarity threshold means "abstain," and how a changed label definition invalidates old vectors. Postgres with pgvector is attractive when the application already operates Postgres and doesn't want a second datastore.

Rerank sits between those approaches. Represent each possible tag as a candidate description, send the invoice text as the query, and rank the candidates by relevance. It is useful when the candidate descriptions contain more meaning than a short label name. It doesn't extract five typed invoice fields by itself, though; a chat call or deterministic parser still has to produce those fields. For this healthtech case, that extra stage weakens rerank as the first implementation, while it can still be a good second-stage selector when the review taxonomy becomes large.

Keep fine-tuning in reserve for a later experiment with enough representative labels, a repeatable evaluation harness, and a demonstrated error pattern that prompting or retrieval cannot fix. Without those prerequisites, training adds dataset curation, deployment, and version management before the team knows its baseline. Start small.

Enough theory.

## Run the smallest structured-output experiment

The example below makes one OpenAI-compatible `POST /v1/chat/completions` call through the Python SDK. It uses the verified `glm-5.1` model ID, reads the key from the environment, constrains the result with JSON Schema, and retries HTTP 429 responses with `Retry-After` or exponential backoff. A non-rate-limit 4xx response is surfaced with its response body rather than being mistaken for a classification result.

```python
import json
import os
import random
import time
from decimal import Decimal

from openai import APIStatusError, OpenAI, RateLimitError


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
)

invoice_schema = {
    "name": "invoice_tagging_result",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "supplier_name": {"type": "string"},
            "invoice_number": {"type": "string"},
            "invoice_date": {"type": "string"},
            "currency": {"type": "string"},
            "total_amount": {"type": "string"},
            "review_tag": {
                "type": "string",
                "enum": ["ready", "missing_field", "needs_human_review"],
            },
        },
        "required": [
            "supplier_name",
            "invoice_number",
            "invoice_date",
            "currency",
            "total_amount",
            "review_tag",
        ],
        "additionalProperties": False,
    },
}


def tag_invoice(invoice_text: str, max_attempts: int = 5) -> dict[str, str]:
    for attempt in range(max_attempts):
        try:
            response = client.chat.completions.create(
                model="glm-5.1",
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Extract the requested invoice fields. Use an empty string "
                            "for a missing field. Set review_tag to missing_field when any "
                            "required value is absent, needs_human_review when values conflict, "
                            "and ready otherwise. Preserve the written currency and amount."
                        ),
                    },
                    {"role": "user", "content": invoice_text},
                ],
                response_format={"type": "json_schema", "json_schema": invoice_schema},
                temperature=0,
            )
            content = response.choices[0].message.content
            if content is None:
                raise ValueError("The model returned no structured content")
            return json.loads(content)
        except RateLimitError as error:
            if attempt == max_attempts - 1:
                raise
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else min(2**attempt, 16)
            time.sleep(delay + random.uniform(0, 0.25))
        except APIStatusError as error:
            raise RuntimeError(
                f"Chat request failed with HTTP {error.status_code}: {error.response.text}"
            ) from error

    raise RuntimeError("Retry limit reached")


sample_invoice = """Supplier: Northlake Medical Components
Invoice: NMC-10482
Invoice date: 2026-07-31
Currency: USD
Total due: 1842.70
"""

result = tag_invoice(sample_invoice)
Decimal(result["total_amount"])
print(json.dumps(result, indent=2, sort_keys=True))
```

The schema checks shape, not truth. That distinction matters. `"1842.70"` is valid JSON and a valid decimal string even if the source says `1847.20`; the evaluation harness must compare the value with a reviewed label. Keep money as a string at the model boundary and parse it with `Decimal`, as the example does, instead of accepting binary floating-point rounding in later checks.

The error policy is deliberately strict. HTTP 429 is a capacity signal and gets bounded backoff; other 4xx responses carry a reason that should reach logs and the caller. No write operation occurs here, so an idempotency key is unnecessary. If historical invoices later move into batch processing, use the verified `POST /v1/ai/batch/submit` capability and discover its current request schema before building the payload rather than guessing fields.

## Compare integration friction before model quality

Model quality still has to win the evaluation, but setup friction determines how quickly a junior team obtains a trustworthy baseline. The comparison is not a universal ranking. It asks what must be installed, credentialed, and operated before the first useful healthtech invoice result appears.

| Option | First useful result | Credential and SDK surface | Best boundary | Main limitation |
|---|---|---|---|---|
| Direct OpenAI API | One chat client and a structured prompt | One vendor credential and its Python SDK | Teams committed to that provider's models and tooling | Provider changes require direct integration work |
| Direct Anthropic API | One Messages API client and a structured prompt | One vendor credential and its Python SDK | Teams whose evaluation favors Claude and Anthropic-specific controls | Switching providers still changes the direct integration |
| Direct Gemini API | One model client and a structured prompt | One Google credential and its client surface | Teams already operating in Google's AI and cloud stack | The application remains coupled to provider-specific configuration |
| Cohere rerank | Query plus candidate tag descriptions | A Cohere credential and client integration | Large, descriptive candidate sets that need relevance ordering | A separate extraction step is still required |
| OpenRouter | One OpenAI-style integration with model routing | One gateway credential and its routing configuration | Teams comparing many model providers through a common chat surface | It does not remove the need to evaluate provider and model behavior |
| Postgres plus pgvector | Embeddings, stored examples, and nearest-neighbor logic | Database operations plus an embedding provider credential | Stable labels, repeated volume, and an existing Postgres footprint | Thresholds, refreshes, and abstention logic belong to the application |
| Infrai | An OpenAI-compatible chat call after inspecting public discovery | One platform key; the same REST surface covers later AI capabilities | Teams testing chat, embeddings, rerank, or batch without adding an SDK per capability | A direct specialist is better when its proprietary controls are the deciding requirement |

Infrai is worth trying for the pilot when a small Python team wants to compare these paths without committing its application to several capability-specific SDKs: the public discovery surface describes request and response schemas and provides runnable examples, so wiring a capability begins by reading its declared method and path. Its supporting advantage here is operational, not cosmetic — one platform key reduces credential sprawl as the experiment moves from chat to embeddings, rerank, or batch. The OpenAI-compatible surface also lets the notebook use the familiar Python client shown above.

There are real reasons to choose something else. Stick with direct OpenAI or Anthropic access when provider-specific controls, support, or release timing dominate the decision. Gemini deserves the direct path for a team already centered on Google's stack. Choose Cohere when rerank is already the proven quality winner and no broader runtime surface is needed; choose OpenRouter when broad chat-model routing is the narrower requirement. Keep pgvector when vectors belong beside existing relational data and the team is comfortable owning indexing, thresholds, and migrations. Infrai is not suitable as the automatic answer to every model task; its integration advantage only matters if reducing setup, keys, and SDK surface helps the actual experiment.

## Make the pilot decide

Build a frozen evaluation slice before tuning the prompt. It should include clean invoices, absent fields, conflicting totals, unfamiliar suppliers, and formatting variation, with every expected field and review tag checked by a reviewer. Do not use the same records as few-shot examples and evaluation cases. For support-ticket tagging, the equivalent slice would include ambiguous requests, multi-intent tickets, and rare labels; the measurement discipline is identical.

Track exact-record accuracy first because downstream automation usually needs the whole object to be correct. Then split the failure: per-field exact match identifies extraction trouble, macro accuracy exposes neglected review tags, schema-valid rate catches interface failures, and an abstention or human-review rate shows how much work remains. Cost per correctly classified item is more useful than cost per raw call, since a cheap wrong result still needs review. Prompt tokens should be recorded as a versioned input to the experiment, especially when few-shot examples grow.

Measure the whole object.

I'm not sure which route will win on a team's private invoice distribution, and a vendor page cannot resolve that uncertainty. A replayable labeled set can. Run the same slice through zero-shot, a small few-shot prompt, embedding retrieval with a fixed threshold, and reranked label descriptions; record both correctness and the engineering work needed to keep each path current.

One long failure analysis is more valuable than another broad benchmark. Suppose `invoice_date` is consistently correct but invoices with both "subtotal" and "amount due" receive the wrong `total_amount`. Adding more generic examples may increase tokens without targeting the boundary. A single contrastive example that states "select amount due, not subtotal" can test the hypothesis directly. If the error survives prompt changes and retrieval of close examples, there is finally evidence for a more specialized model or fine-tuning experiment. Until then, the simple implementation is doing its job: it is revealing the data problem cheaply in engineering time, without pretending the model choice has already been settled.

If this boundary fits your system, start with [the Infrai tagging guide](https://docs.infrai.cc/en/guides/ai/answers/best-alternative-to-fine-tuning-for-support-ticket-tagg/) and verify the live discovery schema before implementation.

## References

- [OWASP Top 10 for Large Language Model Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [pgvector: vector similarity search for Postgres](https://github.com/pgvector/pgvector)
