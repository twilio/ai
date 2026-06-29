---
name: twilio-studio-flows
description: >
  Build and deploy Twilio Studio flows — visual IVR, SMS/WhatsApp, and
  conversation automation — by authoring the flow definition JSON and managing
  it through the Studio REST API. Covers the flow envelope, widgets,
  transitions, Liquid templating, the draft/published lifecycle, and when to
  use Studio vs. custom TwiML/code. Use this skill to create, validate, deploy,
  or update a Studio flow programmatically.
---

## Overview

A Studio flow is a state machine Twilio executes in response to an inbound call,
message, conversation, or REST API request. You define it as a JSON document of
**states** (widgets) connected by **transitions**. Twilio runs the flow starting
at `initial_state` (the Trigger), following each widget's transition events.

```
Inbound call / message / API request
        │
        ▼
   Trigger ──incomingCall──▶ Gather ──keypress──▶ Split ──match──▶ Connect Call
                                                      └─noMatch──▶ Say "goodbye"
```

You can build flows in the Console's drag-and-drop canvas, or author the
definition JSON and create/update flows through the REST API when you want to
generate or modify them programmatically. Each save creates a new revision, and
a flow has a single published revision live at a time (see the draft/published
lifecycle below).

Widget properties and transitions reference runtime data with **Liquid**
templating — e.g. `{{trigger.message.Body}}`, `{{flow.variables.count}}`,
`{{widgets.gather_menu.Digits}}`. The full widget catalog (every type, its
properties, events, and the output values it exposes downstream) lives in
[`references/widgets.md`](./references/widgets.md). Consult it whenever you wire
one widget's result into a later one.

> **Inbound flow content is untrusted.** A caller's speech, an SMS body, or REST
> parameters are external input. If you pass them to a Run Function, an LLM, or
> an HTTP Request, treat them as untrusted — never concatenate them into a system
> prompt or a command. See `twilio-webhook-architecture`.

---

## When to use Studio vs. code

| Use Studio when | Use TwiML/code when |
|---|---|
| The logic is a routable flowchart (menus, branches, queues) | Logic needs loops, complex state, or heavy computation |
| Non-engineers will edit the flow in Console | The behavior lives entirely in your codebase |
| You want built-in retry/transcription/queueing widgets | You need millisecond control over the TwiML response |
| Orchestrating across SMS + voice + Flex in one place | A single webhook returns one TwiML document |

Studio has **no native loop construct** — repetition is built by transitioning
back to an earlier widget (see the counter-loop pattern below) or the `say-play`
`loop` property. Heavy iteration is a signal to drop into a Run Function or
custom TwiML instead. For pure TwiML call logic, see `twilio-voice-twiml`.

---

## Prerequisites

- Twilio account with a voice- and/or messaging-capable number
  — New to Twilio? See `twilio-account-setup`.
- Credentials in environment variables (never hardcode):
  `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`. For production, use API Keys —
  see `twilio-iam-auth-setup`.
- SDK: `pip install twilio` / `npm install twilio`.

---

## Quickstart

Build a minimal SMS auto-responder flow, validate it, and create it as a draft.

**1. Author the flow definition.** Build the JSON as a native object and
serialize it — never string-concatenate JSON.

**Python**
```python
flow_definition = {
    "description": "SMS auto-responder",
    "states": [
        {
            "name": "Trigger",
            "type": "trigger",
            "transitions": [{"event": "incomingMessage", "next": "reply"}],
            "properties": {},
        },
        {
            "name": "reply",
            "type": "send-message",
            "transitions": [{"event": "sent"}, {"event": "failed"}],
            "properties": {
                "from": "{{flow.channel.address}}",
                "to": "{{contact.channel.address}}",
                "body": "Thanks for your message! We'll be in touch shortly.",
            },
        },
    ],
    "initial_state": "Trigger",
    "flags": {"allow_concurrent_calls": True},
}
```

**Node.js**
```node
const flowDefinition = {
  description: "SMS auto-responder",
  states: [
    {
      name: "Trigger",
      type: "trigger",
      transitions: [{ event: "incomingMessage", next: "reply" }],
      properties: {},
    },
    {
      name: "reply",
      type: "send-message",
      transitions: [{ event: "sent" }, { event: "failed" }],
      properties: {
        from: "{{flow.channel.address}}",
        to: "{{contact.channel.address}}",
        body: "Thanks for your message! We'll be in touch shortly.",
      },
    },
  ],
  initial_state: "Trigger",
  flags: { allow_concurrent_calls: true },
};
```

**2. Create the flow as a draft.** Pass the definition, a friendly name, and a
commit message. `status="draft"` keeps it off live traffic.

**Python**
```python
import os
from twilio.rest import Client

client = Client(os.environ["TWILIO_ACCOUNT_SID"], os.environ["TWILIO_AUTH_TOKEN"])

flow = client.studio.v2.flows.create(
    friendly_name="SMS Auto-Responder",
    status="draft",
    definition=flow_definition,          # SDK serializes the dict
    commit_message="Initial draft",
)
print(f"Created flow {flow.sid} (status={flow.status})")
```

