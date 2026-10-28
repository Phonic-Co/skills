# Phonic Agent Skills

Official [Agent Skills](https://agentskills.io) for building voice agents on [Phonic](https://phonic.ai), for use with coding agents that support the Agent Skills standard (Claude Code, Codex, Cursor, GitHub Copilot, and Gemini CLI read `.agents/skills`).

## Install

```bash
npx skills add Phonic-Co/skills
```

Or with Claude Code plugins:

```
/plugin marketplace add Phonic-Co/skills
/plugin install phonic@phonic-skills
```

## Skills

| Skill | Use it for |
| --- | --- |
| [phonic](skills/phonic/SKILL.md) | Choosing an integration and finding the relevant implementation skill |
| [phonic-agents](skills/phonic-agents/SKILL.md) | Agents, phone calls, prompts, and project operations |
| [phonic-websockets](skills/phonic-websockets/SKILL.md) | Browser voice, STS configuration, audio, and reconnects |
| [phonic-tools](skills/phonic-tools/SKILL.md) | Agent tools and human transfers |
| [phonic-post-call](skills/phonic-post-call/SKILL.md) | Webhooks, extractions, and conversation evaluations |
