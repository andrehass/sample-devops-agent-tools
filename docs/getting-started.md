# Getting Started

This guide walks you through deploying a skill from this repository to your AWS DevOps Agent Space.

## Prerequisites

!!! info "Before you begin"
    - An [AWS DevOps Agent Space](https://docs.aws.amazon.com/devopsagent/latest/userguide/getting-started-with-aws-devops-agent-creating-an-agent-space.html) set up with your target AWS account as a cloud source
    - The skill-specific prerequisites documented on each skill's page (IAM permissions, service plans, etc.)

---

## Deploy a Skill

### 1. Clone the Repository

Only needed if you plan to upload a skill as a zip file (Option B) or via the Asset API (Option C). If you're importing directly from GitHub (Option A), you can skip this step.

```bash
git clone https://github.com/aws-samples/sample-code-for-devops-agent-skills.git
cd sample-code-for-devops-agent-skills
```

### 2. Choose a Skill

Browse the [available skills](skills/index.md) and review the skill's page for details on its purpose, prerequisites, and sample prompts.

### 3. Add the Skill to Your Agent Space

You can add a skill to your Agent Space in three ways.

#### Option A: Import from GitHub (recommended)

If you have a [GitHub connection configured](https://docs.aws.amazon.com/devopsagent/latest/userguide/connecting-to-cicd-pipelines-connecting-github.html) in your Agent Space, you can import a skill directly from this repository — no cloning or packaging required. In the DevOps Agent web app, go to Settings → Add Skill → Import from repository, then point to the skill's directory (for example, `skills/aws-health-events`) and select the appropriate agent types. See [Importing a skill from a repository](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html#creating-skills) for full instructions.

!!! note
    You cannot connect the `aws-samples` GitHub organization directly because the GitHub connection setup requires admin rights on the organization. Instead, connect your personal GitHub account and select any repository from it during the connection setup. Once a GitHub connection is established, you can import skills from any public repository — including this one — even if it wasn't selected during the connection setup.

#### Option B: Upload as a zip file

From the `skills/` directory, zip the skill for upload:

```bash
cd skills
zip -r <skill-name>.zip <skill-name>/ \
  -i '*.md' '*.txt' '*.json' '*.yaml' '*.yml' '*.xml' '*.csv' '*.tsv' \
     '*.html' '*.htm' '*.png' '*.jpg' '*.jpeg' '*.gif' '*.svg' '*.webp' '*.pdf' \
  -x '*/.claude/*' '*/scripts/*' '*/README.md' '*/.skilleval.yaml' \
     '*/.skilleval.yml' '*/CHANGELOG.md' '*/evals/*'
```

!!! note
    The zip excludes evaluation files, READMEs, and changelogs since those are development artifacts — only `SKILL.md`, `references/`, and `assets/` are used by the agent at runtime.

Then upload the zip file via the DevOps Agent web app and select the appropriate agent types. See [Uploading a skill](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html#creating-skills) in the AWS DevOps Agent User Guide for detailed steps.

#### Option C: Upload via the Asset API

Use the AWS DevOps Agent Asset API to programmatically manage skills — useful for CI/CD pipelines or automation workflows. Assign the skill to the appropriate agent types. See [Managing a skill end-to-end](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-managing-assets.html#managing-a-skill-end-to-end) for the full API workflow.

### 4. Verify

In the DevOps Agent Chat, try one of the sample prompts listed on the skill's documentation page. The agent should automatically activate the skill based on the context of your request.

## Skill Directory Structure

Each skill follows a consistent structure based on the [Agent Skills specification](https://agentskills.io/home):

```
skills/<skill-name>/
├── SKILL.md          # Main skill instructions with frontmatter (required)
├── README.md         # Documentation, prerequisites, and upload guide
├── CHANGELOG.md      # Version history
├── evals/            # Evaluation queries and benchmarks
├── assets/           # Images, diagrams, data files (optional)
└── references/       # Supplementary reference docs (optional)
```

The `SKILL.md`, `references/`, and `assets/` directories are what AWS DevOps Agent reads at runtime. Everything else supports development, testing, and documentation.

## Writing Your Own Skills

For guidance on creating custom skills for your operational workflows, see the [AWS DevOps Agent skills documentation](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-devops-agent-skills.html).

You can also use the skills in this repository as templates.
