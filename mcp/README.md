# Twilio MCP Docs

Remote MCP server providing real-time search across Twilio's API surface.

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
| `filter.version` | string | No | Version pin, e.g. `"v2"`. Omit for latest. |
| `latest_only` | boolean | No | Default `true`. Set `false` to search all versions. |
| `limit` | integer | No | Max results. Default: 5 |

### `twilio__retrieve`

Fetches complete schemas for specific API operations or full article content.

**Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `ids` | array | Yes | One or more IDs from search results. Prefix: `op::` for API operations |

**Example ID format:** `op::twilio_messaging_v1::CreateMessage`

## How It Works

- In-memory semantic + BM25 hybrid search (< 10MB index)
- Covers all Twilio and SendGrid OpenAPI specs
- Updated when new specs ship — always current
- Read-only: searches and retrieves documentation, cannot execute API calls

## Version Handling

- No filter → returns latest version of each product
- `filter.name` only → latest version of that product
- `filter.name` + `filter.version` → exact version pin
- `latest_only: false` → all versions searchable

## Support

questions-mcp@twilio.com
