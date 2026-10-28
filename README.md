# Phonic Agent Skills and MCP Plugin

Official customer-facing skills and an MCP connection for building, testing, and operating [Phonic](https://phonic.ai) voice agents.

## Install skills

```bash
npx skills add Phonic-Co/skills
```

This installs skills only. Connect the MCP separately, or install the plugin bundle below.

## Install the plugin

The bundle connects to the authenticated Phonic MCP, including documentation search. Sign in to Phonic through your client's OAuth flow. No API keys are stored in this repository.

### Claude Code

```text
/plugin marketplace add Phonic-Co/skills
/plugin install phonic@phonic-skills
```

Use `/mcp` to check connection status and finish authentication. To test an unpublished checkout, launch `claude --plugin-dir /absolute/path/to/skills`.

### Codex

Install the skills with the command above, then add the Phonic MCP connection if it is not already configured:

```bash
codex mcp add phonic --url https://mcp.phonic.ai/mcp
codex mcp login phonic
```

Follow the browser sign-in flow. Use `codex mcp list` to check setup. See [Codex MCP setup](https://developers.openai.com/codex/mcp) for client configuration details.

### Cursor

Install the skills with the command above. Merge this entry into `.cursor/mcp.json` or `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "phonic": { "url": "https://mcp.phonic.ai/mcp" }
  }
}
```

Finish authentication from Cursor’s MCP settings; see [Cursor MCP setup](https://cursor.com/docs/mcp).

Avoid adding a second copy of an already connected server. Documentation search is provided through `search_docs` on the same Phonic connection.

## Get started

In a new chat, ask: **“Get started on phonic.ai via MCP and help me run my first conversation.”** The `phonic-get-started` skill guides connection, project selection, agent setup, and a first conversation. Complete any browser sign-in, client approval, or microphone interaction when prompted.

If your client does not automatically select the skill, explicitly ask it to use `phonic-get-started`. Skills must be installed and loaded by the client; an MCP connection alone does not supply them.

To check individual setup steps, ask:

1. "Which Phonic workspace and projects am I connected to?" Check the returned workspace before making changes.
2. "Search Phonic's docs for how to choose and preview a voice." Expect source links.
3. "Help me configure an agent in my chosen project, then validate an outbound call with dry_run without placing it."

A connected MCP does not automatically install skills. Conversely, installing skills does not authenticate the API connection. If tools are missing, inspect the client's MCP status and restart/reconnect as needed. OAuth selects the workspace for that connection; selecting a different project does not switch workspaces.

## Skills

| Skill | When to use it |
| --- | --- |
| `phonic-get-started` | Connect MCP, build an agent, and run a first conversation. |
| `phonic` | Choose an integration and find the relevant implementation skill. |
| `phonic-agents` | Configure agents, phone calls, prompts, and project operations. |
| `phonic-websockets` | Build browser voice sessions and server audio bridges. |
| `phonic-tools` | Connect tools and transfer callers. |
| `phonic-post-call` | Process webhooks, extractions, and evaluations. |

See [Phonic examples](https://github.com/Phonic-Co/phonic-examples) for runnable applications and [the docs](https://docs.phonic.ai/llms.txt) for current contracts.
