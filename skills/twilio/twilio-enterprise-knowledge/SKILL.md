---
name: twilio-enterprise-knowledge
description: >
  Add knowledge retrieval (RAG) to AI agents using Twilio Enterprise Knowledge.
  Covers provisioning Knowledge Bases, uploading sources from web URLs, PDFs,
  and raw text, and running semantic search to retrieve relevant chunks at
  runtime. Use this skill to ground agent responses in your organization's
  actual approved content — FAQs, policies, product docs — rather than
  hallucinated answers.
---

## Overview

Enterprise Knowledge gives AI agents access to your organization's source material during conversations — FAQs, warranty policies, support scripts, product catalogs. It closes the gap between general model knowledge and how your business actually operates.

```
Your content (web/PDF/text) → Knowledge Base → Indexed chunks
Agent query → Search → Ranked chunks → Inject into LLM prompt
```

Enterprise Knowledge is shared across your organization — it captures institutional content. It is distinct from Conversation Memory (`twilio-conversation-memory`), which is per-customer context. The two are designed to be combined: enterprise content for accuracy, customer memory for personalization.

**Base URL:** `https://knowledge.twilio.com`

**Authentication:** HTTP Basic — `Authorization: Basic {base64(accountSid:authToken)}`

**Rules for agents:**
- Always poll `statusUrl` after any 202 response — all writes are async
- Always wait for Knowledge Base status `COMPLETED` before adding sources
- Always wait for source processing to complete before searching
- Never use `/v1/` paths — all routes use `/v2/` prefix
- Never include auth headers when uploading to presigned URLs — they're already signed
- Never use spaces or underscores in `displayName` — pattern is `^[a-zA-Z0-9-]+$`
- Never exceed 16MB per file upload or 1,048,576 chars per text source

---

## Prerequisites

- Twilio account with Enterprise Knowledge enabled
  — A credit card must be added to the account
  — See `twilio-account-setup` for initial setup
  — See `twilio-iam-auth-setup` for credential best practices
- Environment variables:
  - `TWILIO_ACCOUNT_SID`
  - `TWILIO_AUTH_TOKEN`
- SDK: `pip install twilio` / `npm install twilio`

---

## Quickstart

**Step 1 — Create a Knowledge Base**

**Python**
```python
import os, requests, time

account_sid = os.environ["TWILIO_ACCOUNT_SID"]
auth_token = os.environ["TWILIO_AUTH_TOKEN"]
base_url = "https://knowledge.twilio.com"
auth = (account_sid, auth_token)

res = requests.post(
    f"{base_url}/v2/ControlPlane/KnowledgeBases",
    auth=auth,
    json={"displayName": "product-docs", "description": "Support agent knowledge"}
)

status_url = res.json()["statusUrl"]

while True:
    op = requests.get(status_url, auth=auth).json()
    if op["status"] == "COMPLETED":
        kb_id = op["result"]["id"]
        break
    if op["status"] == "FAILED":
        raise Exception(op["error"]["detail"])
    time.sleep(2)

print(kb_id)  # know_knowledgebase_xxx
```

**Node.js**
```javascript
const accountSid = process.env.TWILIO_ACCOUNT_SID;
const authToken = process.env.TWILIO_AUTH_TOKEN;
const baseUrl = "https://knowledge.twilio.com";
const authHeader = "Basic " + btoa(`${accountSid}:${authToken}`);
const headers = { "Authorization": authHeader, "Content-Type": "application/json" };

const res = await fetch(`${baseUrl}/v2/ControlPlane/KnowledgeBases`, {
    method: "POST",
    headers,
    body: JSON.stringify({ displayName: "product-docs", description: "Support agent knowledge" }),
});

const { statusUrl } = await res.json();

let kbId;
while (true) {
    const op = await fetch(statusUrl, { headers: { "Authorization": authHeader } }).then(r => r.json());
    if (op.status === "COMPLETED") { kbId = op.result.id; break; }
    if (op.status === "FAILED") throw new Error(op.error.detail);
    await new Promise(r => setTimeout(r, 2000));
}
```

**Step 2 — Add a Knowledge Source**

Three source types: **Web** (crawl a URL), **File** (upload PDF/CSV/Markdown/text, max 16MB), **Text** (inline, max 1,048,576 chars).

**Python**
```python
knowledge = requests.post(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Knowledge",
    auth=auth,
    json={
        "name": "Product Documentation",
        "description": "Public product docs",
        "source": {"type": "Web", "url": "https://docs.example.com", "crawlDepth": 3}
    }
).json()

knowledge_id = knowledge["id"]
```

