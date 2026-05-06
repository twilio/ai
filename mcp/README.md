# Twilio MCP

The Twilio MCP (Model Context Protocol) server provides AI coding agents with semantic search over Twilio documentation and API specifications.

> **Public Beta:** Twilio MCP is currently available as a Public Beta product, and the information contained here is subject to change. Some features are not yet implemented and others may be changed before the product is declared as Generally Available. Public Beta products are not covered by the Twilio Support Terms or Twilio Service Level Agreement.

---

## Endpoint

```
https://mcp.twilio.com/docs/mcp
```

Transport: Streamable HTTP (no authentication required)

---

## Available Tools

| Tool | Description | Use when |
|------|-------------|----------|
| `twilio__search` | Semantic search over Twilio docs and API operations | Finding relevant APIs, understanding product capabilities, discovering how-to guides |
| `twilio__retrieve` | Fetch full parameter schemas for a specific API operation | Getting exact request/response schemas, required fields, enum values for an API call |

---

## Setup by IDE

### Claude Code

```bash
# Option 1: Install via marketplace plugin (includes Skills + MCP)
/plugin marketplace add twilio/ai
/plugin install twilio-developer-kit@twilio

# Option 2: Add MCP server directly
claude mcp add twilio-docs -- npx -y @anthropic-ai/mcp-remote https://mcp.twilio.com/docs/mcp
```

### Cursor

Add in **Cursor Settings > MCP**:

```json
{
  "twilio-docs": {
    "url": "https://mcp.twilio.com/docs/mcp"
  }
}
```

### Codex (OpenAI)

```bash
codex mcp add twilio-docs --url https://mcp.twilio.com/docs/mcp
```

### Other MCP-compatible tools

Any tool that supports MCP streamable HTTP transport can connect directly to:

```
https://mcp.twilio.com/docs/mcp
```

No API key or authentication is required.

---

## Example Usage

Once connected, your AI agent can use the MCP tools to answer Twilio development questions:

**Search for relevant APIs:**
> "How do I send an SMS with Twilio?"

The agent calls `twilio__search` with your query and receives relevant documentation snippets and API operation IDs.

**Get full API details:**
> "Show me the parameters for creating a message"

The agent calls `twilio__retrieve` with the operation ID (e.g., `CreateMessage`) and receives the complete parameter schema.

---

## For MCP Listing Submissions

If you are submitting Twilio MCP to an MCP registry or marketplace, use the following metadata:

| Field | Value |
|-------|-------|
| Name | `twilio` |
| Display Name | Twilio MCP |
| Description | Search Twilio documentation and API specifications |
| Endpoint | `https://mcp.twilio.com/docs/mcp` |
| Transport | Streamable HTTP |
| Authentication | None |
| Tools | `twilio__search`, `twilio__retrieve` |
| Repository | `https://github.com/twilio/ai` |
| Documentation | `https://mcp.twilio.com/docs/mcp` |

---

## Feedback

- Questions: questions-skills@twilio.com
- Issues: [GitHub Issues](https://github.com/twilio/ai/issues)
