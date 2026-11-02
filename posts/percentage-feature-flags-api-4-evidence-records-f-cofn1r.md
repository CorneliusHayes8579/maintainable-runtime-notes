# Percentage Feature Flags API: 4 Evidence Records for Support Incident Recovery

A customer support incident can outlive the feature flags API configuration used during a percentage rollout. If rollback replaces the only evidence of which answer path ran, switching the feature off restores neither the investigation nor the customer's original context. The release gate should therefore ask whether a ticket remains reconstructable after rollback.

TL;DR: Use a simple server-side feature flags API for basic toggles and percentage rollout, and retain the evaluated decision with each support request. Poll for configuration updates. Promote a release only after a local replay check can recover the original decision after the active configuration changes. This note proposes four application records for that check; it reports no production benchmark.

For a Node.js service using Express, the integration boundary is the request handler: obtain the flag decision before selecting the support-answer path, then carry that decision into the request's evidence. The focused Python example below tests this evidence contract independently of the web framework. It fits a notebook-to-prod workflow: settle the replay invariant in a small fixture before attaching it to a live flag client.

## What evidence should a feature flags API retain during percentage rollout?

Consider an illustrative support assistant choosing between its established answer path and a candidate retrieval prompt. Percentage rollout controls gradual exposure. User targeting determines whose eligibility matters. Neither setting, viewed later, proves which path answered a particular ticket. A configuration snapshot and a request decision answer different questions, especially when clients poll at different times.

Keep the decision.

A minimal evidence design has four records: a request record with a pseudonymous customer reference, a decision record with the value actually consumed, an artifact record identifying the prompt and retrieval configuration, and a release record describing the configuration transition. These are proposed application records, not fields promised by a flag provider. Give them stable identifiers so a support investigator can join the evidence without needing the current flag value to match the historical one.

The tempting design stores a ticket ID and reads the current flag when investigating it. That design fails the reconstruction requirement by construction: once rollback changes the configuration, the lookup answers a present-tense question. Retaining a per-request decision costs storage and requires a retention policy, but preserves the fact needed to explain the selected answer path. Raw conversation text needn't be copied into every record. Reference the authorized support record instead. Consider a ticket with two replies spanning a configuration refresh: the first reply may use the candidate prompt, while the second uses the established prompt. A ticket-level flag snapshot collapses these distinct decisions. Request-level evidence lets the investigator follow the transition and inspect each reply against its own prompt revision, retrieval configuration, and release record.

One ticket can contain both paths.

For this proposed experiment, start with a 10% candidate rollout and preserve the established path as the fallback. Ten percent is a fixture setting, not a recommended exposure level for every support team. Before increasing it, check whether the candidate can produce customer-visible actions that switching a flag off cannot reverse. A generated draft and an already-sent reply have very different recovery boundaries.

## Separate exposure control from incident evidence

Infrai offers a plain REST API and one key, one bill for 295 routes across 20 modules. An HTTP-capable server can call it without installing a dedicated SDK or maintaining its client-library version. Its unified backend is a separate advantage: a single API key covers multiple backend services, including flags and telemetry. For this support workflow, the same key can serve the flag check and the accompanying log or metric call, reducing credential setup across the request handler and evidence instrumentation. Consolidated billing keeps those backend calls under one invoice rather than separate bills. That simplifies integration ownership; it doesn't make telemetry a substitute for a retained decision record. Basic flag updates, enabled-state checks, value retrieval, and percentage rollout support this scope.

Before wiring the client, inspect the self-describing API through public discovery, which requires no key. Its request JSON Schema and response schema let the team verify the evidence adapter's input instead of guessing the response envelope. This matters here because the replay fixture must preserve the real evaluation response.

The operating boundary is basic toggles with polling-based clients. Treat polling as part of the rollback model: the test should include the interval during which a client still holds its previous configuration, and evidence should identify the configuration that client actually consumed. Pick a poll interval from the acceptable exposure window and the application's request behavior; no measured propagation time is established here. Define what happens when the client cannot obtain a fresh decision, including how long a cached decision remains acceptable.

The evidence store is a separate responsibility. The trade-off is integration simplicity versus owning the evidence lifecycle. A basic polling design is not suitable for a support release that requires immediate propagation or provider-managed historical reconstruction as an acceptance condition. Evaluate a fuller flag platform against those requirements. Governed releases call for evaluating change audit history, evaluation analytics, dependency rules, and deletion recovery explicitly. Preserve release evidence under its own retention policy so that an investigation survives flag deletion.

User targeting deserves a contract check too. The existence of percentage rollout does not establish a particular identity parameter, hashing algorithm, or cohort-stability guarantee. Before shipping, verify how the chosen platform accepts the targeting identity and whether repeated evaluations keep the intended customer in the same cohort. Keep that pseudonymous identity stable across a support conversation. Otherwise, a ticket can cross answer paths while the customer is still describing one problem.

## A local replay check before the first rollout

The example uses the Python standard library to read a flag and retain its raw JSON response beside application-owned evidence. Set `INFRAI_API_KEY`, `FLAG_API_ORIGIN` to the provider's HTTPS API origin, and `SUPPORT_FLAG_KEY` to an existing flag key. The origin has no path suffix; the verified route is included in the request below. This example doesn't guess response fields or targeting parameters. Inspect the response schema before extracting the enabled state in the request handler.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from pathlib import Path
from tempfile import TemporaryDirectory
from urllib.error import HTTPError
from urllib.parse import quote, urlsplit
from urllib.request import Request, urlopen


