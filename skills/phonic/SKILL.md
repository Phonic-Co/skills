---
name: phonic
description: Build, test, and operate voice agents on the Phonic platform using the phonic SDK (npm and PyPI) or REST API. Use when the project or request involves Phonic - creating Phonic agents, placing or receiving calls through Phonic, browser speech-to-speech via Phonic, Phonic tools, webhooks, or conversation analysis, or attaching MCP servers to a Phonic agent. Do NOT use for non-Phonic voice platforms or SDKs, generic text-to-speech or transcription tasks, or building standalone MCP servers.
license: MIT
---

# Building voice agents on Phonic

SDKs: `npm install phonic` / `pip install phonic`. Entry points: `new PhonicClient({ apiKey })` (Node), `Phonic(api_key=...)` / `AsyncPhonic(...)` (Python). REST base: `https://api.phonic.ai/v1` with `Authorization: Bearer $PHONIC_API_KEY`. Always use the `.ai` domain for API and docs URLs — `phonic.co` addresses still resolve for backward compatibility, but all new code should use `phonic.ai`.

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

## Config message

Every conversation begins with a `config` message:

- Minimal `config` message: `{ "type": "config", "agent": "my-cool-agent", "input_format": "pcm_44100", "output_format": "pcm_44100" }` (pick the audio formats that match your use case).
- For prototyping, it is possible to send the config without a named agent by carrying `system_prompt`, `welcome_message`, `voice_id`, `tools`, and more inline (see the config schema in [asyncapi.json](https://docs.phonic.ai/asyncapi.json)). However, for long-term projects we recommend creating an agent to be able to track version changes, reuse the agent config across different conversation mediums, and avoid frequent redeployments.
- Unknown fields and out-of-bounds values are rejected with `{"error": {"message": "Invalid parameters"}}` plus a `param_errors` object naming each offending field — read it, and treat a config rejection as fatal (a reconnect loop must not resend the same config). Field bounds are documented on the docs pages, but the served `asyncapi.json` artifact currently omits constraint keywords (`minimum`/`maximum`/`const`) — a field looking unconstrained in the artifact doesn't mean it is.

## Browser voice

This is the recommended pattern for building Phonic into a browser:

1. Server side, exchange the API key for a session token with the SDK — `client.auth.createSessionToken({ ttl_seconds: 300 })` (`POST /v1/auth/session_token`). The API key stays on the server — in Next.js, never put it in a `NEXT_PUBLIC_*` env var (those are compiled into client code).
2. The browser connects directly to `wss://api.phonic.ai/v1/sts/ws?session_token=<token>` (token in the query string — browsers can't set WebSocket headers). Don't add a relay server in between; the direct connection is the supported, lowest-latency pattern.
3. Send the config message (see [Config message](#config-message) above).
4. Send and receive audio as base64 16-bit PCM `audio_chunk` messages. We recommend to use chunks of ~20ms (chunks longer than ≤40ms are rejected with error); note mic access requires a secure context — HTTPS or localhost.
5. The client can also send messages during the call: `add_system_message`, `update_tools_subset`, `set_external_id`, `mute` / `unmute`, `generate_reply` (force a turn), and `say` — schemas in [asyncapi.json](https://docs.phonic.ai/asyncapi.json). These enable patterns like pushing app state into the conversation or a button that makes the agent speak. Note that `update_system_prompt` is deprecated; use `add_system_message` instead.
6. Turn lifecycle comes from explicit events: `assistant_started_speaking` / `assistant_finished_speaking` and `user_started_speaking` / `user_finished_speaking`. When building a transcript UI, start a new assistant entry on `assistant_started_speaking`, not on the first audio chunk — trailing `audio_chunk`s (often with empty text) can arrive after `assistant_finished_speaking`, and keying entries on chunks creates ghost transcript items. Interrupts are signaled by `user_started_speaking`; flush queued playback there.
7. Long sessions need a reconnect loop (there's no browser SDK to do it for you): on a drop, retry with backoff and mint a fresh session token per attempt; classify failures first — network blips are retryable, auth failures (revoked key, expired grant) must stop retrying and surface to the user.

Reference implementations: [phonic-web-demo](https://github.com/Phonic-Co/phonic-web-demo) (Next.js browser voice) and [phonic-examples](https://github.com/Phonic-Co/phonic-examples) (quick start, webhooks, websocket conversations, websocket tools — TS and Python).

## SDK surface

- Agents: `client.agents.create/get/list/update/upsert/delete`. `upsert` is create-or-update by name (idempotent). Agents are identified by name.
- Outbound calls: `client.conversations.outboundCall(...)` — greeting goes in `config.welcome_message`.
- Server audio bridge: `client.conversations.connect()` (context manager in Python: `with client.conversations.connect() as socket: for message in socket: ...`).
- Tools: `client.tools.create(...)`, then attach by name in agent config `tools: [...]`.

## Tools

Tool types (webhook, websocket, context, built-in transfers, MCP), reserved parameter names, and each type's request/response contract are documented in [the tools docs](https://docs.phonic.ai/docs/using-tools/tools_overview.md) — fetch the page for your tool type before implementing.

- Tool outputs must include every identifier a later tool call needs as a parameter (e.g. return the `job_id` your `update_job` tool expects) — the agent can only pass along values it has been given.
- Tools can be defined inline in the STS config (no pre-created tool): `tools: [{ "type": "custom_websocket", "execution_mode": "sync", "tool_schema": { "type": "function", "function": { ..., "strict": true } } }]` — the `strict: true` inside the function schema is required.
- If you bridge audio server-side, never do blocking I/O in the audio send/receive loop — periodic multi-second stalls in agent audio are almost always blocking code in the bridge.

## Transfers

- Transfer to a human is a built-in tool (`built_in_transfer_to_phone_number`, target number in the tool config) attached to the agent — don't hand-roll Twilio call control.
- Targets are preconfigured per transfer tool; the agent chooses among configured transfer tools and cannot dial an arbitrary number returned by another tool mid-call.
- Reading outcomes from conversation data: the transfer tool's output body has `connected: true|false` — `true` means the callee leg answered (human or voicemail); `false` means the transfer failed and the agent tells the caller.
- Voicemail detection: set `detect_voicemail: true` on the transfer tool (tools API). The agent then listens to the answered leg and a system tool call records the judgment — `tool_transfer_hit_voicemail` or `tool_transfer_successful` (= judged human). Without voicemail detection, `tool_transfer_successful` fires immediately on connect and means only "connected". Detection requires the bridged transfer mode (`use_agent_phone_number: true`, the default) — a SIP REFER handoff leaves the call and can't listen.
- The `conversation.transferred` webhook carries `transferred_to`, `call_info`, and the conversation items up to the transfer — the pre-transfer transcript arrives with the event.

## Webhooks and post-call steps

Webhooks can indicate conversation progress: `conversation.ended`, `conversation.analysis`, `conversation.transferred`. A typical post-call pipeline looks like:

1. **`conversation.ended`** fires when the call ends — log the outcome and kick off your downstream steps from here.
2. **`conversation.analysis`** follows with performance metrics: its `conversation` payload carries `latencies_ms` and `interruptions_count` (also available via `GET /v1/conversations/{id}/analysis`). It does not contain extraction results.
3. **Extractions** turn the transcript into typed fields you define. Create an extraction schema, then run it per conversation with `POST /v1/conversations/{id}/extractions` and read results with the matching GET — typically from your `conversation.ended` handler. Extractions read the transcript, so they fail if transcripts are deleted; run them promptly if you have a retention policy.
4. **Eval prompts** grade calls against plain-language criteria. Create them at the project level (`name` + a criterion like "the assistant made the required disclosure"), evaluate a conversation via the conversation-evals endpoint, and read results per conversation or listed across the project — useful for compliance checks and QA queues.

## Prompting voice agents

Voice prompts differ from chat prompts (speech formatting, silence handling, naturalness) — use [the prompting guide](https://docs.phonic.ai/docs/additional-resources/prompting_guide.md).

## Operations

- **Projects** organize agents/tools/numbers within a workspace; use one project per tenant/product line rather than a new workspace. API keys are workspace-level.
- **Rate limits**: see [the rate limits page](https://docs.phonic.ai/docs/additional-resources/rate_limits.md); high-volume outbound campaigns need client-side concurrency caps.
- **Observability**: per-call outcomes via the webhook events above; live read-only monitoring via the live WebSocket; [audit logs](https://docs.phonic.ai/docs/observability/audit-logs.md).

## Best practices for shipping to prod

These patterns may not be necessary to get a working demo, but are good practices for handling customer traffic, especially as you scale:

- **Webhook receivers** — Phonic delivers webhooks via svix, which retries, so the same event can arrive more than once (and concurrently). Build for that:
  1. Verify the svix signature before touching the body.
  2. Return 2xx immediately; do slow work (DB writes, API calls) in the background.
  3. Store each event id durably (DB unique constraint, not process memory) and skip events you've already processed.
  4. For side effects that must happen exactly once (paging someone, charging a card): durably mark the work "started" before doing it and "done" after, and on startup retry anything started-but-never-finished. The before-mark stops a redelivery from repeating the side effect; the startup retry stops a crash from silently dropping it.
- **Server-side reconnects** The SDK's `conversations.connect()` reconnects automatically on abnormal disconnects and resumes the same conversation — opt in with `reconnectConversationOnAbnormalDisconnect: true` (client or per-connect option).
- **Campaign/batch dialers**: claim each number (durable in-progress state) before dialing and design the claim so a second runner instance can't grab it too — per-run completion state alone doesn't prevent two processes from double-dialing.
