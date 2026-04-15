# Zanovix Skills

Public collection of reusable AI agent skills for Claude Code.

## Skills

| Skill | Description |
|-------|-------------|
| [chatwoot](skills/chatwoot/) | Chatwoot API integration — Application API + Platform API, self-hosted gotchas, sync patterns |

## Installation

Copy a skill folder into `~/.claude/skills/` for global availability:

```bash
cp -r skills/chatwoot ~/.claude/skills/chatwoot
```

Or clone the repo and symlink:

```bash
git clone https://github.com/zanovix/zanovix-skills.git
ln -s $(pwd)/zanovix-skills/skills/chatwoot ~/.claude/skills/chatwoot
```

## License

MIT
