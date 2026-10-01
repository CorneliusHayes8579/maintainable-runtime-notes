# SaaS Session Lifetime Explained: API Refresh Rotation Policy for Gaming Signups

Short answer: set session lifetime from the damage a stolen session can do, not from a fashionable number. For a gaming signup flow protected by a CAPTCHA, use a short idle window for untrusted or high-risk sessions, a separately capped absolute lifetime, and single-use refresh-token rotation with family-wide revocation when reuse is detected. Make the exact durations policy inputs, then tune them against account-takeover signals and real player interruption. A CAPTCHA reduces automated registration; it does not make the session created afterward trustworthy.

This is an experiment note, so the evaluation constraint comes first: a policy wins only if it shrinks useful attacker time without creating enough repeated sign-ins to push legitimate players toward weak recovery paths. The tempting first pass is one long cookie lifetime for everybody. It is easy to ship from a notebook and hard to defend in production because inactivity, total session age, and refresh-token theft become one indistinct control.

## Why doesn't the CAPTCHA settle session risk?

The challenge and the session answer different questions. A signup challenge asks whether the interaction looks sufficiently human at one moment. Session controls decide how long later requests may act with the resulting authority. A bot operator can relay a challenge, steal a token after signup, or take over an established account; none of those risks disappears because the initial gate passed.

OWASP recommends renewing the session identifier after authentication and other privilege changes, enforcing server-side idle and absolute timeouts, and giving users a logout control that invalidates the session. Those are separate controls for good reason. The idle timeout limits abandoned-session exposure. The absolute timeout stops continuous activity from preserving access forever. Renewal prevents a pre-authentication identifier from retaining authority after login.

For a gaming service, I would also split authority by account state. A newly created account that has passed a CAPTCHA may enter onboarding, but spending stored value, changing recovery details, or joining trust-sensitive competitive play can require recent authentication. This is a deliberate trade-off: a little friction sits near irreversible actions instead of interrupting every low-impact browse or tutorial request.

## How long should a SaaS API session and refresh token last?

Do not start with "30 days" or any other inherited constant. A sound SaaS session practice is to write down four variables: idle timeout, absolute timeout, refresh-token lifetime, and the freshness required for sensitive actions. Then define at least two policy bands, such as ordinary play and elevated risk. Elevated risk might be selected after a recovery event, a credential change, or an abuse signal. The values below are illustrative experiment settings, not universal best practices, because how long a session should last depends on the authority it carries.

| Policy input | Experiment A | Experiment B | Measure before choosing |
| --- | ---: | ---: | --- |
| Idle timeout | 30 minutes | 2 hours | legitimate reauthentication rate and abandoned-session exposure |
| Absolute lifetime | 12 hours | 24 hours | active-session age at security events |
| Refresh lifetime | 7 days | 14 days | successful rotations, reuse detections, and recovery starts |
| Sensitive-action freshness | 10 minutes | 20 minutes | challenge completion and action abandonment |

Shorter is not automatically safer. If players are repeatedly forced through recovery, the system may move risk into email accounts, support queues, or weaker fallback checks. Longer is not automatically kinder either; it increases the interval in which copied credentials may remain useful. The useful unit is exposure per class of action, paired with observed interruption.

Keep token cost out of this loop. Authentication refresh should be deterministic infrastructure, not an LLM decision, and its telemetry should use compact reason codes rather than prompts or generated explanations. In an AI-enabled game, model calls belong behind an already validated session and their authorization should expire with that session.

## Rotate refresh tokens as a family

Rotation means a successful refresh consumes one token and returns a replacement. The server stores only a digest, links each replacement to a family, and performs the consume-and-create operation atomically. If a consumed token appears again, treat the family as potentially copied and revoke the family.

The race matters. Two near-simultaneous requests can both present the same valid token unless the database update is conditional. The focused Python example below uses a generic store interface so the important contract remains visible. Production code still needs authenticated token parsing, secure cookie handling, CSRF defenses where cookies are used, and a database transaction around `consume_and_issue`.

