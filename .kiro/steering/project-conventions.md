# Project Conventions

This repository consolidates AWS DevOps Agent skills. Follow these conventions when contributing.

## Repository Structure

```
aws-devops-agent-skill-central/
├── README.md                 # Project overview with skills table
├── .gitignore                # Root-level ignores
├── skills/
│   ├── .gitignore            # Allowlist for DevOps Agent supported extensions only
│   └── <skill-name>/
│       ├── SKILL.md          # Required: main skill instructions with frontmatter
│       ├── README.md         # Skill documentation (purpose, prompts, upload instructions)
│       ├── references/       # Optional: supplementary reference docs
│       ├── assets/           # Optional: images, diagrams, data files
│       └── evals/            # Optional: evaluation queries and benchmarks
```

## Skill Requirements

- Every skill must have a `SKILL.md` with valid frontmatter (`name` and `description` fields).
- Every skill should have a `README.md` describing its purpose, prerequisites, limitations, sample prompts, and upload instructions.
- Skill names use lowercase letters, numbers, and hyphens only (max 64 characters, no leading/trailing hyphens).
- The `description` in frontmatter should be written from the agent's perspective, specifying when and why the skill should activate.

## Allowed File Extensions

Only these extensions are permitted inside skill directories (enforced by `skills/.gitignore` and the DevOps Agent upload validator):

.md, .txt, .json, .yaml, .yml, .xml, .csv, .tsv, .html, .htm, .png, .jpg, .jpeg, .gif, .svg, .webp, .pdf

## Disallowed Content

- `scripts/` directories are not supported by DevOps Agent.
- `.claude/` directories should not be committed.
- `.DS_Store` and other OS files should not be committed.

## Adding a New Skill

1. Create a new directory under `skills/` with the skill name.
2. Add a `SKILL.md` with frontmatter and step-by-step instructions.
3. Add a `README.md` with purpose, prerequisites, limitations, sample prompts, and upload steps.
4. Update the root `README.md` skills table with the new skill's name, agent types, author, and docs link.

## Zipping for Upload

When zipping a skill for upload to DevOps Agent, include only allowed extensions and exclude tooling directories:

```bash
cd skills
zip -r <skill-name>.zip <skill-name>/ -i '*.md' '*.txt' '*.json' '*.yaml' '*.yml' '*.xml' '*.csv' '*.tsv' '*.html' '*.htm' '*.png' '*.jpg' '*.jpeg' '*.gif' '*.svg' '*.webp' '*.pdf' -x '*/.claude/*' '*/scripts/*'
```

## Git Conventions

- Push to a new branch for changes; use merge requests for review.
- Commit messages should be concise and descriptive.
- Do not commit zip files (they are gitignored).
