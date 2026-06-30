---
name: twilio-conversation-memory
description: >
  Store and retrieve conversation context using Twilio Conversation Memory.
  Covers Memory Store provisioning, profile management, traits, observations,
  conversation summaries, and semantic Recall. Use this skill to give AI agents
  or human agents persistent memory of conversations across sessions
  and channels.
---

## Overview

Conversation Memory gives agents persistent, cross-session context about customers. A **Memory Store** holds profiles, observations, summaries, and traits. Data enters two ways:

1. **Automatic:** Link a Conversation Orchestrator config to a store. Profiles are auto-created per caller. At conversation close, observations and summaries are extracted from the transcript and saved. Conflicting data is automatically reconciled.

2. **Direct API:** POST observations, summaries, or traits yourself for any use case.

Data comes out via **Recall** — hybrid semantic + lexical search that surfaces the most relevant profile data. For voice agents where latency is critical, use `GET /Observations` directly instead.

**Base URL:** `https://memory.twilio.com`

**Authentication:** HTTP Basic — `Authorization: Basic {base64(accountSid:authToken)}`

**Rules for agents:**
- Always poll `statusUrl` after any 202 response — all writes are async
- Always create a trait group before writing traits of that group
- Always use E.164 format for phone numbers (`+15558675310`)
- Always verify store status is `ACTIVE` before operating on it
- Never use `/v1/Services/` paths — correct paths are `/v1/ControlPlane/Stores` (config) and `/v1/Stores/{storeId}/...` (data)
- Never exceed 10 observations per batch or 4096 characters per observation
- Never assume writes are immediately readable — indexing is async
- Never send PCI or HIPAA data — not yet compliant

---

## Prerequisites

- Twilio account with Account SID and Auth Token
  — A credit card must be added to the account
  — See `twilio-account-setup` for initial setup
  — See `twilio-iam-auth-setup` for credential best practices
- Environment variables:
  - `TWILIO_ACCOUNT_SID`
  - `TWILIO_AUTH_TOKEN`
- SDK: `pip install twilio` / `npm install twilio`
- For automatic extraction: a Conversation Orchestrator configuration — see `twilio-conversation-orchestrator`

---

## Quickstart

**Step 1 — Create a Memory Store**

**Python**
```python
import os, requests
from requests.auth import HTTPBasicAuth

account_sid = os.environ["TWILIO_ACCOUNT_SID"]
auth_token = os.environ["TWILIO_AUTH_TOKEN"]
base_url = "https://memory.twilio.com"
auth = HTTPBasicAuth(account_sid, auth_token)

store = requests.post(
    f"{base_url}/v1/ControlPlane/Stores",
    auth=auth,
    json={"displayName": "my-store", "description": "Customer memory"}
).json()

print(store["id"])  # mem_store_xxx — save this
# Poll store["statusUrl"] until status is ACTIVE
```

**Node.js**
```node
const axios = require("axios");

const accountSid = process.env.TWILIO_ACCOUNT_SID;
const authToken = process.env.TWILIO_AUTH_TOKEN;
const baseUrl = "https://memory.twilio.com";
const auth = { username: accountSid, password: authToken };

const { data: store } = await axios.post(
    `${baseUrl}/v1/ControlPlane/Stores`,
    { displayName: "my-store", description: "Customer memory" },
    { auth }
);

console.log(store.id);  // mem_store_xxx
// Poll store.statusUrl until status is ACTIVE
```

**Step 2 — Create a profile with identity resolution**

**Python**
```python
profile = requests.post(
    f"{base_url}/v1/Stores/{store['id']}/Profiles",
    auth=auth,
    json={"traits": {"contact": {"phone": "+15558675310", "name": "Alex"}}}
).json()

profile_id = profile["id"]  # mem_profile_xxx
```

**Node.js**
```node
const { data: profile } = await axios.post(
    `${baseUrl}/v1/Stores/${store.id}/Profiles`,
    { traits: { contact: { phone: "+15558675310", name: "Alex" } } },
    { auth }
);

const profileId = profile.id;  // mem_profile_xxx
```

