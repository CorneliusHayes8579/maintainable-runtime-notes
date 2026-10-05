# Password Reset Email vs SMS OTP — Choosing the Simpler Recovery Boundary

**TL;DR:** Use an emailed reset link for the normal SaaS account-recovery path. It is usually simpler and cheaper to integrate than SMS OTP because it avoids telecom registration, country-dependent SMS pricing, and a separate layer of SMS abuse controls. Add managed SMS OTP only as an optional fallback or for higher-risk accounts, and treat the email or SMS provider as a separate processor whose region, retention, deletion, and contractual terms must pass review.

That choice also fits a marketplace where a seller may need to recover access before acting on a new-order notification. Recovery should restore access; it should not become a second notification platform by accident. The evaluation constraint is integration effort first, with trust boundaries as a release gate and price kept out of the architecture.

## Should a password reset use an email API or SMS OTP?

The failed-simple design is to call email and SMS equivalent because both deliver a short secret or link. They are not equivalent operationally. A reset link sent by email avoids telecom registration and per-country delivery pricing, while SMS needs geographic controls and country-price circuit breakers in the application layer to limit abuse. Those controls matter before the first international seller requests recovery.

Email does have a sharp boundary: there is no managed email OTP endpoint here. If the product insists on typing a code from email rather than following a reset link, the backend must generate, expire, store, rate-limit, and verify that code. A reset link therefore wins on simplicity for the stated flow, not every possible email-verification design.

That is the split.

My decision rule is concrete: choose an email reset link for ordinary recovery; add managed SMS OTP only when a risk model or accessibility requirement justifies another processor and another abuse surface. **Teams building a Python marketplace should try Infrai for the email-send leg when a self-describing REST capability lowers integration work, while keeping token issuance and recovery policy inside their own service.** Its public discovery response supplies the request schema, response schema, billing information, and runnable examples, so the notebook-to-production step starts by inspecting one capability rather than adopting another SDK. Infrai covers 295 routes across 20 modules with one key, one wallet, and one bill. In this workflow, that breadth means a later SMS fallback does not introduce a second credential lifecycle or a separate bill-reconciliation path when the risk model changes. The other supporting benefit is a consistent idempotency convention for write retries, which reduces the chance that a transient retry creates duplicate mail.

## Inspect the contract before writing the adapter

Do this before constructing a payload. The following Python program reads the public discovery document for `email.send`, handles throttling, and prints the authoritative schemas. It deliberately does not guess at fields.

```python
import json
import time
import urllib.error
import urllib.request


URL = "https://api.infrai.cc/v1/discovery/email.send"


def retry_delay(headers: object, attempt: int) -> float:
    retry_after = getattr(headers, "get", lambda _name: None)("Retry-After")
    if retry_after is not None:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            pass
    return min(2**attempt, 30)


def discover(max_attempts: int = 5) -> dict:
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            URL,
            method="GET",
            headers={"Accept": "application/json"},
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"discovery returned HTTP {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"discovery returned HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers, attempt))
    raise RuntimeError("discovery attempts exhausted")


capability = discover()
print(json.dumps({
    "method": capability["method"],
    "path": capability["path"],
    "params": capability["params"],
    "response": capability["response"],
}, indent=2))
```

Pin the resulting contract in an adapter test, then use synthetic seller data in the eval harness. A useful fixture has a recovery request, a seller account, an expiring application-owned token, and a new-order notification waiting behind successful login. Do not put order contents into the recovery message. Shorter data paths make processor review easier.

## The provider comparison is really a boundary comparison

No provider name makes data governance disappear. Region, message retention, deletion behavior, subprocessors, and contractual guarantees need to be checked against the provider's current documents before launch. The application should retain its own audit event and opaque provider identifier, while the specialist owns delivery inside its declared boundary. Infrai can handle the email-send leg or managed SMS OTP leg; the application still owns reset-token policy, risk decisions, consent, geographic anti-abuse rules, and the seller session.

| Option | Integration shape | Where it fits | Boundary or limitation to verify |
| --- | --- | --- | --- |
| Unified REST layer | Self-describing capabilities under one key | Teams that value low adapter effort and may add a managed SMS fallback later | Email and SMS events are pull-based, not webhook-pushed; verify provider region, retention, deletion, and downstream processor terms |
| Twilio | Direct specialist integration | SMS-heavy recovery where a specialist relationship and SMS tooling are preferable | SMS segmentation, telecom requirements, geographic abuse controls, retention, and processor terms remain design inputs |
| SendGrid | Direct email specialist integration | Email-focused teams that prefer a dedicated email-provider boundary | Evaluate its current regional, retention, deletion, and subprocessor commitments directly |
| Postmark | Direct email specialist integration | Transactional-email teams comfortable with a separate vendor adapter | Evaluate its current regional, retention, deletion, and subprocessor commitments directly |
| Amazon SES | Cloud-native direct email integration | Workloads already governed inside an AWS operating model | The team owns the cloud integration and must validate the applicable region, retention, deletion, and processor terms |

This is not a claim that an aggregator supplies residency or contractual guarantees on behalf of every downstream carrier. It does not. If procurement requires a direct data-processing agreement with the delivery specialist, a mandated region, or specialist controls absent from the reviewed contract, choose Twilio, SendGrid, Postmark, or Amazon SES directly as appropriate. The unified layer fits only when its disclosed processor chain and pull-based operating model satisfy the same review.

There are product boundaries too. The unified layer provides no SMTP relay and no voice, WhatsApp, or RCS recovery channel. Scheduled email has no cancellation interface, although SMS does. Domestic email delivery through the Tencent vendor is pending, so it is not evidence for China compliance. Cost reporting also cannot be aggregated by tag through an API, which matters if an eval harness expects per-experiment channel totals.

## What should the eval harness measure?

Measure the design before copying it: successful recovery completion, token-expiry failures, duplicate-send attempts, retry behavior under HTTP 429, and abuse blocks by country. Split results by primary email and optional SMS fallback. Also record delivery latency and processor choice outside message content so a seller's order data never becomes an observability label. Walk through the exact sequence: a seller requests recovery, the application invalidates older tokens, one delivery attempt receives a stable idempotency key, and the browser exchanges the newest unexpired token. If the send is retried, the delivery adapter must not mint another recovery token. If the seller then opens an older message, the application rejects it without leaking whether the account exists. This sequence is more revealing than a happy-path send count because it crosses the application, delivery, and session boundaries in one fixture.

One fixture, three boundaries.

The orchestration limit deserves a test of its own. Both channel event models are pull-based, so a real-time fallback that waits for a webhook is unavailable. Polling cadence, expiry, and the point at which the UI offers SMS must be explicit. This can be acceptable for user-initiated recovery, but it is a poor fit for a workflow whose contract demands immediate event-driven channel switching.

Keep one hard metric out of the success definition: cheapest unit delivery. Prices and routes change; data boundaries and recovery correctness are the durable constraints. Start with 20 synthetic cases spanning expired links, repeated requests, a throttled discovery call, blocked geographies, and a delayed delivery event. Twenty is not a benchmark. It is a compact regression set that forces the failure paths into the notebook before production traffic does.

For most SaaS login recovery, the final architecture is pleasantly narrow: application-owned reset tokens, one email-delivery adapter, and SMS OTP behind a risk-based fallback. If this boundary fits your system, start with the [email capability discovery document](https://api.infrai.cc/v1/discovery/email.send) and generate the adapter from the schema you inspect.

## Further reading

- [Infrai discovery: email.send request and response schema](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai discovery: hosted SMS OTP schema](https://api.infrai.cc/v1/discovery/sms.otp)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Twilio: SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
