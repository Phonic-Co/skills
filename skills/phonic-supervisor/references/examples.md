# Supervisor examples (TypeScript)

Minimal sketches. `decide` stands in for your own LLM call, rules, or human queue; it is not a Phonic API. Check field names against [openapi.json](https://docs.phonic.ai/openapi.json) before shipping.

## Create the consult tool and tell the agent when to use it

```typescript
import { PhonicClient } from "phonic";

const secret = process.env.SUPERVISOR_SECRET;
if (!secret) throw new Error("SUPERVISOR_SECRET must be set");

const client = new PhonicClient({ apiKey: process.env.PHONIC_API_KEY! });

await client.tools.create({
  name: "consult_supervisor",
  description:
    "Ask the supervisor before issuing a refund, cancelling an account, or transferring the caller. Describe the situation, what the caller asked for, and the action you intend to take. Follow the returned decision; on deny or hold, don't take the action.",
  type: "custom_webhook",
  execution_mode: "async",
  wait_for_response: true,
  allow_tool_chaining: false,
  require_speech_before_tool_call: false, // true only if the caller must hear an acknowledgement first
  endpoint_method: "POST",
  endpoint_url: "https://your-server.example.com/phonic/consult",
  endpoint_headers: { "X-Supervisor-Secret": secret },
  endpoint_timeout_ms: 20000, // the handler's own deadline below must be well under this
  parameters: [
    {
      name: "question",
      description: "The situation, the caller's request, and the action you intend to take",
      is_required: true,
      type: "string",
      location: "request_body", // required for every parameter of a POST webhook tool
    },
  ],
});
```

Attach it by name (`tools: ["consult_supervisor"]`) and repeat the triggers in the agent's `system_prompt`, for example: "Before any refund, cancellation, or transfer, call consult_supervisor and wait for its answer. If the decision is deny or hold, don't take the action; say the returned `say` text and offer `next_step`."

## Answer the consult (Hono shown)

Phonic POSTs `{ call_info: { conversation_id, ... }, question }`.

```typescript
import { timingSafeEqual } from "node:crypto";
import { Hono } from "hono";
import { contextFor, type Item } from "./watch";

type Decision = { decision: "allow" | "deny" | "hold"; say?: string; next_step?: string };
declare function decide(context: Item[], question: string, signal: AbortSignal): Promise<Decision>;

const secret = process.env.SUPERVISOR_SECRET;
if (!secret) throw new Error("SUPERVISOR_SECRET must be set"); // fail fast, never serve unauthenticated
const secretBytes = Buffer.from(secret);

// Well under the tool's endpoint_timeout_ms (20000), so Phonic always gets an answer.
const DECISION_DEADLINE_MS = 8000;
const HOLD: Decision = { decision: "hold", say: "Let me get a colleague to confirm that for you.", next_step: "offer a callback" };

function authorized(header: string | undefined): boolean {
  if (!header) return false;
  const given = Buffer.from(header);
  return given.length === secretBytes.length && timingSafeEqual(given, secretBytes);
}

function isConsult(body: unknown): body is { call_info: { conversation_id: string }; question: string } {
  const b = body as { question?: unknown; call_info?: { conversation_id?: unknown } } | null;
  return typeof b?.question === "string" && typeof b.call_info?.conversation_id === "string";
}

async function decideWithDeadline(context: Item[], question: string): Promise<Decision> {
  const controller = new AbortController();
  let timer: ReturnType<typeof setTimeout> | undefined;
  const deadline = new Promise<Decision>((resolve) => {
    timer = setTimeout(() => {
      controller.abort(); // pass the signal to your LLM or HTTP client so the work actually stops
      resolve(HOLD);
    }, DECISION_DEADLINE_MS);
  });
  try {
    return await Promise.race([decide(context, question, controller.signal), deadline]);
  } catch {
    return HOLD;
  } finally {
    clearTimeout(timer);
  }
}

const app = new Hono();

app.post("/phonic/consult", async (c) => {
  if (!authorized(c.req.header("X-Supervisor-Secret"))) {
    return c.json({ error: "unauthorized" }, 401);
  }
  const body: unknown = await c.req.json().catch(() => null);
  if (!isConsult(body)) return c.json({ error: "bad request" }, 400);

  // Policy: no fresh transcript means no approval. Drop this if you don't mirror the transcript.
  const context = contextFor(body.call_info.conversation_id);
  if (!context) return c.json(HOLD);

  return c.json(await decideWithDeadline(context, body.question));
});
```

## Mirror the live transcript for context (optional)

```typescript
import WebSocket from "ws";

export type Item = { item_idx: number; role: string; text: string };
const live = new Map<string, { items: Item[]; receivedAt: number }>();

// Snapshots arrive when the conversation changes; tune this to your call pace.
const MAX_CONTEXT_AGE_MS = 60_000;

// undefined when there's no snapshot yet, the socket closed or failed, or the snapshot is stale.
export function contextFor(conversationId: string): Item[] | undefined {
  const entry = live.get(conversationId);
  if (!entry || Date.now() - entry.receivedAt > MAX_CONTEXT_AGE_MS) return undefined;
  return entry.items;
}

export function watchConversation(conversationId: string) {
  const ws = new WebSocket(
    `wss://api.phonic.ai/v1/conversations/${conversationId}/live/ws`,
    { headers: { Authorization: `Bearer ${process.env.PHONIC_API_KEY}` } },
  );

  ws.on("message", (raw) => {
    const message = JSON.parse(raw.toString());
    if (message.type === "conversation-updated") {
      // Full snapshot: replace, don't append.
      live.set(conversationId, { items: message.conversation.items, receivedAt: Date.now() });
    } else if (message.type === "error") {
      console.error(message.error?.message ?? message.message);
    }
  });
  // Transport failures (DNS, TLS, refused connection) arrive here, not as a `type: "error"` message.
  ws.on("error", (err) => console.error(`live socket ${conversationId}:`, err.message));
  // Phonic closes with 1000 after conversation-saved; any close means the context is no longer live.
  ws.on("close", () => live.delete(conversationId));
}
```

Call `watchConversation` with the ID from your `outboundCall` response or your configuration endpoint. If Phonic has no live state for the conversation yet (for example, an outbound call that is still ringing), the socket sends `conversation-saved` and closes immediately; check `GET /v1/conversations/{id}` for `ended_at` and retry with backoff if the call hasn't started.

To end a call from the supervisor: `await client.conversations.cancel(conversationId)`.
