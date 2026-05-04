---
name: twilio-conversation-orchestrator
description: >
  Configure automatic conversation capture and routing with Twilio Conversation
  Orchestrator. Covers Configuration creation, channel capture rules, grouping
  types, status timeouts, Memory Store linkage, Intelligence linkage, and
  conversation lifecycle. Use this skill to automatically capture SMS, voice,
  WhatsApp, RCS, and web chat traffic into unified conversations without
  manually creating conversations or participants.
---

# Conversation Orchestrator

Decision-making guide for Twilio's Conversation Orchestrator (Conversations v2) — automatic conversation capture and routing across Voice, SMS, WhatsApp, RCS, and web chat. Covers Configurations, capture rules, grouping types, channel settings, status timeouts, and linkage to Conversation Memory and Conversation Intelligence.

Evidence date: 2026-04-28. Account prefix: AC...

> **GA** — Conversation Orchestrator v2 is generally available.

## Use Cases

Conversation Orchestrator powers **automatic conversation capture** — replacing manual conversation creation and participant management with declarative rules that capture traffic as it flows through Twilio.

### Unified Customer Context

Capture all channels (voice, SMS, WhatsApp, RCS, web chat) into a single conversation thread per customer. Conversation Memory resolves identity across channels and maintains persistent context. **Start here** — this is the most common pattern.

- **Grouping**: `GROUP_BY_PROFILE`
- **Channels**: SMS + VOICE + WHATSAPP + RCS + CHAT
- **Linkage**: Memory Store (identity resolution) + Intelligence (analysis)

### Channel-Isolated Analytics

Keep voice transcripts separate from SMS threads for per-channel analysis. Intelligence operators run independently on each channel's conversation.

- **Grouping**: `GROUP_BY_PARTICIPANT_ADDRESSES_AND_CHANNEL_TYPE`
- **Channels**: SMS + VOICE (separate conversations)
- **Linkage**: Intelligence (per-channel operators)

### Agent Connect Integration

Capture conversations for AI-to-human escalation via Agent Connect (TAC SDK). Uses address-pair grouping required by the SDK.

- **Grouping**: `GROUP_BY_PARTICIPANT_ADDRESSES`
- **Channels**: SMS or VOICE
- **Linkage**: Memory Store + Intelligence + Agent Connect

### Post-Conversation Memory Extraction

Automatically extract observations from conversations into Conversation Memory. Opt-in — configure once, every conversation feeds the memory loop.

- **Config**: `memoryExtractionEnabled: true`
- **Trigger**: INACTIVE and/or CLOSED lifecycle transitions (configurable)
- **Result**: Observations and summaries written to linked Memory Store profiles

## How It Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. Inbound/outbound traffic arrives (SMS, Voice, WhatsApp, RCS, Web Chat)  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. Capture rules match on phone number patterns                           │
│     - from/to with wildcards (e.g., from: *, to: +15551234567)             │
│     - Per-channel rules (SMS, VOICE, WHATSAPP, RCS, CHAT)                  │
│     - Metadata filters (callType for CLIENT/SIP)                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. Conversation auto-created (or existing one matched via grouping)        │
│     - GROUP_BY_PROFILE: merge by Memory Profile identity                   │
│     - GROUP_BY_PARTICIPANT_ADDRESSES: merge by address pair                │
│     - GROUP_BY_..._AND_CHANNEL_TYPE: separate per channel                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. Linked services activate                                               │
│     - Memory Store: identity resolution, profile auto-creation             │
│     - Intelligence: operators fire per Communication or at close           │
│     - Status timeouts: ACTIVE → INACTIVE → CLOSED lifecycle                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. On conversation close                                                  │
│     - Memory extraction: observations written to Memory Store              │
│     - CONVERSATION_END Intelligence operators fire (Summary, etc.)         │
│     - Status callbacks delivered (if configured)                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key insight**: Conversation Orchestrator replaces manual conversation creation (v1 pattern) with declarative capture rules. You configure once — traffic is captured automatically as it flows.

## Scope

### CAN

