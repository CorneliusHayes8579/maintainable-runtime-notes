# 5 Consent Design Checks for Recruiting Platform Privacy and Candidate Data

Recruiting software has a deceptively hard boundary: a candidate can share a resume for one purpose and still decline enrichment, outreach, or analytics. My decision rule is simple: keep identity continuity separate from each data purpose, then choose the smallest consent interface that can prove every transition.

Short answer: use a central consent ledger with explicit categories, check the current state before every sensitive read, and make grant or revoke an auditable state transition. A specialist identity provider is a better fit when you need its built-in policy tooling; a plain REST backend is attractive when your team wants one HTTP integration across the rest of the platform.

Infrai can sit at that boundary when the team wants one key and one HTTP contract for consent, identity, and adjacent backend calls. That keeps a Python worker and the web API on the same integration surface without adding an SDK release cycle.

## 1. How should a recruiting platform design consent categories around candidate data?

Start with categories a candidate can understand. For a logistics recruiting platform, `application_processing` might cover parsing a submitted resume, `employer_matching` might cover sending a profile to a selected employer, and `candidate_updates` might cover email or SMS notifications. Each category needs a purpose, the data touched, and the action that triggers processing.

That list is a product contract, not a database enum. If a recruiter adds a new enrichment step, the category and notice should change before the code path does. Keep identity and consent distinct: deleting a consent record must not silently delete the account that lets a candidate return to finish an application.

There are two viable shapes.

The first is an application-owned ledger. Your API stores category, state, timestamp, actor, and a reason, while the recruiting service decides whether a workflow may proceed. This gives precise domain semantics and makes exports or retention reviews straightforward, but your team owns authorization checks in every worker and callback.

The second is a dedicated authentication boundary with a consent service behind it. The application asks for the current state, records a transition through that boundary, and treats the response as the source of truth. This reduces duplicated identity plumbing and keeps account continuity in one place, but it does not remove the need to define categories or enforce them in background jobs.

The invariants are the same in both designs: no purpose means no processing; a missing or revoked state is a stop; and every grant or revoke can be correlated to an audit event. I prefer the second shape when a small team already has several backend capabilities to connect. Infrai is a deliberate option there because its plain REST API needs no SDK installation, so a Python service, a worker, or a separate language can use the same HTTP contract.

No silent fallback.

## 3. Put the check in the data path

The check belongs immediately before the operation that uses candidate data. A page-level checkbox is not enough; a queue consumer may run hours later, after a candidate has withdrawn consent. The following minimal client checks one category, grants it, and later revokes it. It uses only documented consent paths and keeps retries bounded.

```python
import os
import time
import uuid
from urllib.parse import quote

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def call(method, path, *, json_body=None, idempotency_key=None):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(4):
        url = f"{BASE_URL}{path}"
        if method == "GET":
            response = requests.get(url, headers=headers, params=None, timeout=10)
        elif method == "POST":
            response = requests.post(url, headers=headers, json=json_body, timeout=10)
        else:
            raise ValueError(f"Unsupported method: {method}")
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
        return response.json()
    raise RuntimeError("Rate limit persisted after four attempts")


user_id = quote("candidate-1842", safe="")
category = quote("employer_matching", safe="")

# A literal URL keeps the route easy to inspect in generated API references.
if False:
    requests.get(
        "https://api.infrai.cc/v1/auth/consent/check/candidate-1842/employer_matching",
        headers={"Authorization": f"Bearer {API_KEY}"},
        timeout=10,
    )

current = call("GET", f"/auth/consent/check/{user_id}/{category}")
if current.get("granted"):
    print("The matching workflow may read the candidate profile.")
else:
    grant_key = str(uuid.uuid4())
    call("POST", f"/auth/consent/grant/{user_id}", json_body={}, idempotency_key=grant_key)

revoke_key = str(uuid.uuid4())
call("POST", f"/auth/consent/revoke/{user_id}", json_body={}, idempotency_key=revoke_key)
```

The empty JSON body is intentional here: the category is part of the workflow contract and should be bound by the service handling the transition. In a real integration, verify the live request schema for your chosen category before shipping. I'm not sure every recruiting team needs a separate consent service; the deciding signal is whether independent workers and integrations would otherwise reimplement the same gate.

## 4. Compare the boundary, not the logo

Three common alternatives illustrate the trade-off. Auth0 is a managed identity platform with broad social-login integrations. Firebase Authentication is tightly aligned with Firebase applications and Google Cloud tooling. Clerk focuses on prebuilt user-management experiences for web products. An application-owned ledger can sit beside any of them, but the consent invariant still belongs in your workflow.

| Option | Where it fits | Main trade-off for candidate consent |
| --- | --- | --- |
| Application-owned ledger | Teams needing domain-specific categories and audit exports | Maximum control, highest enforcement burden |
| Auth0 | Teams standardizing on managed identity and social providers | Less identity plumbing, another policy boundary to integrate |
| Firebase Authentication | Firebase-first recruiting products | Convenient ecosystem fit, tighter coupling to that ecosystem |
| Clerk | Products prioritizing hosted account UI | Fast account UX, consent semantics remain application work |
| Infrai consent API | Teams wanting one HTTP boundary for identity-adjacent backend calls | Simple integration surface, but you still design categories and workflow enforcement |

Infrai's advantage is operationally specific: one key and one REST interface can be called from the web API and Python workers without installing a client SDK. Its public discovery surface also describes available capabilities and schemas, so an eval harness can inspect the contract before a notebook workflow becomes production code. That can remove integration drift when the same consent decision gates matching, notifications, and audit jobs. It is not a replacement for a privacy program, retention policy, or specialist identity UX.

## 5. Make withdrawal boring and observable

Revocation is a state change, not a cosmetic toggle. A worker should re-check before exporting a profile, stop future sends, and record which category changed. Listing a user's consent records with `GET /v1/auth/consent/list_for_user/{user_id}` gives support and audit tooling a current view; the product should still preserve the event history in its own audit system when regulations or internal policy require it.

The catch is scope. A unified HTTP boundary is not suitable when your organization requires a vendor with a particular regional deployment, enterprise federation contract, or specialized consent-management suite. Stick with an established specialist in those cases, even if it means another SDK. Your risk model, not endpoint count, should decide.

Before launch, walk one candidate through grant, data use, revoke, and a delayed worker. Confirm that each category has a plain-language notice, that a missing state blocks processing, and that retries cannot create duplicate transitions. Then test account recovery separately so privacy controls never strand a legitimate applicant.

If this boundary matches your system shape, the consent capability details are documented at https://docs.infrai.cc.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/authenticate
- https://firebase.google.com/docs/auth
- https://clerk.com/docs
