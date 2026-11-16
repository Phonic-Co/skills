---
name: phonic
description: Review existing Phonic conversation history, transcripts, analysis, evaluations, and extractions through the connected Phonic MCP server.
---

# Review Phonic conversations

Use the connected Phonic MCP server. Sign in through its OAuth browser flow if needed. Never request or read API keys, passwords, webhook signing secrets, or other credentials from chat, environment variables, or local files.

1. Use `list_projects` to identify an authorized project. Ask the user to choose when the intended project is unclear. `get_workspace` returns workspace settings and limits, not a workspace name or ID.
2. Use `list_conversations` with the user's intended project and filters. Use `get_conversation` for the selected conversation and its transcript. Preserve full conversation IDs in references.
3. Retrieve existing results with `get_conversation_analysis`, `list_conversation_evaluations`, and `list_conversation_extractions`. Inspect project evaluation prompts or extraction schemas when useful to explain the results. If no results exist, say so; this connector does not run evaluations or extractions.
4. Ground conclusions in the returned transcripts and results. Distinguish observed facts from suggestions, and do not fabricate missing data.

Use `get_doc_page` for parameter and response documentation; omit `page` for the index. Documentation may describe the full Phonic product, but only use tools offered by this connection. Do not use direct HTTP, SDK, CLI, another endpoint, or another MCP profile as a fallback for unavailable operations.

This connection cannot create or modify agents, generate or preview speech, start phone/browser conversations, replay conversations, issue API keys or tokens, or retrieve webhook secrets. Explain this scope when asked for those actions. Suggestions about prompts are text recommendations only; do not apply them through another interface.

`submit_feedback` sends feedback to Phonic. Use it when the user requests a report about an integration problem; include only the details necessary to explain the issue, with no credentials.