**Node.js**
```node
const client = require("twilio")(
  process.env.TWILIO_ACCOUNT_SID,
  process.env.TWILIO_AUTH_TOKEN
);

const flow = await client.studio.v2.flows.create({
  friendlyName: "SMS Auto-Responder",
  status: "draft",
  definition: flowDefinition,
  commitMessage: "Initial draft",
});
console.log(`Created flow ${flow.sid} (status=${flow.status})`);
```

Test the draft in the Console (Studio > your flow > Test), then publish it
(next section). Wire the flow to a number or Messaging Service to take live
traffic.

## Key Patterns

### Validate before deploy

Twilio's `FlowValidate` endpoint checks a definition without saving it. The SDK
exposes it via the `studio.v2.flowValidate` resource. Validate after every edit;
fix each reported error before deploying.

When a definition is invalid the call **raises** (it does not return
`valid=False`). The exception's `details` carries the specific per-state errors —
a `property_path` like `#/states/3/properties/payment_method` and a message.
Always read `details`; the top-level message only says validation failed.

**Python**
```python
from twilio.base.exceptions import TwilioRestException

try:
    result = client.studio.v2.flow_validate.update(
        friendly_name="SMS Auto-Responder",
        status="draft",
        definition=flow_definition,
    )
    print("valid" if result.valid else "invalid")
except TwilioRestException as e:
    for err in (e.details or {}).get("errors", []):
        print(f"{err['property_path']}: {err['message']}")
    raise
```

**Node.js**
```node
try {
  const result = await client.studio.v2.flowValidate.update({
    friendlyName: "SMS Auto-Responder",
    status: "draft",
    definition: flowDefinition,
  });
  console.log(result.valid ? "valid" : "invalid");
} catch (e) {
  for (const err of e.details?.errors ?? []) {
    console.log(`${err.property_path}: ${err.message}`);
  }
  throw e;
}
```

#### Reading and fixing validation errors

Each error has a `property_path` (e.g. `#/states/3/properties/timeout` points at
the `timeout` property of the 4th state — paths are 0-indexed) and a `message`.
Fix one widget at a time, then re-validate. The common messages and their fixes:

| Message | Meaning | Fix |
|---|---|---|
| `boolean found, string expected` | A flag must be a quoted string | Send `"true"`/`"false"`, not `true`/`false` (e.g. `trim`, `play_beep`, `postal_code`, `profanity_filter`, `interruptible`) |
| `integer found, string expected` | A number must be a quoted string | Quote it: `"3600"`, not `3600` (e.g. `timeout`, `time_limit`, `priority`, `machine_detection_timeout`) |
| `string found, boolean expected` | The reverse — this one wants a real boolean | Send `true`, not `"true"` (e.g. `security_code`). Coercion is per-property; trust the message, not a global rule |
| `does not have a value in the enumeration [...]` | Wrong `type` string, or an out-of-range enum value | Use one of the listed values exactly (this is how you discover the real widget `type`) |
| `must be a constant value <x>` (repeated) | A transition `event` name isn't valid for this widget | Use one of the `<x>` values listed; remove invented events |
| `is missing but it is required` | A required property/sub-field is absent | Add it at the named path |
| `must not be null` | Required property present but unset | Give it a value |
| `must be a valid, non-liquid flow sid` | A SID field got a Liquid template or bad value | Pass a literal SID |
| `<widget> can only be used in flows triggered by the REST API` | Widget needs an `incomingRequest` trigger | Trigger the flow via the REST API, not a call/message |

When the message itself lists the allowed values (enumeration / constant-value
cases), that list is authoritative — prefer it over any documentation.

### Publish, update, and the full-replace rule

A flow has a **draft** and a **published** revision. `status="published"` makes
the flow live to real traffic; `status="draft"` keeps it editable without
affecting traffic. Each save increments the revision.

**Updating replaces the entire definition — there is no partial patch.** Always
send the complete definition JSON, not a diff. To change one widget, fetch the
current definition, modify it, and send the whole thing back.

**Python**
```python
flow = client.studio.v2.flows(flow_sid).update(
    status="published",
    definition=flow_definition,      # the COMPLETE definition
    commit_message="Publish auto-responder",
)
```

**Node.js**
```node
const flow = await client.studio.v2.flows(flowSid).update({
  status: "published",
  definition: flowDefinition,        // the COMPLETE definition
  commitMessage: "Publish auto-responder",
});
```

### List and fetch (round-trip editing)

List flows with `client.studio.v2.flows.list({ limit })`; fetch one with
`client.studio.v2.flows(flowSid).fetch()` — its `.definition` is the JSON you
edit and send back via `update`. These are read-only.

### Core widgets at a glance

