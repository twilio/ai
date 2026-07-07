---
name: twilio-migrate-messaging-to-verify
description: >
  Migrate an existing OTP / one-time-passcode flow from the Twilio Programmable
  Messaging API (self-built: generate code → send SMS → store in DB → validate)
  to the Twilio Verify API (managed code generation, delivery, expiry, rate
  limiting, and Fraud Guard). Covers the before/after code mapping, creating a
  Verify Service, replacing send + check logic, E.164 formatting with Lookup,
  bring-your-own custom codes with the Feedback API, error-code translation, and
  a safe incremental cutover. Use when a developer is moving OTP/2FA delivery off
  Programmable Messaging onto Verify, or asks "should I use Verify instead of
  sending my own SMS codes?".
---

## Overview

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

### Before — the Programmable Messaging pattern you're replacing

**Python (self-built OTP — ~20 lines + a database)**
```python
import os, random
from twilio.rest import Client

client = Client(os.environ["TWILIO_ACCOUNT_SID"], os.environ["TWILIO_AUTH_TOKEN"])

# 1. Generate a code yourself
code = f"{random.randint(0, 999999):06d}"

# 2. Persist it with an expiry (your own DB / cache)
db.save_otp(phone="+15558675310", code=code, expires_at=now() + 600)

# 3. Send it over a number you own and registered for A2P 10DLC
client.messages.create(
    from_="+15017122661",
    to="+15558675310",
    body=f"Your verification code is: {code}",
)

# ...later, when the user submits a code, you look it up, compare,
# check expiry, and count attempts yourself.
def check(phone, submitted):
    row = db.get_otp(phone)
    return row and row.code == submitted and row.expires_at > now()
```

### After — the Verify pattern

**Step 1 — Create a Verify Service (one-time)**

Do this once and reuse the SID. Create it in the [Console](https://console.twilio.com/us1/develop/verify/services) or via API:

**Python**
```python
service = client.verify.v2.services.create(friendly_name="My App Verification")
print(service.sid)  # VAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx — save as VERIFY_SERVICE_SID
```

**Node.js**
```node
const service = await client.verify.v2.services.create({ friendlyName: "My App Verification" });
console.log(service.sid); // VAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> `friendlyName` rejects 5+ consecutive digits (error 60200). Use words.

**Step 2 — Replace your "generate + store + send" block with one call**

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
`channel` to `"voice"`, `"whatsapp"`, `"email"`, or `"rcs"`.

**Step 3 — Replace your "look up + compare + expiry check" with one call**

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
- **Must be enabled by Twilio Sales** (`CustomCodeEnabled` on the Service); it is not on by default.
- **Do not pass a custom message body.** Send only the code via `custom_code`; Verify maps it into an approved template. Custom message text returns [error 60243](https://www.twilio.com/docs/api/errors/60243).
- You are billed per **attempted** send, and custom-code users see ~30% more sends than checks (abandoned/re-requested codes) — expect ~30% higher cost than standard Verify.
- You **must** report the outcome via the **Verify Feedback API** so Twilio can monitor routing. Treat custom codes as a transition aid, not the end state — migrating to Verify-generated codes removes all three caveats.

### Translate your error handling

Verify uses different error codes than Messaging. Map them during migration:

| Situation | Verify error | Handle by |
|---|---|---|
| Malformed / non-E.164 number, bad param | 60200 | Fix `to` format (use Lookup) |
| Too many sends to a number | 60202 | Back off; surface "try again later" |
| Too many check attempts | 60203 | Issue a new verification |
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

- **Cannot retrieve the code Verify generated** — never returned by any API (by design). If your old flow logged or displayed the code, that stops working; migrate the UX to "we sent you a code."
- **Cannot reuse your owned number as the Verify sender by default** — Verify picks from managed sender pools per country. (You *can* bring your own number/Messaging Service for specific channels/WhatsApp, but that's not the default.)
- **Cannot pass a custom message body** — only `custom_code`; custom text → error 60243.
- **`custom_code` is not self-serve** — requires Twilio Sales to enable `CustomCodeEnabled`, bills per attempt (~30% higher), and mandates Feedback API reporting.
- **Cannot change the 10-minute token TTL per verification** — configurable only at the Service level, not per request.
- **Cannot re-check an approved verification** — single-use; a second check returns 404. Don't port "verify twice" logic.
- **Cannot send to non-E.164 numbers** — normalize first (Lookup).
- **No push webhook on completion** — poll `verification_checks` (rate-limited: 60/min, 180/hr, 250/day) or use the Events API. If your old design fired a callback on validation, replace it with a poll.
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
