# Changelog

## 1.0.0

- Initial version
- Lean orchestrator system prompt bridging the `redshift-support-specialist` skill to the six `awslabs.redshift-mcp-server` MCP tools
- Enforces active-chat-only execution (never background), overriding the skill's own background-mode option for the Detailed Operational Review
- Hard-stop scope confirmation (cluster/workgroup + database) before any data collection
- Read-only enforcement and per-capability tool mapping (query optimization, operational reviews, disaster recovery, incident detection, cost optimization)