- Automatically capture SMS, voice, WhatsApp, RCS, and web chat into Conversations without manual creation [live-tested]
- Merge multiple channels into one Conversation thread via `GROUP_BY_PROFILE` [live-tested]
- Link Memory Store for automatic identity resolution and observation extraction [live-tested]
- Link multiple Intelligence Configurations for real-time and post-conversation analysis [live-tested]
- Bridge Conversations Classic (v1) Services for browser SDK chat via `conversationsV1Bridge` [live-tested]
- Configure per-channel capture rules with wildcard matching [live-tested]
- Set independent timeout policies per channel [live-tested]
- Add `statusCallbacks` for webhook notifications on conversation state changes [live-tested]
- Pass `conversationConfiguration` in ConversationRelay TwiML to create a conversation (Active TwiML mode) [live-tested]
- Pass `conversationId` in `<Transcription>` TwiML to attach voice to an existing API-created conversation [live-tested]
- Close conversations explicitly via PATCH to trigger Memory extraction and CONVERSATION_END operators [live-tested]
- List and filter Conversations by status, channel, and date range [live-tested]
- Read Communications (messages + voice utterances) within a Conversation [live-tested]
- Authenticate with Account SID/Auth Token or API Key/Secret [live-tested]

### CANNOT

- **Cannot update Configurations with PATCH** — PUT only, full replacement. Omitting fields deletes them. Always re-fetch before updating. [live-tested]
- **Cannot exceed 10 Configurations per account** — Hard limit at GA. Each config supports up to 100 capture rules per channel. Delete unused configs to make room. Plan Configuration topology for large phone number portfolios accordingly (e.g., 101 numbers for one channel requires 2 configs). [live-tested]
- **Cannot change grouping type after creation** — `conversationGroupingType` is immutable on a Configuration. Create a new config if you need a different grouping. [live-tested]
- **Cannot capture CLIENT or SIP voice calls without explicit callType metadata** — PSTN is captured by default. Browser (Client SDK) and SIP calls require `metadata.callType` in capture rules. [live-tested]
- **Cannot combine passive VOICE capture rules with ConversationRelay** — Passive voice capture uses RTT under the hood. If ConversationRelay is also active on the same call, you pay double STT billing. Use Active TwiML mode (`conversationConfiguration` param) for CRelay calls, and passive capture rules only for human-agent scenarios without CRelay. [live-tested]
- **Cannot detect failed Memory linkage** — If `memoryStoreId` points to a deleted or invalid store, capture still works but identity resolution and extraction silently fail. See `twilio-sierra-debugging`.
- **Cannot filter Intelligence operators by participant type** — Operators fire on ALL Communications (customer and agent). Use the operator prompt to specify which participant to analyze. [live-tested]
- **Cannot extract Memory observations mid-conversation (ACTIVE state)** — Extraction is opt-in and can fire on INACTIVE and/or CLOSED lifecycle transitions, but not while the conversation is ACTIVE. For real-time Memory writes during an active conversation, post Observations directly via `twilio-customer-memory`.
- **Cannot have conversations pick up config changes retroactively** — Conversations pin the Configuration version at creation time. Close existing conversations to apply updated rules. [live-tested]
- **Cannot use the POST response to get the Configuration ID** — Creation may return 202 with no ID. Poll the list endpoint to find the new config by `displayName`. [live-tested]
- **No standalone operation** — Requires a Memory Store (even if you don't use Memory features). `memoryStoreId` is mandatory when creating a Configuration. [live-tested]
- **JSON-only API** — All v2 endpoints require `Content-Type: application/json`. Form-encoded bodies are rejected. [live-tested]

## Quick Decision

| Need | Use | Why |
|------|-----|-----|
| Capture all messaging + voice into unified customer threads | Configuration with `GROUP_BY_PROFILE` + Memory Store | Cross-channel identity resolution merges by customer profile |
| Keep voice and SMS conversations separate | Configuration with `GROUP_BY_PARTICIPANT_ADDRESSES_AND_CHANNEL_TYPE` | Channel-isolated threads for per-channel analytics |
| Agent Connect (TAC SDK) integration | Configuration with `GROUP_BY_PARTICIPANT_ADDRESSES` | Required by TAC SDK |
| Auto-extract customer observations from conversations | Set `memoryExtractionEnabled: true` on Configuration | Triggers on conversation close, writes to linked Memory Store |
| Analyze conversations with Intelligence operators | Link `intelligenceConfigurationIds` on Configuration | Operators fire per Communication or at conversation close |
| AI voice agent (ConversationRelay) | Use `conversationConfiguration` param on `<ConversationRelay>` (Active TwiML) — do NOT add passive VOICE capture rules | Avoids double STT billing (CRelay + RTT) |
| Attach voice to existing conversation | Use `conversationId` param on `<Transcription>` | For API-created conversations that need voice transcription added |
| Capture browser voice calls (Client SDK) | Add VOICE capture rule with `metadata.callType: "CLIENT"` | PSTN-only by default; CLIENT needs explicit rule |
| Capture web chat (browser SDK) | Set `conversationsV1Bridge.serviceId` on Configuration | Web chat flows through a Conversations (v1) Service bridged into Orchestrator |

## Decision Frameworks

### Grouping Type Selection

| Grouping Type | Behavior | When to Use | Constraint |
|---------------|----------|-------------|------------|
| `GROUP_BY_PROFILE` | All channels for same Memory Profile merge into one Conversation | Unified cross-channel threads (recommended default) | Requires Memory Store with identity traits |
| `GROUP_BY_PARTICIPANT_ADDRESSES` | Groups by address pair (e.g., +1A ↔ +1B) regardless of channel | Agent Connect (TAC SDK) integration | Simpler identity model, no profile resolution |
| `GROUP_BY_PARTICIPANT_ADDRESSES_AND_CHANNEL_TYPE` | Separate Conversations per channel even for same participants | Channel-isolated analytics or compliance | Most granular, no cross-channel merging |

**Immutable after creation.** Choose before creating the Configuration. To change grouping, create a new Configuration.

### Channel Configuration Matrix

| Channel | Capture Pattern | Timeout Recommendation | Notes |
|---------|----------------|----------------------|-------|
| SMS | Bidirectional rules (from: phone, to: * AND from: *, to: phone) | `inactive: 10, closed: 60` | Both inbound + outbound need explicit rules |
| VOICE (PSTN) | Inbound rule (from: *, to: phone) | `null` (manage by call end) | Captured by default with basic rule |
| VOICE (CLIENT) | Separate rule with `metadata.callType: "CLIENT"` | `null` | Opt-in only — not captured by default VOICE rules |
| VOICE (SIP) | Separate rule with `metadata.callType: "SIP"` | `null` | Opt-in only |
| WHATSAPP | Same pattern as SMS (from/to with WhatsApp-enabled number) | `inactive: 10, closed: 60` | Uses WhatsApp-enabled sender |
| RCS | Same pattern as SMS (from/to with RCS-enabled number) | `inactive: 10, closed: 60` | Uses RCS-enabled sender |
| WEB CHAT | Rule with `metadata.chatService: "ISxxxxxxxx"` | `inactive: 15, closed: 60` | Bridged via Conversations (v1) Service; requires `conversationsV1Bridge` |

## Integration Patterns

Code samples use raw `fetch()` for clarity. All Conversation Orchestrator APIs use Basic Auth — see `twilio-iam-auth-setup`.

### Authentication Helper

```javascript
const CONVERSATIONS_V2_BASE = 'https://conversations.twilio.com/v2';

function getAuthHeaders() {
  const credentials = Buffer.from(
    `${process.env.TWILIO_ACCOUNT_SID}:${process.env.TWILIO_AUTH_TOKEN}`
  ).toString('base64');
  return {
    'Authorization': `Basic ${credentials}`,
    'Content-Type': 'application/json',
  };
}
```

### Create a Configuration

```javascript
const configResponse = await fetch(
  `${CONVERSATIONS_V2_BASE}/ControlPlane/Configurations`,
  {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      displayName: 'my-app-config',
      description: 'Production conversation config',
      conversationGroupingType: 'GROUP_BY_PROFILE',
      memoryStoreId: 'mem_store_...', // Required — create via Memory API first
      memoryExtractionEnabled: true,
      channelSettings: {
        SMS: {
          captureRules: [
            { from: '+15551234567', to: '*', metadata: {} },
            { from: '*', to: '+15551234567', metadata: {} },
          ],
          statusTimeouts: { inactive: 10, closed: 60 },
        },
        VOICE: {
          captureRules: [
            { from: '*', to: '+15551234567', metadata: {} },
          ],
        },
      },
    }),
  }
);
// May return 202 without config ID — poll GET /ControlPlane/Configurations to find by displayName
```

```python
import os, requests

account_sid = os.environ["TWILIO_ACCOUNT_SID"]
auth_token = os.environ["TWILIO_AUTH_TOKEN"]
twilio_phone = os.environ["TWILIO_PHONE_NUMBER"]

config = requests.post(
    "https://conversations.twilio.com/v2/ControlPlane/Configurations",
    auth=(account_sid, auth_token),
    json={
        "displayName": "my-app-config",
        "description": "Production conversation config",
        "conversationGroupingType": "GROUP_BY_PROFILE",
        "memoryStoreId": "mem_store_...",
        "memoryExtractionEnabled": True,
        "channelSettings": {
            "SMS": {
                "captureRules": [
                    {"from": twilio_phone, "to": "*", "metadata": {}},
                    {"from": "*", "to": twilio_phone, "metadata": {}}
                ],
                "statusTimeouts": {"inactive": 10, "closed": 60}
            },
            "VOICE": {
                "captureRules": [
                    {"from": "*", "to": twilio_phone, "metadata": {}}
                ]
            }
        }
    }
).json()
```

### Update a Configuration (PUT — Full Replacement)

```javascript
// Step 1: Fetch current config (ALWAYS re-fetch before updating)
const current = await fetch(
  `${CONVERSATIONS_V2_BASE}/ControlPlane/Configurations/${configId}`,
  { headers: getAuthHeaders() }
).then(r => r.json());

// Step 2: Modify the field you need
current.channelSettings.VOICE.captureRules.push(
  { from: '*', to: '+15551234567', metadata: { callType: 'CLIENT' } }
);

// Step 3: PUT the complete object back
await fetch(
  `${CONVERSATIONS_V2_BASE}/ControlPlane/Configurations/${configId}`,
  {
    method: 'PUT',
    headers: getAuthHeaders(),
    body: JSON.stringify(current),
  }
);
```

```python
# Fetch current config
current = requests.get(
    f"https://conversations.twilio.com/v2/ControlPlane/Configurations/{config_id}",
    auth=(account_sid, auth_token)
).json()

# Modify and PUT the whole thing back
current["channelSettings"]["VOICE"]["captureRules"].append(
    {"from": "*", "to": twilio_phone, "metadata": {"callType": "CLIENT"}}
)

requests.put(
    f"https://conversations.twilio.com/v2/ControlPlane/Configurations/{config_id}",
    auth=(account_sid, auth_token),
    json=current
)
```

### Link Intelligence Configuration

```javascript
// Fetch current config, add Intelligence, PUT back
const current = await fetch(
  `${CONVERSATIONS_V2_BASE}/ControlPlane/Configurations/${configId}`,
  { headers: getAuthHeaders() }
).then(r => r.json());

current.intelligenceConfigurationIds = [intelligenceConfigId];

await fetch(
  `${CONVERSATIONS_V2_BASE}/ControlPlane/Configurations/${configId}`,
  {
    method: 'PUT',
    headers: getAuthHeaders(),
    body: JSON.stringify(current),
  }
);
```

### Read Conversations and Communications

```javascript
// List active conversations
const conversations = await fetch(
  `${CONVERSATIONS_V2_BASE}/Conversations?Status=ACTIVE&PageSize=10`,
  { headers: getAuthHeaders() }
).then(r => r.json());

for (const conv of conversations.conversations ?? []) {
  // List communications (messages + voice utterances)
  const comms = await fetch(
    `${CONVERSATIONS_V2_BASE}/Conversations/${conv.id}/Communications`,
    { headers: getAuthHeaders() }
  ).then(r => r.json());

  for (const comm of comms.communications ?? []) {
    console.log(`[${comm.channel}] ${comm.body}`);
  }
}
```

```python
conversations = requests.get(
    "https://conversations.twilio.com/v2/Conversations",
    auth=(account_sid, auth_token),
    params={"Status": "ACTIVE", "PageSize": 10}
).json()

for conv in conversations.get("conversations", []):
    conv_id = conv["id"]
    comms = requests.get(
        f"https://conversations.twilio.com/v2/Conversations/{conv_id}/Communications",
        auth=(account_sid, auth_token)
    ).json()

    for comm in comms.get("communications", []):
        print(f"  [{comm['channel']}] {comm['body']}")
```

### Close a Conversation

Closing triggers Memory extraction (if enabled) and CONVERSATION_END Intelligence operators.

```javascript
await fetch(
  `${CONVERSATIONS_V2_BASE}/Conversations/${convId}`,
  {
    method: 'PATCH',
    headers: getAuthHeaders(),
    body: JSON.stringify({ status: 'CLOSED' }),
  }
);
```

```python
requests.patch(
    f"https://conversations.twilio.com/v2/Conversations/{conv_id}",
    auth=(account_sid, auth_token),
    json={"status": "CLOSED"}
)
```

### Voice Integration Patterns

**Active TwiML (recommended for AI voice agents):** Pass `conversationConfiguration` on `<ConversationRelay>` to create a new conversation. Do NOT add passive VOICE `captureRules` — this avoids double STT billing (CRelay + RTT).

```xml
<Response>
  <Connect>
    <ConversationRelay
      url="wss://your-relay/voice"
      conversationConfiguration="CONFIG_ID_HERE"
      ttsProvider="ElevenLabs"
      voice="your-voice-id"
    />
  </Connect>
</Response>
```

Still define VOICE in `channelSettings` for lifecycle/timeouts — just omit `captureRules`:
```json
{
  "channelSettings": {
    "VOICE": {
      "statusTimeouts": null
    }
  }
}
```

**Attach voice to an existing conversation (Real-Time Transcription):** Use `<Transcription>` with `conversationId` to add a voice call's transcription to a conversation you created via API:

```xml
<Response>
  <Start>
    <Transcription conversationId="CONVERSATION_ID"/>
  </Start>
  <Say>Welcome to support. How can I help you today?</Say>
</Response>
```

**Passive voice capture (human agent calls):** Use VOICE `captureRules` to automatically capture calls without TwiML changes. Appropriate for human agent scenarios where ConversationRelay is not used:
```json
{
  "VOICE": {
    "captureRules": [
      { "from": "*", "to": "+15551234567", "metadata": {} }
    ]
  }
}
```

> **Warning:** Do NOT combine passive VOICE capture rules with ConversationRelay on the same number. Passive voice capture uses Real-Time Transcription (RTT) under the hood — if CRelay is also active, you pay for both STT engines on the same call.

## Gotchas

### Setup

1. **Memory Store is required.** You cannot create a Configuration without a `memoryStoreId`. Create the Memory Store first via `twilio-customer-memory`. [live-tested]

2. **JSON-only API.** All v2 endpoints require `Content-Type: application/json`. Form-encoded bodies are rejected. This matches Intelligence v3 but differs from most Twilio APIs. [live-tested]

3. **Async creation.** POST to `/ControlPlane/Configurations` may return 202 without a config ID. Poll `GET /ControlPlane/Configurations` and match by `displayName` to find the new config. [live-tested]

### Configuration

4. **PUT replaces everything.** The most common bug: fetching a config, modifying one field, PUTting back — but forgetting to include `channelSettings` or `memoryStoreId`. The API accepts the PUT and silently removes the omitted fields. Always re-fetch, modify, PUT. [live-tested]

5. **Grouping type is immutable.** `conversationGroupingType` cannot be changed after creation. To switch grouping, create a new Configuration and close conversations on the old one. [live-tested]

6. **10 Configuration limit per account.** Hard limit at GA (up to 100 capture rules per channel per config). Delete unused Configurations to make room. For customers with large phone number portfolios, partition numbers across multiple Configurations. [live-tested]

7. **CLIENT voice capture is opt-in.** Browser-originated calls via the Twilio Client SDK are not captured by default VOICE rules. You need a separate capture rule with `"metadata": {"callType": "CLIENT"}`. SIP calls similarly need `{"callType": "SIP"}`. PSTN is the only type captured by default. [live-tested]

### Runtime

8. **Timeout precedence across channels.** If a customer is on a voice call and sends an SMS, both channels are active in the same Conversation (with `GROUP_BY_PROFILE`). When the voice call ends, the SMS channel's timeout still governs — the Conversation won't close until the SMS timeout expires. Channel close events are proposals, not commands. [live-tested]

9. **Config versioning pins at creation.** Intelligence rules and capture rules are pinned to the Configuration version at conversation creation time. Upgrading Intelligence (adding operators, changing rules) doesn't affect existing conversations. Close active conversations to pick up the new version. [live-tested]

10. **ConversationRelay TTS fragmentation.** ConversationRelay writes one Communication per TTS fragment, not per complete utterance. A single agent response may produce 3-5 Communications. Intelligence operators fire per Communication, so operator cost scales with fragment count. [live-tested]

11. **Wildcard VOICE rules cause double-capture.** A rule `{"from": "*", "to": "*"}` for VOICE will match both PSTN and CLIENT calls, potentially creating duplicate Conversations. Use specific `to` addresses. [live-tested]

12. **Active TwiML voice (ConversationRelay) — do NOT add passive VOICE capture rules.** When using `<ConversationRelay conversationConfiguration="CONFIG_ID">`, the param creates the conversation (Active TwiML mode). Do NOT also add VOICE `captureRules` — passive voice capture uses Real-Time Transcription (RTT) under the hood, meaning you'd pay for both ConversationRelay STT AND RTT STT on the same call (double billing). However, you still need VOICE `channelSettings` defined (without `captureRules`) for lifecycle timeouts and grouping. [live-tested]

13. **`conversationConfiguration` (no "Id" suffix) is the correct TwiML attribute name.** The attribute on `<ConversationRelay>` is `conversationConfiguration`, NOT `conversationConfigurationId`. The incorrect name is silently ignored (unrecognized TwiML attributes produce no error), resulting in no conversation being created. [live-tested]

### Observability

14. **Silent Memory linkage failure.** If `memoryStoreId` points to a deleted or invalid store, capture still works but identity resolution and extraction silently fail. No error is returned. See `twilio-sierra-debugging`. [live-tested]

15. **No participant type filtering for Intelligence.** Operators fire on ALL Communications — customer messages AND agent responses. There is no config-level filter. Use the operator prompt to specify which participant to analyze. [live-tested]

16. **Memory extraction is opt-in and fires on INACTIVE and/or CLOSED.** Extraction does not run automatically — it must be enabled. It can be configured to fire on the INACTIVE transition, the CLOSED transition, or both. It does NOT fire while a conversation is ACTIVE. For mid-conversation Memory writes, post directly to the Observations endpoint via `twilio-customer-memory`. [live-tested]

## Related Resources

- [Conversation Intelligence Skill](/.claude/skills/twilio-conversation-intelligence/SKILL.md) — Intelligence Configuration, Language Operators, real-time and post-conversation analysis
- [Customer Memory Skill](/.claude/skills/twilio-customer-memory/SKILL.md) — Memory Store, profiles, traits, observations, Recall
- [ConversationRelay Skill](/.claude/skills/twilio-voice-conversation-relay/SKILL.md) — Voice AI agent setup with WebSocket streaming
- [Agent Connect Skill](/.claude/skills/twilio-agent-connect/SKILL.md) — AI-to-human escalation via TAC SDK
- [Debugging Skill](/.claude/skills/twilio-sierra-debugging/SKILL.md) — Linkage chain verification, silent failures
