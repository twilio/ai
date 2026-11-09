---
name: twilio-migrate-messaging-to-verify
description: >
  Migrate an existing OTP / one-time-passcode flow from the Twilio Programmable
  Messaging API (self-built: generate code → send SMS → validate)
  to the Twilio Verify API (managed code generation, delivery, expiry, rate
  limiting, and Fraud Guard). Covers what Verify replaces, creating a
  Verify Service, replacing send + check logic, E.164 formatting with Lookup,
  bring-your-own custom codes, error-code translation, and
  a safe incremental cutover. Use when a developer is moving OTP/2FA delivery off
  Programmable Messaging onto Verify, or asks "should I use Verify instead of
  sending my own SMS codes?".
---

## Overview

> **Agent safety:** Confirm with the user before switching production OTP traffic to Verify or sending a verification to a real number. Each verification sends a real, billed message. Test credentials do not support Verify, so test with mocks.

If you built OTP on the **Programmable Messaging API**, you are maintaining code
that Twilio Verify manages for you: random code generation, a database of codes
with expiry, resend/rate-limit logic, and fraud filtering. **Verify** replaces
that entire stack with two API calls — send a verification, check a code — and
adds one-click Fraud Guard (SMS pumping protection), managed sender pools, and
automatic A2P/compliance handling.

**What you delete when you migrate:**

| You built (Programmable Messaging) | Verify replaces it with |
|---|---|
| Buy/manage a sending number or Messaging Service | Managed sender pool (short/long code, toll-free, alpha ID) chosen per-country |
| `random.randint(...)` code generation | Built-in generation (4–10 digits, default 6) |
| DB table storing `code`, `phone`, `expires_at` | Stateless — Verify tracks the code and 10-min expiry |
| Custom resend throttling / attempt counting | Built-in rate limits (5 sends & 5 checks per number / 10 min) |
| SMS pumping / fraud filtering | Fraud Guard (geo-permissions, rate anomaly) — one click |
| A2P 10DLC campaign registration for the OTP number | Exempt — Verify traffic needs no 10DLC campaign |
| Per-channel integration (SMS, voice, WhatsApp…) | One API; change the `channel` param |
| Locale-specific message copy | Pre-screened templates in dozens of languages, auto-resolved by country |

> This skill focuses on **the migration**. For full Verify API reference (all
> channels, service config, status values), see `twilio-verify-send-otp`. To
> decide *whether* Verify fits before writing code, see
> `twilio-identity-verification-advisor`.

**When to stay on Programmable Messaging:** you need full control of the message
body/branding beyond what templates allow, custom delivery routing, or you are
sending non-OTP transactional traffic. For standard OTP/2FA, migrate to Verify.

---

## Prerequisites

