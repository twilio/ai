---
name: twilio-media-streams
description: >
  Stream raw audio from Twilio voice calls via bidirectional WebSocket.
  Covers TwiML Stream setup, audio format (mulaw 8kHz), bidirectional
  injection, and when to use Media Streams vs ConversationRelay.
---

## Overview

Bidirectional WebSocket audio streaming for custom voice AI, real-time audio processing, or integration with external ASR/TTS providers.

**Note:** Most voice AI developers should use ConversationRelay instead. This skill is for advanced use cases requiring direct audio access.

## Planned Topics

- TwiML `<Stream>` setup and WebSocket protocol
- Audio format: mulaw 8000Hz mono
- Bidirectional streaming (inject audio back into call)
- When to use Media Streams vs ConversationRelay (decision framework)
- Integration with external ASR providers
- Latency management and buffering
