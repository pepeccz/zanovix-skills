# Zanovix Skills

Public collection of reusable AI agent skills for Claude Code.

## Skills

| Skill | Description |
|-------|-------------|
| [chatwoot](skills/chatwoot-api/) | Chatwoot API integration — Application API + Platform API, self-hosted gotchas, sync patterns |
| [wordpress-theme-standards](skills/wordpress-theme-standards/) | Build custom WordPress themes that pass WordPress Coding Standards + Theme Check and stay upgrade-safe — WPCS tooling recipe, phpcs.xml starter, escaping/nonce discipline, upgrade-safe patterns |

## Installation

Copy a skill folder into `~/.claude/skills/` for global availability:

```bash
cp -r skills/chatwoot-api ~/.claude/skills/chatwoot-api
```

Or clone the repo and symlink:

```bash
git clone https://github.com/zanovix/zanovix-skills.git
ln -s $(pwd)/zanovix-skills/skills/chatwoot-api ~/.claude/skills/chatwoot-api
```

## License

MIT