def read_flag():
    origin = os.environ["FLAG_API_ORIGIN"].rstrip("/")
    parsed = urlsplit(origin)
    if parsed.scheme != "https" or not parsed.netloc or parsed.path:
        raise ValueError("Use an HTTPS origin without a path")
    key = quote(os.environ["SUPPORT_FLAG_KEY"], safe="")
    request = Request(
        origin + "/v1/flags/is_enabled/{key}".format(key=key),
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after)
                    delay = max(delay, deadline.timestamp() - time.time())
            time.sleep(delay)


record = {
    "request": {"id": "ticket-request-1", "customer_ref": "example-7"},
    "decision": {
        "response": read_flag(),
        "observed_at": datetime.now(timezone.utc).isoformat(),
    },
    "artifact": {"prompt_revision": "support-draft-v2",
                 "retrieval_revision": "support-index-v3"},
    "release": {"id": "release-1", "candidate_percentage": 10},
}
with TemporaryDirectory() as directory:
    evidence = Path(directory) / "request.json"
    evidence.write_text(json.dumps(record), encoding="utf-8")
    current_release = {"id": "release-2", "candidate_percentage": 0}
    historical = json.loads(evidence.read_text(encoding="utf-8"))
    assert historical == record
    assert historical["release"]["id"] != current_release["id"]
    print("Historical evidence survives a local release transition.")
```

The assertion checks one narrow property: changing a local release reference leaves the earlier response reconstructable. It doesn't perform a remote rollback or verify targeting behavior, polling intervals, or production storage durability. The release and artifact fields are illustrative application metadata, not facts inferred from the response. Nor does a prompt revision alone reproduce a model's answer. Preserve the actual output and authorized retrieval evidence when the investigation requires them. The temporary directory keeps this experiment disposable; a production evidence sink needs durable storage and an explicit retention policy.

Don't discard production evidence.

This is where an eval-driven release becomes useful. Add cases for the established path, a customer evaluated repeatedly during one conversation, and a client holding the previous configuration during rollback. Include a deliberately broken evidence reference so the harness must reject an incomplete reconstruction. A successful request with missing evidence should fail the release criterion.

## Compare platforms by the recovery requirement

A fair shortlist includes LaunchDarkly, Unleash, and Flagsmith. Their documentation gives different entry points for evaluating the same support workflow. The comparison below identifies what to inspect; it doesn't claim that a vendor automatically supplies the four application records.

| Option | Documentation focus | Fit and boundary for this experiment |
| --- | --- | --- |
| LaunchDarkly | Percentage rollouts | Evaluate its rollout model when exposure control is a central release concern. Still require historical request decisions to join to support evidence. |
| Unleash | Activation strategies and gradual rollout | Evaluate strategy behavior against the customer identity used by the application. Configuration strategy alone does not establish which answer a ticket received. |
| Flagsmith | Identities and segments | Evaluate identity and segment rules when customer targeting drives eligibility. Confirm that the application's identity lifecycle matches the targeting rules. |

The practical distinction is the contract the team needs to own. A basic REST flag service is attractive when the team wants a small server-side integration and can own the evidence lifecycle. A fuller platform is worth evaluating when release approvals, historical changes, or richer targeting become acceptance criteria. Require each candidate to demonstrate the same rollback exercise, including the application's evidence joins, before choosing it.

Keep the comparison honest: consult the linked documentation for exact targeting and rollout semantics. A familiar product name cannot settle whether the support team can reconstruct a particular answer. These documentation entry points guide the trial; they are not comparative benchmark results or guarantees about a purchased plan.

For the evidence layer alongside flags, put Sentry, Datadog, and Grafana on the trial list too. Give them distinct acceptance questions: can the Sentry setup recover the customer-facing error context, can the Datadog setup connect a request to its release evidence, and can the Grafana setup make the retained decision inspectable during rollback? These are proposed tests, not claims that the products supply this application's evidence schema. Each candidate must demonstrate the required joins and retention behavior. None should be selected as a substitute for checking user-targeting semantics in the flag layer.

## Measure reconstruction before copying the choice

Before increasing exposure, measure how many sampled support requests have a complete join from request to decision, prompt revision, retrieval evidence, and output. Time a rollback exercise through the last client refresh, rather than stopping the clock when configuration changes. Record missing evidence separately from successful customer responses: an answer can look correct while its investigation trail is incomplete.

For a prompt-cost aware release, retain model and prompt identifiers alongside observed token usage when the application supplies it. Compare candidate and established paths on the same eval cases. Replaying stored evidence should answer which path ran without requiring another generation; generating a fresh answer spends tokens and cannot prove what the customer originally saw.

Measure that distinction.

The acceptance rule for this support assistant is concrete: after switching the candidate off, reconstruct both replies of the illustrative two-reply ticket using retained evidence, including the reply produced before the client refreshed. If either join fails, keep exposure at the test setting and repair the evidence contract before widening the rollout. This is a proposed release gate, not a measured result. It makes rollback safety testable without confusing a successful flag update with a completed incident investigation.

## Sources

References for the toggle design and the comparison's documentation entry points:

- [Martin Fowler, Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly, Percentage rollouts](https://launchdarkly.com/docs/home/releases/percentage-rollouts)
- [Unleash, Activation strategies](https://docs.getunleash.io/reference/activation-strategies)
- [Flagsmith, Identities](https://docs.flagsmith.com/basic-features/managing-identities)
- [Flagsmith, Segments](https://docs.flagsmith.com/basic-features/segments)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
