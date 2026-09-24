# PDF Format Migration in 2026: Hosted APIs, Local Libraries, and Load Latency

Short answer: choose a hosted PDF API when your team wants the conversion boundary operated for you and can budget network and queue latency; choose a local PDF library when template ownership, data residency, offline work, or a hard tail-latency target matters more than maintenance.

That answer changes in customer support. A ticket may arrive as a bundle of a scanned form, an HTML transcript, and a signed PDF. The job is to split those inputs, migrate them into the support template, then merge the result into one evidence packet. A fast demo hides the hard part: the packet has to remain legally legible when ten agents attach files at once, and a retry must not duplicate a page.

I build these flows around an eval harness before choosing the rendering boundary. The harness keeps fixtures for rotated pages, embedded fonts, annotations, and a 42-page bundle. It measures output fidelity and p50/p95 latency under the same concurrency that production expects. Fancy architecture can wait.

## What changes when template ownership is the decision axis?

Template ownership means more than who edits HTML. It includes the font files, page-break rules, form semantics, accessibility metadata, and the release process that decides when a template changes. With a local library, those assets live beside the worker image. A support team can pin the renderer and review every upgrade, but it also owns the security patches and the regression corpus.

A hosted API moves that operational boundary away from the application. The application still owns the template version and the acceptance test, while the service owns its conversion runtime. That can be a good split when the team ships RAG or agent features and doesn't want PDF binaries competing with model dependencies in the same image. It isn't a free abstraction: every input and output crosses a network boundary, and the remote queue becomes part of the user-visible latency.

For a migration, record one immutable operation key from the source hash and template version. Store it before acknowledging the queue message. If the worker sees the same key again, it should return the verified output rather than merge a second copy of page 17. This is where a seemingly harmless retry becomes a customer-support incident: an agent reopens a ticket, the queue redelivers an attachment, and the merge worker has no memory of the first attempt. The resulting packet may contain two copies of a signed authorization, which is harder to explain than a visibly failed export. I keep the operation record separate from the binary so a status check can answer “already verified” without downloading the document again; the record also carries the page count, output checksum, and template version used for that exact packet.

Keep retries boring.

```python
from dataclasses import dataclass
from hashlib import sha256


@dataclass(frozen=True)
class MigrationKey:
    source_sha256: str
    template_version: str

    @classmethod
    def from_bytes(cls, source: bytes, template_version: str) -> "MigrationKey":
        digest = sha256(source).hexdigest()
        return cls(digest, template_version)


def operation_id(key: MigrationKey) -> str:
    return f"{key.source_sha256}:{key.template_version}"
```

The operation ID is portable across a local worker and a hosted adapter. That matters when a team starts local for a small queue, then adds a remote conversion path for a burst without changing its audit model.

## How should hosted PDF APIs and local PDF libraries handle latency under load?

Do not treat “conversion time” as one number. Break the request into admission, upload, remote queue, conversion, download, and verification. A local call still has parse, render, write, and checksum stages. Emit each duration with the operation ID; otherwise an incident will say only “PDF was slow,” which isn't actionable.

Here is the load test shape I use for the support bundle. It sends the same fixture set through both adapters, warms connections, and records a histogram rather than trusting an average. I once saw a nominal 180 ms conversion hide a 2.8 s p95 after the connection pool hit its limit. That was a capacity signal, not a renderer defect.

```python
from concurrent.futures import ThreadPoolExecutor
from statistics import median
from time import perf_counter
from typing import Callable


def sample_latency(convert: Callable[[bytes], bytes], fixtures: list[bytes], workers: int) -> dict[str, float]:
    def timed(payload: bytes) -> float:
        started = perf_counter()
        result = convert(payload)
        if not result:
            raise ValueError("empty migrated document")
        return perf_counter() - started

    with ThreadPoolExecutor(max_workers=workers) as pool:
        seconds = list(pool.map(timed, fixtures))
    ordered = sorted(seconds)
    p95_index = min(len(ordered) - 1, int(len(ordered) * 0.95))
    return {"p50": median(seconds), "p95": ordered[p95_index]}
```

The test should vary bundle size and concurrency independently. A hosted boundary often shows a queue-shaped tail as concurrency rises; a local boundary shows CPU, memory, or file-descriptor pressure. Both can be healthy. The useful question is whether the tail fits the support agent's wait budget after retries and verification, not whether one path wins a toy benchmark.

My rule is to reserve concurrency per tenant and apply backpressure before a ticket is promised an export. A 429-like throttle signal should pause with jitter. A deadline means the outcome is unknown until the operation record is checked; it does not prove that the remote conversion failed. With a local renderer, write to a temporary file and atomically rename only after the checksum and page count pass.

## Which fidelity and operations checks survive production?

Pixel equality catches visible shifts, but structure checks catch different failures. Compare extracted text, page count, rotation, form-field values, annotation appearance, embedded-font names, and accessibility tags. For scanned pages, compare the OCR text separately from the image. A merged packet can look fine in a browser and still lose a signature field in a downstream archive.

Keep a golden fixture for every template release. When a migration changes, the eval harness should report the first page and object that differs, along with token and byte deltas. I track token cost for the agent that classifies bundle parts too; sending a whole 42-page packet to a model just to find three separators is an avoidable bill.

Observability needs four identifiers: ticket ID, source hash, template version, and renderer or provider request ID. Log them as structured fields, not prose. Redact document contents. Retain enough metadata to replay a fixture without retaining sensitive customer text forever.

There is a practical limit. Local libraries are a poor fit when your team cannot patch native dependencies or when templates change weekly across many formats. Hosted conversion is not suitable when bytes must remain inside a controlled network, the workflow runs offline, or the p99 budget leaves no room for transfer and queueing. Stick with a pinned local library in those cases; change the boundary, not the timeout.

## A decision rule for a customer-support migration

Start local when the support organization owns a stable template set, has strict residency rules, or needs predictable sub-second tail latency inside one region. Pin the library and font bundle, isolate the worker, and make the fixture suite a release gate.

Start hosted when the organization needs several input formats quickly, has a small platform team, and can accept measured network variance. Keep template versions, idempotency, verification, and audit events in your service. The hosted option should be replaceable behind an adapter; your support workflow shouldn't know which renderer produced page 6.

I’m not sure a single threshold can choose for every team. Region, page complexity, and attachment mix move the tail. Publish your own p50, p95, and p99 from representative bundles, then revisit the decision when those distributions change.

Ownership decides the operational burden. Local means you own the machinery. Hosted means you own the boundary contract and the data path. Pick the one whose failure modes your team can observe and explain to a support agent at 09:00 on a busy Monday.

## References

- MDN Web Docs, Blob API: https://developer.mozilla.org/en-US/docs/Web/API/Blob
- PDF Association, PDF/A resources: https://www.pdfa.org/resource/
- ISO 32000-2 overview, PDF 2.0: https://www.iso.org/standard/75839.html
- W3C Web Content Accessibility Guidelines: https://www.w3.org/TR/WCAG22/
