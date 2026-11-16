# Customer Apex Domains: How to Reconcile Property Portal Records Before Cutover

For customer apex domains, support the property portal with A or AAAA records rather than an ordinary CNAME. Switch only after published records match the approved destination, and retain the previous record set as an explicit rollback target.

A successful write isn't proof of publication.

TL;DR: Use A records for IPv4 addresses and AAAA records for IPv6 at a customer apex. An ordinary CNAME cannot coexist with the SOA and NS records required there. A separate www hostname can use a CNAME when it has no conflicting data. For the cutover, compare complete address record sets against intent, keep the previous endpoint serving, and treat rollback as another publication change that needs observation.

Consider a property management portal whose residents use both harbor-court.example and www.harbor-court.example to submit maintenance requests. The following fixture changes the apex address while preserving the www alias. These are documentation names and addresses, not a real deployment or measured incident.

The data flow is small: an approved plan supplies the before and after record sets; a DNS observation adapter supplies published answers; a deterministic comparison emits a decision for the release process. Keep that comparison outside an agent's prompt. A model can explain the result, but the release gate should evaluate DNS data directly.

## Step 1: Can the customer apex use a CNAME?

An ordinary CNAME says that its owner name is an alias. RFC 1034, section 3.6.2, says other data should not be present at that name. A zone apex already needs SOA and NS data, so using a CNAME there conflicts with that model. A records contain IPv4 addresses; AAAA records contain IPv6 addresses. Neither record type accepts a hostname as its address value.

That distinction makes the first implementation choice straightforward. Obtain the destination's supported address configuration, or investigate an apex aliasing mechanism that publishes appropriate answers. Some DNS services expose synthesized alias features; their configuration labels do not make an ordinary apex CNAME valid. Check the published record types and the service's documented update behavior before selecting that path. This walkthrough uses explicit addresses so the expected answer is concrete.

Start in a notebook. This standard-library example runs without credentials or network access. Its inputs are deliberately limited to A, AAAA, and CNAME records for the two application names; SOA, NS, MX, and TXT records remain outside the change. Save the block as a Python file and run it with your Python interpreter. The output should permit the matching fixture, block the mixed fixture, and recognize the previous destination.

```python
from ipaddress import ip_address

APEX = "harbor-court.example."
WWW = "www.harbor-court.example."


def normalize(rows):
    result = {}
    for owner, kind, values in rows:
        owner = owner.lower().rstrip(".") + "."
        kind = kind.upper()
        if kind not in {"A", "AAAA", "CNAME"}:
            raise ValueError("Unexpected record type in address scope")
        if (owner, kind) in result:
            raise ValueError("Supply one complete RRset per key")
        if not values:
            raise ValueError("Omit absent RRsets; do not supply empty ones")
        if kind == "CNAME":
            if owner == APEX or len(values) != 1:
                raise ValueError("Invalid alias plan")
            normalized = frozenset(
                value.lower().rstrip(".") + "." for value in values
            )
        else:
            addresses = [ip_address(value) for value in values]
            version = 4 if kind == "A" else 6
            if any(address.version != version for address in addresses):
                raise ValueError("Address family does not match record type")
            normalized = frozenset(str(address) for address in addresses)
        result[(owner, kind)] = normalized
    for owner, kind in result:
        if kind == "CNAME" and any(
            other_owner == owner and other_kind != "CNAME"
            for other_owner, other_kind in result
        ):
            raise ValueError("Alias conflicts with address data")
    return result


previous = normalize([
    (APEX, "A", ["192.0.2.10"]),
    (WWW, "CNAME", [APEX]),
])
intended = normalize([
    (APEX, "A", ["192.0.2.20"]),
    (WWW, "CNAME", [APEX]),
])


def decide(observed):
    if observed == intended:
        return "DNS_MATCH: continue application checks"
    if observed == previous:
        return "PREVIOUS_TARGET: keep the cutover pending"
    return "DRIFT: block progression and inspect the difference"


fixtures = {
    "matching": intended,
    "mixed": normalize([
        (APEX, "A", ["192.0.2.10", "192.0.2.20"]),
        (WWW, "CNAME", [APEX]),
    ]),
    "previous": previous,
}
for label, observed in fixtures.items():
    print(label, decide(observed))
```

Notice the mixed fixture.

The new address is present, yet the answer still fails equality because the old address remains. A membership test would miss exactly the drift this release needs to catch.

## Step 2: Promote the comparison from notebook to production

The fixture establishes the decision rule; it does not query DNS or prove readiness.

Replace its observations with a DNS collection adapter that preserves the owner name, record type, full RRset, response status, queried server, and observation time. Query authoritative servers to inspect publication, then selected recursive resolvers to inspect the answers their clients receive. Keep the observations separate. Combining answers from different servers into one set can hide disagreement and manufacture an answer no server actually returned. The trade-off is extra observation work because a single aggregate loses the evidence needed to distinguish disagreement from a genuinely mixed RRset. Record each server's answer before deriving the release decision, and retain the discrepancy if two observation points return different addresses.

