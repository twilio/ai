# Twilio MCP Server

Remote MCP server providing real-time search across Twilio's full API surface — OpenAPI specs, documentation, and support content. Always current, including APIs released after your model's training cutoff.

**Server URL:** `https://mcp.twilio.com/docs`

## Tools

### `twilio__search`

Semantic + lexical hybrid search across Twilio and SendGrid OpenAPI specs and documentation.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Natural-language search query |
| `source` | string | No | `"api"` (specs only), `"docs"` (knowledge base), `"all"` (both). Default: `"all"` |
| `topics` | array | No | Filter by topic, e.g. `["twilio_docs"]` |
| `filter.name` | string | No | Product name filter, e.g. `"conversations"` |
| `filter.version` | string | No | API version filter, e.g. `"v2"` |
| `latest_only` | boolean | No | Only return latest API version. Default: `true` |
| `limit` | integer | No | Max results to return. Default: `5` |

### `twilio__retrieve`

Fetches complete schemas for specific API operations: full parameter definitions, request bodies, response fields, and intent guidance.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `ids` | array | Yes | Operation IDs from search results, e.g. `["op::twilio_messaging_v1::CreateMessage"]` |

## Connect the MCP Server

### Claude Code (CLI)

```bash
claude mcp add --scope user twilio-docs \
  --transport http \
  https://mcp.twilio.com/docs
```

### Claude Desktop / Cowork

Open the Connector window within Claude and add the **Twilio** connector to your workspace.

### Cursor

Add in **Cursor Settings > MCP Servers**:

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

### Codex (OpenAI)

Add to your project config:

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

## Filtering

- No filters → searches all products, latest versions only
- `filter.name` only → latest version of that product
- `filter.name` + `filter.version` → exact version pin
- `latest_only: false` → all versions searchable

## Support

questions-mcp@twilio.com
