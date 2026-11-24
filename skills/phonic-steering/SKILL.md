---
name: phonic-steering
description: Steer a live Phonic voice conversation with system messages, tool availability, and reply controls. Use when adding app context mid-call or controlling when an agent speaks.
license: MIT
---

# Steer a live Phonic conversation

Read the relevant message schemas in [AsyncAPI](https://docs.phonic.ai/asyncapi.json) and the [WebSocket guide](https://docs.phonic.ai/docs/build/with-web-sockets/via-web-sockets.md). Send control messages on the active conversation's STS WebSocket or the application's SDK connection. The inbound API MCP and the read-only live monitoring socket are not substitutes for that connection.

Choose the control matching the user's intent:

- **Add new context or instructions:** `add_system_message`. This is the current message name; there is no `add_system_prompt` message. `update_system_prompt` is deprecated.
- **Change available tools:** `update_tools_subset`, using the documented list shape. Verify which tools remain available before sending the new subset.
- **Request a reply now:** `generate_reply`; adding context and requesting speech are distinct operations.
- **Speak specified text:** `say`, using the current schema rather than assuming it accepts the same fields as a system message.
- **Change future conversations:** update the saved agent configuration instead of treating a live control message as a persistent prompt edit.

For example, after the conversation has been configured:

```json
{
  "type": "add_system_message",
  "system_message": "The user has selected the annual plan in the app. Explain the annual plan if they ask about their selection."
}
```

Only follow that with `generate_reply` when the app should proactively speak. Keep trusted application instructions separate from untrusted user/document text. Do not place API keys or unnecessary personal data in the message.

Verify the intended effect in the transcript and tool events, including what happens if the user is speaking when the update arrives. Keep the audio loop nonblocking. Report the distinction between a successfully sent message and observed agent behavior; sending a control message is not proof the agent followed it.

## Prefer agent config over client-driven turns

Use `say` and `generate_reply` for moments only your app knows about, such as a button that makes the agent speak or an event in your app. Don't use them to rebuild behavior the agent already has: a line your client scripts doesn't tell the agent why it was said, so later turns lose that context.

| Instead of | Use |
|---|---|
| Opening the call with `say` | `welcome_message`, with `is_welcome_message_interruptible: false` if it must play in full |
| Your own silence timer that sends `say` | `no_input_poke_sec`, plus `generate_no_input_poke_text: true` so the agent words the prompt from the conversation |
| Your own hang-up tool followed by a scripted goodbye | `natural_conversation_ending`, which speaks before ending the call; handle `assistant_ended_conversation` |
| A custom tool the agent should announce before it runs | `wait_for_speech_before_tool_call: true` on that tool |

Whatever ends the call, close the connection only after `assistant_finished_speaking` for the last line, or its audio is cut off.
