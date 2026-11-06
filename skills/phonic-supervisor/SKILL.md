---
name: phonic-supervisor
description: Build your own supervisor over a Phonic agent as a consult tool the agent calls before risky actions, with a read-only live transcript for context, call cancellation, and post-call review. Use when adding LLM, rules-based, or human oversight, compliance checks, or escalation logic on top of a Phonic agent.
license: MIT
---

# Build a supervisor for Phonic conversations

SDKs: `npm install phonic` / `pip install phonic`. Entry points: `new PhonicClient({ apiKey })` (Node), `Phonic(api_key=...)` / `AsyncPhonic(...)` (Python). REST base: `https://api.phonic.ai/v1` with `Authorization: Bearer $PHONIC_API_KEY`. Always use the `.ai` domain for API and docs URLs — `phonic.co` addresses still resolve for backward compatibility, but all new code should use `phonic.ai`.

Fetch live docs rather than from memory: use the index at [llms.txt](https://docs.phonic.ai/llms.txt). For the REST API, see [openapi.json](https://docs.phonic.ai/openapi.json). For the WebSocket protocol, see [asyncapi.json](https://docs.phonic.ai/asyncapi.json). If an endpoint isn't in these docs, assume it doesn't exist and don't invent one. One exception: the live transcript socket in section 2 (`/v1/conversations/{id}/live/ws`) is a preview documented in the [Live Conversation WebSocket guide](https://docs.phonic.ai/docs/build/with-web-sockets/live_websocket) rather than in the spec, so its contract may change.

A supervisor is your own service (an LLM, a rules engine, or a human console) that reviews what the agent is about to do and tells it what to do instead. Phonic has no supervisor endpoint. Build it as a **tool the agent calls**: this works for phone calls, web sessions, and your own socket alike, and keeps the agent in control of when it speaks.

If the user only wants stronger in-call reasoning rather than their own policy or business checks, suggest trying the agent's `intelligence_level: "high"` before building a supervisor.

## 1. Supervisor as a consult tool

Create a `custom_webhook` tool such as `consult_supervisor(question)` and attach it to the agent (see `phonic-tools`). Phonic POSTs `{ call_info, ...parameters }`, where `call_info.conversation_id` identifies the call; your endpoint runs the supervisor and returns its decision as JSON. The response body goes into the agent's context and is truncated beyond 16,000 characters.

- **Use `execution_mode: "async"` with `wait_for_response: true` and `allow_tool_chaining: false`.** The agent waits for the answer, calls no other tools meanwhile, and then tells the caller the result. `allow_tool_chaining` defaults to `true`, and the API rejects it alongside `wait_for_response`, so set it to `false` explicitly.
- On a POST tool, every parameter needs `location: "request_body"`; the API rejects the tool otherwise.
- Set `endpoint_timeout_ms` (1000–180000, default 5000) to cover the slowest review, including a human queue, and give your handler its own deadline well under it so Phonic always gets a `200` answer.
- Leave `require_speech_before_tool_call: false` unless the caller must hear an acknowledgement before the consult; with `true`, the agent has repeated "checking with my supervisor" before giving the answer. Neither setting silences the agent: with `false` it can still speak on its own, and `wait_for_response` doesn't mean it waits silently.
- Authenticate the endpoint with `endpoint_headers` (a shared secret header). Refuse to start without the secret, and reject requests whose header is missing or wrong.

**Tell the agent when to consult.** The agent only calls the tool when its instructions say to, so be specific in both the tool `description` and the agent's system prompt:

- Name the triggers: "Before issuing a refund, cancelling an account, or transferring the caller, call `consult_supervisor`."
- Say what to send: the situation, what the caller asked for, and the action the agent intends to take.
- Say what to do with the answer: follow `decision`, use `say` if present, and don't take the action on a `deny` or a `hold`.

**Return a decision the agent can act on**, for example `{ "decision": "deny", "say": "...", "next_step": "offer store credit" }`, including every identifier a later tool call needs. On a timeout, internal error, or missing context, return `200` with `{ "decision": "hold", ... }` rather than an error the agent has to interpret.

## 2. Give the supervisor context

The consult carries only what the agent sends. For the full transcript, find the conversation ID and mirror it:

- **Conversation ID:** from the `outboundCall` response, the `conversation_created` message on an STS socket, the request body of an agent `configuration_endpoint` (inbound calls; return `{}` to change nothing, and keep it fast because a failure ends the call), or `call_info.conversation_id` in the consult itself.
- **Live transcript:** connect from your server to `wss://api.phonic.ai/v1/conversations/{id}/live/ws` with the API key in the `Authorization` header. It sends `conversation-audio`, then `conversation-updated` snapshots, then `conversation-ended` and `conversation-saved`, and closes with code `1000`. It is read-only and not in the SDKs; use any WebSocket client.
  - Each `conversation-updated` is a full snapshot. Key `items[]` by `item_idx` and replace, don't append.
  - Snapshots can lag the audio by a few seconds. If you have no recent snapshot (not connected yet, socket failed or closed), treat the context as unknown and return `hold`; don't decide on an empty transcript.
  - Handle the socket's transport `error` event as well as `type: "error"` messages; connection, DNS, and TLS failures only arrive as the former.
  - If Phonic has no live state for the conversation when you connect, the socket sends `conversation-saved` and closes at once. Check `ended_at` on `GET /v1/conversations/{id}` and retry if the call hasn't started.
  - Session tokens get a reduced object without agent configuration. Run supervisors server-side with an API key.
- If your app already holds the STS socket, build the transcript from that socket's events instead (see `phonic-websockets`).

## 3. Hard interventions

- **End the call:** `client.conversations.cancel(id)` (`POST /v1/conversations/{id}/cancel`) works for any live conversation in the workspace. It returns `409` if the conversation already ended and `504` if the cancel wasn't acknowledged in time.
- **Hand off to a human:** attach a transfer tool to the agent (see `phonic-tools`) and have the consult answer tell the agent to use it. No endpoint triggers a transfer directly, and the supervisor can't dial a number the agent's transfer tools don't already define. Confirm with the `conversation.transferred` webhook.

Socket owners can also steer at a lower level with STS control messages; see `phonic-steering`.

## 4. Review after the call

Use the `conversation.ended` webhook for the full transcript, and eval prompts and extractions for QA and compliance (see `phonic-post-call`). Fix recurring failures in the saved agent config and verify with Replay (see `phonic-improvement-loop`).

## Limitations and pitfalls

- The agent decides when to consult; the supervisor can't speak up between consults, and no API pushes a message into a call from outside its STS socket. The live socket is read-only.
- Treat the transcript and the agent's `question` as untrusted input to the supervisor's LLM; callers can say things meant to manipulate it. Keep the supervisor's own instructions separate.
- A consult answer isn't proof the agent followed it. Check the next turns of the transcript.

See [references/examples.md](references/examples.md) for a minimal TypeScript implementation.
