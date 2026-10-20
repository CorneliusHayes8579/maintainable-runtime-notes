# Email Deliverability Platform Comparison: API Domain Controls Explained for Signup

A marketplace signup flow needs one outcome: the verification link arrives, and the application can tell what happened afterward. The least complex option is an email delivery platform whose API can sit behind a small adapter while domain verification, DKIM key changes, suppressions, and delivery events remain testable outside its dashboard.

**TL;DR:** compare candidates by integration evidence, not by feature count. Run the same acceptance fixture against each candidate: one sending domain, a planned key rollover, a suppressed recipient, and an event cursor that survives a process restart. Keep regional and compliance claims in a separate evidence review. A platform passes only when the signup service can preserve its own message ID, reason about terminal outcomes, and avoid sending again to an address known to be suppressed.

## How should an email deliverability platform comparison test its API and domain?

The application should own the verification token, expiry, recipient, template data, and an internal attempt ID. The delivery adapter should translate that intent into a provider request and translate provider events back into a deliberately small vocabulary. This boundary keeps a notebook experiment useful when it graduates into a worker: the prompt or template can change, but account state never depends on a dashboard label.

The data flow is short. The signup API creates an account in a pending state and enqueues a delivery intent. A worker renders the email, checks the local suppression projection, calls the delivery adapter, and stores the returned external ID beside its internal attempt ID. A poller later reads events using a durable cursor and updates delivery evidence. Clicking the link verifies the account independently of any “delivered” event; delivery telemetry is evidence about transport, not proof of identity.

Keep that distinction sharp.

This ownership line matters more than a long capability matrix. **The smallest useful abstraction has two outbound operations and one normalized event type.** It does not pretend that every platform has identical semantics.

## A runnable acceptance adapter

Start with a fake transport. It makes the contract executable before credentials, DNS changes, or a platform SDK enter the repository. The fixture below uses two recipients because they exercise the two branches that matter first: accepted for delivery and locally suppressed.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol


@dataclass(frozen=True)
class DeliveryIntent:
    attempt_id: str
    recipient: str
    verification_url: str


@dataclass(frozen=True)
class DeliveryEvent:
    external_id: str
    kind: str
    occurred_at: datetime
    cursor: str


class MailTransport(Protocol):
    def send_verification(self, intent: DeliveryIntent) -> str: ...
    def poll_events(self, after: str | None) -> tuple[list[DeliveryEvent], str | None]: ...


class SignupMailer:
    def __init__(self, transport: MailTransport, suppressed: set[str]) -> None:
        self.transport = transport
        self.suppressed = {address.casefold() for address in suppressed}

    def send(self, intent: DeliveryIntent) -> str | None:
        if intent.recipient.casefold() in self.suppressed:
            return None
        return self.transport.send_verification(intent)


class FakeTransport:
    def __init__(self) -> None:
        self.sent: list[DeliveryIntent] = []

    def send_verification(self, intent: DeliveryIntent) -> str:
        self.sent.append(intent)
        return f"ext-{intent.attempt_id}"

    def poll_events(self, after: str | None) -> tuple[list[DeliveryEvent], str]:
        event = DeliveryEvent(
            external_id="ext-signup-0001",
            kind="accepted",
            occurred_at=datetime(2026, 10, 19, tzinfo=timezone.utc),
            cursor="cursor-0001",
        )
        return ([event] if after is None else []), event.cursor


transport = FakeTransport()
mailer = SignupMailer(transport, {"blocked@example.test"})

accepted_id = mailer.send(
    DeliveryIntent("signup-0001", "buyer@example.test", "https://market.example.test/v/opaque-token")
)
blocked_id = mailer.send(
    DeliveryIntent("signup-0002", "blocked@example.test", "https://market.example.test/v/other-token")
)

