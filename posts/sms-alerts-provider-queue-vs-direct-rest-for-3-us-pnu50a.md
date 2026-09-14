# SMS Alerts Provider: Queue vs Direct REST for 3 US/EU Appointment and Shipping Events

For a property-management marketplace, the best SMS alerts provider design is usually queue-first when a seller must hear about a new order reliably; the same decision covers appointment reminders, shipping alerts, and account activity. Direct REST sending is reasonable for a tiny, low-risk flow where a few seconds matter more than replay control. The choice is about failure handling, not a vendor badge.

Short answer: put the order event in a durable queue, render a versioned template, apply suppressions before sending, and record an auditable delivery state for every US or EU recipient. Use direct REST only when the request can safely be retried and the business accepts weaker recovery after a timeout.

I build RAG and agent features in Python, so I tend to start in a notebook and then ask an eval harness to punish optimistic assumptions. SMS alerts deserve the same treatment. “The API returned 200” is not a delivery SLO. It is one observation in a longer chain: order commit, enqueue, policy check, provider acceptance, carrier handoff, and a seller actually seeing the message.

## What should a simple REST API prove for SMS alerts, templates, and suppressions?

Start with three concrete events in the marketplace: a new order, an appointment reminder for a property visit, and an account-activity warning such as a changed payout destination. Give each event an immutable ID and an event time. The notification record should point to that ID, recipient region, consent state, template version, and a reason code if the message is suppressed.

The API contract needs fewer promises than most demos imply. It should accept an idempotency key supplied by your service, return a provider message identifier, and expose a status-read operation or a documented callback. If status cannot be queried, a timeout after submission is an ambiguity, not proof of failure. Retrying blindly can turn one order into two texts, and a `429` response should consume a bounded retry budget rather than hold the checkout request open.

Templates should be data, not strings scattered through handlers. Keep a US and an EU variant when sender identity, language, or legal copy differs. Render from the same order facts used by the dashboard, then validate length, currency, property address truncation, and a human-readable support path in CI. Never let a model invent the order number or appointment time; those fields come from the event payload.

Suppressions belong before the network call. Check opt-out state, quiet hours, duplicate event IDs, blocked destinations, and regional policy. A suppression is a successful policy decision and needs its own metric. Otherwise an on-call engineer cannot tell “we respected an opt-out” from “the SMS service vanished.”

Here is a deliberately small Python boundary. It keeps transport details out of the state machine and makes the retry key visible in tests.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class OrderAlert:
    event_id: str
    phone_e164: str
    region: str
    body: str


class SmsGateway(Protocol):
    def send(self, *, to: str, body: str, idempotency_key: str) -> str:
        """Return the remote message id after acceptance."""


def send_order_alert(alert: OrderAlert, gateway: SmsGateway) -> str:
    key = f"order-alert:{alert.event_id}"
    return gateway.send(
        to=alert.phone_e164,
        body=alert.body,
        idempotency_key=key,
    )
```

The snippet is an interface, not a claim about any particular service. Your adapter must map its real REST path and status vocabulary, and tests should assert that the same event produces the same key after a worker restart.

## How do queue-first and direct REST paths behave under US/EU delivery pressure?

The direct path sends during the order request. It is easy to trace and can shave queue latency, but a slow network call now shares the checkout timeout. A caller retry, load balancer retry, or lost response can repeat the message unless the remote API and your database agree on idempotency.

Queue-first commits the order and notification intent together, then lets a worker perform policy checks and send. The seller may receive the alert a little later. In exchange, the worker can back off on rate limits, isolate regional credentials, replay a transient failure, and keep checkout available while the messaging system is being repaired.

That trade is visible in a small table:

| Path | Strong point | Failure to measure | Better fit |
|---|---|---|---|
| Direct REST | Lowest scheduling latency | Ambiguous timeout or duplicate on caller retry | Low-volume, non-critical notices |
| Durable queue | Replay, backoff, and checkout isolation | Stale queue items can miss an appointment window | New orders and account activity |
| Hybrid | Immediate attempt plus durable reconciliation | More states and harder evaluation | High-value events with a tight deadline |

For a new marketplace order, I would choose queue-first and set a deadline derived from the seller promise, not from the HTTP request timeout. For an appointment reminder, the deadline is the appointment start minus a useful reading window. For account activity, a duplicate is annoying but a missing warning can be serious; that usually favors durable reconciliation.

Keep the state monotonic: `created`, `suppressed`, `accepted`, `delivered`, `failed_permanent`, or `expired`. “Unknown” deserves a separate state when the submission response was lost. It should trigger reconciliation, not a hopeful success label. A compare-and-swap transition around `accepted` prevents two workers from sending the same event after a queue redelivery.

## A delivery evaluation that survives notebook-to-prod

Before selecting a provider or architecture, build a fixture set of order events: short and long property names, accented EU names, currencies, duplicate webhook deliveries, an opted-out seller, and a phone number that changes region metadata. Replay the fixtures through a fake gateway, then through a sandbox with synthetic destinations. Record acceptance latency, terminal-status latency, duplicate attempts, suppression accuracy, and the age of the oldest pending job.

I once treated a green send response as the finish line in an eval. It passed every happy-path assertion and still left no way to explain a missing notification. The fix was unglamorous: persist the event before sending, include a correlation ID in structured logs, and test a lost response as its own branch. We then replayed the same fixture through a worker restart, a delayed status read, a `429`, and a duplicate queue delivery; each branch had to end in one auditable state, while the original order remained visible to the seller. That test caught a 17-second retry gap in our harness; your mileage may vary because carrier reporting and regional routing differ.

Measure message usefulness, too. A shipping-style alert that omits the order ID forces the seller to open the dashboard, while a verbose address can trigger extra SMS segments. Track rendered character count, segment count, and the fraction of alerts that are acknowledged in the seller application. Do not put phone numbers or message bodies in metric labels.

Measure first.

The evaluation should include cost as a constraint, not as the headline. Token cost is a concern in my agent work, and SMS has its own budget: retries, long Unicode text, and duplicated sends consume it quickly. A design that looks inexpensive per request can be expensive when its ambiguity policy is “send again.”

## Where the recommendation stops applying

Queue-first is not suitable when the event is disposable, the volume is tiny, and adding a worker would create more operational surface than the notification warrants. Direct REST is also a poor fit when the checkout request cannot tolerate a provider timeout, or when the provider does not document idempotent submission and status retention.

Stick with an email or in-app notice when a seller has not consented to SMS, when local quiet-hour rules cannot be evaluated, or when the message contains a secret that should not travel by text. For account recovery and one-time codes, follow the rate-limit, single-use, and non-disclosure guidance in the OWASP cheat sheet rather than copying an order-alert template.

The practical decision rule is simple: choose the path whose failure evidence you can observe and repair before the business deadline. A REST API, templates, and suppressions are useful parts; they are not the reliability strategy by themselves.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://senders.yahooinc.com/best-practices/
- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc9110
