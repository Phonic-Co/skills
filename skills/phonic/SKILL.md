---
name: phonic
description: Build, test, and operate voice agents on the Phonic platform using the phonic SDK (npm and PyPI) or REST API. Use when the project or request involves Phonic - creating Phonic agents, placing or receiving calls through Phonic, browser speech-to-speech via Phonic, Phonic tools, webhooks, or conversation analysis, or attaching MCP servers to a Phonic agent. Do NOT use for non-Phonic voice platforms or SDKs, generic text-to-speech or transcription tasks, or building standalone MCP servers.
license: MIT
---

# Building voice agents on Phonic

SDKs: `npm install phonic` / `pip install phonic`. Entry points: `new PhonicClient({ apiKey })` (Node), `Phonic(api_key=...)` / `AsyncPhonic(...)` (Python). REST base: `https://api.phonic.ai/v1` with `Authorization: Bearer $PHONIC_API_KEY`. Always use the `.ai` domain for API and docs URLs — `phonic.co` addresses still resolve for backward compatibility, but all new code should use `phonic.ai`.

For API details not covered here, fetch live docs rather than from memory: use the index `https://docs.phonic.ai/llms.txt`. If an endpoint appears in neither the docs nor the OpenAPI spec (`docs.phonic.ai/openapi.json`), it doesn't exist, do NOT invent one.

## Choosing a conversation surface

- **Outbound phone call**: `client.conversations.outboundCall({ to_phone_number, config })`
- **Inbound phone number**: create/update the agent with `phone_number: "assign-automatically"`
- **Voice in a browser or mobile app**: direct STS WebSocket `wss://api.phonic.ai/v1/sts/ws?session_token=...` (see Browser voice below)
- **Bridge your own telephony/audio server-side**: `client.conversations.connect()`
- **Watch a live conversation (read-only)**: `wss://api.phonic.ai/v1/conversations/{id}/live/ws`

Mistakes to avoid when picking:

- Don't open the browser STS socket to place a phone call — phone calls are one API call (`outboundCall`); the STS socket streams a user's microphone.
- Don't relay browser audio through your own server on the way to Phonic — it works, but adds a network hop of latency to every utterance; browsers connect directly using a session token. The session-token flow exists because an API key can't live in a browser; your own server holds the key, so server-side code uses `conversations.connect()` directly and never needs session tokens.
- Don't confuse the two WebSockets: `sts/ws` is the live voice conversation; `conversations/{id}/live/ws` is a read-only monitoring feed (`conversation-updated` / `conversation-ended` events only) — it cannot send audio or speak.

## Browser voice

This is the recommended pattern — connect the browser directly to Phonic:

