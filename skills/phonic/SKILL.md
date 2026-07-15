---
name: phonic
description: Build, test, and operate voice agents on the Phonic platform using the phonic SDK (npm and PyPI) or REST API. Use when the project or request involves Phonic - creating Phonic agents, placing or receiving calls through Phonic, browser speech-to-speech via Phonic, Phonic tools, webhooks, or conversation analysis, or attaching MCP servers to a Phonic agent. Do NOT use for non-Phonic voice platforms or SDKs, generic text-to-speech or transcription tasks, or building standalone MCP servers.
license: MIT
---

# Building voice agents on Phonic

SDKs: `npm install phonic` / `pip install phonic`. Entry points: `new PhonicClient({ apiKey })` (Node), `Phonic(api_key=...)` / `AsyncPhonic(...)` (Python). REST base: `https://api.phonic.ai/v1` with `Authorization: Bearer $PHONIC_API_KEY`. Always use the `.ai` domain for API and docs URLs — `phonic.co` addresses still resolve for backward compatibility, but all new code should use `phonic.ai`.

For API details not covered here, fetch live docs rather than coding REST calls from memory: `https://docs.phonic.ai/llms.txt` is the index; append `.md` to any docs page URL for markdown.

## Choosing a conversation surface

| You want | Use | Not |
|---|---|---|
| Agent calls someone (outbound) | `client.conversations.outboundCall({ to_phone_number, config })` | the browser socket |
| Agent answers a phone number (inbound) | create/update the agent with `phone_number: "assign-automatically"` | — |
| User talks from a browser or mobile app | direct STS WebSocket `wss://api.phonic.ai/v1/sts/ws?session_token=...` | proxying audio through your own server (adds latency); `conversations.connect()` |
| Bridge your own telephony/audio server-side | `client.conversations.connect()` | the session-token flow — that exists for browsers/mobile apps where the API key can't live; your own server holds the key and connects directly |
| Watch a live conversation (read-only) | `wss://api.phonic.ai/v1/conversations/{id}/live/ws` | this socket cannot send audio or speak |

The two WebSockets are easy to confuse: `sts/ws` is the voice conversation; `conversations/{id}/live/ws` is a read-only monitoring feed (`conversation-updated` / `conversation-ended` events only).

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
- Transfer to a human is a built-in tool (`built_in_transfer_to_phone_number` with the target number) attached to the agent — don't hand-roll Twilio call control. Transfer targets are preconfigured on the tool; the agent selects among configured transfer tools and cannot dial an arbitrary number returned by another tool mid-call.
- Reading transfer outcomes from conversation data: the transfer tool's output body has `connected: true|false` — `true` means the callee leg answered (human **or** voicemail); `false` means the transfer failed and the agent informs the caller. With voicemail detection enabled (`detect_voicemail: true` on the transfer tool — settable via the tools API), the agent listens to the answered leg and a system tool call records the judgment: `tool_transfer_hit_voicemail` or `tool_transfer_successful` (= judged human; without voicemail detection it fires immediately on connect, meaning only "connected"). Voicemail detection requires the bridged transfer mode (`use_agent_phone_number: true`, the default) — a SIP REFER handoff can't listen.
- External MCP servers attach at the agent level via `mcp_servers: [...]` in the agent config; Phonic discovers their tools automatically — don't recreate MCP tools as individual custom tools.
- If you bridge audio server-side, never do blocking I/O in the audio send/receive loop — periodic multi-second stalls in agent audio are almost always blocking code in the customer's bridge, not Phonic.

## Webhooks and post-call analysis

Webhook event names: `conversation.ended`, `conversation.analysis`, `conversation.transferred`. Analysis and extraction results arrive in the `conversation.analysis` event, or via `GET /v1/conversations/{id}/analysis`. The `conversation.transferred` payload carries `transferred_to`, `call_info`, and the conversation items up to the transfer — so the pre-transfer transcript arrives with the event.

For structured data out of every call, define an **extraction schema** (typed fields, not free text) via the extraction-schemas API and read results per conversation. For quality/QA, projects support **conversation eval prompts** (`name` + a plain-language criterion like "the assistant made the required disclosure") that conversations are evaluated against — create via the projects eval-prompts API, results via the conversation evaluations endpoints.

## Prompting voice agents

Voice prompts differ from chat prompts — full guide at `docs.phonic.ai/docs/additional-resources/prompting_guide.md`. The rules that matter most:

- To confirm spelled input: "spell letter by letter using caps with spaces" (no periods between letters). Alphanumeric IDs and phone numbers are spoken correctly without special formatting; wrap URLs in `[slower]` tags.
- For naturalness: instruct disfluencies/casual speech and "do not repeat the same sentence twice" to avoid robotic loops.
- The agent can stay silent deliberately: have it output exactly `(())` when no reply is warranted.
- Behavioral rules that must always hold ("we never accept customer-owned trailers") belong in the system prompt or a context tool — not buried in one webhook tool's description.

## Operations

- **Projects** organize agents/tools/numbers within a workspace (default project: `main`); use one project per tenant/product line rather than a new workspace. API keys are workspace-level.
- **Rate limits**: see `docs.phonic.ai/docs/additional-resources/rate_limits.md`; high-volume outbound campaigns need client-side concurrency caps and retry-with-backoff — capacity errors ("insufficient capacity", declined calls) are expected at burst and must be retried, not dropped.
- **Observability**: per-call outcomes via the webhook events above; live read-only monitoring via the live WebSocket; audit logs at `docs.phonic.ai/docs/observability/audit-logs.md`.
- **Billing/usage** has no public API — it's managed in the dashboard; don't invent billing endpoints.

## Production patterns

For anything beyond a demo, these orderings are load-bearing — getting them subtly wrong passes tests and fails in production:

- **Webhook receivers**: verify the svix signature before trusting the body; acknowledge with 2xx fast and do slow work asynchronously and idempotently; and commit dedup state (durable, keyed on event id) **before** performing side effects like paging or charging — commit-after-send means a crash in between double-sends on redelivery. Claiming-before-send alone trades duplicates for drops: record each pending side effect (outbox) and retry incomplete ones on startup, so a crash mid-sequence neither double-sends nor silently drops.
- **Reconnects, server-side: don't hand-roll them.** The SDK's `conversations.connect()` reconnects automatically on abnormal disconnects and resumes the same conversation — opt in with `reconnectConversationOnAbnormalDisconnect: true` (client or per-connect option). **Browser sockets still need a hand-written loop** (no browser SDK yet): classify the failure before retrying — transient network errors get backoff + a **fresh session token** per attempt; auth/permission failures (revoked key, expired grant) must stop retrying and surface, or you'll hammer the API forever.
- **Campaign/batch dialers**: claim each number (durable in-progress state) before dialing and design the claim so a second runner instance can't grab it too — per-run completion state alone doesn't prevent two processes from double-dialing.

## Defaults

- Create referenced resources instead of stopping to ask: if the task mentions an agent that may not exist, create it (or `upsert` it) in the setup path.
- Dashboard steps shown in docs (API keys aside) have API equivalents — prefer code.
