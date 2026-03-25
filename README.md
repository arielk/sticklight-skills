# sticklight-skills

A collection of [Agent Skills](https://agentskills.io/) for Claude. Share these skills with your team to extend Claude's capabilities on specialized tasks.

## What are Skills?

Skills are folders of instructions, scripts, and resources that Claude loads dynamically. Each skill is self-contained in its own directory with a `SKILL.md` file that tells Claude **what** the skill does and **how** to use it.

For more background, see:
- [Agent Skills specification](https://agentskills.io/)
- [Skills in Claude Code](https://code.claude.com/docs/en/skills)
- [Anthropic's skills repo](https://github.com/anthropics/skills)

## Available Skills

| Skill | Description |
|-------|-------------|
| [seo](./skills/seo/) | SEO best practices and optimization guidelines |
| [auth](./skills/auth/) | Authentication & authorization patterns and implementation guidelines |

## Repository Structure

```
sticklight-skills/
├── skills/
│   ├── seo/              # SEO skill
│   │   └── SKILL.md
│   └── auth/             # Auth skill
│       └── SKILL.md
├── template/             # Template for creating new skills
│   └── SKILL.md
└── README.md
```

## Usage

### In Claude Code (as a Plugin)

Register this repo as a plugin marketplace:

```
/plugin marketplace add arielk/sticklight-skills
```

Or install skills directly:

```
/plugin install sticklight-skills
```

### In Claude.ai

Upload a `SKILL.md` file directly in Claude's settings to make a skill available in your conversations. See [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude).

### Locally (Personal Skills)

Copy a skill folder into your personal skills directory to make it available across all your projects:

```bash
cp -r skills/seo ~/.claude/skills/seo
```

### In a Project

Copy a skill folder into your project's `.claude/skills/` directory:

```bash
cp -r skills/seo your-project/.claude/skills/seo
```

## Creating a New Skill

1. Copy the template:

```bash
cp -r template skills/my-new-skill
```

2. Edit `skills/my-new-skill/SKILL.md`:
   - Set the `name` field (lowercase, hyphens for spaces)
   - Write a clear `description` so Claude knows when to use it
   - Add your instructions, examples, and guidelines in the markdown body

3. (Optional) Add supporting files like templates, examples, or scripts in the skill directory.

See [`template/SKILL.md`](./template/SKILL.md) for the starting point.

## Skill Format

Every skill needs a `SKILL.md` with YAML frontmatter and markdown instructions:

```markdown
---
name: my-skill
description: A clear description of what this skill does and when to use it
---

# My Skill

Instructions Claude will follow when this skill is active.

## Guidelines
- Guideline 1
- Guideline 2

## Examples
- Example 1
- Example 2
```

For advanced options (tool restrictions, subagent execution, dynamic context injection), see the [Claude Code skills docs](https://code.claude.com/docs/en/skills).
