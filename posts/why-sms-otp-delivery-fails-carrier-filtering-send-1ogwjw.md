# Why SMS OTP delivery fails: carrier filtering, sender registration, and shared routes

An SMS one-time passcode that never arrives is rarely a bug in your login handler. In short, the message dies in a layer you don't own: sender registration you never finished, carrier filtering that scored the traffic as bulk, shared routes carrying somebody else's reputation, or an anti-fraud throttle that read your 2FA burst as fraud. SMS OTP delivery is probabilistic, and US and EU networks fail it in different ways.

Plan for that and the login stops looking flaky.

The system behind this note is unglamorous. A newsroom analytics service renders a nightly PDF for each desk and mails it as an attachment, and the same service guards staff sign-in with a 2FA code over SMS. Two generated messages, two channels, one question that turned out to decide almost everything: who owns the template? For the emailed report, the answer is "we do, completely." For the OTP, the honest answer is "we co-own it with a registry and a carrier," and that single fact explains most of the delivery behaviour below. The bar we set for the login channel was a fixed panel of live test numbers per country, with a terminal delivery status expected inside 30 seconds — not an average delivery rate across a month, which hides exactly the segment that's broken.

## What actually causes SMS OTP delivery failure on some carriers during 2FA login?

Because the path to a handset is four independent systems, and each one filters on different evidence. Your provider accepts the request and hands it to an aggregator. The aggregator picks a route — a direct interconnect with that operator, or a cheaper transit hop shared with thousands of other senders. The operator scores the traffic against its own spam and anti-fraud models. Then the handset has to be reachable at all.

US and EU networks disagree about which of those layers does the gatekeeping. In the US, application-to-person traffic on a 10-digit long code has to be registered: a brand, then a campaign describing the use case with sample message copy, filed through your provider into The Campaign Registry. Unregistered or mis-registered traffic gets filtered, throttled to a trickle, or surcharged. Toll-free numbers run their own verification track, and shared short codes were retired by the US carriers years ago, so a "shared sender" today means a shared pool of long codes whose reputation you inherit from whoever else is on it.

Europe has no single registry, and that's the trap. Sender ID rules are national: several countries require pre-registration of an alphanumeric sender ID, others let it pass unregistered, and a few silently rewrite it. Alphanumeric sender IDs are also one-way, which means the reply-based opt-out flow you built for the US does not exist there, and the phone number your provider substitutes may vary per message.

| What you see | What is actually happening | What changes it |
| --- | --- | --- |
| Accepted, never delivered, no error | Carrier filtering on unregistered or shared routes | Finish sender registration; move OTP to a dedicated sender |
| Delivered in the US, silent in one EU country | Sender ID not permitted or not registered in that country | Per-country sender profile, registered where required |
| Sudden spike in sends to one country code | Artificially inflated traffic against your OTP endpoint | Country allowlist plus per-number and per-country rate caps |
| "Delivered" but the user has nothing | Transit-carrier receipt, not a handset acknowledgement | Treat the receipt as weak evidence; measure code entry instead |

Handset reality adds the last few percent: roaming, dual-SIM phones that keep the second line asleep, MVNOs with their own filters, and users who enabled network-level message blocking after one too many marketing blasts.

## Sender registration, EU sender IDs, and the template you only think you own

Here's the part that surprised the team most, and the reason template ownership became our primary decision axis. US campaign registration includes sample message copy and a declared use case. Once that's filed, your OTP text is effectively co-owned: change the wording, add a promotional line, drop in a link from a public URL shortener, and the traffic can drift outside the approved campaign and straight into filtering. Public shorteners in particular are widely treated as a spam signal, and a link in a login message is a phishing pattern in the first place.

So OTP copy is frozen. It lives in the repository as a versioned constant with a test asserting the exact string, and changing it is a review plus a re-filing, not a hotfix at 23:00.

The emailed report sits at the other end of the ownership spectrum. Nobody registers your HTML with a carrier. You own the template end to end, you can localize it per desk, and you can regenerate it every night — but you also own authentication and shape. Alignment across SPF, DKIM and DMARC, a bounce and complaint feed you actually consume, an attachment under the recipient's size ceiling, and a file type that corporate scanners don't strip. The failure mode moves from "the network filtered your copy" to "your own records and payload are wrong," and the second is far easier to debug because every signal is yours.

That gives a decision rule worth stealing: put frequently changing, experiment-driven copy only on channels whose templates you fully own, and keep registered channels boring, fixed, and single-purpose. If a product manager wants to A/B test the sign-in message, the answer is no — test the screen around it instead.

## From notebook code to the version that shipped

The first cut was the obvious one: one shared long code for everything, the message string built inline in the request handler, a single attempt, and an HTTP 202 from the provider treated as success. It looked fine in a notebook against two colleagues' phones. In production it produced the worst possible support ticket — "the code never came," with nothing in the logs to contradict the user, because the only thing recorded was that the API accepted the request.

