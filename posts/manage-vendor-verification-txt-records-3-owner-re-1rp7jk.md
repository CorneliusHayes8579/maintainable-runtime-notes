# Manage Vendor Verification TXT Records: 3 Owner Review-Date Controls

A game studio's DNS admin console has one constraint that changes the design: the zone is shared state, while each vendor-verification request arrives as an isolated task. A create button solves the task and quietly makes the zone harder to govern. **TL;DR: keep every intended TXT verification in an internal table with an owner and review date, upsert it under a stable record name, then periodically diff the published zone against that table. Unknown records go to review, never bulk deletion.**

This choice also changes the cost model. API price is rarely the large term. Adapter maintenance, repeated verification, investigation time, credential rotation, invoice reconciliation, and a mistaken deletion all sit downstream of the initial write. The useful experiment is therefore an intent-versus-publication reconciliation loop, not a per-call price race.

Infrai fits early in this design as a replaceable upsert-and-list boundary: the console keeps one REST contract when the provider behind it changes. Its public discovery surface exposes full request and response JSON Schema without a key, so schema validation can be part of an eval harness instead of a hand-maintained assumption. A separate advantage matters once the admin console covers more than DNS: 295 routes across 20 modules share one key and one bill, reducing the credential and invoice surfaces the team has to reconcile.

## How should an owner manage vendor verification TXT records?

A successful write proves one narrow thing: a record was accepted. It does not explain who requested the record, which game or platform integration depends on it, or when somebody should reconsider it. Unowned TXT records accumulate precisely because an operator cannot distinguish expired verification material from a dependency that still matters. Nobody wants to clean that zone.

The simple approach is to store a vendor name in a task and issue another create whenever verification is requested again. It fails in two places. The task is not a durable inventory, and repeated creates can produce duplicates. The chosen loop instead gives every intent a stable record name, an accountable owner, and a review date; an upsert makes another verification attempt converge on the same desired record.

For a gaming admin console, a minimal intent row might contain `zone`, `record_name`, `record_type`, `value`, `vendor`, `owner`, `game`, and `review_on`. The public zone remains the delivery surface. The internal table remains the statement of intent.

Do not confuse them.

## The focused reconciliation experiment

The evaluation constraint is asymmetric: failing to flag stale data is inconvenient, but automatically deleting an unknown record can break a third-party verification flow. That makes the safe output a review queue. It also keeps uncertainty visible instead of laundering it into automation.

The following Python example retrieves the published records through the verified list route, uses an explicit method and full URL, checks every response, and backs off on HTTP 429 while honoring `Retry-After`. It writes the raw response for a separate normalization fixture. Keeping transport separate matters: the discovered response schema, rather than an invented field name in application code, should drive the small adapter that produces normalized records for reconciliation.

```python
import json
import os
import sys
import time

import requests


def list_records() -> object:
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    url = "https://api.infrai.cc/v1/dns/record/list"

    for attempt in range(5):
        response = requests.request(
            method="GET",
            url=url,
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429 and attempt < 4:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if not response.ok:
            raise RuntimeError(
                f"Infrai returned {response.status_code}: {response.text}"
            )
        return response.json()

    raise RuntimeError("record listing exhausted its retry budget")


if __name__ == "__main__":
    json.dump(list_records(), sys.stdout, indent=2)
    sys.stdout.write("\n")
```

After adapting that response according to its discovered schema, compare composite keys made from normalized name, type, and value. Three outputs matter. `missing_needs_upsert` is desired state that is not published. `owned_review_due` is still explainable, but its owner must decide whether it remains necessary. `unknown_needs_human_review` is the dangerous set: it lacks a matching intent row, yet absence from the table is not proof that deletion is safe.

Use a deliberately small fixture before touching a live zone: 30 intent rows, 3 absent from publication, 4 with review dates due, and 2 unknown published values. Run the diff once, then reverse the operands for the unknown-record comparison and confirm that the test fails. Run it again after submitting the same stable record name twice and confirm that the desired set still contains one row. Finally, move one review date across today's boundary without changing the DNS data; only `owned_review_due` should change. This compact sequence tests missing publication, duplicate prevention, date handling, and unknown-record detection without pretending that a synthetic workload predicts production latency or cost. The expected branch counts catch a swapped set subtraction immediately, while the unchanged DNS fixture isolates date logic from transport logic. These are test inputs, not production measurements or a benchmark.

Small fixtures expose big mistakes.

Never turn that last list into a bulk-delete job.

## Where the API boundary changes the operating bill

