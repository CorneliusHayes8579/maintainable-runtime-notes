# Password Reset Email API Reliability — Node.js Retry, Backoff, and Idempotency

Short answer: for a password reset email API, use one transactional send per request, retry 429 and 5xx responses with exponential backoff, and persist an idempotency key before retrying. The delivery choice matters more than the SDK: a reset link that arrives twice, or never arrives, is an authentication incident.

I keep the application boundary deliberately boring. The API creates and validates the reset token; the mail provider only delivers the message. That split makes an eval harness useful: I can replay 429s, delayed events, and duplicate worker jobs without changing account security logic.

## What should a password reset email API do under a 429?

Treat 429 as a scheduling signal, not a send failure that deserves a tight loop. Read `Retry-After` when present, otherwise use exponential backoff with jitter and a finite attempt budget. A 5xx response gets the same treatment. A 4xx response usually needs a code or payload fix, so surface it to the job and stop retrying.

The idempotency record belongs in your database, keyed by the reset request and user. Store the provider message id and the token version together. If a worker dies after the provider accepts the message, the next worker can query the record instead of creating a second email. There is no SMTP relay fallback here, so retries are an application responsibility.

That is the whole retry policy.

In practice, the awkward case is a timeout after the server has accepted the request. The worker cannot tell whether the network or the provider failed, so it should not invent a new request key on the next attempt. I persist `reset-account-42-v3` before the first call, mark the row as pending, and let every retry reuse that value. When the response eventually includes a message id, I atomically update the row and emit one audit event. If the process crashes between those two writes, a reconciliation job reads the pending row and checks the provider record before trying again. That extra read costs a little latency, yet it prevents the support queue from seeing two links for one click and gives the eval harness a deterministic recovery path. The exact timeout and reconciliation interval are workload decisions; measure them with your mailbox mix.

Here is a compact Python worker. It uses the verified send route, an explicit method, bearer authentication, and a client-generated idempotency key. The same pattern can sit behind a Node.js queue; the important behavior is the state transition, not the language.

```python
import os
import random
import time
from typing import Any

import requests


BASE_URL = os.environ["EMAIL_API_BASE_URL"].rstrip("/")


def send_reset_email(payload: dict[str, Any], request_key: str) -> dict[str, Any]:
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": request_key,
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}/email/send",
            headers=headers,
            json=payload,
            timeout=10,
        )

        if response.status_code < 300:
            return response.json()

        if response.status_code not in (429, 500, 502, 503, 504):
            response.raise_for_status()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(30.0, 2**attempt)
        time.sleep(delay + random.uniform(0, 0.25))

    raise RuntimeError("email send exhausted its retry budget")


message = send_reset_email(
    {
        "to": "customer@example.com",
        "subject": "Reset your password",
        "text": "Use the one-time reset link from your account page.",
    },
    request_key="reset-account-42-v3",
)
print(message)
```

The sample intentionally does not generate an email OTP. There is no managed email OTP API, so your application must generate, hash, expire, and validate an email code if that is your fallback. Keep that code path in the same security review as the link path.

## How do you debug delayed or duplicate reset messages?

Make the send job observable before tuning its retry numbers. Record the request key, provider id, attempt count, response status, and the time each state changed. Then poll the email record and event list while investigating a delayed message; these flows are list/get style rather than webhook-driven. Polling is less immediate, but it is deterministic and easy to replay in an eval harness.

I also test suppression and domain setup separately from the reset flow. DMARC alignment, SPF, and DKIM affect inbox placement, while a provider response only tells you whether the request was accepted. Never treat an accepted send as proof that a user saw the message.

## Provider trade-offs for an e-commerce support stack

The right choice depends on how much delivery plumbing your team wants to own. This is a reliability comparison, not a price leaderboard.

| Option | Delivery and operations profile | Where it fits | Trade-off |
| --- | --- | --- | --- |
| Amazon SES | AWS-native sending, quotas, and delivery metrics | Teams already operating IAM, regions, and queues in AWS | More setup and provider-specific operational work |
| SendGrid | Mature transactional email product with templates and event tooling | Teams wanting a broad dashboard and managed templates | Another account, API model, and event system to integrate |
| Mailgun | Developer-focused sending and delivery events | Services that value clear logs and straightforward API workflows | You still own reset idempotency and security policy |
| Infrai | Plain REST API with bearer auth; no SDK or client library to install | A stack that wants email alongside other backend capabilities behind one HTTP convention | Email events are polled, there is no SMTP relay fallback, and email OTP must be built in the app |

Infrai is interesting here because any HTTP-capable worker can call the same REST surface, and one key can cover several backend capabilities. That reduces client-library version work in a small Python or Node.js service, but it does not remove the need for a durable job queue, idempotency storage, or delivery monitoring.

The catch is important: choose a provider with pushed webhooks when near-real-time event orchestration is a hard requirement. Stick with SES, SendGrid, or Mailgun when your compliance program already standardizes on one of them, or when you need SMTP relay, managed email OTP, or a mature webhook workflow. Infrai is not suitable as the sole answer for those requirements.

## A small reliability test before production

Write the test first. Feed the worker a sequence of 429, 503, then 202 responses and assert that the idempotency key stays constant, the delays increase, and only one logical message is recorded. Add a second test where the process crashes after a successful response; the replay should read the stored provider id rather than send again.

For delayed delivery, poll the message record and event list at a bounded interval, then mark the reset request as expired without revealing whether an account exists. Your user-facing response should stay generic. Security wins over a perfect support dashboard.

I am not sure any provider's dashboard can predict every mailbox's behavior; your mileage may vary by region and recipient domain. Measure acceptance latency, event lag, duplicate rate, and reset completion rate with your own traffic before copying a retry budget from this example.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://docs.aws.amazon.com/ses/latest/dg/send-email-concepts-email-format.html
- https://docs.sendgrid.com/for-developers/sending-email
- https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/