**Node.js**
```javascript
const knowledge = await fetch(`${baseUrl}/v2/KnowledgeBases/${kbId}/Knowledge`, {
    method: "POST",
    headers,
    body: JSON.stringify({
        name: "Product Documentation",
        description: "Public product docs",
        source: { type: "Web", url: "https://docs.example.com", crawlDepth: 3 },
    }),
}).then(r => r.json());

const knowledgeId = knowledge.id;
```

**Step 3 — Wait for processing**

Sources are processed asynchronously. Poll until `status` is `COMPLETED`.

**Python**
```python
while True:
    k = requests.get(
        f"{base_url}/v2/KnowledgeBases/{kb_id}/Knowledge/{knowledge_id}", auth=auth
    ).json()
    if k["status"] == "COMPLETED":
        break
    if k["status"] == "FAILED":
        raise Exception(f"Processing failed: {k}")
    time.sleep(3)
```

**Node.js**
```javascript
while (true) {
    const k = await fetch(
        `${baseUrl}/v2/KnowledgeBases/${kbId}/Knowledge/${knowledgeId}`,
        { headers: { "Authorization": authHeader } }
    ).then(r => r.json());
    if (k.status === "COMPLETED") break;
    if (k.status === "FAILED") throw new Error(JSON.stringify(k));
    await new Promise(r => setTimeout(r, 3000));
}
```

Statuses: `SCHEDULED` → `QUEUED` → `PROCESSING` → `COMPLETED` / `FAILED`

**Step 4 — Search and inject into LLM prompt**

**Python**
```python
results = requests.post(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Search",
    auth=auth,
    json={"query": "How do I reset my password?", "top": 5}
).json()

chunks = "\n\n".join(c["content"] for c in results["chunks"])

system_prompt = f"""You are a helpful support agent.

Relevant knowledge:
{chunks}

Answer using only the above content."""
```

**Node.js**
```javascript
const results = await fetch(`${baseUrl}/v2/KnowledgeBases/${kbId}/Search`, {
    method: "POST",
    headers,
    body: JSON.stringify({ query: "How do I reset my password?", top: 5 }),
}).then(r => r.json());

const chunks = results.chunks.map(c => c.content).join("\n\n");
const systemPrompt = `You are a helpful support agent.\n\nRelevant knowledge:\n${chunks}`;
```

---

## Key Patterns

### Combine with Conversation Memory

For the best agent responses, combine Enterprise Knowledge (company content) with Conversation Memory Recall (individual customer history).

**Python**
```python
recall_res = requests.post(
    f"https://memory.twilio.com/v1/Stores/{store_id}/Profiles/{profile_id}/Recall",
    auth=auth,
    json={"query": user_query, "observationsLimit": 5}
).json()

search_res = requests.post(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Search",
    auth=auth,
    json={"query": user_query, "top": 3}
).json()

customer_context = "\n".join(o["content"] for o in recall_res.get("observations", []))
knowledge = "\n\n".join(c["content"] for c in search_res.get("chunks", []))

system_prompt = f"""Customer history:\n{customer_context}\n\nDocumentation:\n{knowledge}"""
```

**Node.js**
```javascript
const [recallRes, searchRes] = await Promise.all([
    fetch(`https://memory.twilio.com/v1/Stores/${storeId}/Profiles/${profileId}/Recall`, {
        method: "POST",
        headers,
        body: JSON.stringify({ query: userQuery, observationsLimit: 5 }),
    }).then(r => r.json()),
    fetch(`${baseUrl}/v2/KnowledgeBases/${kbId}/Search`, {
        method: "POST",
        headers,
        body: JSON.stringify({ query: userQuery, top: 3 }),
    }).then(r => r.json()),
]);

const customerContext = recallRes.observations.map(o => o.content).join("\n");
const knowledge = searchRes.chunks.map(c => c.content).join("\n\n");
```

### File Upload (PDF/CSV/Markdown)

File sources return a presigned URL. Upload the file there — do not include auth headers.

**Python**
```python
knowledge = requests.post(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Knowledge",
    auth=auth,
    json={"name": "Handbook", "description": "Employee handbook", "source": {"type": "File"}}
).json()

upload_url = knowledge["source"]["importUrl"]

with open("handbook.pdf", "rb") as f:
    requests.put(upload_url, data=f, headers={"Content-Type": "application/pdf"})
```

**Node.js**
```javascript
const knowledge = await fetch(`${baseUrl}/v2/KnowledgeBases/${kbId}/Knowledge`, {
    method: "POST",
    headers,
    body: JSON.stringify({ name: "Handbook", description: "Employee handbook", source: { type: "File" } }),
}).then(r => r.json());

const file = await fs.promises.readFile("handbook.pdf");
await fetch(knowledge.source.importUrl, {
    method: "PUT",
    headers: { "Content-Type": "application/pdf" },
    body: file,
});
```

### Filter Search to Specific Sources

When your Knowledge Base has multiple sources, target search to specific ones:

**Python**
```python
results = requests.post(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Search",
    auth=auth,
    json={"query": "cancellation policy", "top": 5, "knowledgeIds": [policy_source_id]}
).json()

