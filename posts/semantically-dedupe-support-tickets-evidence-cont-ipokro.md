# Semantically Dedupe Support Tickets: Evidence Contracts for Hosted Scoring APIs

The best way to semantically dedupe support tickets through an API is to treat its score as evidence-bound reranking, not permission to merge. Retrieve a small candidate set, rerank each candidate against the new ticket, and expose the matching text plus its source identifier before a person or policy decides what to do.

**TL;DR:** For an education support queue, a managed scoring API can remove model-hosting work, but it cannot define what counts as a duplicate. Keep normalization, candidate eligibility, thresholds, and citation records in your application. Measure false merges separately from missed duplicates, because joining two students' unrelated enrollment or billing cases is usually the more damaging error.

This architecture also answers the grounding problem. A score without inspectable evidence is only a sorting signal; a duplicate suggestion should point back to the exact prior ticket fields that produced it.

## How should an API dedupe support tickets semantically?

Two tickets can share vocabulary while asking for different remedies. "I cannot open module 4" and "module 4 shows the wrong completion status" mention the same course unit, yet one concerns content access and the other concerns progress tracking. Names, copied assignment instructions, automated signatures, and long error templates can dominate the representation while the decisive sentence gets diluted.

The reverse happens too. "My quiz submission vanished" and "the assessment reset after I clicked submit" may describe one incident with little lexical overlap. Semantic retrieval is valuable precisely because keyword matching can miss that relationship, but a vector distance does not prove identity. The original retrieval-augmented generation work separates parametric generation from retrieved non-parametric memory and conditions output on retrieved passages. The practical lesson here is narrower: preserve retrieved records as evidence rather than collapsing their information into an unexplained label.

Eligibility filters belong before semantic comparison. Compare tickets within a defensible scope such as institution, product area, access boundary, and a time window chosen from the support policy. These rules are application semantics, so delegating them to a model endpoint would hide the most important part of the decision.

It can't decide policy.

## Build the evidence path before tuning the threshold

The data flow is compact. Normalize stable boilerplate, retrieve plausible earlier tickets from the permitted partition, rerank those candidates against the new issue, and return a structured suggestion containing the score, matched excerpts, and immutable ticket identifiers. A downstream rule may auto-link low-risk records, request review, or decline to act. It should never infer a merge merely because the first result exists.

Here is a runnable local shape for that contract. The token-overlap scorer is deliberately modest; replace only `score_pair` with an API adapter during a notebook experiment. The surrounding evidence and policy code should survive that replacement.

```python
from dataclasses import asdict, dataclass
import re
from typing import Iterable


@dataclass(frozen=True)
class Ticket:
    ticket_id: str
    institution_id: str
    category: str
    subject: str
    body: str


@dataclass(frozen=True)
class MatchEvidence:
    candidate_id: str
    score: float
    query_excerpt: str
    candidate_excerpt: str
    decision: str


def terms(value: str) -> set[str]:
    return set(re.findall(r"[a-z0-9]+", value.lower()))


def score_pair(query: Ticket, candidate: Ticket) -> float:
    query_terms = terms(f"{query.subject} {query.body}")
    candidate_terms = terms(f"{candidate.subject} {candidate.body}")
    union = query_terms | candidate_terms
    return len(query_terms & candidate_terms) / len(union) if union else 0.0


def rerank(query: Ticket, candidates: Iterable[Ticket]) -> list[dict]:
    evidence = []
    for candidate in candidates:
        if candidate.institution_id != query.institution_id:
            continue
        if candidate.category != query.category:
            continue
        score = score_pair(query, candidate)
        decision = "review" if score >= 0.45 else "distinct"
        evidence.append(MatchEvidence(
            candidate_id=candidate.ticket_id,
            score=round(score, 3),
            query_excerpt=query.body[:160],
            candidate_excerpt=candidate.body[:160],
            decision=decision,
        ))
    ranked = sorted(evidence, key=lambda item: item.score, reverse=True)
    return [asdict(item) for item in ranked]


new_ticket = Ticket(
    "T-104", "school-7", "assessment", "Quiz answer missing",
    "My submitted quiz reset when I reopened the lesson.",
)
prior = [
    Ticket(
        "T-087", "school-7", "assessment", "Submission disappeared",
        "The quiz reset after I clicked submit and returned to the lesson.",
    ),
    Ticket(
        "T-091", "school-8", "assessment", "Quiz reset",
        "A learner says a completed attempt is now blank.",
    ),
]
print(rerank(new_ticket, prior))
```

