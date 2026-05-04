# Twilio for AI

Official tools for AI coding agents building on Twilio. Two complementary resources that work across Claude Code, Cursor, Codex, and any MCP-compatible client.

| Resource | What it does | Think of it as... |
|----------|-------------|-------------------|
| **[Skills](#twilio-skills)** | Guides agents through the *right* approach — which products to combine, in what order, what to avoid | An experienced Solutions Engineer |
| **[MCP Server](#twilio-mcp-server)** | Live search across 1,800+ API endpoints — parameters, response fields, versions | The reference manual |

Together: MCP gives your agent the **what**, Skills give it the **how**.

---

## Twilio Skills

Skills are structured packages of procedural knowledge for AI coding agents. They follow a progressive disclosure architecture: your agent sees lightweight metadata for all available skills, loads the relevant skill when a task matches, and drills into reference material on demand — keeping your context window lean while providing deep domain expertise exactly when needed.

### What Skills help with

- **Product selection:** "I need to verify users" — the agent reasons through Twilio Verify vs. custom OTP, whether to add Lookup for fraud scoring, before any code gets written
- **Architecture patterns:** "Build me an AI voice agent" — the agent asks questions to understand your use case complexity, then recommends the right combination of products (e.g., Conversation Relay + Memory + Conversation Intelligence + TaskRouter)
- **What to avoid:** Skills carry explicit `CANNOT` sections that reduce hallucination by documenting hard constraints and common pitfalls

### Available skill categories

| Type | Purpose | Examples |
|------|---------|---------|
| **Setup** | Account, auth, and number configuration | `twilio-account-setup`, `twilio-iam-auth-setup` |
| **Planner** | Use-case qualification and product selection | `twilio-identity-verification-advisor`, `twilio-marketing-promotions-advisor` |
| **Product** | How to use one Twilio product correctly | `twilio-verify-send-otp`, `twilio-sms-send-message`, `twilio-sendgrid-email-send` |
| **Guardrail** | Operational patterns preventing failures | `twilio-security-hardening`, `twilio-reliability-patterns`, `twilio-compliance-traffic` |

Covers: SMS, MMS, WhatsApp, RCS, Voice, Verify, SendGrid, Conversations, Messaging Services, Compliance (A2P 10DLC, Toll-Free, STIR/SHAKEN), and more.

### Install Twilio Skills

#### Claude Code

```bash
# Add the Twilio marketplace and install the plugin (includes Skills + MCP)
/plugin marketplace add twilio/ai
/plugin install twilio-developer-toolkit@twilio
```

Skills activate automatically when your prompt matches a covered use case. You can also invoke them directly:

```
/twilio-verify-send-otp
/twilio-sms-send-message
```

#### Cursor

Skills follow the open [Agent Skills](https://agentskills.io) standard. Cursor auto-discovers them from `.agents/skills/` in your workspace:

```bash
# Clone into your project
git clone https://github.com/twilio/ai.git .twilio-ai
cp -r .twilio-ai/skills/.agents/skills/ .agents/skills/
```

Type `/skills` in Cursor's Agent mode to see available Twilio skills.

Also add the MCP server in **Cursor Settings > MCP Servers**:

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

#### Codex (OpenAI)

Skills are file-based. Place them in any of these locations:

```bash
# Project-level (committed to your repo)
.agents/skills/<skill-name>/SKILL.md

# User-level (available across all projects)
~/.agents/skills/<skill-name>/SKILL.md
```

Clone and copy:

```bash
git clone https://github.com/twilio/ai.git
cp -r ai/skills/.agents/skills/ ~/.agents/skills/
```

Add the MCP server to your project config:

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

#### Other tools (GitHub Copilot, Gemini CLI, JetBrains, etc.)

Twilio Skills use the open [Agent Skills](https://agentskills.io) standard (`SKILL.md` format). Any tool supporting this standard can load them from the `skills/` directory in this repo.

---

## Twilio MCP Server

Real-time search across Twilio's full API surface — OpenAPI specs, documentation, and support content. Always current, including APIs released after your model's training cutoff.

**Server URL:** `https://mcp.twilio.com/docs`

### Tools

| Tool | Purpose | Example query |
|------|---------|---------------|
| `twilio__search` | Semantic + lexical hybrid search across all Twilio APIs and docs | "how do I send an SMS?", "conversation memory API" |
| `twilio__retrieve` | Full schema for a specific API operation — every parameter, response field, and version | `op::twilio_messaging_v1::CreateMessage` |

### Connect the MCP Server

#### Claude Code (CLI)

```bash
claude mcp add --scope user twilio-docs \
  --transport http \
  https://mcp.twilio.com/docs
```

#### Claude Desktop / Cowork

Open the Connector window within Claude and add the **Twilio** connector to your workspace.

#### cURL (direct testing)

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

## When to use what

| You're asking... | Use | Why |
|-----------------|-----|-----|
| "Which Twilio product solves X?" | Skills | Product selection requires judgment, not just API specs |
| "What parameters does this endpoint accept?" | MCP | Precise, versioned schema lookup |
| "What's the right architecture for Y?" | Skills | Architecture decisions need context on tradeoffs |
| "Show me the full API for Z" | MCP | Complete request/response definitions |
| "What compliance steps do I need?" | Skills | Regulatory guidance requires procedural knowledge |
| "What error code is 30007?" | MCP | Error reference lookup |

---

## Verify it's working

After setup, ask your agent:

> "How do I send an SMS with Twilio?"

Your agent should:
1. Invoke `twilio__search` (not rely on training data)
2. Reference specific API endpoints (e.g., `POST /2010-04-01/Accounts/{AccountSid}/Messages.json`)
3. If Skills are installed, provide guidance on Messaging Services, A2P compliance, and error handling

If your agent pulls live API specs instead of guessing from memory, you're connected.

---

## Feedback

- Skills: questions-skills@twilio.com
- MCP and Connector: questions-mcp@twilio.com
- Issues: [GitHub Issues](https://github.com/twilio/ai/issues)

---

## License

MIT
