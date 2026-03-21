# aviary-skills

Skill content for [NeuBird SkillsHub](https://github.com/neubirdai/aviary). Community contributions welcome via PR.

## Structure

```
skills/
├── sev-hunt/
│   └── SKILL.md
├── remediation-steps/
│   └── SKILL.md
└── ...
```

Each skill lives in its own directory under `skills/`. The directory name must match the `name` field in the SKILL.md frontmatter.

## Skill Format

```yaml
---
name: my-skill
description: "What this skill does and when to use it."
tags: [incident, proactive]
homepage: https://example.com
metadata: { "neubird": { "emoji": "🔍", "requires": { "bins": ["neubird"] } } }
---

Skill body here (markdown, under 500 lines)...
```

### Required Fields

- `name` — lowercase letters, digits, hyphens only (max 64 characters)
- `description` — what the skill does and when to invoke it

### Optional Fields

- `tags` — list of strings for search and filtering
- `homepage` — link to docs or source
- `metadata.neubird` — NeuBird-specific fields (emoji, requires, install specs)

## Contributing

1. Fork this repo
2. Create a new directory under `skills/` with your skill name
3. Add a `SKILL.md` following the format above
4. Open a PR — CI will validate your skill automatically

## License

MIT