Require successful collection before calling the comparison. A timeout, refused query, or failed response must produce an unknown observation, not an empty address set. For a successful response with no AAAA data, omit that RRset in this example's representation; for an unexpected AAAA answer, include it so equality catches the extra destination. If IPv6 is supported intentionally, put its full RRset in both the plan and the observation. Otherwise, residents reaching an old IPv6 destination could be missed by an IPv4-only check.

Keep scope explicit.

The example validates only address and alias data. Before publishing, inspect the full owner data for a proposed www CNAME, because an existing TXT or other conflicting record is outside this validator's view. Preserve unrelated apex records in the change operation. A www alias and an apex address are separate DNS configurations even when the alias ultimately resolves through the apex. An HTTP redirect is a separate application decision again; it requires a working connection to the hostname receiving the request.

For an eval-driven notebook-to-prod workflow, add cases for an unexpected IPv6 address, a missing alias, and a collection failure before connecting a write adapter. This small, independently runnable regression check captures the most consequential comparison mistake without depending on a live zone:

```python
def matches(expected, observed):
    return expected == observed


def test_address_sets():
    expected = frozenset({"192.0.2.20"})
    assert matches(expected, frozenset({"192.0.2.20"}))
    assert not matches(
        expected, frozenset({"192.0.2.10", "192.0.2.20"})
    )
    assert not matches(expected, frozenset())


test_address_sets()
print("Exact-set regression checks passed")
```

Deterministic comparisons also keep this workflow prompt-cost aware. Send a compact mismatch summary to an assistant only when explanation is useful. Store the full DNS evidence in the release record, where an operator can inspect it without spending tokens repeatedly summarizing unchanged answers. This is an architecture choice, not a claim about measured savings.

## Step 3: Make rollback an observed state change

TTL is a cache lifetime, not a global completion timer.

Lowering it immediately before an address change does not rewrite answers already cached with the previous TTL. If the release depends on a shorter cache interval, publish that change ahead of the address switch and account for the previous value. Resolver samples still describe only those observation points; they cannot prove every resident has the same answer.

That limitation drives the rollback design. Keep the previous endpoint available during the transition, record the exact previous RRsets, and define which evidence triggers restoration. Publishing those previous values is the start of rollback. Some clients may continue using a cached new address until their cached data expires, so both destinations need an explicit service plan while answers differ.

A subtle operational trap is overwriting a customer's newer edit with a stale rollback snapshot. Bind the approved change to the record state it was reviewed against. Immediately before a write, reread that state and stop if it differs; when the control interface offers a conditional update, use it to close the gap between reading and writing. Without such a mechanism, a reread alone does not eliminate the race. Escalate conflicting intent instead of guessing which address the customer meant to keep.

DNS agreement is only one release gate. Check certificate coverage, hostname routing, and the maintenance-request flow on the destination before declaring the portal ready. For a portal backed by a RAG assistant or agent, use a synthetic request that confirms the hostname reaches the intended property context; avoid putting resident records into DNS diagnostics or model prompts. Treat that check as an application test, with its own result and retention policy.

Keep mail configuration outside the address cutover.

DMARC policy discovery uses DNS TXT records under the _dmarc name described by RFC 7489; moving the portal's A record is not a mail authentication migration. The approved record diff should show exactly which owner/type pairs change so an address release cannot silently become a zone replacement.

Before the switch, the operator should be able to identify the approved destination, the previous RRsets, the observation sources, and the person responsible for conflicting customer edits. During the switch, retain timestamped answers and application checks together. Close the release only when the agreed DNS observations match intent and the portal checks pass; close rollback only after the restored intent has its own evidence. That makes the property team's decision reviewable even after the notebook is gone.

## References

- RFC 1034, section 3.6.2: CNAME behavior and coexistence constraints. https://datatracker.ietf.org/doc/html/rfc1034#section-3.6.2
- RFC 1035: DNS resource record formats, SOA, NS, A, and TTL. https://datatracker.ietf.org/doc/html/rfc1035
- RFC 3596: AAAA records for IPv6 addresses. https://datatracker.ietf.org/doc/html/rfc3596
- RFC 2308: DNS negative caching and negative response handling. https://datatracker.ietf.org/doc/html/rfc2308
- RFC 7489, section 6.6.3: DMARC policy discovery. https://datatracker.ietf.org/doc/html/rfc7489#section-6.6.3
- RFC 2606: reserved example domain names. https://datatracker.ietf.org/doc/html/rfc2606
- RFC 5737: IPv4 address blocks reserved for documentation. https://datatracker.ietf.org/doc/html/rfc5737
