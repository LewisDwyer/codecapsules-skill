# Code Capsules Skill

Manage [Code Capsules](https://codecapsules.io) from any AI coding agent. List spaces, create and delete capsules, bind databases, and check build logs from the terminal.

Pure markdown skill, no MCP server or Python required. The agent calls the Code Capsules REST API directly using `curl`.

## Install

Clone this repo and copy the skill into your agent's skills directory:

```bash
git clone https://github.com/codecapsules-io/codecapsules-skill
cp -r codecapsules-skill/skills/codecapsules ~/.claude/skills/codecapsules
```

For other agents, copy into the equivalent skills or rules directory.

## Setup

Add your Code Capsules credentials to `~/.claude/settings.json`:

```json
{
  "env": {
    "CC_EMAIL": "you@example.com",
    "CC_PASSWORD": "yourpassword"
  }
}
```

Or export `CC_EMAIL` and `CC_PASSWORD` from your shell profile (for example, `~/.zshrc` or `~/.bashrc`).

For a full walkthrough, see the [deployment guide](https://docs.codecapsules.io/tutorials/use-codecapsules-with-an-agent).

## What's included

```
skills/codecapsules/
├── SKILL.md              # Auth flow, curl examples, workflows
└── references/
    └── api.md            # Payload structures, manifest types, defaults
```

## Capabilities

- **List** spaces, capsules, repos, and plans
- **Create** capsules (backend, frontend, docker, agent, WordPress, databases)
- **Delete** capsules (with confirmation)
- **Bind** data capsules to app capsules
- **Get** build logs for debugging

**Capsule types:** backend, frontend, docker, agent, wordpress, mysql, mongodb, documentdb, postgresql, redis, storage