for chunk in results["chunks"]:
    print(f"[{chunk['score']:.3f}] {chunk['content'][:100]}")
```

**Node.js**
```javascript
const results = await fetch(`${baseUrl}/v2/KnowledgeBases/${kbId}/Search`, {
    method: "POST",
    headers,
    body: JSON.stringify({ query: "cancellation policy", top: 5, knowledgeIds: [policySourceId] }),
}).then(r => r.json());

for (const chunk of results.chunks) {
    console.log(`[${chunk.score.toFixed(3)}] ${chunk.content.slice(0, 100)}`);
}
```

Omit `knowledgeIds` to search across all sources. Max 100 IDs per request.

### Refresh a Web Source

Re-crawl without changing config. Set `crawlPeriod` for automatic recrawling.

**Python**
```python
requests.patch(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Knowledge/{knowledge_id}?refresh=true",
    auth=auth,
    json={"name": "Product Documentation"}
)

requests.patch(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Knowledge/{knowledge_id}",
    auth=auth,
    json={"name": "Product Documentation", "source": {"type": "Web", "crawlPeriod": "WEEKLY"}}
)
```

**Node.js**
```javascript
await fetch(`${baseUrl}/v2/KnowledgeBases/${kbId}/Knowledge/${knowledgeId}?refresh=true`, {
    method: "PATCH",
    headers,
    body: JSON.stringify({ name: "Product Documentation" }),
});

await fetch(`${baseUrl}/v2/KnowledgeBases/${kbId}/Knowledge/${knowledgeId}`, {
    method: "PATCH",
    headers,
    body: JSON.stringify({ name: "Product Documentation", source: { type: "Web", crawlPeriod: "WEEKLY" } }),
});
```

Crawl period options: `WEEKLY` | `BIWEEKLY` | `MONTHLY` | `NEVER`

### Inspect Chunks

Audit what was indexed from a source:

**Python**
```python
chunks = requests.get(
    f"{base_url}/v2/KnowledgeBases/{kb_id}/Knowledge/{knowledge_id}/Chunks",
    auth=auth,
    params={"pageSize": 50}
).json()

for chunk in chunks["chunks"]:
    print(f"[{chunk['metadata']['sourceType']}] {chunk['content'][:100]}")
```

**Node.js**
```javascript
const chunks = await fetch(
    `${baseUrl}/v2/KnowledgeBases/${kbId}/Knowledge/${knowledgeId}/Chunks?pageSize=50`,
    { headers: { "Authorization": authHeader } }
).then(r => r.json());

for (const chunk of chunks.chunks) {
    console.log(`[${chunk.metadata.sourceType}] ${chunk.content.slice(0, 100)}`);
}
```

Paginate with `pageToken` from `chunks.meta.nextToken`.

---

## CANNOT

- Cannot exceed 5 Knowledge Bases per account
- Cannot exceed 10 knowledge sources per Knowledge Base
- Cannot add sources before Knowledge Base is active — poll `statusUrl` until `COMPLETED`
- Cannot use v1 endpoints — all routes use `/v2/` prefix on `knowledge.twilio.com`
- Cannot include auth header when uploading to presigned URL — `importUrl` is already signed
- Cannot search before source processing completes — poll source status first
- Cannot exceed 16 MiB (16,777,216 bytes) per file upload
- Cannot exceed 1,048,576 characters (~1MB) per text source pushed via API
- Cannot exceed 2048 characters in a search query
- Cannot exceed 2048 characters in a web source URL
- Cannot retrieve more than 20 search results per query (`top` max is 20)
- Cannot exceed 100 `knowledgeIds` in a search filter
- Cannot set crawl depth beyond 1–10 levels for web sources
- Cannot use custom crawl schedules — locked to fixed intervals: `WEEKLY`, `BIWEEKLY`, `MONTHLY`, or `NEVER`
- Cannot use spaces or underscores in `displayName` — alphanumeric and hyphens only
- Cannot use `name` longer than 30 characters for knowledge sources
- Cannot modify immutable fields (`id`, `type`, `status`, `url`, `createdAt`, `updatedAt`) via PATCH
- Cannot use Knowledge for per-customer context — use `twilio-conversation-memory` for that

---

## Next Steps

- **Per-customer context:** `twilio-conversation-memory` — combine with Enterprise Knowledge for full agent context
- **Background transcript intelligence:** `twilio-conversation-intelligence`
- **Voice agent with ConversationRelay:** `twilio-voice-conversation-relay`
- **TAC SDK integration:** `twilio-agent-connect`
- **Debug integration issues:** `twilio-debugging-observability`
