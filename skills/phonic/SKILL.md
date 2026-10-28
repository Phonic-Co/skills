---
name: phonic
description: Choose a Phonic voice integration and find the relevant implementation skill. Use when starting a Phonic integration or deciding between phone calls, browser voice, and server audio.
license: MIT
---

# Build voice agents on Phonic

SDKs: `npm install phonic` / `pip install phonic`. REST base: `https://api.phonic.ai/v1`. Always use the `.ai` domain for new API and docs URLs.

For operations through this plugin, use the connected Phonic MCP and its OAuth authorization. Do not search the user's local files or environment for credentials, or forward an application API key to the MCP connection.

When implementing an application with the SDK or REST API, guide the user to configure their own Phonic API key in the application's server-side secret configuration. Do not ask them to paste it into chat. Application requests authenticate to Phonic's API using that explicitly configured key; keep it out of browser code, logs, and committed files. Read the current SDK authentication documentation when constructing the integration.

Fetch live docs rather than from memory: use the index at [llms.txt](https://docs.phonic.ai/llms.txt). For the REST API, see [openapi.json](https://docs.phonic.ai/openapi.json). For the WebSocket protocol, see [asyncapi.json](https://docs.phonic.ai/asyncapi.json). If an endpoint does not appear in these docs, it doesn't exist, do not invent one.

## Choosing a conversation surface

- **Outbound phone call**: `client.conversations.outboundCall({ to_phone_number, config })`
- **Inbound phone number**: create/update the agent with `phone_number: "assign-automatically"`
- **Voice in a browser or mobile app**: direct STS WebSocket `wss://api.phonic.ai/v1/sts/ws?session_token=...`
- **Bridge your own telephony/audio server-side**: `client.conversations.connect()`
- **Watch a live conversation (read-only)**: `wss://api.phonic.ai/v1/conversations/{id}/live/ws`

Mistakes to avoid when picking:

- Don't open the browser STS socket to place a phone call — phone calls are one API call (`outboundCall`); the STS socket streams a user's microphone.
- Don't relay browser audio through your own server on the way to Phonic — it works, but adds a network hop of latency to every utterance; browsers connect directly using a session token. The session-token flow exists because an API key can't live in a browser; your own server holds the key, so server-side code uses `conversations.connect()` directly and never needs session tokens.
- Don't confuse the two WebSockets: `sts/ws` is the live voice conversation; `conversations/{id}/live/ws` is a read-only monitoring feed (`conversation-updated` / `conversation-ended` events only) — it cannot send audio or speak.

## Continue with the relevant skill

Use the matching skill when installed. If it is unavailable, follow the live documentation links above.

| Task | Skill |
| --- | --- |
| Agents, phone calls, prompts, and project operations | [phonic-agents](../phonic-agents/SKILL.md) |
| Browser voice, STS configuration, audio, and reconnects | [phonic-websockets](../phonic-websockets/SKILL.md) |
| Agent tools and human transfers | [phonic-tools](../phonic-tools/SKILL.md) |
| Webhooks, extractions, and conversation evaluations | [phonic-post-call](../phonic-post-call/SKILL.md) |
