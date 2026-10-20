# SeggWat Agent Skills

[Agent skills](https://agentskills.io) for [SeggWat](https://seggwat.com), the user feedback platform. They teach coding agents (Claude Code, Cursor, Codex, …) to add feedback collection to an app and to work with the feedback afterwards.

| Skill | What it does |
|---|---|
| [`seggwat-setup`](skills/seggwat-setup/SKILL.md) | Adds the feedback widget to a web app through the SeggWat MCP server and verifies the first submission |
| [`seggwat-cli`](skills/seggwat-cli/SKILL.md) | Manages feedback, projects and ratings with the `seggwat` CLI |
| [`seggwat-changelog`](skills/seggwat-changelog/SKILL.md) | Drafts and publishes changelog entries from git history |

## Install

```bash
npx skills add SeggWat/skills
```

Or one skill: `npx skills add SeggWat/skills --skill seggwat-setup`.

Browse them at https://seggwat.com/skills.

## License

MIT