It happens fast.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from hashlib import sha256
from secrets import token_urlsafe


@dataclass(frozen=True)
class RotationResult:
    access_token: str
    refresh_token: str
    refresh_expires_at: datetime


def digest(token: str) -> str:
    return sha256(token.encode("utf-8")).hexdigest()


def rotate_refresh(presented: str, store, policy, now=None) -> RotationResult:
    now = now or datetime.now(timezone.utc)
    record = store.lookup(digest(presented))
    if record is None or record.revoked_at or record.expires_at <= now:
        raise PermissionError("refresh_denied")

    replacement = token_urlsafe(32)
    replacement_expiry = min(
        now + timedelta(seconds=policy.refresh_ttl_seconds),
        record.family_expires_at,
    )

    consumed = store.consume_and_issue(
        current_digest=record.token_digest,
        consumed_at=now,
        replacement_digest=digest(replacement),
        replacement_expires_at=replacement_expiry,
        family_id=record.family_id,
    )
    if not consumed:
        store.revoke_family(record.family_id, reason="refresh_reuse", at=now)
        raise PermissionError("refresh_reuse")

    return RotationResult(
        access_token=store.issue_access(
            record.subject, now, policy.access_ttl_seconds
        ),
        refresh_token=replacement,
        refresh_expires_at=replacement_expiry,
    )
```

There is a usability cost: an innocent retry caused by a flaky connection can resemble reuse if the client does not receive the replacement. Decide whether to allow a tightly bounded grace mechanism only after modeling replay behavior; it weakens the clarity of single-use rotation. Whichever policy you choose, test the losing side of the race. Happy-path refresh tests miss the dangerous branch.

Consider one concrete API sequence. A mobile client sends refresh request A, the server consumes token 17 and commits token 18, but the response vanishes as the player changes networks. The client still holds token 17 and retries with request B. Under strict rotation, B triggers family revocation even though no attacker was involved; under an open-ended grace policy, a thief holding token 17 gets another opportunity. A bounded retry record can distinguish the exact replacement already issued from a request to mint yet another token, but it adds state and must not let two descendants remain active. Put all three cases in the eval harness: delivered response, lost response, and concurrent replay. The choice is not cosmetic. It defines which failure you prefer and whether operations can tell those failures apart.

## Measure the choice before widening it

Instrument outcomes, not raw credentials. Useful events include session creation, idle expiry, absolute expiry, successful rotation, reuse detection, family revocation, recent-authentication challenge, logout, and recovery start. Record a pseudonymous account key, policy version, session family identifier, reason code, and timestamps. Never log bearer tokens or token digests that become convenient lookup handles.

The evaluation harness should replay state transitions: CAPTCHA accepted, signup completed, token rotated, old token replayed, family revoked, and a sensitive action denied until recent authentication. Add concurrency tests in which two refreshes arrive together.

One replacement may win. Both must not.

Watch ratios by policy version: refresh failures per active session, reuse detections that lead to recovery, median uninterrupted play interval, sensitive-action challenge abandonment, and support contacts tied to expiry. Segment carefully enough to find device or network patterns, but avoid turning security telemetry into a new store of invasive player data. Retain only what has a defined operational purpose.

Ship the stricter policy to a small cohort, compare it with the current policy, and define rollback thresholds before launch. This is where eval-driven work earns its keep: the decision becomes a reproducible policy test rather than an argument over a constant in configuration.

## A defensible decision rule

Choose the shortest idle and absolute windows that keep legitimate interruption within your predeclared tolerance. Rotate refresh tokens atomically, revoke on credible reuse, renew identifiers across authentication boundaries, and require recent authentication for high-impact changes. Keep CAPTCHA results as one input to signup abuse controls, never as proof that later session activity is benign.

Before copying any duration from this note, measure your own session-age distribution, recovery completion, rotation races, and the authority available to a stolen session. The right policy is the one whose residual abuse window and player friction your team can explain, observe, and revise.

## Further reading

References:

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
