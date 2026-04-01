# Claude Code Toolkit

Community skills, references, and tools for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

Built and maintained by [Codependent AI](https://codependentai.io).

## What's here

### Skills

Skills are markdown files that Claude Code loads automatically. They give your agent domain knowledge and trigger on relevant questions.

| Skill | Description |
|-------|-------------|
| [cc-architecture](skills/cc-architecture/SKILL.md) | Internal architecture reference — boot sequence, query loop, 46 built-in tools, permission system, context management, 88 feature flags, and all major subsystems |

### Installing a skill

Copy the skill folder into your Claude Code skills directory:

```bash
# Global (all projects)
cp -r skills/cc-architecture ~/.claude/skills/

# Project-local
cp -r skills/cc-architecture .claude/skills/
```

The skill auto-triggers when you ask about Claude Code internals, build plugins/hooks/agents, or debug CC behavior.

## Contributing

PRs welcome. Each skill should be a folder under `skills/` containing a `SKILL.md` with frontmatter:

```yaml
---
skill: your-skill-name
description: "One-line description for trigger matching"
trigger: "when to activate this skill"
---
```

## License

MIT