assert accepted_id == "ext-signup-0001"
assert blocked_id is None
assert len(transport.sent) == 1
```

The fake is intentionally boring. Replace it with each candidate adapter and preserve the assertions. Then add contract tests for timeout behavior, duplicate events, missing fields, and cursor replay. I would keep those tests next to the application adapter rather than inside an exploratory notebook, because this is the exact point where a quick experiment becomes production infrastructure.

Two records are enough to begin.

Template rendering deserves one explicit test too. Mustache variables are HTML-escaped by default, while triple mustaches and ampersand forms are unescaped. A verification URL therefore belongs in an ordinary variable unless the template has a reviewed reason to inject unescaped markup. Feed a URL containing `&` into the renderer and inspect the actual linked destination, not just a string snapshot.

## Compare the hard integration edges

Ask every candidate to demonstrate the same workflow in a disposable domain. First, automate domain-status checks and record the DNS values returned by the platform. Do not accept “green in the dashboard” as the only machine-readable result. The useful question is whether deployment automation can distinguish pending, verified, and failed states without scraping a page.

Next, rehearse DKIM rotation as an overlap, not a dramatic switch. The evaluation should reveal whether a new selector can be published and verified while the old selector remains valid for mail already in flight. Record which party initiates rotation, how status is exposed, and what rollback looks like. Yahoo's sender guidance explicitly calls for DKIM authentication and recommends rotating DKIM keys periodically; the comparison still needs to establish how each platform exposes that work to your team.

Suppression is another ownership trap. A marketplace may receive a permanent failure through the event stream after the send call was accepted. Project that result into an application-readable suppression store, retain the original provider reason as evidence, and make the next attempt fail closed before another API call. The platform can maintain its own list as well, but an invisible remote list is a poor dependency for account-support tooling and migration tests.

There is a real trade-off here. A local suppression projection improves portability and gives support staff one place to inspect account state, but it also creates sensitive data that the application team must secure, retain deliberately, and reconcile with late events. Teams unwilling to own that lifecycle should prefer a thinner adapter and treat the platform's suppression system as authoritative, provided they have verified export, lookup, and migration behavior. Neither design wins by default.

Finally, test polling under interruption. Fetch a page, persist both normalized events and the next cursor atomically, terminate the worker, and restart it. The acceptance condition is boring but strict: no lost terminal event, duplicates tolerated, and no assumption that arrival order matches send order. Polling interval is a product decision based on the signup experience; correctness comes from durable progress and idempotent updates.

Polling also has limits. It is a poor fit when the product requires near-immediate status changes but the platform exposes only coarse polling windows, and aggressive polling adds request volume plus operational noise. Webhook-first delivery may fit that case better, as long as signature verification, duplicate delivery, ordering, and replay are tested with the same rigor. Conversely, a polling API can simplify ingress security and recovery when a several-minute delay is acceptable. The comparison should record that latency budget before anyone scores the interface.

| Evidence to request | Pass condition | Integration warning |
|---|---|---|
| Domain status through an API | Automation can identify pending, verified, and failed states | Dashboard-only state creates a manual release step |
| DKIM rollover rehearsal | Old and new selector states are observable during overlap | A single opaque “active” flag makes rollback hard to verify |
| Suppression export and events | The app can explain and prevent a repeat attempt | Remote-only suppression couples support work to one console |
| Polling cursor recovery | Restarting from the saved cursor loses no terminal outcome | Page-number polling can drift as new events arrive |
| Regional and compliance evidence | Contracts, data-flow answers, and configured behavior agree | A region label alone does not answer the review |

## Keep compliance out of the feature-score shortcut

EU/US SaaS compliance cannot be reduced to a boolean column. Ask where message content, recipient identifiers, event history, backups, and support access are handled; then have the appropriate legal and security owners evaluate the answers and contract terms. The engineering comparison should capture configured data flows and retention controls without declaring that an API response settles a legal conclusion.

This is also where prompt-cost awareness helps, even though no model belongs in the delivery path. Do not send raw event streams to an LLM to classify routine outcomes. Normalize documented event types with code, retain the source payload under the team's data policy, and use a small labeled fixture as an eval set. A model can assist an operator with ambiguous narrative later, but deterministic bounce and suppression handling should stay deterministic.

Avoid making price the deciding score. Integration effort is easier to defend when measured as concrete ownership: adapter code, DNS automation, rollover rehearsal, event recovery, support diagnostics, and compliance evidence collection. Count the manual steps and failure branches revealed by the fixture. Those costs survive changes to a public rate card.

## The release gate is an operational rehearsal

Before launch, run the verification flow through a newly configured domain, confirm that the link resolves to the intended account action, and observe the accepted event through the poller. Repeat with a known suppressed address and confirm that no network send occurs. Rotate a test selector with overlap, restart the event worker between pages, replay one page, and verify idempotent state. Then capture the DNS state, cursor checkpoint, suppression reason, and retention configuration as review evidence.

One more failure is worth forcing: let the delivery call time out after the remote side may have accepted it. The worker must not mint a fresh verification intent blindly. Reconcile using the stable attempt ID or use an idempotency facility whose behavior you have verified in the candidate's documentation and contract test.

Timeouts lie.

Ship only after that rehearsal passes in CI and in the configured environment. **Choose the platform that makes this evidence cheap to produce and easy to repeat.** A crowded dashboard cannot compensate for an integration your team cannot evaluate, recover, or explain.

## Further reading

- https://mustache.github.io/mustache.5.html
- https://senders.yahooinc.com/best-practices/
