# sticklight-skills

A collection of skills for the **sticklight.com** app builder, tailored for the **React + Vite + Tailwind CSS + Supabase** stack.

Skills teach AI how to follow your conventions and best practices. Share them with your team so everyone gets the same quality guidance.

## What is a Skill?

A skill is a single markdown file with three fields:

| Field | Purpose |
|-------|---------|
| **title** | Short name shown in the skill library |
| **description** | When to use this skill (helps the AI decide if it's relevant) |
| **content** | The full instructions, patterns, examples, and guidelines |

These three fields are defined in YAML frontmatter (`title`, `description`) and the markdown body (content):

```markdown
---
title: My Skill
description: When and why to use this skill.
---

# My Skill

Instructions, code examples, and guidelines go here.
```

Users fetch the skill content as text into the sticklight skill library. No folders, scripts, or supporting files needed — just the markdown.

## Stack

All skills in this repo are written for:

| Layer | Technology |
|-------|-----------|
| Frontend | React (with TypeScript) |
| Build | Vite |
| Styling | Tailwind CSS |
| Backend | Supabase (Auth, Database, Edge Functions, Storage) |

## Available Skills

| Skill | File | Description |
|-------|------|-------------|
| SEO | [`skills/seo/SKILL.md`](./skills/seo/SKILL.md) | SEO for React + Vite + Tailwind SPAs — static fallbacks, `@unhead/react`, sitemaps, structured data, semantic HTML, performance |
| Auth | [`skills/auth/SKILL.md`](./skills/auth/SKILL.md) | Authentication & authorization with Cloud Backend — email/password, protected routes, RLS, role-based access |

## Creating a New Skill

1. Copy the template folder:

```bash
cp -r skills/template skills/my-new-skill
```

2. Edit `skills/my-new-skill/SKILL.md`:
   - Set `title` — short name for the skill library
   - Set `description` — explain when the AI should use this skill
   - Write the content — instructions, code examples, guidelines

See [`skills/template/SKILL.md`](./skills/template/SKILL.md) for the starting point.

## Repository Structure

```
sticklight-skills/
├── skills/
│   ├── seo/
│   │   └── SKILL.md      # SEO skill
│   ├── auth/
│   │   └── SKILL.md      # Auth skill
│   └── template/
│       └── SKILL.md      # Template for new skills
└── README.md
```

Each skill lives in its own folder. The skill content is always in `SKILL.md`.