The shipped version pins a registered sender per country, freezes the template, polls to a terminal status, and treats a timeout as a routing signal rather than a user error.

```python
import os, time, requests

GATEWAY = os.environ["SMS_GATEWAY_URL"]
AUTH = {"Authorization": f"Bearer {os.environ['SMS_GATEWAY_TOKEN']}"}

# Frozen copy: this exact string is what the campaign was registered with.
# Editing it is a code review plus a re-filing, never a hotfix.
OTP_TEMPLATE = "{code} is your Newsroom sign-in code. It expires in 10 minutes."

TERMINAL = {"delivered", "undelivered", "rejected", "blocked", "expired"}

def send_otp(to_e164: str, code: str, country: str) -> str:
    payload = {
        "to": to_e164,
        "sender_profile": f"otp-{country.lower()}",   # registered sender, one per country
        "template_id": "otp_login_v3",
        "body": OTP_TEMPLATE.format(code=code),
    }
    r = requests.post(f"{GATEWAY}/dispatch", headers=AUTH, json=payload, timeout=10)
    r.raise_for_status()
    return r.json()["message_id"]

def await_terminal(message_id: str, budget_s: int = 30) -> tuple[str, float]:
    started, delay = time.monotonic(), 1.0
    while time.monotonic() - started < budget_s:
        status = requests.get(f"{GATEWAY}/status/{message_id}", headers=AUTH, timeout=10).json()["status"]
        if status in TERMINAL:
            return status, time.monotonic() - started
        time.sleep(delay)
        delay = min(delay * 1.6, 5.0)
    return "timeout", budget_s
```

Everything interesting happens after that function returns. A terminal status other than `delivered`, or a timeout, flips the login screen to a second factor the carrier can't touch: an authenticator code, or a passcode mailed to the address already on file. The re-send button is rate limited with a visible cooldown, capped per number per hour, and backed by a suppression list so a mistyped number doesn't get hammered. Two channels, one decision, and the user is never left staring at an empty inbox tray on their phone.

## Measure it like an eval set, not like an uptime chart

Anyone who has built model evaluation harnesses already has the right instinct here: a single aggregate number tells you nothing about the segment that's broken. Log one row per attempt with country, sender profile, template id, terminal status, and seconds-to-terminal, then slice p95 time-to-terminal and terminal-failure rate by country and sender profile. Global averages stay green while one country code sits at zero — that's the whole failure pattern, and it's invisible unless you segment.

Keep a small golden panel: real SIMs on the operators that matter, one canary send per country per hour, alerting on a segment regression rather than a global one. Track the metric the user experiences too, which is codes successfully entered before expiry, not codes accepted by an API.

Watch the money, since every retry is a billable message and anti-fraud pumping turns your login form into someone else's revenue stream. A country allowlist, per-number caps, and a per-country hourly ceiling cost an afternoon to build and are the cheapest control in this entire stack. I'm not sure cross-country delivery numbers are ever truly comparable — receipt semantics differ per operator, and a transit-level receipt is a weaker claim than a handset acknowledgement — so treat inter-country comparisons as directional and trust the in-country trend line.

## When SMS is the wrong hammer

SMS earns its place on reach and zero enrollment friction. That's the trade-off it buys, and it costs you a delivery path you cannot inspect. NIST's digital identity guidelines classify out-of-band authentication over the public telephone network as a restricted authenticator, which means it comes with a risk assessment and a migration plan attached, not an indefinite blessing.

For a newsroom account that can publish to the front page, that's not a good fit. A time-based code from an authenticator app removes carriers from the loop entirely, and a passkey removes the shared secret as well; stick with those for anyone with publishing or billing rights, and keep SMS as the enrollment ramp and the recovery path for people who lose a device. Email as a fallback carries its own failure modes — spam placement, corporate link rewriting, attachment stripping — so it needs the same status feed and the same segmented measurement, not blind trust.

The general rule that survived all of this: choose the channel by who owns the template, then instrument the layer you don't own until its failures are visible in your own data.

## Further reading

- The Campaign Registry (US A2P brand and campaign registration): https://www.campaignregistry.com/
- US A2P 10DLC compliance documentation: https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
- Amazon SES developer guide (email sending, authentication, bounce and complaint events): https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- NIST SP 800-63B, Digital Identity Guidelines — Authentication and Lifecycle Management: https://pages.nist.gov/800-63-3/sp800-63b.html
- RFC 6238, TOTP: Time-Based One-Time Password Algorithm: https://datatracker.ietf.org/doc/html/rfc6238
- W3C Web Authentication (WebAuthn) Level 2: https://www.w3.org/TR/webauthn-2/
- ITU-T Recommendation E.164, the international public telecommunication numbering plan: https://www.itu.int/rec/T-REC-E.164/en
