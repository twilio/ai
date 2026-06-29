# Studio Widget Catalog Reference

This reference covers the Studio Flow definition schema, the global context
namespaces available everywhere in a flow, and a per-widget catalog of inputs,
transition events, and outputs.

## Flow definition schema

Authoritative reference: https://www.twilio.com/docs/studio/rest-api/v2/flow

A flow definition is a JSON object with this envelope:

```json
{
  "description": "Human-readable name of the flow",
  "states": [ /* array of widget state objects */ ],
  "initial_state": "Trigger",
  "flags": { "allow_concurrent_calls": true }
}
```

### Top-level keys

| Key | Type | Required | Notes |
|-----|------|----------|-------|
| `description` | string | no | Recommended flow description. |
| `states` | array | yes | One object per widget. Must include the Trigger. |
| `initial_state` | string | yes | Name of the starting state — almost always the Trigger. |
| `flags` | object | no | Optional flow flags, e.g. `allow_concurrent_calls`. May be omitted. |

### State (widget) object

```json
{
  "name": "greet_caller",
  "type": "send-message",
  "transitions": [ {"event": "sent", "next": "next_widget_name"} ],
  "properties": { /* widget-specific — see the catalog below */ }
}
```

- `name` — unique within the flow; referenced by transitions' `next`.
- `type` — the widget type string (see the catalog below).
- `transitions` — array of `{event, next}`. `event` is a widget-specific
  outcome (e.g. `sent`, `noReply`, `audioComplete`); `next` is the target
  state name. To end a branch, omit the `next` field entirely (the form used
  in Twilio's docs); an empty string `""` is also accepted.
- `properties` — widget-specific configuration.

### Transitions and events

Each widget type defines its own set of transition events. A transition that
omits `next` (or sets it to an empty string) ends the branch. Every non-empty
`next` must name an existing state in the flow.

Studio has no native loop construct — to repeat a step (e.g. retry a prompt),
route a transition back to an earlier state.

### Liquid templating

Widget properties support Liquid template syntax to reference flow/runtime
variables, e.g. `{{flow.data}}`, `{{trigger.message.Body}}`,
`{{widgets.greet_caller.Body}}`. Widget outputs are addressed directly as
`{{widgets.<name>.<key>}}` — there is no `.outputs.` segment. See:
https://www.twilio.com/docs/studio/user-guide/working-with-variables-in-studio

## Global context (available everywhere, not scoped to a widget)

Besides `widgets.*`, every flow has four global namespaces. `trigger.*` is
populated once, by the event that started the execution.

### `trigger.*` — the triggering event

Each trigger type populates exactly one subtree. The inbound payload is a
verbatim passthrough, so the key set is open-ended; the entries below are the
common, load-bearing ones.

| Trigger event | Subtree | Notable keys |
|---|---|---|
| `incomingCall` | `trigger.call.*` | `From`, `To`, `Caller`, `CallSid`, `AccountSid`, `CallStatus`, `Direction`, geo (`FromCity`, `FromState`, …), `AddOns` (JSON-parsed) |
| `incomingMessage` | `trigger.message.*` | `From`, `To`, `Body` (truncated if oversized), `MessageSid`, `AccountSid`, `NumMedia`, `MediaUrl0…`, `ChannelAttributes` (JSON-parsed), `AddOns` (JSON-parsed) |
| `incomingConversationMessage` | `trigger.conversation.*` | `From`, `Body` (truncated), `ConversationSid`, `MessageSid`, `AccountSid`, `Author`, `ParticipantSid` |
| `incomingRequest` (REST API) | `trigger.request.*` | `from`, `to`, `parameters.<custom>` (open-ended), and `Param_FlexInstanceSid`/`Param_ResourceId` (only in the Flex case) |

Notes:
- The conversation trigger event is `incomingConversationMessage` (not
  `incomingConversation`).
- For REST-API triggers, caller-supplied custom variables appear in **two**
  places: `trigger.request.parameters.<name>` and `flow.data.<name>`.

### `flow.*` — flow-level data

| Key | Meaning |
|---|---|
| `flow.sid` | The **execution** SID (the key is literally `sid`) |
| `flow.flow_sid` | The flow SID |
| `flow.channel.address` | Twilio-side address (call/message `To`; REST `from`; messaging-service SID for RCS) |
| `flow.data.<name>` | REST-API custom variables (open-ended) |
| `flow.variables.<key>` | Variables written by Set Variables |
| `flow.add_ons` | Add-on data lifted from a widget event, when present |
| `flow.channel.*` | For conversation-channel executions, enriched by Start/Resume Conversation with `sid`, `type` (`conversation`), `status`, `details.{chat_service_sid, friendly_name, participants, webhook_sid}` |

> Naming quirk: `flow.sid` holds the execution SID; `flow.flow_sid` holds the
> flow SID.

### `contact.*` — the end user

| Key | Meaning |
|---|---|
| `contact.channel.address` | The contact address (call/message `From`; REST `to`; `rcs:` prefix stripped for messaging-service channels) |

Only `contact.channel.address` is populated.

## How widget outputs work

A widget exposes downstream variables under the namespace
`{{widgets.<name>.<key>}}`, where `<name>` is the widget's `name`
in the flow JSON (not its type). A few things worth knowing:

- **Callback-driven widgets pass through the raw Twilio request.** Voice and
  messaging widgets that transition on an inbound webhook (Gather, Record,
  Connect Call To, Make Outgoing Call, Send & Wait For Reply, …) expose the
  entire callback payload. The keys listed per widget are the load-bearing
  ones, but the actual set is whatever Twilio's Voice/Messaging layer posts —
  it is open-ended, and more keys are usually present than are listed here.
- **Failure branches are usually empty.** Most widgets emit their failure event
  with an empty parameter map, so `{{widgets.<name>.*}}` resolves to nothing on
  the error transition. Exceptions are called out per widget (notably Send
  Message / Send & Wait For Reply on a true API send failure, which still carry
  `outbound.*`).

## Widget catalog

### Trigger
- **Type:** `trigger`
- **Purpose:** Entry point for every flow; initiates execution in response to an incoming call, message, conversation, or REST API request.
- **Key properties:** `offset` — visual canvas position; no user-configurable trigger properties (Webhook URL and REST API URL are auto-populated and read-only).
- **Transition events:** `incomingCall`, `incomingMessage`, `incomingConversationMessage`, `incomingRequest` (REST API)
- **Outputs** (`{{widgets.<name>.<key>}}`): none scoped to the widget — see the `trigger.*` global namespace above.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/trigger-start

### Split Based On…
- **Type:** `split-based-on`
- **Purpose:** Branches the flow by testing a variable against one or more conditions, routing to different widgets based on which condition matches.
- **Key properties:** `input` — the Liquid expression or variable to test (e.g. `{{widgets.gather_menu.Digits}}`); `conditions` — array of condition objects each with a `friendly_name`, `arguments` (array, holds the tested value/expression), `type` (the operator — see table below), and `value` (the comparison operand)
- **Condition `type` values:** the operator is one of these exact strings. List operators (`matches_any_of` / `does_not_match_any_of`) take a **comma-separated** `value` (e.g. `"sales, support, billing"`). Blank checks (`is_blank` / `is_not_blank`) ignore `value`.

  | `type` | Matches when the value… |
  |---|---|
  | `equal_to` | equals `value` |
  | `not_equal_to` | does not equal `value` |
  | `matches_any_of` | equals any entry in the comma-separated `value` list |
  | `does_not_match_any_of` | equals none of the comma-separated `value` list |
  | `contains` | contains `value` as a substring |
  | `does_not_contain` | does not contain `value` |
  | `starts_with` | starts with `value` |
  | `does_not_start_with` | does not start with `value` |
  | `is_blank` | is empty or whitespace-only |
  | `is_not_blank` | has non-whitespace characters |
  | `regex` | matches the `value` regex (case-insensitive, whole string) |
  | `less_than` / `greater_than` | is numerically less/greater than `value` |
  | `is_before_date` / `is_after_date` | is a date (`YYYY-MM-DD`) before/after `value` |
  | `is_before_time` / `is_after_time` | is a time (24h `HH:MM`) before/after `value` |
- **Transition events:** `match` (with the matching condition's `friendly_name` carried inside the transition's `conditions` array) and `noMatch` when nothing matches
- **Outputs** (`{{widgets.<name>.<key>}}`): none
- **Docs:** https://www.twilio.com/docs/studio/widget-library/split-based-on

**Shape (this is what trips people up):** `input` is the only entry in
`properties`. Each `match` is a **separate transition** carrying its own
`conditions` array — conditions live on the transition, not in `properties`. Use
one `match` transition per branch, plus one `noMatch`. `arguments` is an array.

```json
{
  "name": "route_choice",
  "type": "split-based-on",
  "properties": { "input": "{{widgets.gather_menu.Digits}}" },
  "transitions": [
    { "event": "noMatch", "next": "say_goodbye" },
    { "event": "match", "next": "connect_sales",
      "conditions": [
        { "friendly_name": "pressed 1",
          "arguments": ["{{widgets.gather_menu.Digits}}"],
          "type": "equal_to", "value": "1" } ] },
    { "event": "match", "next": "connect_support",
      "conditions": [
        { "friendly_name": "pressed 2",
          "arguments": ["{{widgets.gather_menu.Digits}}"],
          "type": "equal_to", "value": "2" } ] }
  ]
}
```

### Set Variables
- **Type:** `set-variables`
- **Purpose:** Stores one or more key/value pairs as flow-scoped variables accessible via `{{flow.variables.<key>}}` throughout the rest of the flow.
- **Key properties:** `variables` — array of `{key, value}` objects; `value` supports Liquid expressions; optionally `parse_as_json: true` to treat the value as a JSON object
- **Transition events:** `next`
- **Outputs:** `{{flow.variables.<key>}}` — each configured key/value, interpolated and type-cast (null becomes empty string). The same pairs are also written under `{{widgets.<name>.<key>}}`.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/set-variables

### Say/Play
- **Type:** `say-play`
- **Purpose:** Plays a spoken text-to-speech message, a pre-recorded audio file, or DTMF tones to a caller.
- **Key properties:** `say` — text to speak (supports Liquid and SSML); `play` — URL of audio file to play; `digits` — DTMF sequence to send; `voice` — TTS voice name; `language` — TTS language/dialect; `loop` — repetition count (1–99)
- **Transition events:** `audioComplete`
- **Outputs** (`{{widgets.<name>.<key>}}`): none usable as data — only call-status params from the `audioComplete` redirect pass through.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/sayplay

### Gather Input On Call
- **Type:** `gather-input-on-call`
- **Purpose:** Prompts a caller (via TTS or audio file) and collects either DTMF keypress digits or speech input, then routes based on what was received.
- **Key properties:** `say` — prompt text; `play` — prompt audio URL; `voice`, `language` — TTS settings; `timeout` — seconds to wait for input (0–30); `finish_on_key` — keypress that ends input early; `num_digits` — max digits to collect; `speech_recognition_language` — language for voice recognition; `hints` — comma-separated expected phrases; `profanity_filter` — redact profanity from transcripts, as the **string** `"true"`/`"false"` (FlowValidate rejects a JSON boolean here); `loop` — prompt repeat count
- **Transition events:** `keypress`, `speech`, `timeout`
- **Outputs** (`{{widgets.<name>.<key>}}`): `Digits` — DTMF keys pressed (on `keypress`); `SpeechResult` — transcribed speech (on `speech`); `Confidence` — ASR confidence 0.0–1.0 (on `speech`); `CallStatus` — call status at callback; plus standard voice-request params passed through verbatim (`AccountSid`, `CallSid`, `From`, `To`, geo fields, …). `timeout` carries only standard call params (no input keys); `hangup`/`failed` set no input keys.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/gather-input-call

### Connect Call To
- **Type:** `connect-call-to`
- **Purpose:** Bridges an in-progress call to another phone number, Voice SDK client, SIP endpoint, or conference.
- **Key properties:** `noun` — destination type, lowercase (`number`, `client`, `sip`, `conference`); `to` — destination address (phone number, client identity, SIP URI, or Conference SID); `caller_id` — caller ID shown to the recipient; `record` — enable call recording; `timeout` — seconds to wait for answer (1–600); `time_limit` — max call duration in seconds, as a **string** (FlowValidate rejects a JSON integer here)
- **Transition events:** `callCompleted`, `hangup`
- **Outputs** (`{{widgets.<name>.<key>}}`, from the `<Dial>` action callback): `DialCallStatus` — outcome of the dialed leg (`completed`/`busy`/`no-answer`/`failed`/`answered`); `DialCallSid` — SID of the dialed (child) leg; `DialCallDuration` — dialed-leg duration in seconds; `RecordingUrl` — recording URL, when recording is enabled; plus standard call params (`CallSid`, `CallStatus`, `From`, `To`, `AccountSid`, …)
- **Docs:** https://www.twilio.com/docs/studio/widget-library/connect-call

### Make Outgoing Call
- **Type:** `make-outgoing-call-v1` or `make-outgoing-call-v2` (there is no bare `make-outgoing-call` type)
- **Note:** `make-outgoing-call-v1` validates with integer/boolean-free defaults; `-v2` additionally requires `trim` and `machine_detection_timeout` as strings.
- **Purpose:** Dials out to a phone number and delivers a voice message or connects the call, enabling proactive outbound voice interactions.
- **Key properties:** `to` — number to call (default: `{{contact.channel.address}}`); `from` — caller ID (default: `{{flow.channel.address}}`); `record` — enable recording; `recording_channels` — `mono` or `dual`; `recording_status_callback` — callback URL when recording ready; `trim` — remove silence, as the **string** `"true"`/`"false"` (FlowValidate rejects a JSON boolean); `machine_detection` — detect answering machine (`Enable`, `DetectMessageEnd`); `machine_detection_timeout` — seconds for detection (3–120), as a **string** (FlowValidate rejects a JSON integer); `send_digits` — DTMF tones post-connect; `timeout` — ring duration (0–600)
- **Transition events:** `answered`, `busy`, `noAnswer`, `failed`. `make-outgoing-call-v1` additionally exposes `answeredByMachine` (when `machine_detection` is enabled) and `hangup`; `-v2` folds those outcomes into the four base events.
- **Outputs** (`{{widgets.<name>.<key>}}`, raw call/status callback params): `CallSid` — outbound call SID; `CallStatus` — call status driving transitions; `AnsweredBy` — answering-machine-detection result; plus `AccountSid`, `From`, `To`, `Direction`, `MachineDetectionDuration`
- **Docs:** https://www.twilio.com/docs/studio/widget-library/make-outgoing-call

### Record Voicemail
- **Type:** `record-voicemail`
- **Purpose:** Records a voicemail from a caller with optional transcription, configurable silence detection, and keypress-to-stop.
- **Key properties:** `silence_timeout` — seconds of silence before ending recording (default: 5); `finish_on_key` — key that stops recording (0–9, #, *); `max_length` — max recording duration in seconds (1–14400, default: 3600); `transcribe` — enable transcription; `transcribe_callback` — URL for transcription results; `trim` — remove trailing silence, as the **string** `"true"`/`"false"` (FlowValidate rejects a JSON boolean here); `play_beep` — play beep before recording, as the **string** `"true"`/`"false"` (FlowValidate rejects a JSON boolean here)
- **Transition events:** `recordingComplete`, `noAudio`, `hangup`
- **Outputs** (`{{widgets.<name>.<key>}}`, event `recordingComplete` unless noted): `RecordingUrl` — URL of the recording; `RecordingDuration` — length in seconds; `RecordingSid` — recording SID; `CallStatus` — call status (also on `hangup`); `Digits` — finish key pressed, or `"hangup"` on a hangup-ended recording; plus standard voice-request params. `noAudio` (silence/timeout) and `failed` set no recording keys.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/record-voicemail

### Call Recording
- **Type:** `record-call`
- **Purpose:** Toggles call recording on or off at any point during a voice call flow.
- **Key properties:** `record_call` — JSON boolean `true` to start recording, `false` to stop (required; this one is a real boolean, not a string); `recording_status_callback` — URL for recording event notifications; `recording_status_callback_method` — `GET` or `POST` (default: `POST`); `recording_status_events` — which events trigger callbacks (`in-progress`, `completed`, `absent`); `recording_channels` — `mono` or `dual` (default: `dual`); `trim` — `trim-silence` or `do-not-trim`
- **Transition events:** `success`, `failed`
- **Outputs** (`{{widgets.<name>.<key>}}`, event `success`, built from the Recording resource): `Sid`, `AccountSid`, `CallSid`, `Status`, `StartTime`, `Duration`, `DateCreated`, `DateUpdated`, `ErrorCode`, `Price`, `PriceUnit`, `Channels`, `Source`, `Uri`. Note: the recording URL key is `Uri` (not `RecordingUrl`). `failed` exposes no variables.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/call-recording

### Enqueue Call
- **Type:** `enqueue-call`
- **Purpose:** Places the current call into a TaskRouter call queue, playing hold music until the task is dequeued by a worker.
- **Key properties:** `queue_name` — name of the queue (created on demand, max 64 chars); `task_router_workspace` — TaskRouter Workspace SID; `task_router_workflow` — Workflow SID for routing; `task_attributes` — JSON task metadata (max 1024 chars); `priority` — task priority (higher = served first, default: 0); `timeout` — seconds before task expires; `hold_music_url` — custom hold music TwiML URL; `twiml_request_method` — `GET` or `POST` for hold music requests
- **Transition events:** `callComplete`, `failedToEnqueue`, `callFailure`
- **Outputs** (`{{widgets.<name>.<key>}}`, from the `<Enqueue>` action callback): `QueueResult` — dequeue outcome (drives the transition); `QueueSid` — queue SID; `QueueTime` — seconds spent in queue; `CallSid`, `CallStatus`; plus standard call params (`AccountSid`, `Direction`, `From`, `To`, …)
- **Docs:** https://www.twilio.com/docs/studio/widget-library/enqueue-call

### Capture Payments
- **Type:** `capture-payments`
- **Purpose:** Securely captures credit card or ACH payment details from a caller using PCI-compliant DTMF collection, then tokenizes or processes the payment via a connected Payment Gateway.
- **Key properties:** `payment_connector` — Marketplace Pay Connector name; `timeout` — seconds awaiting additional digit input (default: 5); `max_attempts` — retry count on failure (default: 2); `language` — language for cardholder prompts; `valid_card_types` — accepted card brands; `security_code` — require CVV, JSON boolean (default: true); `postal_code` — require postal code, as the **string** `"true"`/`"false"` (FlowValidate rejects a JSON boolean here, unlike `security_code`); `payment_token_type` — `one-time` or `reusable`; `charge_amount` — transaction amount; `currency` — transaction currency (default: USD); `payment_method` — `CREDIT_CARD` or `ACH_DEBIT`; `bank_account_type` — ACH account category
- **Transition events:** `success`, `maxFailedAttempts`, `providerError`, `payInterrupted`, `hangup`, `validationError`
- **Outputs** (`{{widgets.<name>.<key>}}`): `Result` — Pay outcome string; drives every transition; `PaymentToken` — payment/reusable token; `PaymentConfirmationCode` — confirmation code; `ProfileId` — profile id, on the token-as-profile-id path (on `success`); `PaymentCardNumber`, `PaymentCardType`, `PaymentCardPostalCode`, `ExpirationDate`, `SecurityCode` — card fields, when the callback includes them (on `success`). Error transitions still carry `Result`/`PaymentToken`/`PaymentConfirmationCode` (usually empty).
- **Docs:** https://www.twilio.com/docs/studio/widget-library/capture-payments

### Fork Stream
- **Type:** `fork-stream`
- **Purpose:** Starts or stops a real-time audio stream of a call, forwarding audio to a WebSocket or SIPREC endpoint.
- **Key properties:** `stream_action` — `start` or `stop` (lowercase; the property is `stream_action`, NOT `action`, and capitalized values are rejected); `stream_name` — friendly name for the stream; `stream_transport_type` — `websocket` or `siprec` (lowercase; the property is `stream_transport_type`, NOT `stream_type`); `stream_url` — destination WebSocket URL (`wss://`), required when `stream_transport_type` is `websocket` (the key is `stream_url`, NOT `url`); `stream_track` — audio to stream: `inbound_track`, `outbound_track`, or `both_tracks`; `stream_connector_name` — connector name, required for `siprec`. A `stop` action needs only `stream_action` + `stream_name`.
- **Transition events:** `next`
- **Outputs** (`{{widgets.<name>.<key>}}`): none — the single `next` transition exposes no variables.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/fork-stream

### Connect Virtual Agent (v2)
- **Type:** `connect-virtual-agent-v2`
- **Purpose:** Connects a Voice call or Conversation to a Google Dialogflow CX virtual agent for conversational AI handling, with session resumption and live agent handoff.
- **Key properties:** `channel` — `voice` or `conversations`; `connector_name` — the Dialogflow CX connector name (required for the voice channel); `session_behavior` — `new_session` or resume an existing one; `resume_session_identification_method` — how a paused session is matched on resume (e.g. `widget`); `status_callback_method` — `GET` or `POST` (default: `POST`); `timeout` — session timeout in seconds
- **Transition events:** `completed`, `live-agent-handoff`, `hangup`, `failed`, `paused`, `timeout`
- **Outputs** (`{{widgets.<name>.<key>}}`): in-flight bookkeeping (Conversations) `ParticipantSid`, `onConversationStateUpdatedWebhookSid`; (Voice) `ConnectorName`. On transition: `VirtualAgentProvider`, `VirtualAgentStatus` (the event name), `VirtualAgentProviderData` (nested; e.g. `VirtualAgentProviderData.AgentHandoffParameters` on handoff, `.EndUserId`, `.ConversationId`, `.PauseParameters` on Voice), `VirtualAgentError`, `VirtualAgentErrorCode`. `timeout`/`paused`/`hangup` expose no params.
- **Note:** A deprecated v1 widget exists — use v2 (`connect-virtual-agent-v2`) for all new flows.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/connect-virtual-agent

### Send Message
- **Type:** `send-message`
- **Purpose:** Sends an SMS or chat message to a recipient without waiting for a reply; use Send & Wait For Reply when a response is needed.
- **Key properties:** `body` — message text (supports Liquid variables); `from` — sending phone number or Messaging Service SID (default: flow channel address); `to` — recipient address (default: contact channel address); `media_url` — optional media attachment URL; `messaging_service_sid` — Messaging Service for link shortening; `chat_service_instance_sid` — Programmable Chat Service SID; `chat_channel_sid` — Chat Channel SID; `attributes` — JSON metadata for chat messages
- **Content templates:** to send a pre-approved Content API template (required for WhatsApp and other template-gated channels) instead of a freeform `body`, set `message_type: "content_template"` (the freeform default is `message_type: "custom"`) and supply:
  - `content_sid` — the Content template SID (`HXxxxx`), or a Liquid expression resolving to one
  - `content_variables` — array of `{key, value}` pairs filling the template's numbered placeholders, where `key` is the placeholder token (`"{{1}}"`, `"{{2}}"`, …) and `value` is the substituted text (Liquid allowed), e.g. `[{"key": "{{1}}", "value": "{{flow.data.first_name}}"}]`
- **Transition events:** `sent`, `failed`
- **Outputs** (`{{widgets.<name>.<key>}}`, all under `outbound.`, events `sent`/`failed`): SMS/MMS — `outbound.Sid`, `outbound.Status`, `outbound.Body`, `outbound.From`, `outbound.To`, `outbound.Direction`, `outbound.NumMedia`, `outbound.NumSegments`, `outbound.Price`, `outbound.PriceUnit`, `outbound.ErrorCode`, `outbound.ErrorMessage`, `outbound.MessagingServiceSid`, `outbound.AccountSid`, and more. Conversation channel — `outbound.Sid`, `outbound.ConversationSid`, `outbound.Body`, `outbound.Author`, `outbound.Index`, `outbound.Attributes`, `outbound.Media`, `outbound.ParticipantSid`, and more. A true API send failure still carries the full `outbound.*` (with `ErrorCode`/`ErrorMessage`); early-exit failures emit an empty map.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/send-message

### Send & Wait For Reply
- **Type:** `send-and-wait-for-reply`
- **Purpose:** Sends an outgoing SMS or chat message and pauses the flow until the recipient replies or the timeout expires.
- **Key properties:** `body` — outgoing message text; `from` — sending phone number or Messaging Service SID; `to` — recipient address; `timeout` — seconds to wait for a reply (default: 3600); `media_url` — optional media attachment URL; `messaging_service_sid` — Messaging Service SID; `chat_service_instance_sid` — Programmable Chat Service SID; `chat_channel_sid` — Chat Channel SID; `attributes` — JSON metadata for chat messages
- **Transition events:** `incomingMessage`, `timeout`, `deliveryFailure`
- **Outputs** (`{{widgets.<name>.<key>}}`): `outbound.*` — the sent message (same field sets as Send Message); present on all transitions once sent. `inbound.*` — the reply; populated only on `incomingMessage` (verbatim passthrough of the inbound webhook; notable keys `inbound.Body`, `inbound.From`, `inbound.To`, `inbound.MessageSid`, `inbound.NumMedia`, `inbound.MediaUrl0…`). `timeout`/`deliveryFailure` expose only `outbound.*`. Replies over a Conversation channel route to `incomingConversationMessage` and land under `trigger.conversation`, not `inbound.*`.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/send-wait-reply

### Run Function
- **Type:** `run-function`
- **Purpose:** Invokes a Twilio Serverless Function from within a flow, allowing custom business logic to run without leaving Studio.
- **Key properties:** `url` — URL of the Twilio Function to invoke; `service_sid` — Functions Service SID; `environment_sid` — Environment SID; `parameters` — key/value pairs passed to the function via its event object (supports Liquid variables)
- **Transition events:** `success` (2xx/3xx response within 10 s), `fail` (4xx/5xx, timeout, or error)
- **Outputs** (`{{widgets.<name>.<key>}}`, events `success`/`fail`): `body` — full response body string; `content_type` — response Content-Type; `status_code` — HTTP status code; `parsed` — body parsed to an object, present **only** when `content_type` is `application/json` (nested via `parsed.<key>`); `headers` — contains only `x-twilio-function-concurrency` when present (access as `headers["x-twilio-function-concurrency"]`).
- **Docs:** https://www.twilio.com/docs/studio/widget-library/run-function

### HTTP Request
- **Type:** `make-http-request`
- **Purpose:** Makes an outbound HTTP request to an external API or service, making the response available to downstream widgets via Liquid variables.
- **Key properties:** `method` — `GET` or `POST` (default: `GET`); `url` — request endpoint; `content_type` — `application/x-www-form-urlencoded` or `application/json`; `body` — request body text or Liquid template; `parameters` — key/value pairs appended as query parameters; `send_twilio_credentials` — include Twilio auth credentials
- **Transition events:** `success` (2xx within 10 s), `failed` (4xx/5xx, timeout, or network error)
- **Outputs** (`{{widgets.<name>.<key>}}`, events `success`/`failed`): `body` — full response body string; `content_type` — response Content-Type; `status_code` — HTTP status code; `parsed` — body parsed to an object, present **only** when `content_type` is exactly `application/json` (nested via `parsed.<key>`). Both events carry the same keys; network/timeout/too-large failures emit an empty map. There is no `headers` key.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/http-request

### TwiML Redirect
- **Type:** `add-twiml-redirect`
- **Purpose:** Redirects a call or message to external TwiML hosted outside Studio, with optional return of control back to the flow.
- **Key properties:** `url` — TwiML endpoint URL; `method` — `POST` or `GET` (default: `POST`); `timeout` — seconds to wait for control to return (0–14400; 0 means do not wait), as a **string** (FlowValidate rejects a JSON integer)
- **Transition events:** `return` (TwiML includes `FlowEvent=return`), `timeout` (timer expires), `fail` (URL invalid or unparseable)
- **Outputs** (`{{widgets.<name>.<key>}}`): open-ended pass-through — any parameter the external TwiML posts back on the `return` URL becomes a key (on `return`). No fixed key set. `timeout` and `failed` expose no variables.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/twiml-redirect

### Run Subflow
- **Type:** `run-subflow`
- **Purpose:** Invokes another Studio Flow as a reusable subflow, passing parameters in and receiving variables back when it completes.
- **Key properties:** `flow_sid` — SID of the Studio Flow to run as a subflow; `flow_revision` — `LatestPublished` or `LatestDraft`; `parameters` — key/value pairs passed into the subflow (optionally parsed as JSON)
- **Transition events:** `completed`, `failed`
- **Outputs** (`{{widgets.<name>.<key>}}`, event `completed`): each variable set inside the subflow surfaces **flat** under the widget — subflow sets `foo` → parent reads `{{widgets.<name>.foo}}`. `failed` exposes no variables. The subflow also lifts its `add_ons`/`channel` to the parent's `flow.add_ons`/`flow.channel`. Inside the subflow, parent data is available as `trigger.parent.*` (`parameters`, `step_sid`, `execution_sid`, `flow_sid`, `trigger_type`).
- **Docs:** https://www.twilio.com/docs/studio/widget-library/run-subflow

### Send To Flex
- **Type:** `send-to-flex`
- **Purpose:** Routes an incoming call, message, or conversation to Twilio Flex by creating a TaskRouter task; voice calls are enqueued with hold music while awaiting an agent.
- **Key properties:** `workflow` — TaskRouter Workflow SID (required); `channel` — Task Channel (required); `priority` — task priority as a **string** (FlowValidate rejects a JSON integer); `timeout` — seconds task remains active (default: 3600); `attributes` — JSON task attributes (max 1024 chars); `url_method` — HTTP method for hold music requests; `hold_music_url` — custom hold music TwiML URL
- **Transition events:** `callComplete`, `failedToEnqueue`, `callFailure`, `composerStarted`
- **Outputs** (`{{widgets.<name>.<key>}}`, event `created` — shape depends on channel): Conversations/Interactions — `sid`, `channel`, `routing`, `url`, `FlexContextSid` (when present); also writes `flow.channel.status="handed off"` for active outbound conversations. Chat (TaskRouter task) — `sid`, `account_sid`, `age`, `assignment_status`, `attributes`, `addons`, `date_created`, `date_updated`, `priority`, `reason`, `task_queue_sid`, `task_queue_friendly_name`, `task_channel_sid`, `task_channel_unique_name`, `timeout`, `workflow_sid`, `workflow_friendly_name`, `workspace_sid`, `url`, `links`. Voice — the `<Enqueue>` action callback params (`QueueResult`, `QueueSid`, `QueueTime`, `CallStatus`), plus `FlexContextSid` when present.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/send-flex

### Start Conversation
- **Type:** `start-conversation`
- **Purpose:** Creates a new Conversation and adds a participant, enabling conversation-channel executions.
- **Note:** Usable only in flows triggered by the REST API. Requires `conversation_service_sid` and `participants`.
- **Key properties:** `conversation_service_sid` — Conversations Service the Conversation belongs to (required); `participants` — array of participant objects, each requiring a `type` (required) — for `type: "chat"` add `identity`; for `type: "sms"` add `messaging_binding_address` and `messaging_binding_proxy_address`; `friendly_name` — display name for the Conversation; `attributes` — JSON metadata attached to the Conversation; `chat_service_sid` — Conversations (Chat) Service the Conversation belongs to; `messaging_service_sid` — Messaging Service for SMS/MMS-bound participants; `timers` — auto-close/inactive timers
- **Transition events:** `created`, `failed`
- **Outputs** (`{{widgets.<name>.<key>}}`, event `created`; non-null fields only): `conversation.{sid, friendly_name, unique_name, state, attributes, bindings, chat_service_sid, messaging_service_sid, date_created, date_updated, timers, links, url}`; `conversation.participants[N].{sid, identity, attributes, role_sid, conversation_sid, date_created, date_updated, last_read_message_index, last_read_timestamp, messaging_binding.address, messaging_binding.proxy_address, url}` (this widget creates one participant, so `participants[0]`); `webhook_sid` — onMessageAdded webhook SID; `close_channel_on_execution_end` — boolean. `failed` exposes no variables. The same data is also merged into `flow.channel.*`.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/start-conversation

### Resume Conversation
- **Type:** `resume-conversation`
- **Purpose:** Resumes an existing Conversation, reattaching the flow to an active conversation and its participants.
- **Note:** Usable only in flows triggered by the REST API. Requires `conversation_service_sid`.
- **Key properties:** `conversation_service_sid` — Conversations Service the Conversation belongs to (required); `conversation_sid` — SID of the existing Conversation to resume; `webhook_filters` — which Conversation events resume the flow; optional `attributes` updates. Use this when an execution needs to re-enter a Conversation that Start Conversation (or another flow) created earlier.
- **Transition events:** `resumed`, `failed`
- **Outputs** (`{{widgets.<name>.<key>}}`, event `resumed`): same shape as Start Conversation, but `conversation.participants[N]` lists all existing participants (up to 1000). `failed` exposes no variables. Also merged into `flow.channel.*`.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/resume-conversation

### Run ConversationRelay
- **Type:** `run-conversation-relay`
- **Purpose:** Connects a Voice call to a ConversationRelay session for real-time AI-driven voice interactions, with handoff support.
- **Key properties:** `url` — WebSocket URL (`wss://`) of the application driving the conversation; `welcome_greeting` — text spoken to the caller when the session starts; `tts_provider` / `voice` / `language` — text-to-speech settings; `transcription_provider` / `speech_model` — speech-to-text (STT) settings; `dtmf_detection` — capture keypad input; `interruptible` — allow the caller to interrupt playback, as the **string** `"true"`/`"false"` (FlowValidate rejects a JSON boolean); `parameters` — additional key/value pairs sent to the session on connect
- **Transition events:** `success` (end of session), `failed`
- **Outputs** (`{{widgets.<name>.<key>}}`): `HandoffData` (handoff payload), plus the raw end-session webhook params (`SessionId`, `SessionStatus`, `SessionDuration`, …). `failed` exposes no curated variables.
- **Docs:** https://www.twilio.com/docs/studio/widget-library/run-conversationrelay
