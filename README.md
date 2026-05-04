# Twilio for AI

Official Twilio Skills for AI coding agents. Works across Claude Code, Cursor, Codex, and any tool supporting the [Agent Skills](https://agentskills.io) standard.

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
# Add the Twilio marketplace and install the plugin
/plugin marketplace add twilio/ai
/plugin install twilio-skills@twilio-skills
```

Skills activate automatically when your prompt matches a covered use case. You can also invoke them directly:

```
/twilio-verify-send-otp
/twilio-sms-send-message
```

#### Cursor

Skills follow the open [Agent Skills](https://agentskills.io) standard. Cursor auto-discovers them from `.agents/skills/` in your workspace:

```bash
git clone https://github.com/twilio/ai.git .twilio-ai
cp -r .twilio-ai/skills/ .agents/skills/
```

Type `/skills` in Cursor's Agent mode to see available Twilio skills.

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
cp -r ai/skills/ ~/.agents/skills/
```

#### Other tools (GitHub Copilot, Gemini CLI, JetBrains, etc.)

Twilio Skills use the open [Agent Skills](https://agentskills.io) standard (`SKILL.md` format). Any tool supporting this standard can load them from the `skills/` directory in this repo.

---

## Verify it's working

After setup, ask your agent:

> "How do I send an SMS with Twilio?"

Your agent should provide guidance on Messaging Services, A2P compliance, error handling, and recommend specific skills for deeper implementation — rather than relying solely on training data.

---

## Coming soon

- **Twilio MCP Server** — Real-time search across 1,800+ Twilio API endpoints, always current

---

## Feedback

- Questions and feedback: questions-skills@twilio.com
- Issues: [GitHub Issues](https://github.com/twilio/ai/issues)

---

## License

MIT