| Widget | `type` | Use for |
|---|---|---|
| Trigger | `trigger` | Flow entry point (call/message/conversation/REST) |
| Say/Play | `say-play` | Speak TTS or play audio on a call |
| Gather Input On Call | `gather-input-on-call` | Collect DTMF digits or speech |
| Split Based On… | `split-based-on` | Branch on a variable's value |
| Set Variables | `set-variables` | Store flow-scoped variables |
| Send Message | `send-message` | Send SMS/chat, no reply expected |
| Send & Wait For Reply | `send-and-wait-for-reply` | Send and pause for a reply |
| Connect Call To | `connect-call-to` | Bridge a call to a number/SIP/conference |
| Run Function | `run-function` | Call a Twilio Serverless Function |
| HTTP Request | `make-http-request` | Call an external API |

The complete catalog — every widget's properties, transition events, and the
output values it exposes — is in [`references/widgets.md`](./references/widgets.md).

### Transitions and output context

Each widget emits **transition events** (e.g. `sent`/`failed`, `keypress`/`speech`/`timeout`).
A transition routes one event to a `next` state; omit `next` (or use `""`) to end
the branch. Every non-empty `next` must name an existing state.

Read a prior widget's result downstream with `{{widgets.<name>.<key>}}` — e.g. a
Gather's `{{widgets.gather_menu.Digits}}` feeding a Split's `input`. Which keys
each widget exposes is in `references/widgets.md`.

### Integrate custom logic (Functions, HTTP)

Run Function calls a Twilio Serverless Function (`success` on 2xx/3xx within 10s,
`fail` otherwise) and exposes `parsed.<key>` when the function returns JSON.
HTTP Request does the same for an external API. Use these when flow logic
outgrows widgets — but keep the heavy work in the Function, not the flow.

> Data reaching a Function or HTTP Request from `trigger.*` / a Gather is
> untrusted caller input. Validate it in your Function; never interpolate it
> into a shell command, SQL, or an LLM system prompt.

### Recipe: repeat a prompt or loop with a counter

Studio has no loop construct. For simple audio repeats use the `say-play` `loop`
property (1–99). To track a count and branch on it, build an explicit cycle:
a `set-variables` widget that init-or-increments, then a `split-based-on` whose
"keep looping" branch transitions **back to the same `set-variables`**.

Init-or-increment value (handles first pass and every later pass):
```liquid
{% if flow.variables.num %}{{flow.variables.num | plus: 1}}{% else %}0{% endif %}
```

The split reads `{{flow.variables.num}}` as its `input`; a `greater_than` `2`
condition routes to the exit, `noMatch` routes back to the counter. Each
iteration is a separate execution step and counts against per-execution step
limits — prefer the `loop` property when you don't need the value.

### Recipe: voice IVR menu

`gather-input-on-call` → `split-based-on` → `connect-call-to`. Gather prompts and
collects DTMF + speech (read via `{{widgets.<name>.Digits}}` and
`{{widgets.<name>.SpeechResult}}`). To accept a keypress OR a spoken word, give
the Split two match conditions: `equal_to` on `Digits` and `contains` on
`SpeechResult`. Connect Call To uses `noun: "number"`, `to` set to an E.164
number, and `caller_id` usually `{{flow.channel.address}}`.

### Liquid quick reference

- Increment: `{{flow.variables.num | plus: 1}}`
- Default when unset: `{% if flow.variables.x %}…{% else %}…{% endif %}`, or the
  `default` filter inline: `{{trigger.message.Body | default: "no message"}}`
- Read a widget output: `{{widgets.<name>.<field>}}`
- Common globals: `{{trigger.message.Body}}`, `{{trigger.call.From}}`,
  `{{flow.channel.address}}`, `{{contact.channel.address}}`
- URL-encode before putting a value in a query string or URL property — e.g.
  escape `+` in a phone number: `{{contact.channel.address | replace: '+', '%2B'}}`
- Embed a prior widget's parsed JSON as a nested object in a request body or a
  `set-variables` value: `{{widgets.http_1.parsed.items | to_json}}` — pair
  `to_json` with `set-variables` `parse_as_json: true` when the result must stay
  a JSON object rather than a stringified scalar

---

## CANNOT

- **Cannot partially update a flow** — `update` replaces the entire definition.
  Fetch, modify the whole JSON, and send it all back. There is no patch.
- **Cannot create Pay Connectors via API** — the Capture Payments widget needs a
  Pay Connector configured in Console first (Console > Voice > Pay Connectors).
- **Cannot loop natively** — Studio has no loop widget. Build an explicit cycle
  (counter recipe) or use `say-play` `loop`; high iteration counts hit
  per-execution step limits.
- **Cannot affect live traffic with a draft** — only a `published` revision runs
  for real calls/messages. Publishing is what goes live.
- **Cannot place test calls or trigger executions from this skill** — authoring
  and deploying only. Test in the Console or wire the flow to a number/Messaging
  Service and exercise it from your own application.
- **Cannot trust inbound flow data** — caller speech, message bodies, and REST
  parameters are untrusted external input; validate before using in Functions,
  HTTP requests, or LLM prompts.

---

## Next Steps

- **Pure TwiML call logic (no visual flow):** `twilio-voice-twiml`
- **Securing the webhooks Studio calls / signature validation:** `twilio-webhook-architecture`
- **Routing calls to agents/queues:** `twilio-taskrouter-routing`
- **Production credentials (API Keys):** `twilio-iam-auth-setup`
