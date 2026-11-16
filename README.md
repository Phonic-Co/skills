# Phonic conversation review for Claude

Review existing [Phonic](https://phonic.ai) conversations, transcripts, analysis, evaluations, and extractions through MCP.

This is the `claude-plugin` review distribution. The [main branch](https://github.com/Phonic-Co/skills/tree/main) contains the full multi-client plugin.

## Connect

1. Sign in to your Phonic account and choose the workspace containing the conversations you want to review.
2. Install this plugin and complete the MCP OAuth browser sign-in. No API key is needed.
3. Ask: **“Help me review recent Phonic conversations in my project.”**

MCP URL: `https://mcp.phonic.ai/mcp?profile=review`

The server change enabling this profile must be deployed before this distribution is submitted. The profile shares the existing Phonic OAuth resource and workspace/project permissions; it limits tools, not the permissions of your Phonic account.

## Included capabilities

- Read workspace settings and list authorized projects.
- Browse existing conversations and transcripts.
- Read existing conversation analysis, evaluation results, and extractions.
- Inspect evaluation prompts and extraction schemas.
- Read Phonic documentation and submit integration feedback when requested.

This distribution does not create or modify agents, generate audio, place calls, start browser conversations, replay conversations, run new evaluations/extractions, create or rotate API keys, issue session/conversation tokens, or retrieve webhook secrets. It does not use API or SDK fallbacks.

## Data handling

The declared MCP connection sends requests to Phonic and returns data from your authorized workspace. Conversations and transcripts may contain personal data. Feedback submitted through `submit_feedback` is sent to Phonic. Skills do not send data through separate API connections or read credentials from your machine.

## Reviewer checks

Use a dedicated account with access to a project containing synthetic, completed conversations and existing analysis, evaluation, and extraction results.

1. Connect through OAuth and list authorized projects. Workspace settings do not include the workspace name or ID.
2. List conversations in the selected project and retrieve one transcript.
3. Retrieve its existing analysis, evaluations, and extractions. Missing results should be reported honestly.
4. Ask to create an API key or retrieve a webhook secret: the connector must not offer or execute those operations.
5. Ask to preview speech, place a call, create an agent, or replay a conversation: the connector must explain that those operations are unavailable and must not attempt a fallback.

These are reviewer instructions, not a claim that production testing has been completed.

## Links

- [Website](https://phonic.ai)
- [Documentation and support](https://docs.phonic.ai)
- [Privacy policy](https://phonic.ai/legal/privacy-policy)
- [Terms of service](https://phonic.ai/legal/terms-of-service)
