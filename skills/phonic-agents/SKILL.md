---
name: phonic-agents
description: Create and configure Phonic agents and phone calls using MCP, SDKs, or REST. Use for saved agents, inbound numbers, outbound calls, prompting, and project operations.
license: MIT
---

# Configure Phonic agents and calls

SDKs: `npm install phonic` / `pip install phonic`. REST base: `https://api.phonic.ai/v1`. Always use the `.ai` domain for new API and docs URLs.

For operations through this plugin, use the connected Phonic MCP and its OAuth authorization. Do not search the user's local files or environment for credentials, or forward an application API key to the MCP connection.

When implementing an application with the SDK or REST API, guide the user to configure their own Phonic API key in the application's server-side secret configuration. Do not ask them to paste it into chat. Application requests authenticate to Phonic's API using that explicitly configured key; keep it out of browser code, logs, and committed files. Read the current SDK authentication documentation when constructing the integration.

Fetch live docs rather than from memory: use the index at [llms.txt](https://docs.phonic.ai/llms.txt). For the REST API, see [openapi.json](https://docs.phonic.ai/openapi.json). For the WebSocket protocol, see [asyncapi.json](https://docs.phonic.ai/asyncapi.json). If an endpoint does not appear in these docs, it doesn't exist, do not invent one.

## SDK surface

- Agents: `client.agents.create/get/list/update/upsert/delete`. `upsert` is create-or-update by name (idempotent). Agents are identified by name.
- Outbound calls: `client.conversations.outboundCall(...)` — greeting goes in `config.welcome_message`.

## Prompting voice agents

Voice prompts differ from chat prompts (speech formatting, silence handling, naturalness) — use [the prompting guide](https://docs.phonic.ai/docs/build/agents/prompting_guide.md).

## Operations

- **Projects** organize agents/tools/numbers within a workspace; use one project per tenant/product line rather than a new workspace. API keys are workspace-level.
- **Rate limits**: see [the rate limits page](https://docs.phonic.ai/docs/platform/rate_limits.md); high-volume outbound campaigns need client-side concurrency caps.
- **Observability**: per-call outcomes via the webhook events above; live read-only monitoring via the live WebSocket; [audit logs](https://docs.phonic.ai/docs/platform/audit-logs.md).

## Campaign and batch dialers

- **Campaign/batch dialers**: claim each number (durable in-progress state) before dialing and design the claim so a second runner instance can't grab it too — per-run completion state alone doesn't prevent two processes from double-dialing.