The `0.45` value makes the program executable; it is not a production recommendation. Calibrate the real boundary on labeled cases from the same queue and category. More important, return evidence even when the score is excellent. This catches a common notebook-to-production mistake: optimizing ranking metrics, then shipping an opaque boolean that reviewers cannot audit.

The API boundary can remain small: accept versioned text inputs and candidate identifiers, then receive scores with explicit IDs. Record the adapter version, input-normalization version, and policy version beside every suggestion. Timeouts should yield "no suggestion" rather than "duplicate," and retries need a stable request identifier so an intermittent response cannot create multiple actions.

A Node.js service can call that boundary just as easily as a Python worker; the contract matters more than the client runtime. Model hosting stays outside the application, while authorization and merge policy stay inside it.

## Make false merges visible

A single accuracy number conceals the operational cost. Build a frozen evaluation set with confirmed duplicates, confirmed non-duplicates, and hard negatives that share courses, error strings, or assignment text. Split results by category and institution policy where permitted. Report precision among suggested duplicates, recall over known duplicate pairs, and the review rate at each threshold.

Keep the retrieval stage and reranking stage separable in the harness. If the correct prior ticket never enters the candidate set, changing the reranker cannot repair recall. If it enters at rank 20 but the reranker promotes a boilerplate-heavy false match, inspect normalization and examples before buying more context tokens. This separation makes experiments useful. It also keeps prompt and token cost attributable: count candidate pairs and submitted characters per resolved ticket rather than staring at one aggregate invoice.

There is a subtle labeling trap. Tickets closed under the same incident are useful positive evidence, but closure metadata is not automatically ground truth; agents may bulk-close related yet distinct requests. Sample disagreements, retain adjudication notes, and version the labeled set.

No invented certainty.

This approach has real limitations. It is not suitable when policy forbids sending ticket text to an external scoring API, when the queue is too small to produce a representative labeled set, or when exact identifiers already determine duplicates without semantic ambiguity. In those cases, keep processing inside the approved environment or use deterministic matching. The trade-off is clear: hosted scoring reduces model operations, but adds a data-processing boundary and dependence on another service's latency and version behavior.

## Operate the matcher as a reversible suggestion

Deploy in shadow mode first, writing suggestions and evidence without changing ticket state. Review the highest-scoring false positives and the lowest-scoring confirmed duplicates. Once the policy is stable, prefer a reversible link over destructive consolidation, especially when ticket histories contain student-specific access or assessment details.

The production checklist is short enough to keep in prose. Confirm that authorization filters execute before retrieval, raw text retention matches policy, every suggestion carries source ticket IDs and excerpts, and missing or malformed API responses fail closed. Alert on candidate-count shifts, score-distribution shifts, review acceptance, latency, and input volume. Re-run the frozen set whenever normalization, provider adapters, thresholds, or category rules change. Pin those versions in the decision record so a later audit can reproduce why a pair was surfaced.

Finally, give reviewers a real escape hatch: "related but distinct" should be a first-class outcome, not a comment field. That label is useful future evaluation data and prevents teams from forcing nuanced support work into a brittle binary.

## Choose by contract, not leaderboard position

For an externally hosted scorer, compare candidates on data handling, regional processing, retention controls, batch limits, timeout behavior, stable identifiers, versioning, and the ability to run a representative evaluation without rewriting the application. Model quality matters, but only on the queue's labeled cases and inside the eligibility policy. A leaderboard cannot tell you the cost of a false merge involving two different learners.

The durable design is provider-agnostic: retrieval proposes, reranking orders, policy decides, and evidence explains. Keep those boundaries explicit, and changing a scoring service becomes an adapter and re-evaluation task rather than a rewrite of support operations.

## References

- https://arxiv.org/abs/2005.11401
