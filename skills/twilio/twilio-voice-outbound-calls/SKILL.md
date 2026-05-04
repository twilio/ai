---
name: twilio-voice-outbound-calls
description: >
  Make outbound phone calls via Twilio's Programmable Voice REST API. Covers
  initiating calls with TwiML instructions, inline TwiML, tracking call status,
  and recording calls. Use this skill whenever an agent needs to place an
  outbound voice call through Twilio.
---

## Overview

> **Agent safety:** Before placing an outbound call, always confirm the recipient number and intent with the user. Outbound calls are irreversible and may incur charges. For automated systems, implement TCPA compliance: obtain prior express consent, respect quiet hours (8 AM–9 PM recipient local time), and maintain a Do Not Call list.

Every outbound call requires a `from` Twilio number, a `to` recipient, and TwiML instructions that define what happens when the call is answered — either as a webhook URL or inline.

---

## Prerequisites

- Twilio account with a voice-capable phone number
  — New to Twilio? See `twilio-account-setup` for signup, getting a number, and trial limitations
  — Trial accounts can only call verified numbers
- Environment variables:
  - `TWILIO_ACCOUNT_SID`
  - `TWILIO_AUTH_TOKEN`
  — See `twilio-iam-auth-setup` for credential setup and best practices
- SDK: `pip install twilio` / `npm install twilio`

---

## Quickstart

**Python**
```python
import os
from twilio.rest import Client

client = Client(os.environ["TWILIO_ACCOUNT_SID"], os.environ["TWILIO_AUTH_TOKEN"])

call = client.calls.create(
    from_="+15017122661",    # Your Twilio number (E.164)
    to="+15558675310",       # Recipient (E.164)
    twiml="<Response><Say>Your order has shipped. Goodbye.</Say></Response>"
)

print(call.sid)     # CAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
print(call.status)  # queued | ringing | in-progress | completed | failed
```

**Node.js**
```node
const twilio = require("twilio");
const client = twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);

const call = await client.calls.create({
    from: "+15017122661",
    to: "+15558675310",
    twiml: "<Response><Say>Your order has shipped. Goodbye.</Say></Response>",
});

console.log(call.sid);
console.log(call.status);
```

---

## Key Patterns

### Use a Webhook URL for Dynamic Call Handling

Pass a `url` instead of inline `twiml` — Twilio POSTs to your server when the call connects and executes the TwiML you return.

**Python**
```python
call = client.calls.create(
    from_="+15017122661",
    to="+15558675310",
    url="https://yourapp.com/twiml/welcome"
)
```

**Node.js**
```node
const call = await client.calls.create({
    from: "+15017122661",
    to: "+15558675310",
    url: "https://yourapp.com/twiml/welcome",
});
```

**Python (Flask) — TwiML webhook handler**
```python
from flask import Flask
from twilio.twiml.voice_response import VoiceResponse

app = Flask(__name__)

@app.route("/twiml/welcome", methods=["POST"])
def welcome():
    response = VoiceResponse()
    response.say("Hello! Press 1 to hear your account balance.")
    response.gather(num_digits=1, action="/twiml/handle-input")
    return str(response)
```

**Node.js (Express) — TwiML webhook handler**
```node
const { VoiceResponse } = require("twilio").twiml;

app.post("/twiml/welcome", (req, res) => {
    const response = new VoiceResponse();
    response.say("Hello! Press 1 to hear your account balance.");
    response.gather({ numDigits: 1, action: "/twiml/handle-input" });
    res.type("text/xml").send(response.toString());
});
```

For all TwiML verbs (Say, Gather, Dial, Record, Conference), see `twilio-voice-twiml`.

### Track Call Status

**Python**
```python
call = client.calls.create(
    from_="+15017122661",
    to="+15558675310",
    url="https://yourapp.com/twiml/welcome",
    status_callback="https://yourapp.com/call-status",
    status_callback_method="POST"
)
```

**Node.js**
```node
const call = await client.calls.create({
    from: "+15017122661",
    to: "+15558675310",
    url: "https://yourapp.com/twiml/welcome",
    statusCallback: "https://yourapp.com/call-status",
    statusCallbackMethod: "POST",
});
```

Status transitions: `queued → ringing → in-progress → completed` (or `failed`/`busy`/`no-answer`).

### Record a Call

**Python**
```python
call = client.calls.create(
    from_="+15017122661",
    to="+15558675310",
    url="https://yourapp.com/twiml/welcome",
    record=True,
    recording_status_callback="https://yourapp.com/recording-ready"
)
```

**Node.js**
```node
const call = await client.calls.create({
    from: "+15017122661",
    to: "+15558675310",
    url: "https://yourapp.com/twiml/welcome",
    record: true,
    recordingStatusCallback: "https://yourapp.com/recording-ready",
});
```

Recordings accessible at `https://api.twilio.com/2010-04-01/Accounts/{AccountSid}/Recordings`.

---

## Response Fields

| Field | Description |
|-------|-------------|
| `sid` | Call identifier (`CA...`) |
| `status` | `queued`, `ringing`, `in-progress`, `completed`, `failed`, `busy`, `no-answer` |
| `duration` | Length in seconds (after completion) |
| `price` | Cost (populated after completion) |

---

## Common Errors

| Code | Meaning | Fix |
|------|---------|-----|
| 21211 | Invalid `to` number | Validate E.164 format |
| 13224 | Invalid TwiML at webhook URL | Validate TwiML response from your server |
| 13225 | TwiML URL returned non-200 status | Fix your webhook endpoint |

---

## CANNOT

- **Cannot use a private TwiML URL** — Must be publicly accessible. Use ngrok for local development.
- **Cannot exceed ~4,096 characters in inline `twiml` parameter** — Use a TwiML URL for longer responses
- **Cannot call unverified numbers on trial accounts** — Upgrade to paid or verify recipient numbers first

---

## Next Steps

- **TwiML verb reference (Say, Gather, Dial, Record, Conference):** `twilio-voice-twiml`
- **AI voice agents with real-time speech/LLM:** `twilio-voice-conversation-relay`
