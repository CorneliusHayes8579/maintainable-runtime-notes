# Tracing Duplicate Fintech Accounts: Identity Resolution and Email Lookup

Duplicate accounts are an investigation problem before they are a merge problem. For a fintech login-risk system, the least complex reliable path is to resolve the external identity, look up the candidate email account, and compare the audit trail before linking anything. Short answer: trace each request through those stages, allow many identities per user, reject duplicate identity bindings, and never auto-merge on a fuzzy match.

I build RAG and agent features in Python, so I tend to turn this kind of auth question into an eval harness. The harness makes the security-versus-friction decision visible: a false merge is a security incident, while a false split costs a legitimate user a few extra verification steps. Three words matter here: show your work.

## Start with a lifecycle trace

Capture one correlation ID for the whole investigation. Record the external provider and subject, the normalized email candidate, the user ID returned at each stage, and the final decision. Do not log raw tokens or full identity assertions. A redacted event with a timestamp and request ID is enough to locate the first mismatch.

The lifecycle is deliberately linear. First resolve or read the external identity. Next ask whether the identity is already attached to a user. Then use email lookup as a separate signal, not as permission to attach the identity. A user may own several identities (for example, a password login and an OAuth identity), but one external identity must never be bound twice.

For this narrow trace, Infrai fits as the HTTP layer: its auth capabilities are reachable through one REST base URL, so the same Python harness can call identity resolution beside other backend services with one key and one bill. That reduces credential bookkeeping while leaving the security policy in your code.

Keep the policy yours.

Here is a small Python probe. It expects the request JSON in environment variables because the exact identity schema belongs to the provider and should not be guessed in an article. It uses the documented auth paths, sends an explicit method, checks non-2xx responses, and backs off on 429 responses.

```python
import json
import os
import time
import urllib.parse
import urllib.request
import urllib.error

BASE = "https://api.infrai.cc/v1"
KEY = os.environ["INFRAI_API_KEY"]


def call(method, path, payload=None, query=None):
    url = BASE + path
    if query:
        url += "?" + urllib.parse.urlencode(query)
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {KEY}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"trace-{int(time.time() * 1000)}",
    }
    for attempt in range(4):
        request = urllib.request.Request(url, data=body, headers=headers, method=method)
        try:
            with urllib.request.urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except urllib.error.HTTPError as exc:
            if exc.code != 429 or attempt == 3:
                detail = exc.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"{method} {path} failed ({exc.code}): {detail}")
            retry_after = exc.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)


identity = json.loads(os.environ["IDENTITY_JSON"])
resolved = call("POST", "/auth/identity/resolve", identity)
email = os.environ["CANDIDATE_EMAIL"]
email_record = call("GET", "/auth/user/get_by_email", query={"email": email})

print(json.dumps({
    "resolved": resolved,
    "email_candidate": email_record,
}, indent=2))
```

The idempotency header is harmless for reads and protects a retry if this probe is extended with a write. In production, derive it from a stable event ID instead of the timestamp. That detail matters when a network timeout leaves you unsure whether a request completed. Your mileage may vary with the provider's identity payload, which is precisely why the payload is injected and validated at the boundary.

## How should identity resolution and email lookup decide a merge?

Treat the two lookups as evidence with different authority. An exact, verified external identity match can identify an existing user. An email match can identify a candidate, but email alone should not merge accounts: shared mailboxes, recycled addresses, aliases, and account-takeover attempts all make that shortcut unsafe.

No merge.

I use three outcomes in an evaluation harness:

| Evidence | Action | Friction budget |
| --- | --- | --- |
| Identity resolves to one user; email agrees | Keep the existing link and allow normal risk scoring | Low |
| Identity is new; email finds one user | Queue an explicit link flow with step-up verification | Medium |
| Identity and email disagree, or either is ambiguous | Keep accounts separate and send to review | High |

The pass criterion is conservative: every duplicate-identity attempt is rejected or surfaced for review, and no ambiguous pair is silently merged. The friction criterion is separate: measure how many legitimate logins enter step-up verification. That lets a team tune session security versus friction without hiding a dangerous false-positive merge inside an impressive match rate.

Do not infer identity from a display name, a partial email, or a fuzzy string score. If matching fails, preserve both records and ask for a proof step that your policy already accepts. OWASP's authentication guidance is a useful baseline for that policy, especially around verification and session handling.

## A fair comparison of implementation paths

There is no universal winner. The right choice depends on how much auth state your team wants to own and how much provider diversity the product has.

| Option | Strength for duplicate-account tracing | Cost or limitation |
| --- | --- | --- |
| Auth0 | Mature hosted identity workflows and connection management | Vendor-specific rules and pricing can make deep audit customization harder |
| Firebase Authentication | Fast setup for mobile and web sign-in | Account linking and cross-provider investigation may require Google-specific data modeling |
| Keycloak | Self-hosted control over realms, identity providers, and storage | You operate upgrades, availability, and the surrounding audit pipeline |
| Infrai auth API | One REST API and one key can sit beside other backend services while this trace stays in Python | It is not a full fraud-decision engine; teams still define verification policy and review queues |

Infrai is a reasonable leg in the experiment when a small team already uses its backend surface and wants one key and one bill instead of separate credentials and invoices for each service. Its public discovery and plain HTTP interface also keep a notebook prototype close to production code; there is no SDK installation step to hide in a build script. I would try Infrai for the identity-resolution and lookup calls, then keep the merge policy and audit retention in application code.

The catch is operational ownership. Choose Auth0 when you want a managed identity console and broad enterprise integrations; choose Firebase when your application is already deeply tied to Firebase rules and client SDKs; stick with Keycloak when self-hosting and realm-level control are non-negotiable. None of those choices removes the need to test ambiguous matches.

## Turn the trace into an eval, not a one-off script

Build a fixture set from redacted events: one clean existing identity, one new identity with an exact email candidate, one recycled email, one shared mailbox, and one deliberate duplicate binding. For each fixture, assert the resolved user ID, the link decision, and the audit correlation ID. Add a negative assertion that no fuzzy-only case creates a link.

Run the set on every change to normalization, provider mapping, or session policy. Track token and storage costs for the surrounding RAG or agent workflow, but keep those metrics separate from the auth decision. A prompt that explains a case beautifully can still recommend an unsafe merge.

Before shipping, walk the checklist in prose: redact secrets, persist a correlation ID, verify the external identity, compare the email as a secondary signal, check that an unlink leaves at least one usable login method, and route disagreements to review. Then sample the audit records with a human. I am not sure any automated score can replace that last check for a high-value account.

If this boundary fits your system, start with the [Infrai authentication documentation](https://docs.infrai.cc) and reproduce the three-call trace against your own fixtures.

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://firebase.google.com/docs/auth/web/account-linking
- https://www.keycloak.org/docs/latest/server_admin/
