---
name: AKM
description: Use when managing skills during an LLM session — searching, loading, unloading, or checking available skills
---

# AKM — Agent Kit Manager

AKM manages reusable agent skills. During a session, use these commands to find and activate skills on the fly.

## Key Commands

```bash
akm skills search <query>   # Find skills by keyword
akm skills list              # Browse all available skills
akm skills load <id>         # Activate a skill in the current session
akm skills unload <id>       # Deactivate a skill from the current session
akm skills loaded            # Show what's currently active
akm skills status            # Full overview (library, project, session)
```

## Workflow

1. **Search** for what you need: `akm skills search testing`
2. **Load** it into your session: `akm skills load test-driven-development`
3. **Use** the skill — it's now available as context
4. **Unload** when done: `akm skills unload test-driven-development`

## More Help

- `akm skills --help` for all skills subcommands
- `akm help` for the full CLI reference
- https://akm.raphaelsimon.fr for documentation