**Step 3 — Post observations (quick test)**

Post an observation directly to quickly test Recall in step 4. In production, the preferred path is connecting your Memory Store to a Conversation Orchestrator configuration — observations are then automatically extracted from conversations without manual POSTs. See `twilio-conversation-orchestrator`.

**Python**
```python
requests.post(
    f"{base_url}/v1/Stores/{store['id']}/Profiles/{profile_id}/Observations",
    auth=auth,
    json={"observations": [{
        "content": "Customer prefers email communication over phone calls",
        "source": "support-agent",
        "occurredAt": "2025-01-15T10:30:00Z"
    }]}
)
# 202 — poll statusUrl for completion
```

**Node.js**
```node
await axios.post(
    `${baseUrl}/v1/Stores/${store.id}/Profiles/${profileId}/Observations`,
    { observations: [{
        content: "Customer prefers email communication over phone calls",
        source: "support-agent",
        occurredAt: "2025-01-15T10:30:00Z",
    }] },
    { auth }
);
// 202 — poll statusUrl for completion
```

**Step 4 — Recall relevant context**

**Python**
```python
recall = requests.post(
    f"{base_url}/v1/Stores/{store['id']}/Profiles/{profile_id}/Recall",
    auth=auth,
    json={"query": "communication preferences"}
).json()

for obs in recall["observations"]:
    print(f"[{obs['score']:.2f}] {obs['content']}")
```

**Node.js**
```node
const { data: recall } = await axios.post(
    `${baseUrl}/v1/Stores/${store.id}/Profiles/${profileId}/Recall`,
    { query: "communication preferences" },
    { auth }
);

for (const obs of recall.observations) {
    console.log(`[${obs.score.toFixed(2)}] ${obs.content}`);
}
```

---

## Key Patterns

### Recall Modes

| Mode | Request | When to use |
|------|---------|-------------|
| Explicit query | `{"query": "shipping preferences"}` | You control relevance |
| Conversation context | `{"conversationId": "conv_conversation_xxx"}` | Active Orchestrator conversation — auto-generates query from last 10 messages |
| Most recent | `{}` (omit both) | Chronological order, no relevance ranking |

Additional Recall parameters: `observationsLimit` (default 20, 0–100), `summariesLimit` (default 5), `communicationsLimit` (default 0, requires conversationId), `relevanceThreshold` (0–1), `beginDate`/`endDate`.

### Profile Resolution

| I have... | Use | Behavior |
|-----------|-----|----------|
| Profile ID | `PATCH /v1/Stores/{storeId}/Profiles/{profileId}` | Direct update |
| An identifier (phone, email) | `POST /v1/Stores/{storeId}/Profiles` | Identity resolution finds or creates |
| Bulk contact data | `PUT /v1/Stores/{storeId}/Profiles/Bulk` | Up to 1000, identity-resolved |
| CSV file | `POST /v1/Stores/{storeId}/Profiles/Imports` | Presigned URL upload |

Check for 308 redirects on profile endpoints — indicates a merged profile.

### Trait Groups

Every trait must belong to a trait group. You must create the group (or add the trait to an existing group) before writing trait values to a profile.

Each trait has a `dataType` (STRING, NUMBER, BOOLEAN, ARRAY). Set `idTypePromotion` on a trait (e.g., `"phone"`) to make it an identifier for Lookup and identity resolution.

**Add a new trait to an existing group:**

Use `PATCH /v1/ControlPlane/Stores/{storeId}/TraitGroups/{traitGroupName}` — pass the new trait definition in the request body. Only the new traits need to be included; existing traits are unchanged.

**Python**
```python
requests.patch(
    f"{base_url}/v1/ControlPlane/Stores/{store['id']}/TraitGroups/contact",
    auth=auth,
    json={"traits": {
        "preferred_channel": {"dataType": "STRING", "description": "Preferred contact method"}
    }}
)
# 202 — adds "preferred_channel" to the existing "contact" group
```

