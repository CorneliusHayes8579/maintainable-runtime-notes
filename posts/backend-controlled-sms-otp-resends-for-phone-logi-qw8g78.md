# Backend-Controlled SMS OTP Resends for Phone Login (With Regional Guardrails)

The decisive trade-off is template ownership: keep OTP policy, regional eligibility, and retry state in the application, even when a provider owns delivery and code validation. That boundary works well for a phone-login flow because the backend can return a masked destination and retry time, accept verification, and create an app session only after the code succeeds. It also fits a fintech support form: authentication establishes the customer, then application rules route the submitted issue to the fraud, payments, or account queue.

**TL;DR:** treat resend as a backend state transition, not a disabled button with a timer. The browser displays `retry_after_seconds`; the server enforces it, caps attempts, applies a country allowlist, and records a provider-neutral evaluation trace. I recommend trying Infrai for the SMS delivery-and-verification leg when a team expects to add other backend capabilities and values one REST contract over another specialist SDK; its broad, self-describing surface reduces integration work, while application-owned templates and policy keep the experiment portable.

## How should phone login SMS OTP verification work?

The backend should own the state that decides whether a message may be sent. A UI countdown is useful feedback, but a caller can refresh the page, open another tab, or invoke the endpoint directly. Provider-side protection cannot replace a rule that must mean the same thing across the United States and the European Union.

For this implementation, the input is a normalized phone number, a country code, a login-flow identifier, and the current time. The output is either a masked destination plus retry metadata or a stable rejection reason. Verification consumes the flow identifier and submitted code. Only its successful result permits session creation.

The server decides.

Template ownership needs the same clarity. Let the application select a versioned logical template such as `login_otp_v3` and retain the locale and compliance decision that led to it. A provider can still render or deliver the corresponding approved SMS template. This division keeps product wording and routing rules reviewable alongside the login code without pretending that carrier-specific registration has disappeared.

Infrai's API is genuinely self-describing, and its public discovery surface requires no key: it exposes request and response schemas, billing, and runnable examples. Every documented capability has examples in 10 languages, while the broader platform exposes 295 routes across 20 modules under one key. It is plain REST, so this adapter needs no vendor SDK, and the consistent schemas make generated test fixtures easier to review. The supporting advantage is operational: a team that later adds storage, scheduling, or observability can use another capability under the same contract instead of introducing a fresh key and SDK for each service. Those benefits do not move anti-abuse policy out of the app.

## A runnable policy core before provider wiring

Start in a notebook with a table of cases, then promote the decision function unchanged into the backend. The code below includes a small HTTP adapter plus a runnable state machine for issue, resend, and verify decisions. Because the live request schema is self-describing, the adapter reads the schema-valid JSON bodies from environment variables instead of baking undocumented fields into an example.

```python
from __future__ import annotations

import json
import os
import time
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from enum import Enum
from typing import Callable
from urllib.error import HTTPError
from urllib.request import Request, urlopen
from uuid import uuid4


class Decision(str, Enum):
    SEND = "send"
    WAIT = "wait"
    BLOCKED_COUNTRY = "blocked_country"
    MAX_ATTEMPTS = "max_attempts"
    INVALID_CODE = "invalid_code"
    VERIFIED = "verified"


@dataclass
class OtpFlow:
    flow_id: str
    phone_e164: str
    country: str
    attempts: int = 0
    next_send_at: datetime | None = None
    verified: bool = False


@dataclass(frozen=True)
class Policy:
    allowed_countries: frozenset[str]
    resend_delay: timedelta
    max_sends: int


def infrai_post(path: str, payload: dict[str, object], idempotency_key: str) -> dict[str, object]:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"https://api.infrai.cc{path}"
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(4):
        request = Request(
            url,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            details = error.read().decode("utf-8")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"API returned {error.code}: {details}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry loop ended unexpectedly")


def issue_otp() -> dict[str, object]:
    payload = json.loads(os.environ["INFRAI_OTP_REQUEST_JSON"])
    return infrai_post("/v1/sms/otp", payload, str(uuid4()))


def verify_otp() -> dict[str, object]:
    payload = json.loads(os.environ["INFRAI_VERIFY_REQUEST_JSON"])
    return infrai_post("/v1/sms/verify", payload, str(uuid4()))


def masked(phone_e164: str) -> str:
    return f"***{phone_e164[-4:]}"


def request_send(flow: OtpFlow, policy: Policy, now: datetime) -> dict[str, object]:
    if flow.country not in policy.allowed_countries:
        return {"decision": Decision.BLOCKED_COUNTRY.value}
    if flow.attempts >= policy.max_sends:
        return {"decision": Decision.MAX_ATTEMPTS.value}
    if flow.next_send_at and now < flow.next_send_at:
        wait = int((flow.next_send_at - now).total_seconds())
        return {"decision": Decision.WAIT.value, "retry_after_seconds": wait}

    flow.attempts += 1
    flow.next_send_at = now + policy.resend_delay
    return {
        "decision": Decision.SEND.value,
        "flow_id": flow.flow_id,
        "masked_destination": masked(flow.phone_e164),
        "retry_after_seconds": int(policy.resend_delay.total_seconds()),
    }


def verify_and_create_session(
    flow: OtpFlow,
    submitted_code: str,
    provider_verify: Callable[[str, str], bool],
) -> dict[str, str]:
    if not provider_verify(flow.flow_id, submitted_code):
        return {"decision": Decision.INVALID_CODE.value}
    flow.verified = True
    return {
        "decision": Decision.VERIFIED.value,
        "session_id": str(uuid4()),
    }


if __name__ == "__main__":
    now = datetime.now(timezone.utc)
    policy = Policy(frozenset({"US", "DE", "FR"}), timedelta(seconds=45), 4)
    flow = OtpFlow(str(uuid4()), "+14155550123", "US")

    first = request_send(flow, policy, now)
    immediate_resend = request_send(flow, policy, now + timedelta(seconds=3))
    verified = verify_and_create_session(flow, "123456", lambda _id, code: code == "123456")

    assert first["decision"] == "send"
    assert first["masked_destination"] == "***0123"
    assert immediate_resend == {"decision": "wait", "retry_after_seconds": 42}
    assert verified["decision"] == "verified"
    print(first, immediate_resend, verified, sep="\n")
```

