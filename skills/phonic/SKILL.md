---
name: phonic
description: Choose a Phonic voice integration and find the relevant implementation skill. Use when finding or viewing Phonic conversations, starting a Phonic integration, or deciding between phone calls, browser voice, and server audio.
license: MIT
---

# Build voice agents on Phonic

SDKs: `npm install phonic` / `pip install phonic`. Entry points: `new PhonicClient({ apiKey })` (Node), `Phonic(api_key=...)` / `AsyncPhonic(...)` (Python). REST base: `https://api.phonic.ai/v1` with `Authorization: Bearer $PHONIC_API_KEY`. Always use the `.ai` domain for API and docs URLs — `phonic.co` addresses still resolve for backward compatibility, but all new code should use `phonic.ai`.

Fetch live docs rather than from memory: use the index at [llms.txt](https://docs.phonic.ai/llms.txt). For the REST API, see [openapi.json](https://docs.phonic.ai/openapi.json). For the WebSocket protocol, see [asyncapi.json](https://docs.phonic.ai/asyncapi.json). If an endpoint does not appear in these docs, it doesn't exist, do not invent one.

## Sharing conversations

When the user asks to find, list, show, or open conversations, return clickable conversation-viewer links with brief identifying details (such as date, agent, or duration), rather than dumping JSON, full transcripts, or tool-call payloads. If it is unclear whether they want viewer links, a summary/analysis, or raw data, ask a short clarifying question. Honor explicit requests for analysis, transcripts, or raw data, returning only the relevant details alongside the viewer link.

Build viewer links as `https://phonic.ai/conversations/{workspace}/{project}/{conversationId}` using the returned conversation's `workspace`, `project.name`, and full `id`; URL-encode each path segment. Do not use `project.id`, an admin-only URL, or the temporary `audio_url`. If the required fields are missing or ambiguous, look them up through the available conversation tools or ask the user; never guess a workspace/project or fabricate a link. These links use the viewer's existing Phonic access and do not make the conversation public.


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
