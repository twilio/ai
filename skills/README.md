# Twilio Skills

Coming soon. Skills will be published here across the following categories:

## Skill Types

| Type | Purpose | Example |
|------|---------|---------|
| **Setup** | Account, auth, and number configuration | `twilio-account-setup`, `twilio-iam-auth-setup` |
| **Planner** | Use-case qualification and product selection | `twilio-identity-verification-advisor`, `twilio-customer-support-architect` |
| **Product** | How to use one Twilio product correctly | `twilio-verify-send-otp`, `twilio-sms-send-message` |
| **Guardrail** | Operational patterns preventing failures | `twilio-security-hardening`, `twilio-reliability-patterns` |

## Products Covered

SMS, MMS, WhatsApp, RCS, Voice, Verify, SendGrid, Conversations, TaskRouter, Messaging Services, Compliance (A2P 10DLC, Toll-Free, STIR/SHAKEN), and more.

## Format

Each skill is a `SKILL.md` file with YAML frontmatter:

```markdown
---
name: skill-name
description: >
  What this skill does and when to use it.
---

## Overview
## Prerequisites
## Quickstart
## Key Patterns
## CANNOT
## Next Steps
```

Compatible with Claude Code, Cursor, and OpenAI Codex.