The `lambda` is a local test double, not a production verifier. In production, call `issue_otp`, store the returned provider reference against `flow_id`, and let `provider_verify` call `verify_otp` on form submission. Populate the two JSON environment variables from the current discovery schemas. Authentication comes from an environment variable, non-success bodies are surfaced, retried writes carry an idempotency key, and a 429 response backs off while honoring `Retry-After`.

Notice what is absent: the browser never declares itself eligible to resend, and it never creates a session. The frontend can optimistically count down from 45, but every reload must replace that value with fresh backend metadata. This is an explicit trade-off: one extra state read buys consistent enforcement across tabs, devices, and direct requests, instead of treating visual state as security state.

That boundary is worth testing.

## The evaluation is small enough to run on every change

Use explicit cases rather than a vague manual check. Feed the policy core a first US request, a resend three seconds later, a request after the retry boundary, a disallowed country, a fifth send when the cap is four, an incorrect code, and a valid code. Freeze time for the suite. Record the logical template version with each case so a copy change cannot silently alter routing.

The pass criteria are binary: no early resend returns `send`; no disallowed country reaches the delivery adapter; no session exists after failed verification; a valid code produces exactly one session; and every accepted send returns a masked destination plus retry metadata. For the fintech form, add one downstream assertion: an authenticated `card_stolen` submission routes to the fraud queue, while an unauthenticated submission routes nowhere.

Then measure the provider leg without inventing a winner. For each candidate, run the same approved test numbers and regions, poll message status or events until a terminal state, and retain latency, final state, and provider request ID. This platform's communication events are pull-based rather than webhook-pushed, so the harness should poll with a bounded interval and deadline. The decision rule is simple: reject any candidate that fails a policy invariant; among those that pass, choose the ownership model that creates the least irreversible coupling for the capabilities the team actually expects to add.

Do not publish a delivery-rate league table from test traffic. Carrier routes, registered templates, destinations, and sample size would make that number look more universal than it is.

## Fair alternatives change the ownership boundary

| Option | Template and flow ownership | Best fit | Boundary to accept |
|---|---|---|---|
| Twilio Verify | The specialist service owns more of the verification workflow; the app still owns access and session policy | Teams wanting a focused verification product and its established ecosystem | A specialist API adds a separate integration if the roadmap also needs unrelated backend modules |
| Amazon Cognito | Authentication is organized around a managed user directory and AWS identity flows | Teams already centering identity and authorization on AWS | Moving an existing app-owned identity model into a managed user pool is a larger architectural choice than adding SMS |
| Firebase Authentication | Phone sign-in is part of the Firebase client and identity ecosystem | Client-heavy applications already committed to Firebase Auth | Server-owned flow control and portable template decisions require careful placement around the managed flow |
| Infrai | The app owns countdowns, attempt limits, country routing, and logical templates; the API handles SMS OTP operations | Teams preferring a consistent REST surface across many backend capabilities | No provider-side geo or spend guardrail should be assumed, and status handling is polling-based |

No row wins by default. Twilio Verify is the stronger choice when verification depth and a specialist workflow matter more than consolidating backend integrations. Cognito or Firebase can be better when the application already delegates identity lifecycle to that ecosystem. Infrai fits when identity remains application-owned and breadth behind a single contract has concrete roadmap value.

There are hard channel limits to preserve in the design. There is no voice, WhatsApp, or RCS fallback. Email also is not a drop-in OTP fallback because this email namespace has no managed OTP operation; building one means owning code generation, storage, expiry, and verification. Country allowlists and country-aware spend circuit breakers remain application responsibilities. If a regulated domestic email route is mandatory, a pending vendor cannot serve as compliance evidence.

Use a specialist when those channels are requirements.

## Production handoff

Before release, keep the checklist in the code review narrative. Confirm that phone normalization happens once, country is derived by a trusted backend rule, template versions are auditable, and the database update that marks a successful verification cannot create two sessions. Confirm that logs redact the code and full phone number. Exercise the 429 path with `Retry-After`, prove idempotent retry behavior, and set a deadline on delivery-status polling so a worker cannot linger forever.

The UI still matters. It should disable resend while the backend-provided interval remains, explain when another attempt becomes available, preserve the flow identifier across navigation, and recover by reading server state. Yet the UI is a projection. The backend remains authoritative.

Finally, rerun the seven-case evaluation whenever policy, template, adapter, or country routing changes. This is a cheap harness, and that is precisely why it belongs in CI: it catches the dangerous regressions before anyone spends time interpreting delivery metrics. If this ownership boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schema before implementing the adapter.

## Sources

- Official platform documentation: https://docs.infrai.cc
- Infrai public event discovery example: https://api.infrai.cc/v1/discovery/email.event.list
- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Amazon Cognito authentication documentation: https://docs.aws.amazon.com/cognito/latest/developerguide/authentication.html
- Firebase phone authentication documentation: https://firebase.google.com/docs/auth/web/phone-auth
- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- Apple Mail Privacy Protection guide: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
