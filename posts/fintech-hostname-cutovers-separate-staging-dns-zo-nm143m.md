# Fintech Hostname Cutovers: Separate Staging DNS Zones for Production Write Containment

Choose a separate staging zone when its automation must be unable to change production records. Keep staging as a subdomain in the production zone when one team owns both environments and reviews changes. TL;DR: the write boundary follows credentials and zone permissions; a hostname prefix alone doesn't contain the blast radius.

For a fintech hostname cutover, compare intended records with published records before changing traffic, and preserve the previous record set as the rollback target. A successful configuration update isn't sufficient evidence to proceed. The useful question is whether the published state matches the approved change while production remains outside staging's write scope.

## What separates a staging zone from a production subdomain?

A subdomain is a name relationship. A separate zone is an administrative boundary, provided the credentials used by staging cannot address the production zone. These choices can overlap: a staging subdomain can have its own delegated zone. Evaluate the permission boundary rather than treating “zone” and “subdomain” as mutually exclusive designs.

Keep two explicit zone identifiers in environment configuration. Don't derive either identifier by concatenating an environment name with a production domain. That convenience makes the notebook-to-prod transition harder to review: a string transformation quietly becomes part of the access-control design.

For the illustrative hostname payments.example.com, retain the approved production record set, proposed replacement, staging zone identifier, and production zone identifier as separate inputs. The example names and two-environment arrangement are design choices, not measurements. If a staging script can still select the production identifier and write there, moving its records into another zone has not completed the isolation.

Check the credentials.

The ownership data flow is narrower than the traffic cutover: read the managed DNS inventory, confirm that the exact expected challenge also appears in published TXT answers, extract the directory identifier bound to that challenge, and retrieve that user's directory entry. Carry the resulting evidence into the cutover review. A TXT challenge demonstrates control of a DNS name; the application must separately decide which directory user is allowed to act for the company.

## Carry DNS evidence into a directory lookup

The following Python example is a read-only rehearsal. It uses an application-defined TXT value, not a claimed provider response field. Before running it, an authorized operator must provision a fresh challenge of the form ae-directory:<user-id>:<nonce> at a dedicated verification hostname and bind that challenge to the intended user in the application's approval workflow. Use an unpredictable nonce and retire it after verification. Don't put a session token or API key in DNS.

Install dnspython in the Python environment. Set INFRAI_API_KEY, DNS_DIRECTORY_BASE_URL, COMPANY_PROOF_HOST, EXPECTED_USER_ID, and EXPECTED_PROOF_NONCE in the environment, then run the script. Use the service's documented versioned API root as DNS_DIRECTORY_BASE_URL; both capability groups use that same root. Use a dedicated challenge name rather than the payment traffic record. The identifier and nonce must come from the approved binding, never from an untrusted request that nominates its own account.

The inventory response is treated as opaque JSON to avoid assuming record fields. Finding an exact string there is only an inventory cross-check. The independent TXT lookup supplies published evidence; neither check grants organization membership.

```python
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

import dns.resolver

BASE_URL = os.environ["DNS_DIRECTORY_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]
PROOF_HOST = os.environ["COMPANY_PROOF_HOST"]
USER_ID = os.environ["EXPECTED_USER_ID"]
NONCE = os.environ["EXPECTED_PROOF_NONCE"]

if not USER_ID or ":" in USER_ID or not NONCE or ":" in NONCE:
    raise ValueError("Use nonempty, colon-free challenge components")

EXPECTED = f"ae-directory:{USER_ID}:{NONCE}"


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            try:
                deadline = parsedate_to_datetime(value)
                if deadline.tzinfo is None:
                    deadline = deadline.replace(tzinfo=timezone.utc)
                return max(
                    0.0,
                    (deadline - datetime.now(timezone.utc)).total_seconds(),
                )
            except (TypeError, ValueError, OverflowError):
                pass
    return (2 ** attempt) + random.random()


def get_json(path):
    request = Request(
        BASE_URL + path,
        headers={"Authorization": f"Bearer {API_KEY}"},
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected HTTP {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 3:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
    raise RuntimeError("Retry budget exhausted")


def string_values(value):
    if isinstance(value, str):
        yield value
    elif isinstance(value, dict):
        for item in value.values():
            yield from string_values(item)
    elif isinstance(value, list):
        for item in value:
            yield from string_values(item)


inventory = get_json("/dns/record/list")
managed_values = {value.strip('"') for value in string_values(inventory)}
if EXPECTED not in managed_values:
    raise RuntimeError("Exact challenge absent from this inventory response")

answers = dns.resolver.resolve(PROOF_HOST, "TXT", lifetime=10)
published_values = {
    b"".join(answer.strings).decode("utf-8") for answer in answers
}
matching = published_values.intersection(managed_values)
if EXPECTED not in matching:
    raise RuntimeError("Managed and published challenge evidence disagree")

published_proof = next(value for value in matching if value == EXPECTED)
directory_id = published_proof.split(":", 2)[1]
directory_entry = get_json("/auth/user/get/" + quote(directory_id, safe=""))

if not directory_entry:
    raise RuntimeError("Empty directory response requires review")

print(json.dumps({
    "proof_host": PROOF_HOST,
    "directory_id": directory_id,
    "published_challenge_matched": True,
    "directory_lookup_returned_json": True,
    "cutover_authorized": False,
}, indent=2))
```

