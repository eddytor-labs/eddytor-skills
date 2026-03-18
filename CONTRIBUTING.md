# Contributing to Eddytor Skills

Thanks for helping improve Eddytor's skill library! This guide explains how to add or modify skills.

## Skill structure

Every skill lives in `skills/<topic>/SKILL.md` and follows this template:

```markdown
```yaml
name: eddytor-<topic>
description: >
  Use this skill when the user wants to... Also use when the user mentions
  "keyword1", "keyword2" — even if they don't explicitly say "<topic>."
```

# Skill Title

Brief intro (1-2 sentences).

## Default procedure

Step-by-step workflow with tool calls.

## Tool usage examples

Concrete JSON examples with realistic data.

## Gotchas

Bullet list of common mistakes and edge cases.

## Guidelines

Short list of best practices.
```

## Principles

1. **Be concrete** — include real tool call examples with realistic parameters
2. **Lead with the default** — show the recommended workflow first, alternatives second
3. **Document gotchas** — every non-obvious behavior should be called out
4. **Stay provider-agnostic** — skills are plain markdown, no provider-specific syntax
5. **One topic per skill** — cross-cutting concerns go in `guides/`

## Adding a new skill

1. Create `skills/<topic>/SKILL.md` following the template above
2. Add an entry to `schemas/skill-manifest.json`
3. Update `README.md` if the skill adds a new category
4. Submit a PR with a clear description of the use case

## Modifying an existing skill

- Keep changes focused — one concern per PR
- If adding a new MCP tool reference, verify the tool exists in the current endpoint
- Update the YAML description block if trigger conditions change

## Guides vs skills

- **Skills** (`skills/`) teach the LLM how to accomplish a specific task domain
- **Guides** (`guides/`) provide cross-cutting reference material (tool catalog, provider setup, error handling)

## Code of conduct

Be helpful, be accurate, be concise. Skills are read by LLMs — every word counts.
