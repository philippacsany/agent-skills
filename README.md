# Agent Skills

Skills for AI coding agents like Claude Code, from my YouTube videos. Each skill is a
folder with a `SKILL.md` file that your agent loads when a task calls for it.

Skills follow the [Agent Skills](https://agentskills.io) format, so they work with
Claude Code, Codex, Cursor, and other agents that support it.

## Available skills

| Skill | What it does |
| --- | --- |
| [openrouter-jev](skills/openrouter-jev/) | Calls TypeSafe's Jev decisions model through OpenRouter, so one OpenRouter API key covers both your agent and Jev |

Each skill folder has its own README with setup steps and a link to its video. Start
there, because some skills need an API key or other configuration before they work.

## Install

Install a single skill with the [`skills`](https://github.com/vercel-labs/skills) CLI:

```bash
npx skills add philippacsany/agent-skills --skill openrouter-jev
```

The CLI asks which agent to install to. Add `-g` to install the skill for your user
instead of the current project, and `-a claude-code` to skip the agent prompt.

To install by hand, copy the skill folder into your agent's skills directory. For
Claude Code, that's `~/.claude/skills/` for your user, or `.claude/skills/` in a
project:

```bash
git clone https://github.com/philippacsany/agent-skills.git
cp -r agent-skills/skills/openrouter-jev ~/.claude/skills/
```

## License

[MIT](LICENSE). Skills adapted from other projects keep their original copyright notice
in the skill's own `LICENSE` file.