- An existing Programmable Messaging OTP implementation (the thing you're replacing)
- Twilio account — New to Twilio? See `twilio-account-setup`
- Environment variables:
  - `TWILIO_ACCOUNT_SID`
  - `TWILIO_AUTH_TOKEN`
  - `VERIFY_SERVICE_SID` (created in Step 1 below)
  — See `twilio-iam-auth-setup` for credential setup
- SDK: `pip install twilio` / `npm install twilio` (same SDK you already use)
- Verify needs **no separate product activation** — just create a Service

---

## Migration steps

The self-built pattern you're replacing does three things by hand: **generate** a
random code, **store** it with an expiry, **send** it over a number you own — then
later **look up + compare + check expiry** yourself. Verify collapses all of that
into the two calls below. (The "What you delete" table above maps each piece to its
Verify replacement.)

### The Verify pattern

**Step 1 — Create a Verify Service (one-time)**

Do this once and reuse the SID. Create it in the [Console](https://1console.twilio.com/go?to=/account/__account__/us1/verify/services) or via API:

**Python**
```python
import os
from twilio.rest import Client

client = Client(os.environ["TWILIO_ACCOUNT_SID"], os.environ["TWILIO_AUTH_TOKEN"])

service = client.verify.v2.services.create(friendly_name="Acme")  # your business name — shown in the SMS
print(service.sid)  # VAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx — save in environment variable as VERIFY_SERVICE_SID
```

**Node.js**
```node
const twilio = require("twilio");
const client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);

const service = await client.verify.v2.services.create({ friendlyName: "Acme" }); // your business name — shown in the SMS
console.log(service.sid); // VAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> `friendlyName` appears in the default verification SMS (e.g. "Your Acme verification code is 123456"), so set it to your business name — not an internal label. Avoid embedding 5 or more digits in the name — it can trigger error 60200.

**Step 2 — Send the verification (replaces your existing messaging logic)**

This single `verifications.create` call replaces steps you did by hand:
Verify generates the code, tracks it with a 10-min expiry, and delivers it over a
managed sender — so whatever you used to produce the code, persist it, and send
the SMS can all go. A `status` of `pending` means the code is on its way.

**Python**
```python
verification = client.verify.v2 \
    .services(os.environ["VERIFY_SERVICE_SID"]) \
    .verifications \
    .create(to="+15558675310", channel="sms")

print(verification.status)  # pending
```

**Node.js**
```node
const verification = await client.verify.v2
    .services(process.env.VERIFY_SERVICE_SID)
    .verifications.create({ to: "+15558675310", channel: "sms" });

console.log(verification.status); // pending
```

No code generation, no DB write, no owned number. Switch channel by changing
`channel` to `"call"`, `"whatsapp"`, or `"email"` (also `"sna"` and `"auto"`).
RCS isn't a `channel` value — Verify upgrades an `sms` verification to RCS where
the device supports it and falls back to SMS automatically.

**Step 3 — Check the code (replaces your existing validation logic)**

**Python**
```python
check = client.verify.v2 \
    .services(os.environ["VERIFY_SERVICE_SID"]) \
    .verification_checks \
    .create(to="+15558675310", code="123456")

if check.status == "approved":
    grant_access()
else:
    reject()  # "pending" = wrong/not-yet-submitted, "expired" = past TTL
```

**Node.js**
```node
const check = await client.verify.v2
    .services(process.env.VERIFY_SERVICE_SID)
    .verificationChecks.create({ to: "+15558675310", code: "123456" });

if (check.status === "approved") grantAccess();
else reject();
```

> A wrong code does **not** throw — it returns `status: "pending"`. You must
> test `status === "approved"` explicitly. After approval the verification is
> single-use; re-checking it returns 404. Delete your OTP database table once
> traffic is fully on Verify.

---

## Key patterns

### Format numbers as E.164 (Verify is strict)

Programmable Messaging tolerated loose formats; **Verify requires E.164**
(`+15558675310`). If your DB stores national-format numbers, normalize them
before sending or you'll get error 60200. Use the **Lookup API** to convert user
input to E.164 and capture the ISO country code — useful for fraud allow/block
lists. See `twilio-lookup-phone-intelligence`.

### Keep your own codes (bring-your-own OTP)

If you can't rip out your existing generation logic on day one, pass your code
via `custom_code` so Verify still handles delivery, templates, and localization.

**Python**
```python
verification = client.verify.v2 \
    .services(os.environ["VERIFY_SERVICE_SID"]) \
    .verifications \
    .create(to="+15558675310", channel="sms", custom_code="123456")
```

Caveats — read before relying on this:
- **Enable it per-Service first** — it is not on by default. Set `CustomCodeEnabled=true` when creating/updating the Service via the API (`services.update(custom_code_enabled=True)`), or toggle "Enable Custom Verification Code" in the Console (**Verify → Services → your Service → General**).
- **Do not pass a custom message body.** Send only the code via `custom_code`; Verify maps it into an approved template. Custom message text returns [error 60243](https://www.twilio.com/docs/api/errors/60243).
- You are billed per **attempted** send (not per completed verification), and custom-code flows tend to generate more sends than checks (abandoned/re-requested codes) — expect higher cost than standard Verify.
- **Report the outcome so Twilio can keep deliverability high.** Verify's route-intelligence engine uses whether a code gets validated as a signal — it automatically retries different carriers/routes when codes go unvalidated or delivery underperforms. Tell Verify the result by updating the verification's status to `approved` so Twilio can keep optimizing routing:

**Python**
```python
client.verify.v2 \
    .services(os.environ["VERIFY_SERVICE_SID"]) \
    .verifications(verification.sid) \
    .update(status="approved")  # after your own code check passes
```

Treat custom codes as a transition aid, not the end state — migrating to Verify-generated codes removes all these caveats.

### Translate your error handling

Verify uses different error codes than Messaging. Map them during migration:

| Situation | Verify error | Handle by |
|---|---|---|
| Malformed / non-E.164 number, bad param | 60200 | Fix `to` format (use Lookup) |
| Too many sends to a number | 60203 | Back off; surface "try again later" |
| Too many check attempts | 60202 | Issue a new verification |
| Service SID wrong | 60212 | Check `VERIFY_SERVICE_SID` |
| Country not enabled | 60410 | Enable geo-permission in Console |
| Custom message passed | 60243 | Use `custom_code` only, not a body |

Full Verify error handling and built-in protections: `twilio-verify-send-otp`.

### Rate limits & fraud — delete your custom throttling

Verify enforces, per phone per service, out of the box: **5 sends / 10 min** and
**5 check attempts** per verification. Codes live **10 minutes** and are
immutable within that window (a re-request inside 10 min returns the *same*
code). This replaces hand-rolled resend counters. Need different limits? Define
custom [Rate Limits / Buckets](https://www.twilio.com/docs/verify/api/service-rate-limits)
on the Service. Enable **Fraud Guard** in the Console to retire your SMS-pumping
defenses. See `twilio-security-hardening` for the pumping-attack background.

### Safe incremental cutover

Don't flip 100% of traffic at once:

1. **Stand up Verify in parallel.** Keep the Messaging path live; route a small % of new OTP requests (or one region / internal users) through Verify.
2. **Dual-run the check.** If a user was issued a legacy code, validate against your DB; if issued via Verify, use `verification_checks`. Track which system issued each code so checks hit the right validator.
3. **Drain, then delete.** Once no legacy codes remain within their 10-min TTL, stop issuing them, ramp Verify to 100%, then decommission the OTP number/Messaging Service and drop the OTP DB table.
4. **Reuse one Verify Service** across the app — do not create a Service per request.

For testing without tripping rate limits during the cutover, see Twilio's
[Verify testing docs](https://www.twilio.com/docs/verify/quickstarts/test).

---

## CANNOT

- **Cannot retrieve the code Verify generated** — never returned by any API (by design for security). If your old flow logged or displayed the code, that stops working.
- **Cannot reuse your owned number as the Verify sender by default** — Verify picks from managed sender pools per country. (You *can* bring your own number/Messaging Service for specific channels/WhatsApp, but that's not the default.)
- **Cannot pass a raw custom message body** — the `customMessage` param is deprecated; raw text → error 60243. To customize the copy, use an approved template (`TemplateSid`) instead.
- **`custom_code` has strings attached** — must be enabled per-Service (Console → Verify → Services → General), bills per attempt (higher cost than standard Verify), and mandates updating Verification Status manually.
- **Cannot change the 10-minute token TTL yourself** — to change the default (adjustable between 2 minutes and 24 hours), you must contact Twilio Support to set it on your Service.
- **Cannot re-check an approved verification** — single-use; a second check returns 404.
- **Cannot send to non-E.164 numbers** — normalize first (Lookup).
- **No push webhook on completion** — poll `verification_checks` (rate-limited: 60/min, 180/hr, 250/day) or use the Verify Events API.
- **Cannot skip WhatsApp sender setup** — as of March 1, 2024 there is no shared Verify WhatsApp sender; bring your own. See `twilio-whatsapp-manage-senders`.
- **Verify traffic is A2P-exempt, but your remaining non-OTP Messaging is not** — keep 10DLC registration for any other traffic on that number.

---

## Next Steps

- **Full Verify API reference (all channels, service config):** `twilio-verify-send-otp`
- **Decide product fit / architecture first:** `twilio-identity-verification-advisor`
- **Normalize numbers to E.164 + fraud signals:** `twilio-lookup-phone-intelligence`
- **Bring a WhatsApp sender for WhatsApp OTP:** `twilio-whatsapp-manage-senders`
- **Retire SMS-pumping defenses safely:** `twilio-security-hardening`
- **Credential setup:** `twilio-iam-auth-setup`
- **Source blog post:** [Migrate from Programmable Messaging to Verify](https://www.twilio.com/en-us/blog/migrate-programmable-messaging-to-verify)