1. Server side, exchange the API key for a session token with the SDK — `client.auth.createSessionToken({ ttl_seconds: 300 })` (`POST /v1/auth/session_token`). The API key stays on the server — in Next.js, never put it in a `NEXT_PUBLIC_*` env var (those are compiled into client code).
2. The browser connects directly to `wss://api.phonic.ai/v1/sts/ws?session_token=<token>` (token in the query string — browsers can't set WebSocket headers). Don't add a relay server in between; the direct connection is the supported, lowest-latency pattern.
3. First message: `{ "type": "config", "agent": "<agent-name>", "input_format": "pcm_44100", "output_format": "pcm_44100" }`.
4. Send mic audio as base64 16-bit PCM in `{ "type": "audio_chunk", "audio": ... }` (capture via AudioWorklet) — **each chunk must be ≤40ms of audio** (the server rejects longer chunks with an error; batch ~20ms frames). The server streams agent audio back as `audio_chunk` messages too (same type name, both directions) — play them back by scheduling gaplessly on one shared AudioContext.
5. Turn lifecycle comes from explicit events: `assistant_started_speaking` / `assistant_finished_speaking` and `user_started_speaking` / `user_finished_speaking` (all bare `{type}` markers). When building a transcript UI, start a new assistant entry on `assistant_started_speaking`, not on the first audio chunk — **trailing `audio_chunk`s (often with empty text) can arrive after `assistant_finished_speaking`**, and keying entries on chunks creates ghost transcript items. Barge-in is signaled by `user_started_speaking`; flush queued playback there.

Reference implementations: **github.com/Phonic-Co/phonic-web-demo** (Next.js browser voice) and **github.com/Phonic-Co/phonic-examples** (quick start, webhooks, websocket conversations, websocket tools — TS and Python).

## SDK surface

- Agents: `client.agents.create/get/list/update/upsert/delete`. `upsert` is create-or-update by name (idempotent). Agents are referenced everywhere by name.
- Outbound calls: `client.conversations.outboundCall(...)` — greeting goes in `config.welcome_message`.
- Server audio bridge: `client.conversations.connect()` (context manager in Python: `with client.conversations.connect() as socket: for message in socket: ...`).
- TTS: `client.tts.stream({ text, ... })` — use the SDK rather than a hand-rolled `POST /v1/tts`. Output is raw PCM with no WAV container; add a header yourself if writing a playable file.
- Tools: `client.tools.create(...)`, then attach by name in agent config `tools: [...]`.

## Custom tools

- Tool names are snake_case.
- These parameter names are reserved — Phonic injects them, and defining them yourself silently breaks the tool: `call_info`, `conversation_id`, `from_phone_number`, `to_phone_number`, `twilio_call_sid`. Name your own parameters differently (e.g. `customer_phone_number`).
- Tool outputs must include every identifier a later tool call needs as a parameter (e.g. return the `job_id` your `update_job` tool expects) — the agent can only pass along values it has been given.
- Webhook tool endpoints need no response envelope — return plain JSON and it enters the agent's context — but the response is **silently truncated past 16,000 serialized characters**, so return only what the agent needs. POST tools receive parameters at the body root with call context nested under `call_info`; GET tools receive everything as query params (URL-encode phone numbers: `+` → `%2B`). `endpoint_timeout_ms` defaults to 5000; exceeding it fails the invocation.
- Besides webhook/websocket tools there are **context tools** (inject knowledge/instructions mid-conversation; heavily used for scenario-style flows) — see `docs.phonic.ai/docs/using-tools/context_tools.md`.
- External MCP servers attach at the agent level via `mcp_servers: [...]` in the agent config; Phonic discovers their tools automatically — don't recreate MCP tools as individual custom tools.
- If you bridge audio server-side, never do blocking I/O in the audio send/receive loop — periodic multi-second stalls in agent audio are almost always blocking code in the customer's bridge, not Phonic.

## Transfers

- Transfer to a human is a built-in tool (`built_in_transfer_to_phone_number`, target number in the tool config) attached to the agent — don't hand-roll Twilio call control.
- Targets are preconfigured per transfer tool; the agent chooses among configured transfer tools and cannot dial an arbitrary number returned by another tool mid-call.
- Reading outcomes from conversation data: the transfer tool's output body has `connected: true|false` — `true` means the callee leg answered (human **or** voicemail); `false` means the transfer failed and the agent tells the caller.
- Voicemail detection: set `detect_voicemail: true` on the transfer tool (tools API). The agent then listens to the answered leg and a system tool call records the judgment — `tool_transfer_hit_voicemail` or `tool_transfer_successful` (= judged human). Without voicemail detection, `tool_transfer_successful` fires immediately on connect and means only "connected". Detection requires the bridged transfer mode (`use_agent_phone_number: true`, the default) — a SIP REFER handoff leaves the call and can't listen.
- The `conversation.transferred` webhook carries `transferred_to`, `call_info`, and the conversation items up to the transfer — the pre-transfer transcript arrives with the event.

## Webhooks and post-call analysis

Webhook event names: `conversation.ended`, `conversation.analysis`, `conversation.transferred`. Analysis and extraction results arrive in the `conversation.analysis` event, or via `GET /v1/conversations/{id}/analysis`.

For structured data out of every call, define an **extraction schema** (typed fields, not free text) via the extraction-schemas API and read results per conversation. For quality/QA, projects support **conversation eval prompts** (`name` + a plain-language criterion like "the assistant made the required disclosure") that conversations are evaluated against — create via the projects eval-prompts API, results via the conversation evaluations endpoints.

## Prompting voice agents

Voice prompts differ from chat prompts (speech formatting, silence handling, naturalness) — fetch and follow the prompting guide at `docs.phonic.ai/docs/additional-resources/prompting_guide.md` before writing a system prompt. One placement rule the guide doesn't cover: behavioral rules that must always hold ("we never accept customer-owned trailers") belong in the system prompt or a context tool — not buried in one webhook tool's description.

## Operations

- **Projects** organize agents/tools/numbers within a workspace (default project: `main`); use one project per tenant/product line rather than a new workspace. API keys are workspace-level.
- **Rate limits**: see `docs.phonic.ai/docs/additional-resources/rate_limits.md`; high-volume outbound campaigns need client-side concurrency caps and retry-with-backoff — capacity errors ("insufficient capacity", declined calls) are expected at burst and must be retried, not dropped.
- **Observability**: per-call outcomes via the webhook events above; live read-only monitoring via the live WebSocket; audit logs at `docs.phonic.ai/docs/observability/audit-logs.md`.

## Production patterns

For anything beyond a demo, these orderings are load-bearing — getting them subtly wrong passes tests and fails in production:

- **Webhook receivers** — Phonic delivers webhooks via svix, which retries, so the same event can arrive more than once (and concurrently). Build for that:
  1. Verify the svix signature before touching the body.
  2. Return 2xx immediately; do slow work (DB writes, API calls) in the background.
  3. Store each event id durably (DB unique constraint, not process memory) and skip events you've already processed.
  4. For side effects that must happen exactly once (paging someone, charging a card): durably mark the work "started" before doing it and "done" after, and on startup retry anything started-but-never-finished. The before-mark stops a redelivery from repeating the side effect; the startup retry stops a crash from silently dropping it.
- **Reconnects, server-side: don't hand-roll them.** The SDK's `conversations.connect()` reconnects automatically on abnormal disconnects and resumes the same conversation — opt in with `reconnectConversationOnAbnormalDisconnect: true` (client or per-connect option).
- **Browser sockets still need a hand-written loop** (no browser SDK yet): classify the failure before retrying — transient network errors get backoff + a **fresh session token** per attempt; auth/permission failures (revoked key, expired grant) must stop retrying and surface, or you'll hammer the API forever.
- **Campaign/batch dialers**: claim each number (durable in-progress state) before dialing and design the claim so a second runner instance can't grab it too — per-run completion state alone doesn't prevent two processes from double-dialing.

## Defaults

- Create referenced resources instead of stopping to ask: if the task mentions an agent that may not exist, create it (or `upsert` it) in the setup path.
- Dashboard steps shown in docs (API keys aside) have API equivalents — prefer code.
