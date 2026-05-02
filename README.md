# Twilio AI

Tools for AI agents building on Twilio. Two complementary resources that work across Claude Code, Cursor, Codex, and any MCP-compatible client.

## What's Here

### Skills

Procedural knowledge that guides AI agents to the right Twilio APIs for a given use case. Skills answer the questions that API specs can't: which products to combine, in what order, and what to avoid.

Covers: SMS, Voice, WhatsApp, RCS, Verify, SendGrid, Conversations, TaskRouter, Compliance, and 30+ products.

### MCP Server (Twilio Docs)

Real-time search across Twilio's full API surface — OpenAPI specs, public documentation, and support articles. Two tools:

- **`twilio__search`** — Semantic + lexical hybrid search. Given a natural-language query, returns ranked API operations and doc snippets with version awareness and product filtering.
- **`twilio__retrieve`** — Fetches complete schemas for specific API operations: full parameter definitions, request bodies, response fields, and intent guidance.

Always current. Covers APIs released after your model's training cutoff.

**Server URL:** `https://mcp.twilio.com/docs`

---

## Quick Start

### Claude Code

```bash
# Add MCP server (one-time)
claude mcp add --scope user twilio-docs \
  --transport http \
  https://mcp.twilio.com/docs
```

Skills are installed separately — see the `skills/` directory README once available.

### Cursor

Add the MCP server in Cursor Settings > MCP Servers:

```json
{
  "mcpServers": {
    "twilio-docs": {
      "type": "remote",
      "url": "https://mcp.twilio.com/docs"
    }
  }
}
```

Or install the full plugin (skills + MCP) via the Cursor plugin marketplace once available.

### Codex (OpenAI)

Add to your project's `.codex-plugin/plugin.json` or global config:

```json
{
  "mcpServers": {
    "twilio-docs": {
      "type": "remote",
      "url": "https://mcp.twilio.com/docs"
    }
  }
}
```

### OpenCode

Add to `~/.config/opencode/opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "twilio-docs": {
      "type": "remote",
      "url": "https://mcp.twilio.com/docs",
      "enabled": true,
      "oauth": false
    }
  }
}
```

### cURL (direct testing)

The server speaks JSON-RPC 2.0 over HTTP POST:

```bash
# Search for an API
curl -s -X POST https://mcp.twilio.com/docs \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "twilio__search",
      "arguments": {
        "query": "send an SMS message",
        "source": "all",
        "limit": 5
      }
    }
  }' | jq '.result.content[0].text | fromjson'

# Retrieve full schema for a specific operation
curl -s -X POST https://mcp.twilio.com/docs \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "twilio__retrieve",
      "arguments": {
        "ids": ["op::twilio_messaging_v1::CreateMessage"]
      }
    }
  }' | jq '.result.content[0].text | fromjson'
```

---

## What to Use When

| Question type | Use | Example |
|--------------|-----|---------|
| "What parameters does this endpoint accept?" | MCP (`twilio__search` + `twilio__retrieve`) | "What fields does messages.create() take?" |
| "Which Twilio product should I use for X?" | Skills | "I need to verify users — Verify vs custom OTP?" |
| "How do I call this specific API?" | MCP | "Show me the Conversations v2 AddParticipant schema" |
| "What's the right architecture for Y?" | Skills | "Build a contact center with call recording and transcription" |
| "What error code is 30007?" | MCP (`twilio__search`) | Error code lookups |
| "What compliance steps do I need?" | Skills | "What do I register before sending US SMS?" |

**Think of it as:** MCP is the reference manual. Skills are the experienced developer who knows which pages to read and in what order.

---

## Coming Soon

- **Skills directory** — Full skill set for Claude Code, Cursor, and Codex (shipping soon)
- **Plugin manifests** — One-click install for Cursor and Codex marketplaces

---

## Support

- MCP questions and feedback: questions-mcp@twilio.com
- Issues: [GitHub Issues](https://github.com/twilio/ai/issues)

---

## License

MIT
