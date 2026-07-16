# Redshift Support Specialist — Custom Agent

## Purpose

This custom agent is a lean orchestrator for the [`redshift-support-specialist`](../../skills/redshift-support-specialist/) skill. It bridges that skill's domain knowledge (query optimization, operational reviews, disaster recovery guidance, incident detection guidance, cost optimization) to the six tools exposed by the connected `awslabs.redshift-mcp-server` MCP server, and adds one important behavior override: it always runs in the active chat session and never switches to background execution, even for the multi-step Detailed Operational Review.

## Key Capabilities

- Diagnoses slow Redshift queries live (EXPLAIN plan, disk spill, distribution/sort key issues) with concrete SQL/config fixes
- Runs a quick PASS/WARN/FAIL operational review using cluster inventory data
- Runs a full Detailed Operational Review (storage, WLM, table design, Advisor recommendations) and produces both an in-chat Markdown summary and a downloadable HTML report
- Offers disaster recovery and CloudWatch alarm guidance as reference material where live AWS CLI/CloudWatch access isn't available through the MCP server
- Performs cost optimization analysis, including provisioned-to-serverless RPU sizing

## Important behavior note

The skill's own `SKILL.md` lets the user choose between background mode and step-by-step execution for the Detailed Operational Review (Capability 3). This custom agent's system prompt (Section 0) overrides that choice and unconditionally forbids background mode — everything runs in the active foreground chat so the user can watch progress and intervene at any point. This is an intentional agent-level override, not a bug: if you expect background-mode behavior from the skill's documentation, note that this custom agent will not offer it.

## Prerequisites

- An AWS DevOps Agent space
- The [redshift-support-specialist skill](../../skills/redshift-support-specialist/) uploaded to your Agent Space. Important note: for the skill to be used by the custom agent, choose "All agents" in the "Agent Type" field when importing the skill, even though the skill's README instructs to choose "Chat"
- The `awslabs.redshift-mcp-server` MCP server connected as a capability provider — see the skill's [`deployment/README.md`](../../skills/redshift-support-specialist/deployment/README.md) for a ready-to-use serverless deployment
- No AWS CLI or CloudWatch access is required for the agent itself; all Redshift access goes through the connected MCP server

## Creating the Agent

1. In the DevOps Agent web app, go to the "Agents" menu (on the bottom left pane)
2. Click "Create agent" (on the right side), then on the new menu that popped up, click "Form" (the left-most option)
3. In the "Name" field, use "redshift-support-specialist"
4. Copy the content of the "SYSTEM_PROMPT.md" file from this directory, and paste it into the "System prompt" field in the custom agent creation form
5. In the "Skills" drop-down list, select the "redshift-support-specialist" skill, and click "Create agent"
6. Connect the `awslabs.redshift-mcp-server` MCP server as a capability provider for this agent (see the skill's deployment guide linked above) — this agent has no other way to reach Redshift

## Executing the Agent

You can execute the custom agent on-demand from the custom agent page or using chat. Follow the [Executing custom agents guide](https://docs.aws.amazon.com/devopsagent/latest/userguide/custom-agents-executing-custom-agents.html) for more information. Because this agent always runs in the active chat session (never in the background), it's best suited to interactive use rather than scheduled runs — ask it things like "why is this Redshift query slow?" or "run a detailed operational review on my cluster."

## Related

- [redshift-support-specialist skill](../../skills/redshift-support-specialist/) — domain knowledge for Redshift query optimization, operational reviews, and cost optimization
- [AWS DevOps Agent custom agents documentation](https://docs.aws.amazon.com/devopsagent/latest/userguide/working-with-devops-agent-custom-agents-index.html)
