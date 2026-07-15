# Phonic Agent Skills

Official [Agent Skills](https://agentskills.io) for building voice agents on [Phonic](https://phonic.ai), for use with Claude Code and any harness that supports the Agent Skills standard (Codex, Cursor, GitHub Copilot, and Gemini CLI read `.agents/skills` natively).

## Install

```bash
npx skills add Phonic-Co/skills
```

Or with Claude Code plugins:

```
/plugin marketplace add Phonic-Co/skills
```

## What's here

The `phonic` skill: decision guidance and hard-won platform facts for building Phonic voice agents with the [`phonic`](https://www.npmjs.com/package/phonic) SDK (npm / PyPI) — choosing the right conversation surface, browser voice done correctly, custom tools, webhooks, transfers, and production patterns.

The skill was developed eval-first: every line traces to an observed coding-agent failure, a docs gap, or a live-verified platform fact. The eval suite and its results are coming in a follow-up release.
