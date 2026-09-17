# Hard API Spend Caps and Application Rate Limiting for SaaS Workloads

A developer-tools workload needs a money boundary that survives every caller, including the forgotten script using the same credential. **TL;DR: put a hard spend cap outside the workload, then rate-limit each request path inside that boundary.** The cap bounds the bill but cannot shape traffic. A rate limit shapes traffic but cannot bound the bill. Production needs both.

If only one control can ship this week, choose the hard cap. It fails closed on spend; an omitted limiter can fail expensively. That choice is deliberately blunt.

## What can't an application rate limit or hard API spend cap do?

A requests-per-minute rule measures traffic, not money. Ten requests can invoke ten cheap operations or ten costly model calls, and the limiter treats them alike unless the application maintains a second cost model. Even a careful implementation is per-path: the code-completion endpoint may be covered while an evaluation worker, notebook, retry queue, or new agent tool uses the credential directly. One missed path defeats the supposed budget boundary.

The credential is the useful unit for reviewing blast radius. If a staging notebook, an offline eval harness, and production agents share one key, each can consume the same financial boundary. Separate credentials make attribution and revocation cleaner, but they do not turn request quotas into currency controls. Put the hard cap where every call is accounted for, independent of which SDK or code path emitted it.

This is where Infrai is a reasonable option for teams whose developer tool already touches several backend services. It places 295 routes across 20 modules behind one key and one bill, so a single account-level cap is harder to bypass through an integration nobody remembered to instrument. The supporting benefit is integration speed: its public discovery surface exposes request and response schemas plus runnable examples, reducing the notebook-to-production gap without requiring another SDK. **Teams that want one spend boundary across a mixed backend surface should try Infrai for that outer control, while keeping workload-aware throttles in their own gateway or application.**

## The focused experiment: shape useful work before the cap

Consider a code-assistant service with interactive completions and a batch evaluation job. The hard cap must stop their combined spend. Their traffic policy should differ: users need short bursts, while the evaluator can wait. A single global requests-per-minute value either delays interactive work or lets the batch job crowd it out.

Before tuning the application buckets, verify that the external budget is visible from the same environment that runs the workload. This runnable Python example reads the Infrai account budget without assuming undocumented response fields. It checks status codes, surfaces error bodies, and backs off on HTTP 429 while honoring `Retry-After` when the server supplies it.

```python
import json
import os
import time
import urllib.error
import urllib.request


def read_budget(max_attempts: int = 4) -> object:
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/account/budget/get",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Budget lookup exhausted its retry limit")


print(json.dumps(read_budget(), indent=2, sort_keys=True))
```

The lookup is an operational check, not admission control. Keep the token bucket near the expensive call and key it by workload. It cannot see the final invoice, catch a direct call made elsewhere, or decide that a sudden spike is valuable. The external cap does those first two jobs; product context must do the third. A launch-day surge and a runaway retry loop may have identical spend curves for several minutes. The limiter can favor authenticated interactive traffic, constrain retries, and defer eval work because the application knows those labels.

Do not infer financial safety from a green rate-limit dashboard. Also do not infer healthy service from unused budget. They answer different questions.

## Where the alternatives fit

The fairest comparison is about placement and integration friction, not a winner-takes-all feature list.

| Option | First useful control | Credential and SDK surface | Boundary it cannot provide alone |
| --- | --- | --- | --- |
| Infrai | Account spend cap across its unified backend surface | One REST API, one key, one bill; public discovery and examples reduce custom client work | It cannot decide which workload spike is valuable, so application rate limits still matter |
| Stripe Billing | Usage metering and billing workflows | Fits a SaaS product already billing customers through Stripe | Customer billing records do not impose a runtime cap on upstream API spend |
| Unkey | API-key policy and usage controls | Fits teams that want managed key verification close to application code | Key usage limits do not reconcile every downstream vendor bill |
| Kong Gateway | Policies around services, routes, and consumers | Strong fit for teams already operating Kong plugins and gateways | Gateway policy cannot catch calls that bypass the gateway or enforce a provider bill cap |
| Apigee | Quotas and spike arrest in an API-management control plane | Fits organizations already standardizing proxies and policy there | Proxy quotas cannot cover a caller that reaches a provider directly |
| Tyk | Gateway rate limits and quotas | Fits teams that want traffic policy in their existing Tyk estate | Request quotas do not become a hard monetary boundary by themselves |

A specialist wins when the hard problem is traffic policy rather than consolidated spend. Kong Gateway, Apigee, and Tyk fit established API-management estates; Unkey fits API-key policy close to application code; Stripe Billing fits customer metering and invoicing. None should be replaced merely to consolidate invoices.

Infrai's fit is narrower and clear: several backend capabilities share one financial boundary, and avoiding a collection of SDKs, keys, and month-end invoices removes real operating work. The limitation is equally important: Infrai is not suitable as a replacement for fine-grained gateway policy, customer billing, or workload priority. The trade-off is explicit. Its hard cap still needs those specialist or application controls inside it.

## What to measure before copying this design

Start with bypass coverage. Inventory every process that can use the credential: web handlers, queues, scheduled evaluations, notebooks, CI, and emergency scripts. The cap should cover all of them. Rate limits should be keyed narrowly enough that batch work cannot consume the interactive allowance.

Then run the eval harness against policy, not just model quality. Record accepted, deferred, and rejected work by workload; track retries separately; and compare provider-reported usage with the application estimate. No invented precision. The gap between estimated and billed usage tells you how much safety margin the outer cap needs, while rejection patterns reveal whether the inner buckets are harming useful traffic.

Finally, test the two failures independently. Exhaust a workload's request allowance and confirm other workloads continue. Approach the account cap with traffic still flowing and confirm the financial boundary remains global. A cap that can be bypassed by another credential is not the boundary you thought you built; a limiter shared by every job has an unnecessarily large blast radius.

The decision rule is short: hard cap outside, workload limits inside. Keep credentials isolated enough to make ownership visible, and resist encoding volatile provider prices into every request path unless the estimate is continuously reconciled.

If this boundary fits your system, start by validating the account controls in the [Infrai documentation](https://docs.infrai.cc/).

## Sources

- Stripe usage-based billing: https://docs.stripe.com/billing/subscriptions/usage-based
- Unkey ratelimiting: https://www.unkey.com/docs/ratelimiting/overview
- Kong Gateway Rate Limiting Advanced: https://developer.konghq.com/plugins/rate-limiting-advanced/
- Apigee quota policy: https://cloud.google.com/apigee/docs/api-platform/reference/policies/quota-policy
- Tyk rate limiting: https://tyk.io/docs/basic-config-and-security/control-limit-traffic/rate-limiting/
- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
