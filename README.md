# Radius Skills

Agent skills for modeling applications with [Radius](https://github.com/radius-project/radius).

## Installation

```bash
npx skills add radius-project/skills
```

## Available Skills

| Skill | Description |
|---|---|
| [app-modeling](skills/app-modeling/SKILL.md) | Generates `.radius/app.bicep` and, when needed, repository-local custom Resource Types, Azure Recipes, a Bicep extension, and a Recipe Pack. |

## How It Works

Each skill is a self-contained set of instructions that an AI agent reads and follows. Skills can include reference files with architecture patterns, naming conventions, and validation rules to produce consistent, high-quality output.

## License

This project is licensed under the [Apache License 2.0](LICENSE).