**Node.js**
```node
await axios.patch(
    `${baseUrl}/v1/ControlPlane/Stores/${store.id}/TraitGroups/contact`,
    { traits: {
        preferred_channel: { dataType: "STRING", description: "Preferred contact method" },
    } },
    { auth }
);
// 202 — adds "preferred_channel" to the existing "contact" group
```

To remove a trait from a group, PATCH with `"dataType": ""` for that trait name.

### Automatic Extraction (Orchestrator Integration)

1. Create a Memory Store
2. Create a Conversation Orchestrator Configuration with capture rules
3. Link the store to the config

Once linked: profiles are auto-created per caller, and extraction is tied to the conversation lifecycle configured in Orchestrator. By default, observations are extracted when a conversation goes inactive or ends. Conversation summaries are generated when a conversation ends. These lifecycle transitions are configurable in your Orchestrator configuration — how you define conversation status timeouts determines when memory extraction runs. Conflicting information is automatically reconciled. See `twilio-conversation-orchestrator`.

### Lookup by Identifier

**Python**
```python
result = requests.post(
    f"{base_url}/v1/Stores/{store['id']}/Profiles/Lookup",
    auth=auth,
    json={"idType": "phone", "value": "+15558675310"}
).json()

profile_id = result["profiles"][0]["id"]
```

**Node.js**
```node
const { data: result } = await axios.post(
    `${baseUrl}/v1/Stores/${store.id}/Profiles/Lookup`,
    { idType: "phone", value: "+15558675310" },
    { auth }
);

const profileId = result.profiles[0].id;
```

### Voice Agent Latency Optimization

For voice agents where latency matters, skip semantic search and fetch observations directly:

**Python**
```python
observations = requests.get(
    f"{base_url}/v1/Stores/{store['id']}/Profiles/{profile_id}/Observations",
    auth=auth,
    params={"source": "support-agent", "createdAfter": "2025-01-01T00:00:00Z"}
).json()
```

**Node.js**
```node
const { data: observations } = await axios.get(
    `${baseUrl}/v1/Stores/${store.id}/Profiles/${profileId}/Observations`,
    { auth, params: { source: "support-agent", createdAfter: "2025-01-01T00:00:00Z" } }
);
```

### Async Polling Pattern

All 202 responses include a `statusUrl`. Poll until `status` is `COMPLETED` or `FAILED`:

**Python**
```python
import time

status_url = store["statusUrl"]
while True:
    op = requests.get(status_url, auth=auth).json()
    if op["status"] in ("COMPLETED", "FAILED", "CANCELLED"):
        break
    time.sleep(2)
```

**Node.js**
```node
let op;
do {
    await new Promise(r => setTimeout(r, 2000));
    op = (await axios.get(store.statusUrl, { auth })).data;
} while (!["COMPLETED", "FAILED", "CANCELLED"].includes(op.status));
```

---

## CANNOT

- Cannot exceed 15 Memory Stores per account — use sub-accounts beyond this
- Cannot batch more than 10 observations per request
- Cannot exceed 4096 characters per observation
- Cannot write traits to a group that doesn't exist — create the trait group first
- Cannot recover deleted profiles or observations — deletion is irreversible
- Cannot read data immediately after write — indexing is async
- Cannot auto-extract observations without a linked Orchestrator config
- Cannot exceed 1000 profiles per bulk upsert
- Cannot include auth headers when uploading to presigned import URLs — they're pre-signed
- Cannot use spaces or underscores in store displayName — pattern is `^[a-zA-Z0-9-]+$`

---

## Next Steps

- **Automatic conversation capture:** `twilio-conversation-orchestrator`
- **Background transcript intelligence (script adherence, NBR, scoring):** `twilio-conversation-intelligence`
- **Enterprise knowledge retrieval (RAG):** `twilio-enterprise-knowledge`
- **Voice agent with ConversationRelay:** `twilio-voice-conversation-relay`
- **TAC SDK middleware:** `twilio-agent-connect`
- **Debug issues:** `twilio-debugging-observability`