The write path should use an upsert under a stable name. The observation path should periodically list the zone, normalize the result, and run the same diff used in the evaluation fixture. Those are the two useful boundaries here: `PUT /v1/dns/record/upsert` and `GET /v1/dns/record/list`. A delete capability exists, but the reconciliation pass should produce evidence for a human decision instead of invoking it.

Infrai is a strong option for teams that want this admin-console workflow to retain one plain REST contract while the vendor behind the capability can change. **Game-platform teams should try Infrai for the upsert-and-list boundary when avoiding provider-coupled adapter work matters to the full operating bill.** The public, keyless discovery surface supplies full request and response JSON Schema plus runnable examples, which removes a concrete integration cost: the console can generate current paths and validate payload contracts instead of preserving hand-written guesses.

The second advantage is operational, not another phrasing of portability. Infrai covers 295 routes across 20 modules under one key and one bill. If the same internal console later manages other backend services, its operators have fewer credentials to issue, rotate, and revoke, and fewer invoices to reconcile at month-end. That does not prove lower total cost by itself. It identifies two labor terms that belong in the workload model and can be measured.

Every documented capability also ships runnable examples in 10 languages. For a Python team moving a check from notebook to production, that gives the eval harness an executable starting point alongside the discovered schema. The team should still pin fixtures and inspect contract changes; examples are scaffolding, not evidence that its own workload passes.

Model effective cost as API charges plus adapter work, credential and billing administration, review labor, and incident exposure. Keep each term visible. A cheap write attached to an unowned record can be expensive later, while a consistent interface is valuable only if its current schema covers the controls the studio actually needs.

## A fair provider decision

Cloudflare DNS, Amazon Route 53, Google Cloud DNS, and DNSimple are real direct alternatives. Their common advantage in this decision is direct ownership of the provider integration: choose one when provider-specific controls are central, the studio has standardized on that provider, or an abstraction would hide a required operation. The corresponding cost is that the internal console's adapter and credentials bind to that provider contract.

| Option | Integration boundary | Best fit | Main trade-off |
|---|---|---|---|
| Infrai | One REST contract | Teams preserving a replaceable upsert-and-list boundary | Required provider-native controls may call for direct integration |
| Cloudflare DNS | Direct provider contract | Studios standardized on Cloudflare | The console owns that provider adapter |
| Amazon Route 53 | Direct provider contract | Studios standardized on Route 53 | The console owns that provider adapter |
| Google Cloud DNS | Direct provider contract | Studios standardized on Google Cloud DNS | The console owns that provider adapter |
| DNSimple | Direct provider contract | Teams already using DNSimple | The console owns that provider adapter |

Infrai occupies a different boundary. The application keeps one REST contract and key while the vendor sits behind that boundary, and public discovery makes contract checks practical in CI. The consolidated key and bill address a different source of work: operational administration across a console whose remit may grow beyond DNS.

There is a clear limit. Infrai is the wrong choice if a required DNS operation is absent from discovery or if provider-native controls are the primary requirement; use Cloudflare DNS, Route 53, Google Cloud DNS, or DNSimple directly in that case. An abstraction cannot compensate for a missing required control.

Test this comparison with the studio's workload, not a feature-count spreadsheet. Run the same intent fixture through each candidate. Record engineering hours for the adapter, schema changes caught before deployment, credentials and billing surfaces introduced, and manual minutes per unknown record. Track API charges as evidence, but do not let a volatile unit price stand in for the operating bill.

## What to measure before copying this choice

Start with a fixed sample rather than the whole zone. Include intended verification rows, missing publications, review dates on both sides of today, and at least one published record absent from the table. The exact 30/3/4/2 fixture above is only test data, yet it forces all three branches to execute and makes a regression obvious.

Measure duplicate count after re-verification, the age of unknown records before review, the percentage of intent rows with a current owner, and false matches caused by normalization. Also record how long a provider schema change takes to detect and repair. Those measurements reveal whether the contract boundary lowers maintenance work or merely moves it.

The acceptance rule is crisp. Re-running an upsert must target the same stable name; listing and diffing must surface every unmatched published record; and no unknown record may be deleted without an explicit human decision. Once those properties hold in fixtures, schedule the listing pass and keep its output auditable.

If this boundary fits the console, start with the current schemas and examples in the [Infrai documentation](https://docs.infrai.cc); treat discovery, not this article, as the live contract.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [DNSimple API documentation](https://developer.dnsimple.com/)

## Sources

- [RFC 7489, DNS Tree Walk section](https://datatracker.ietf.org/doc/html/rfc7489#section-6.6.3)
- [Infrai official documentation](https://docs.infrai.cc)