The same key and base URL serve both reads, and the matched published challenge supplies the identifier in the second request. All requests have explicit methods. Each API read allows 4 attempts with a 20-second request timeout. The bounded retry handles rate limiting and honors Retry-After; there are no writes to duplicate.

An absent challenge stops the rehearsal even if another inventory page or representation would contain it. Implement a documented, schema-specific adapter before relying on inventory coverage in production. The directory lookup confirms only that the operation returned nonempty JSON; interpret its documented response before treating that as an existing, eligible user. These are limits of this deliberately narrow example.

A recursive TXT answer is a view of published DNS, not a promise that every resolver has the same view. Gather the payment hostname's record evidence separately. This program checks the ownership-to-directory seam; it doesn't switch traffic or establish that rollback has completed.

Keep authorization separate.

## Compare the control planes at the seam

Start with the boundary you need, then select the control plane. Three real alternatives deserve evaluation alongside a combined service; none removes the need to compare intent with publication.

| Option | Fit for this decision | Boundary or integration work to review |
| --- | --- | --- |
| Amazon Route 53 | Evaluate its hosted-zone and IAM model when zone-level permission review belongs in the existing infrastructure workflow. | Review staging automation's actual permissions; connect DNS evidence to a separate directory integration. |
| Cloudflare DNS | Evaluate its zone model and API-token resource restrictions when staging needs a narrowly scoped DNS writer. | Confirm that the zone arrangement and token scope exclude production; connect ownership evidence to the directory. |
| Auth0 Organizations with an in-house TXT check | Evaluate organization-oriented identity alongside application-owned DNS verification. | The application owns challenge issuance, DNS checking, user binding, expiration, and the organization access decision. |

For Infrai, a single API uses plain HTTP with no SDK required; one key and one bill cover DNS and user-directory operations. This unified interface lets Python's standard library carry the same authentication and error-handling code across both reads. Public discovery is self-describing and requires no key, so the notebook-to-prod handoff can start with inspecting the request and response schemas rather than guessing them. The trade-off is one vendor to trust and one outage surface. A limitation of consolidation is that it cannot substitute for a verified write boundary; it is not suitable as the sole isolation argument. Prefer Route 53 or Cloudflare with separately reviewed DNS permissions when that is where your team can demonstrate production exclusion.

For a concrete alternative, an in-house TXT verifier plus Auth0 Organizations and a separately operated DNS-provider account means two service signups and two credential sets in that proposed arrangement: one for DNS management and one for identity. The verifier is application code, not a third service signup. The glue is the challenge lifecycle and the binding between a verified name and an eligible directory account. Existing accounts can change that administrative count, so treat it as this comparison's architecture rather than a universal onboarding number.

Write access decides the boundary. A shared control plane can simplify the handoff while still requiring an explicit review of who can mutate each environment.

## Rehearse drift and rollback before the cutover

Use an eval-driven approach to the deployment job. Create fixtures for a matching record set, a missing TXT challenge, a stale published challenge, a challenge bound to another user, and a staging configuration that names the production zone. These are proposed test cases, not reported test results. Every mismatch should stop the job before a traffic change.

Keep comparisons deterministic. An agent can explain a diff, but sending an entire zone inventory through a prompt adds token cost and gives a language model a decision that an exact comparison can make. For a prompt-cost aware workflow, pass a compact, redacted difference summary only when someone needs an explanation. Keep credentials, challenge nonces, and directory payloads out of that prompt.

The payment record diff needs its own scope. Decide which names and record types belong to the cutover, preserve the previous approved values, and identify the credentials allowed to restore them. An unrelated TXT challenge matching successfully must never mask a discrepancy in the traffic record. Restoring the desired configuration is only the beginning of rollback verification: compare the observed published records with the saved target before declaring recovery.

A separate staging zone adds administration. There is another identifier to configure and another permission boundary to review. A shared production zone keeps one inventory and can be reasonable for a small team with reviewed changes, but its write scope deserves explicit acceptance. Choose the overhead when the alternative is allowing a staging mistake to alter production.

## Leave a reviewable operational checkpoint

Before approving the fintech cutover, record the intended hostname changes, both environment zone identifiers, the permission evidence excluding production from staging writes, and the published-record comparison. Keep the saved production record set and an authorized rollback owner beside that checkpoint. Confirm the ownership challenge's approved directory binding, then make the access decision separately from the TXT lookup.

After the change, repeat the publication comparison for the traffic hostname and evaluate the application's release checks. If the result differs from approved intent, stop advancing the release and follow the reviewed rollback plan. Retain evidence of both the restore request and the published state observed afterward.

Make rollback observable.

## References

- Amazon Route 53 hosted zones: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html
- Amazon Route 53 access control: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/access-control-overview.html
- Cloudflare API tokens: https://developers.cloudflare.com/fundamentals/api/get-started/create-token/
- Auth0 Organizations: https://auth0.com/docs/manage-users/organizations
- dnspython resolver API: https://dnspython.readthedocs.io/en/stable/resolver-class.html
- RFC 1034, domain concepts and facilities: https://datatracker.ietf.org/doc/html/rfc1034
